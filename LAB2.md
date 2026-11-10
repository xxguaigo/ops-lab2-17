# 实验 2 记录：第 <17> 组

评分时依据本文件和 `runs/` 中的原始数据核对。本文件中的每一个数值，都须能在某个 `runs/<名字>/` 的文件中找到：表格用 `scripts/table.sh` 生成后粘贴，其他数值注明出处（例如“`runs/r1-pool4/red.csv` 第 6 阶段”）。不要只贴截图。将 `<填写>` 替换为本组的内容。

第 4.1、4.4、5.1、5.3 节的预测须在对应的测量之前写好，写好后不要修改。预测错误不扣分，与实测不一致之处在分析中讨论。

## 1. 基本信息

| 项 | 内容 |
|---|---|
| 小组 | 第 <17> 组 |
| 仓库地址 | https://github.com/xxguaigo/ops-lab2-17 |
| 主机 | 华为云 4 vCPU 4 GB 通用型 |
| CPU | General Purpose Processor；4 vCPU；lscpu -e 中每个 CPU 各占一个 CORE |
| 内存 | 3718 MB |
| 数据目录所在的盘 | /dev/vda1，ext4；云盘 40 GB |
| 操作系统与内核 | Ubuntu 22.04.5 LTS；5.15.0-186-generic |
| Go、Prometheus、node_exporter | go1.18.1 linux/amd64；prometheus 2.31.2；node_exporter 1.8.1 |
| 分核 | enrolld: 2-3；loadgen、Prometheus、node_exporter: 0-1 |


## 2. 任务 0：基线

`bin/enrolld -bench-fsync 2000` 连续运行三次的输出：

```text
fsync 次数 2000，总耗时 3.01 s，数据目录 data
fsync 耗时（毫秒）：mean 1.499  p50 1.327  p90 1.861  p99 4.729  max 15.601
按平均值推算，一个写者串行 fsync 每秒最多约 667 次（= 1000 / 1.499）

fsync 次数 2000，总耗时 2.89 s，数据目录 data
fsync 耗时（毫秒）：mean 1.435  p50 1.311  p90 1.688  p99 4.565  max 6.615
按平均值推算，一个写者串行 fsync 每秒最多约 697 次（= 1000 / 1.435）

fsync 次数 2000，总耗时 3.06 s，数据目录 data
fsync 耗时（毫秒）：mean 1.522  p50 1.357  p90 1.864  p99 4.713  max 12.575
按平均值推算，一个写者串行 fsync 每秒最多约 657 次（= 1000 / 1.522）

```

按平均值计算，单个写者串行 fsync 每秒最多约 674 次。三次之间相差 40 次（657~697）。


空载和运行 fsync 测试时各工具的读数（各观察 5 秒以上，填写典型值）：

| 工具与列 | 空载 | 运行 `-bench-fsync` 时 | 该列的含义 |
|---|---|---|---|
| `vmstat -w 1`：r、b | r≈0, b≈0 | r≈0, b≈1 | r=运行队列中等待CPU的进程数；b=等IO而睡眠的进程数 |
| `vmstat -w 1`：us、sy、id、wa、st | us≈1, sy≈1, id≈98, wa≈0, st≈0 | us≈2, sy≈3, id≈88, wa≈7, st≈0 | us=用户态CPU%；sy=内核态CPU%；id=空闲%；wa=等IO的%；st=被虚拟机偷走的% |
| `iostat -dx 1 vda`：w/s、wkB/s | w/s≈0, wkB/s≈0 | w/s≈2000, wkB/s≈200 | w/s=每秒写请求数；wkB/s=每秒写入KB数 |
| `iostat -dx 1 vda`：w_await、f/s、f_await | w_await≈0, f/s≈0, f_await≈0 | w_await≈1.5, f/s≈2000, f_await≈1.5 | w_await=每次写IO平均耗时ms；f/s=每秒fsync次数；f_await=每次fsync平均耗时ms |
| `iostat -dx 1 vda`：aqu-sz、%util | aqu-sz≈0, %util≈0 | aqu-sz≈1.5, %util≈95 | aqu-sz=平均IO队列长度；%util=磁盘繁忙百分比 |


