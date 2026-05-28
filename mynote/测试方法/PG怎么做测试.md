# PG怎么做测试
可以通过make check来测试PG的功能，棒棒的。方法就是
make check

会看到类似这种输出:
 parallel group (18 tests):  compression_lz4 numa hash_part reloptions explain predicate partition_info memoize compression eager_aggregate partition_merge partition_split stats partition_aggregate partition_prune tuplesort partition_join indexing
ok 214       + partition_merge                          1699 ms
ok 215       + partition_split                          1932 ms
ok 216       + partition_join                           2613 ms
ok 217       + partition_prune                          2525 ms
ok 218       + reloptions                                176 ms
ok 219       + hash_part                                 122 ms
ok 220       + indexing                                 2738 ms
ok 221       + partition_aggregate                      2441 ms
ok 222       + partition_info                            338 ms
ok 223       + tuplesort                                2583 ms
ok 224       + explain                                   224 ms
ok 225       + compression                               580 ms
ok 226       + compression_lz4                            36 ms
ok 227       + memoize                                   561 ms
ok 228       + stats                                    1939 ms
ok 229       + predicate                                 243 ms
ok 230       + numa                                       47 ms
ok 231       + eager_aggregate                           986 ms
 parallel group (2 tests):  oidjoins event_trigger
ok 232       + oidjoins                                  339 ms
ok 233       + event_trigger                             375 ms
ok 234       - event_trigger_login                        29 ms
ok 235       - fast_default                              206 ms
ok 236       - tablespace                                592 ms


也可以只测试一个功能，比如：
make check TESTS=tuplesort VERBOSE=1

