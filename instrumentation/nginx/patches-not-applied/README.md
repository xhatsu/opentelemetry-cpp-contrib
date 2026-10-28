# Patches documented but not applied

This fork carries one change for old nginx (`ngx_http_null_variable` fallback, needed to build against
nginx 1.12). The three patches below are fixes found while running the module on nginx 1.12 packages.
They are **not applied** on this branch. Apply one with `patch -p1 < patches-not-applied/<file>` from
`instrumentation/nginx`. All three apply cleanly (0003 and 0004 with a 5-line offset) on upstream
`eab86ac`.

Evidence below comes from nginx 1.12.2 (EPEL package), aarch64, run natively, with the same module source
(the older pinned upstream `cff3cb6`, opentelemetry-cpp 1.17) built with and without each patch. It was
not re-run against upstream head (opentelemetry-cpp 1.30.0).

| patch | problem | measured effect | severity |
|---|---|---|---|
| `0002-fix-bsp-schedule-delay-directive.patch` | `opentelemetry_bsp_schedule_delay_millis` writes `maxExportBatchSize` instead of `scheduleDelayMillis` | with the directive set to 300 ms, first export reached the collector after 2.998 s unpatched, 0.106 s patched | the directive has no effect. Same bug as open PR #775 |
| `0003-shutdown-sdk-on-worker-exit.patch` | the SDK is torn down by C++ static destructors when nginx `dlclose`s the module, while the exporter thread may still run and log an error through the destroyed internal log handler | collector down (connection refused), 15 reloads: unpatched 16 worker crashes (signal 11) per run, patched 0. No client request failed (8 clients, ~11,000 requests per run, including 600 KB downloads in flight). Hung, slow and 503 collectors did not crash in the test | an `[alert] worker process exited on signal 11` per reload when the collector is down, possible core files, possible loss of queued spans. A race, so intermittent |
| `0004-destroy-main-conf-with-its-pool.patch` | `OtelMainConf` is placement-new'd into the config pool with no cleanup, so its `std::string` members (endpoint, service name, sampler) are never destroyed | master under valgrind: "definitely lost" 456 B after 5 reloads, 2,056 B after 30 (about 64 B and 2 blocks per reload); patched constant 72 B (protobuf static init). Master RSS over 600 reloads was too coarse to show it | small leak per reload, relevant only to servers that reload often |

## How 0003 happens

1. A worker exits (reload or quit). nginx `dlclose`s the module while the exporter thread is running.
2. `dlclose` runs the static destructors, destroying the SDK's global internal log handler.
3. The exporter thread, failing to reach the collector, logs an error through that handler: SIGSEGV.
4. The patch registers an `exit_process` hook that calls `TracerProvider::Shutdown()` first, which stops the
   exporter thread and flushes queued spans while the SDK is intact.

With a collector that hangs, the thread is blocked on the network and never logs, which is why only the
"collector down" case crashed.