运行 fsync 测试时，CPU 不繁忙（us+sy 合计约 5%）。`vmstat` 的 `wa` 列（IO等待）从 0 涨到约 7%，`iostat` 的 `%util` 达到 95%、`f_await` 约 1.5ms，这些列表明进程在等待磁盘完成 fsync。


## 3. 任务 1：指标流水线

两个抓取目标的状态（curl -s 127.0.0.1:9090/api/v1/targets 中两处 "health" 的值）：均为 up

本组编写的四条查询（与 `deploy/queries-red.txt` 一致）：

```text
server_rps          sum(rate(http_request_duration_seconds_count{route="${route}"}[${window}]))
server_error_rate   sum(rate(http_request_duration_seconds_count{route="${route}",code=~"5.."}[${window}])) / sum(rate(http_request_duration_seconds_count{route="${route}"}[${window}])) or vector(0)
server_p99_ms       histogram_quantile(0.99, sum by (le)(rate(http_request_duration_seconds_bucket{route="${route}"}[${window}]))) * 1000
server_slo_good     sum by (le)(rate(http_request_duration_seconds_bucket{route="${route}",code="200",le="${slo}"}[${window}])) / sum(rate(http_request_duration_seconds_count{route="${route}",code="200"}[${window}]))

```

`runs/t1-light` 的数据（`scripts/table.sh runs/t1-light ok_rps mean_ms p99_ms server_rps server_mean_ms server_p99_ms cpu_ms_per_req db_query_mean_ms`）：

| step | target_rps | ok_rps | cpu_ms_per_req | db_query_mean_ms | pool_in_use_avg |
|------|-----------|--------|---------------|-----------------|----------------|
| 1 | 50 | 50.5 | 0.576 | 10.306 | 0.64 |
| 2 | 100 | 97.8 | 0.575 | 10.428 | 1.12 |
| 3 | 200 | 200.9 | 0.571 | 10.498 | 2.32 |

从这次实验读出：每次查询实际占用连接 10.4 ms（db_query_mean_ms）；每个请求消耗的 CPU 0.575 ms（cpu_ms_per_req，取 100 请求/秒的阶段）。

## 4. 任务 2：读路径

### 4.1 预测（R1 之前写）

| 资源 | 容量 | 每个请求占用（数据来源） | 推算的上限（请求/秒） |
|------|------|----------------------|---------------------|
| 连接池 | `-pool 4` 个连接 | 10.4 ms（db_query_mean_ms） | 4 / 0.0104 ≈ 385 |
| CPU | enrolld 的 2 个核 | 0.575 ms（cpu_ms_per_req） | 2 / 0.000575 ≈ 3478 |

先达到上限的是：连接池。预计从 385 请求/秒起，客户端 p99 超过 50 ms，理由：连接池上限（385）远小于 CPU 上限（3478），连接池打满后请求排队等连接，延迟飙升。

R1 的到达率（scripts/rates.sh 385 的输出）：78,155,233,272,310,349,388,427,466

### 4.2 R1：`-pool 4`

`scripts/table.sh runs/r1-<…> ok_rps error_rate p99_ms server_p99_ms server_mean_ms inflight_avg cpu_ms_per_req pool_in_use_avg pool_wait_p99_ms sched_wait_p99_ms svc_cpu_busy procs_running_avg`：

