# Week 6 – Performance Evaluation and Analysis

## 1. Testing Approach and Methodology
This week, I conducted detailed performance tests on the applications selected in Week 3: `stress-ng` (CPU/memory), `fio` (disk I/O), `iperf3` (network), and `nginx` (web server). For each workload I ran:

- **Baseline**: idle system measurements.
- **Load test**: controlled stress for ~60s.
- **Analysis**: comparison of CPU, memory, disk and network metrics.
- **Optimisation**: nginx tuning and configuration changes followed by repeat tests.

All monitoring commands (`vmstat`, `iostat`, `free`, `df`) were executed remotely from the workstation over SSH, consistent with the earlier monitoring strategy.
<img width="865" height="257" alt="image" src="https://github.com/user-attachments/assets/559eac15-ea62-42ec-98dd-82335e1bea94" />

## 2. Performance
The following summarises the key metrics collected for each scenario. CPU values are derived from `vmstat` (100 − average idle), memory from `free -h`, disk throughput from `iostat`/`fio`, network throughput from `iperf3`, and HTTP performance from ApacheBench (`ab`).

```bash
cat nginx_baseline_ab.txt
```
<img width="957" height="912" alt="image" src="https://github.com/user-attachments/assets/bf549373-6cd6-4b0f-b17e-6a8e60acb7a8" />

```bash
cat nginx_baseline_vmstat.txt
```
<img width="958" height="736" alt="image" src="https://github.com/user-attachments/assets/296a4e7d-6d72-4fd4-a481-077a203d3615" />


## 3. Testing Evidence
To make the results easier to interpret, I generated several charts from the measurement data:

- CPU utilisation by scenario 
- Disk throughput for `fio` sequential write vs baseline
- HTTP requests per second for nginx baseline vs tuned configuration

```bash
stress-ng --cpu 2 --timeout 60s
stress-ng --vm 2 --vm-bytes 75% --timeout 60s
```
<img width="852" height="407" alt="image" src="https://github.com/user-attachments/assets/d7198905-b930-49d8-8003-f567555cb236" />

```bash
fio --name=seqwrite --filename=/tmp/fio-testfile --size=512M --bs=1M --rw=write --direct=1
```
<img width="1890" height="699" alt="image" src="https://github.com/user-attachments/assets/0a835491-39de-4f6e-abdb-13a8ff9df64e" />

```bash
cat nginx_baseline_ab.txt | grep -E 'Requests per second|Time per request'
cat nginx_tuned_ab.txt | grep -E 'Requests per second|Time per request'
```
<img width="1073" height="210" alt="image" src="https://github.com/user-attachments/assets/d879ac76-5a9c-4eeb-a024-d2540be3074f" />

## 4. Network Performance Analysis
Using `iperf3` and `ping`, I measured end-to-end throughput and latency between the workstation and the Ubuntu server:

```bash
iperf3 -s                 #server
iperf3 -c 192.168.1.171   #client
ping 192.168.1.171 -c 20
```
<img width="893" height="973" alt="image" src="https://github.com/user-attachments/assets/ab985ab6-7675-4852-8386-9a44c557fe98" />

## 5. Optimisation Analysis and Results
The main optimisation work focused on the nginx web server configuration.
Baseline configuration used the default number of worker processes. I changed:

```nginx
worker_processes auto;
events {
    worker_connections 1024;
}
```
and restarted nginx. Running the same ApacheBench workload (ab -n 200 -c 10) before and after tuning produced:
| Scenario       | Requests per second | Time per request (ms) |
| -------------- | ------------------- | --------------------- |
| Baseline nginx | 2833.82 [/sec]                   | 3.529 [ms]                  |
| Tuned nginx    | 3111.92 [/sec]                  | 3.213 [ms]                   |

This shows an improvement of approximately 9.90% in throughput and a corresponding reduction in average response time.
## 6. Reflection
I​‍​‌‍​‍‌ could practically see in week 6 how the OS behaves when it is put under different pressures than just reading about it. It was made very clear to me how fast the CPU could be saturated by the operation of the system when I ran `stress-ng`: during the CPU test, the `vmstat` showed user + system time close to 100%, while memory and disk were almost idle. On the other hand, the memory test made the “used” RAM in `free-h` increase substantially, although CPU usage was moderate. The `fio` benchmark was instrumental in uncovering yet another bottleneck since `iostat` was reporting high write throughput and `%util` on the disk while the CPU was relatively low. Network instruments like `iperf3` and `ping` were very helpful in confirming that latency was minimal, and bandwidth was not the main limiting factor inside the virtual lab.

The experiments with nginx were the icing on the cake because I could associate resource consumption directly with an application that users would actually interact with. The baseline ApacheBench results demonstrated decent performance, but after modifying the nginx worker configuration, the requests-per-second were increased, and the time-per-request was reduced, thereby validating that tiny configuration changes can have a tangible effect. To sum up, this week has been a great learning experience for me to regard performance as different trade-offs between CPU, memory, disk, and network rather than just a single “fast or slow” label and to depend on tools and measurements instead of making bottleneck identification and optimisation testing ​‍​‌‍​‍‌guesses.
