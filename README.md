# parallel-logram

A high-performance Rust implementation of Logram— extracting structure from unstructured log files. 

Accelerates n-gram dictionary construction using multi-threading and concurrent hash maps, enabling fast identification of static and dynamic log tokens in massive logs. 

___

Most code lives in parser.rs. A bit of code is in main.rs.

You can run cargo test to run the test cases.

Here's how you can invoke the program itself.

```
cargo run --release -- --raw-spark data/from_paper.log --to-parse "17/06/09 20:11:11 INFO storage.BlockManager: Found block rdd_42_20 locally" --before "split: hdfs://hostname/2kSOSP.log:29168+7292" --after "Found block" --cutoff 3
cargo run --release -- --raw-linux data/Linux_2k.log --to-parse "Jun 23 23:30:05 combo sshd(pam_unix)[26190]: authentication failure; logname= uid=0 euid=0 tty=NODEVssh ruser= rhost=218.22.3.51  user=root" --before "rhost=<*> user=root" --after "session opened" --cutoff 100
cargo run --release -- --raw-hdfs data/HDFS_2k.log --to-parse "081109 204925 673 INFO dfs.DataNode$DataXceiver: Receiving block blk_-5623176793330377570 src: /10.251.75.228:53725 dest: /10.251.75.228:50010" --before "size <*>" --after "BLOCK* NameSystem.allocateBlock:"
cargo run --release -- --raw-hpc data/HPC_2k.log --to-parse "inconsistent nodesets node-31 0x1fffffffe <ok> node-0 0xfffffffe <ok> node-1 0xfffffffe <ok> node-2 0xfffffffe <ok> node-30 0xfffffffe <ok>" --before "running running" --after "configured out"
cargo run --release -- --raw-hpc data/HPC.log --to-parse "inconsistent nodesets node-31 0x1fffffffe <ok> node-0 0xfffffffe <ok> node-1 0xfffffffe <ok> node-2 0xfffffffe <ok> node-30 0xfffffffe <ok>" --before "running running" --after "configured out" --cutoff 106
cargo run --release -- --raw-hpc data/HPC.log --to-parse "58717 2185 boot_cmd new 1076865186 1 Targeting domains:node-D1 and nodes:node-[40-63] child of command 2176" --before-line "58728 2187 boot_cmd new 1076865197 1 Targeting domains:node-D2 and nodes:node-[72-95] child of command 2177" --after-line "58707 2184 boot_cmd new 1076865175 1 Targeting domains:node-D0 and nodes:node-[0-7] child of command 2175" --cutoff 106
cargo run --release -- --raw-proxifier data/Proxifier_2k.log --to-parse "[10.30 16:54:08] chrome.exe - proxy.cse.cuhk.edu.hk:5070 close, 3637 bytes (3.55 KB) sent, 1432 bytes (1.39 KB) received, lifetime 00:01" --before "proxy.cse.cukh.edu.hk:5070 HTTPS" --after "open through" --cutoff 10
cargo run --release -- --raw-healthapp data/HealthApp_2k.log --to-parse "20171223-22:15:41:672|Step_StandReportReceiver|30002312|REPORT : 7028 5017 150539 240" --before "calculateAltitudeWithCache totalAltitude=240" --after "onStandStepChanged 3601"
cargo run --release -- --raw-healthapp data/HealthApp.log --to-parse "20171223-22:15:41:672|Step_StandReportReceiver|30002312|REPORT : 7028 5017 150539 240" --before "calculateAltitudeWithCache totalAltitude=240" --after "onStandStepChanged 3601" --cutoff 10
```

You'll need to untar `OpenStack.tar.gz` to try this one (but it doesn't work well anyway):
```
cargo run --release -- --raw-openstack data/openstack_normal2.log --to-parse "nova-compute.log.2017-05-17_12:02:35 2017-05-17 12:02:30.397 2931 INFO nova.virt.libvirt.imagecache [req-addc1839-2ed5-4778-b57e-5854eb7b8b09 - - - - -] image 0673dd71-34c5-4fbb-86c4-40623fbe45b4 at (/var/lib/nova/instances/_base/a489c868f0c37da93b76227c91bb03908ac0e742): in use: on this node 1 local, 0 on other nodes sharing this instance storage"
```

___

You can attempt your own implementation and benchmark it against mine

**Command used:** (be sure to update --num-threads!!)

```
hyperfine 'cargo run --release -- --raw-hpc data/HPC.log --to-parse "58717 2185 boot_cmd new 1076865186 1 Targeting domains:node-D1 and nodes:node-[40-63] child of command 2176" --before-line "58728 2187 boot_cmd new 1076865197 1 Targeting domains:node-D2 and nodes:node-[72-95] child of command 2177" --after-line "58707 2184 boot_cmd new 1076865175 1 Targeting domains:node-D0 and nodes:node-[0-7] child of command 2175" --cutoff 106 --single-map --num-threads=x'
```

**My results:** (may vary machine to machine)

**Benchmark 1:** `--num-threads=1`

  Time (mean ± σ):      4.602 s ±  0.029 s    [User: 4.566 s, System: 0.031 s]

  Range (min … max):    4.563 s …  4.654 s    10 runs

**Benchmark 2:** `--single-map --num-threads=8` (default)

  Time (mean ± σ):     912.0 ms ±  17.9 ms    [User: 3032.7 ms, System: 52.8 ms]

  Range (min … max):   897.3 ms … 959.7 ms    10 runs

**Benchmark 2:** `--num-threads=8` (default)

  Time (mean ± σ):     789.7 ms ±  14.1 ms    [User: 3190.1 ms, System: 56.1 ms]

  Range (min … max):   771.5 ms … 809.5 ms    10 runs