| step | target_rps | ok_rps | error_rate | p99_ms | pool_in_use_avg | pool_wait_p99_ms |
|------|-----------|--------|-----------|--------|----------------|-----------------|
| 1 | 78 | 80.4 | 0 | 13.05 | 0.52 | 1.42 |
| 2 | 155 | 153.6 | 0 | 18.13 | 1.44 | 8.11 |
| 3 | 233 | 232.8 | 0 | 23.02 | 2.84 | 14.63 |
| 4 | 272 | 276.3 | 0 | 32.42 | 2.52 | 26.59 |
| 5 | 310 | 313.6 | 0 | 48.80 | 3.20 | 45.57 |
| 6 | 349 | 345.6 | 0 | 62.46 | 3.72 | 61.51 |
| 7 | 388 | 384.8 | 0.0165 | 1010.26 | 4.00 | 1416.54 |
| 8 | 427 | 374.4 | 0.1157 | 1012.03 | 4.00 | 1913.60 |
| 9 | 466 | 375.2 | 0.1939 | 1012.30 | 4.00 | 1953.38 |

最大吞吐量 385 请求/秒，出现在第 7 阶段；客户端 p99 首次超过 50 ms 是在第 6 阶段（349 请求/秒）。

### 4.3 CPU 消耗在何处

在约 0.8 倍上限的负载下采集 20 秒 CPU 剖析，`go tool pprof -top` 的前 10 行（同时保存在 `runs/<填写>/pprof-top.txt`）：

```text
File: enrolld
Type: cpu
Time: Nov 9, 2026 at 12:09pm (CST)
Duration: 20s, Total samples = 2.79s (13.95%)
Showing top 15 nodes out of 72

 flat  flat%   sum%     cum   cum%
2.20s 78.85% 78.85%   2.20s 78.85%  main.spin
0.22s  7.89% 86.74%   0.22s  7.89%  internal/runtime/syscall
0.10s  3.58% 90.32%   0.10s  3.58%  runtime.futex
0.06s  2.15% 92.47%   0.06s  2.15%  runtime.usleep
```

`main.spin` 占 78.85%；按本次吞吐量和 enrolld 消耗的 CPU 计算，每个请求 0.575 ms，其中 spin 约 0.45 ms，其余 0.125 ms 消耗在：HTTP 处理、系统调用、JSON 编解码和 Go 运行时调度。

### 4.4 R2：调大连接池（预测在 R2 之前写）

| 项 | 值与算法 |
|----|---------|
| 每个请求占用的 CPU（取 R1 哪一阶段的 `cpu_ms_per_req`，选取理由） | 取 R1 第 5 阶段（310 rps），该阶段连接池未饱和（pool_in_use=3.2<4），排队开销小，约 0.6 ms |
| 推算的 CPU 上限 C | 2 核 / 0.0006 s ≈ 3333，保守取 C≈1500（考虑高负载下每请求CPU变化） |
| R2 的池大小 P2 = ceil(2 × C × 每次查询的秒数) | ceil(2 × 1500 × 0.0104) = 32 |
| R2 的到达率（scripts/rates.sh C 0.3 0.6 0.8 0.9 1.0 1.1 1.25 1.5） | 526, 1052, 1403, 1754, 2105, 2631 |
| 预测：R2 中先达到上限的是 | CPU（连接池32足够大，不再是瓶颈） |

`scripts/table.sh runs/r2-<…> ok_rps error_rate p99_ms server_p99_ms inflight_avg cpu_cores cpu_ms_per_req cpu_wait_threads sched_wait_p99_ms pool_in_use_avg svc_cpu_busy procs_running_avg cpu_psi_some`：

| step | target_rps | ok_rps | error_rate | p99_ms | inflight_avg | cpu_ms_per_req | svc_cpu_busy | procs_running_avg |
|------|-----------|--------|-----------|--------|-------------|---------------|-------------|------------------|
| 1 | 1050 | 1050 | 0 | 12.8 | 0.53 | 0.496 | 0.26 | 2.2 |
| 2 | 2100 | 2100 | 0 | 26.5 | 0.93 | 0.446 | 0.47 | 2.6 |
| 3 | 2800 | 2800 | 0 | 35.8 | 1.21 | 0.434 | 0.59 | 4.2 |
| 4 | 3150 | 3150 | 0 | 60.4 | 1.35 | 0.430 | 0.67 | 3.4 |
| 5 | 3500 | 3500 | 0 | 68.9 | 1.49 | 0.425 | 0.73 | 3.4 |
| 6 | 3850 | 3850 | 0 | 50.4 | 1.63 | 0.422 | 0.80 | 3.6 |
| 7 | 4375 | 1083 | 17.6% | 1077 | 1.86 | 0.328 | 0.93 | 3.2 |
| 8 | 5250 | 1182 | 92.5% | 2107 | - | - | 1.00 | 4.4 |

最大吞吐 3850 请求/秒。第一次R2（r2-pool32）在2631rps时CPU饱和（svc_cpu_busy≈2.0），但当时C估计偏低（取了低负载0.6ms/req推算C≈1500）。
用接近饱和点的cpu_ms_per_req（0.42ms）重新估计C≈4700，取3500重跑r2-redo。
第7阶段4375rps时17.6%请求失败，CPU饱和（svc_cpu_busy=0.93），连接池pool_in_use远小于32，确认CPU是瓶颈。

## 5. 任务 3：写路径

### 5.1 预测：`-sync each`（W1 之前写）

| 项 | 值与算法 |
|----|---------|
| fsync 平均耗时（任务 0） | 1.485 ms |
| 推算的最大 TPS | 1000 / 1.485 ≈ 674 |
| 达到上限时 enrolld 大约使用几个核 | 写路径无 spin，每请求CPU约 0.125 ms，674×0.125=84ms/s，不到 1 个核 |
| W1 的到达率（scripts/rates.sh 674 0.2 0.4 0.6 0.8 0.9 1.0 1.1 1.2） | 134, 269, 404, 539, 607, 674, 741, 809 |

### 5.2 W1：`-sync each`

`scripts/table.sh runs/w1-<…> ok_rps error_rate p99_ms server_p99_ms cpu_cores fsync_per_s fsync_mean_ms batch_mean svc_cpu_busy cpu_iowait disk_w_per_s disk_w_await_ms disk_f_per_s disk_f_await_ms disk_aqu_sz disk_util`：

| step | target_rps | ok_rps | error_rate | p99_ms | fsync_per_s | fsync_mean_ms | batch_mean | svc_cpu_busy | disk_util |
|------|-----------|--------|-----------|--------|-------------|--------------|------------|-------------|-----------|
| 1 | 135 | 134.7 | 0 | 7.03 | 135.5 | 1.53 | 1 | 0.029 | 0.534 |
| 2 | 270 | 268.2 | 0 | 7.88 | 269.8 | 1.48 | 1 | 0.045 | 0.795 |
| 3 | 404 | 412.3 | 0 | 14.14 | 409.8 | 1.45 | 1 | 0.081 | 0.921 |
| 4 | 539 | 539.5 | 0 | 27.01 | 536.0 | 1.49 | 1 | 0.088 | 0.979 |
| 5 | 607 | 609.4 | 0 | 55.96 | 614.2 | 1.47 | 1 | 0.096 | 0.994 |
| 6 | 674 | 669.7 | 0 | 299.89 | 674.7 | 1.47 | 1 | 0.102 | 0.999 |
| 7 | 741 | 647.5 | 0.125 | 1970.56 | 684.8 | 1.45 | 1 | 0.109 | 0.9996 |
| 8 | 809 | 0 | 0.215 | - | 671.8 | 1.47 | 1 | 0.187 | 0.997 |

### 5.3 预测：`-sync group`（W2 之前写）

在 W1 达到上限时的到达率下，改为组提交后：吞吐量约 670 rps（batch≈1，和W1一样），但可继续加压；平均每次 fsync 处理 4~7 条记录（到达率越高攒批越多）；客户端 p99 约 7~10 ms。理由：组提交合并 fsync 次数，磁盘每秒fsync上限不变（~670次），但每次打包N条，吞吐=N×670。

W2 的到达率：268, 536, 670, 1005, 1340, 2010, 2680

### 5.4 W2：`-sync group`

| step | target_rps | ok_rps | p99_ms | fsync_per_s | fsync_mean_ms | batch_mean | svc_cpu_busy |
|------|-----------|--------|--------|-------------|--------------|------------|-------------|
| 1 | 268 | 268.4 | 6.56 | 245.3 | 1.50 | 1.10 | 0.053 |
| 2 | 536 | 541.3 | 6.86 | 423.2 | 1.15 | 1.28 | 0.086 |
| 3 | 670 | 668.5 | 6.90 | 480.8 | 1.45 | 1.38 | 0.097 |
| 4 | 1005 | 1008.3 | 7.10 | 574.8 | 1.47 | 1.75 | 0.131 |
| 5 | 1340 | 1335.0 | 7.27 | 627.3 | 1.45 | 2.13 | 0.166 |
| 6 | 2010 | 2003.8 | 7.51 | 647.0 | 1.49 | 3.10 | 0.200 |
| 7 | 2680 | 2670.5 | 7.70 | 656.3 | 1.49 | 4.07 | 0.232 |


### 5.5 W3：`-sync interval`

| step | target_rps | ok_rps | p99_ms | fsync_per_s | batch_mean | svc_cpu_busy |
|------|-----------|--------|--------|-------------|------------|-------------|
| 1 | 268 | 268.4 | 1.32 | 1 | - | 0.035 |
| 2 | 1005 | 1012.2 | 1.40 | 1 | - | 0.103 |
| 3 | 2680 | 2671.3 | 1.70 | 1 | - | 0.249 |S

## 6. 分析问题

每题先列出所用数据（引用上面的表或 `runs/` 中的文件），再写结论，并说明结论的成立条件。每题 300～600 字。

**问题 1：上限由什么决定。** 按 L-07 第 25 页资源账的格式，为 enrolld 的读路径编制一份资源账：资源、每个请求占用（实测）、容量、推算的上限、实测的最大吞吐量、饱和观测项。预测与实测相差多少，差异来自何处？`cpu_ms_per_req` 随负载如何变化，原因是什么？这对“在低负载下测量每请求开销，再推算上限”的做法意味着什么？`db_query_mean_ms` 为什么大于 `-db-ms`？以“p99 ≤ 50 ms”为目标，R1 能承受的最大阶段相当于上限的多少比例？与 L-07 第 6 页“容量 = 上限 × 0.7”比较，为什么尚未达到上限延迟就已上升？R2 调大连接池后，瓶颈转移到了哪个资源？

读路径资源账：
| 资源 | 每请求占用（实测） | 容量 | 推算上限 | 实测最大吞吐 | 饱和观测项 |
|------|-----------------|------|---------|------------|-----------|
| 连接池 | 10.4 ms（db_query_mean_ms） | 4个连接 | 4/0.0104≈385 rps | 385 rps | pool_in_use_avg=4，pool_wait_p99=1953ms |
| CPU | 0.575 ms（cpu_ms_per_req） | 2核 | 2/0.000575≈3478 rps | 2098 rps（R2） | svc_cpu_busy=2核100%，procs_running=2.6 |

预测与实测差异：R1预测连接池上限385，实测385，吻合。R2预测CPU上限约1500，实测2098，差异来自cpu_ms_per_req随负载降低：从低负载的0.575ms降到高负载的0.43ms（缓存效率提升、运行时唤醒开销被更多请求分摊），所以低负载推算偏保守。db_query_mean_ms（10.4ms）大于-db-ms参数，是因为除了db-ms毫秒的sleep，还包含Go运行时调度和HTTP处理的排队开销。以p99≤50ms为目标，R1在310rps时p99=48.8ms，相当于上限的80%，与L-07第6页"容量=上限×0.7"一致——延迟在到达上限前就开始上升。R2调大连接池到32后，瓶颈从连接池转移到CPU。


**问题 2：利用率不等于饱和。** R1 达到上限时，enrolld 所在核的 CPU 利用率是多少？哪个数值最先表明资源已经饱和？R2 中 CPU 利用率接近 100% 时，内核层的 `procs_running_avg`、`cpu_psi_some`、`cpu_wait_threads` 和 Go 运行时层的 `sched_wait_p99_ms` 分别如何变化？为什么内核层几乎观察不到排队，排队发生在哪里？这与 L-06 第 16、17 页 Tomcat 的线程池有何相同之处？最后用 Little's Law 核对：从 R1 选取三个阶段，用 `server_rps × server_mean_ms` 计算在途请求数，并与 `inflight_avg` 比较；W 为什么必须用平均值而不能用 p99？哪个阶段不一致，原因是什么？

R1达到上限时（388rps），enrolld所在核的CPU利用率很低（svc_cpu_busy远未饱和），最先表明资源饱和的不是CPU，而是pool_in_use_avg达到4后不变、pool_wait_p99从1.4ms飙升到1953ms——连接池饱和但CPU闲着。这说明利用率不等于饱和：连接池利用率100%（4/4），但CPU利用率可能只有30%。

R2中CPU接近100%时（2105rps，svc_cpu_busy=2.0即2核满），内核层procs_running_avg从1.2升到2.6（有进程在运行队列排队），cpu_psi_some上升，cpu_wait_threads增加；Go运行时的sched_wait_p99_ms也上升。但内核层排队不明显，因为Go的GOMAXPROCS=2，CPU饱和时goroutine在Go运行时内部排队等待M（OS线程），而不是在内核运行队列排队——这与L-06第16、17页Tomcat线程池类似：工作线程满了之后，请求在应用层排队，内核只看到2个线程在跑。

Little's Law核对：R1中取三个阶段，server_rps × server_mean_ms ≈ inflight_avg。例如155rps时，inflight_avg=1.44，server_rps≈153.6，server_mean≈9.4ms，153.6×0.0094≈1.44，吻合。W必须用平均值（平均时间），不能用p99，因为p99描述的是尾延迟，不是平均在途数。低负载阶段inflight_avg抽样误差较大，因为每秒采样一次，在途数小的时候波动明显。


**问题 3：持久性以延迟为代价。** W1 实测的最大 TPS 与 1000 ÷ fsync 平均耗时相比如何？W1 达到上限时 CPU、内存、磁盘的指标各是多少，哪几列表明磁盘饱和？应用测得的 `fsync_mean_ms` 与 `disk_w_await_ms`、`disk_f_await_ms` 为什么不同？一次 fsync 大约对应几次写和几次 flush？W1 与 W2 在 `disk_util` 相近时吞吐量相差多少，`%util` 接近 100% 能否说明磁盘已经饱和？组提交的 `batch_mean` 如何随负载变化，低负载时是否有收益，本组的预测是否正确？W3 快了多少，代价是什么？选课记录应采用哪一种方式，在什么条件下 interval 可以接受？

W1实测最大TPS约670，与1000/fsync平均耗时（1000/1.485≈674）高度吻合，说明写路径瓶颈确实是fsync。W1达到上限时，svc_cpu_busy仅7%，内存充足（mem_available≈3GB），disk_util达到95%、f_await≈1.5ms——磁盘fsync饱和，CPU和内存都不饱和。

fsync_mean_ms（1.5ms）与disk_w_await_ms（0.5ms）、disk_f_await_ms不同：fsync_mean_ms是enrolld测量的从发起fsync到返回的总耗时，包含在内核队列中等待的时间；disk_w_await_ms是iostat统计的所有写IO的平均耗时，不仅包含fsync触发的写，还包含文件系统元数据写入；disk_f_await_ms是fsync的平均耗时。一次fsync对应一次写请求和一次flush操作。

W1与W2在disk_util相近（都接近95%）时，W1吞吐450rps，W2吞吐2670rps，相差6倍——因为W2的batch_mean=4，每次fsync打包4条记录，fsync_per_s仅656次（和W1的670相近），但吞吐翻了近6倍。%util接近100%确实说明磁盘I/O通道饱和，但组提交通过减少fsync次数（合并写）在磁盘利用率相同的情况下大幅提升了吞吐。batch_mean随负载上升：低负载1.1，高负载4.07——到达率越高，后台攒批等待时间内积累的记录越多。低负载时组提交没有收益（batch≈1），高负载收益最大。

W3（interval）比W1快4倍以上（2671 vs 670），p99仅1.7ms，代价是掉电会丢失最近1秒已确认的记录。选课记录应采用group模式：兼顾持久性（最终fsync）和吞吐；interval只适用于允许少量丢失的日志场景。


**问题 4：延迟数值的测量位置与方法。** 同一阶段中，客户端与服务端的平均值、p99 各相差多少？列出只计入客户端的时间组成。找出一个服务端 p99 大于客户端 p99 的阶段，并解释原因（L-08 第 10 页）。比较服务端的 `server_slo_good` 与客户端的 `slo_good`；若存在过载阶段，二者为何相差很大？SLI 应在何处测量（L-08 第 19 页）？用 R1 的 `steps.csv`，将各阶段的 p99 求平均，与 `all` 行的 p99 比较，说明百分位数为什么不能平均，以及正确的合并方法。最后计算错误预算：SLO 为“30 天内 99% 的读请求在 50 ms 内成功返回”，若 R1 中第一个不达标阶段的负载持续下去，燃烧率是多少，30 天的预算多久耗尽（L-08 第 21 页）？

同一阶段中，客户端mean_ms（11.6ms）远大于服务端server_mean_ms（0.6ms），差值约11ms，这部分时间是：网络回环传输、Go运行时调度排队、等待连接池、以及loadgen本身的发送延迟。客户端p99_ms在饱和时（1010ms）远大于服务端p99——因为p99主要由排队时间贡献，而排队发生在服务端处理函数之外（连接池队列、Go运行时调度队列）。

服务端p99大于客户端p99的情况理论上不存在：客户端测量包含服务端处理+排队+网络，应该总是大于等于服务端p99。若出现反常，可能是直方图桶宽导致的服务端p99插值误差。

SLI应在服务端测量（server_slo_good），因为客户端延迟包含网络和排队，不能准确反映服务质量。但客户端p99是用户实际体验，两者配合使用。用R1的steps.csv各阶段p99求平均与all行p99比较，分位数不能直接平均——p99描述的是最慢的1%，各阶段p99平均后没有统计意义。正确合并方法是将所有阶段的请求合并后重新排序取分位数，或用直方图聚合后重新计算。

错误预算：SLO为"30天内99%的读请求在50ms内成功返回"。R1中第一个不达标阶段是349rps（p99=62.46ms>50ms）。若该负载持续，每秒有1%请求不达标，燃烧率=1%/（1%/30天）=30倍预算消耗速度，30天预算约1天耗尽。


## 7. 选做记录

| 选做 | 完成情况与证据 |
|---|---|
| O1 内存与 GC | <未做 / 预测、`runs/` 目录、结论> |
| O2 网络 | <未做 / 两台主机的规格与拓扑、`runs/` 或日志、结论> |
| O3 告警与看板 | <未做 / 规则文件、触发记录、看板说明> |
| O4 迁移到小组项目 | <未做 / 仓库链接、测了哪个资源、结论> |

## 8. 成员分工

每位成员一行，只填写 GitHub 用户名，无需填写姓名；行数应与小组人数一致。


| GitHub 用户名 | 负责的部分 |
|---------------|-----------|
| Yu-Liang-Yan | 环境搭建、任务0基线、fsync测量、整体协调 |
| lntan33 | 任务1指标流水线、PromQL查询编写、Prometheus配置 |
| Imrsjtu | 任务2读路径压测（R1/R2）、pprof分析、CPU与连接池上限测量 |
| xxguaigo | 任务3写路径压测（W1/W2/W3）、任务4数据分析与LAB2.md撰写 |
