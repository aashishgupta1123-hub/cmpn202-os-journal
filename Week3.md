# Week 3 – Application Selection for Performance Testing

## 1. Application Selection Matrix

The goal is to cover a range of workload types (CPU, memory, disk I/O, network, and a real server application) so that I can observe how the system behaves under different conditions in later phases.

| Application | Workload Type        | Reason for Selection |
|------------|----------------------|----------------------|
| `stress-ng` | CPU-intensive        | Designed specifically to stress CPU cores with different workloads. It allows me to control the number of workers and load level, making it ideal for observing CPU utilisation and scheduling behaviour. |
| `stress-ng` (memory mode) | RAM-intensive | Provides options to allocate and touch large amounts of memory (`--vm` workers), which makes it easy to generate predictable memory pressure and observe swapping and cache behaviour. |
| `fio`       | Disk I/O-intensive   | A flexible disk benchmarking tool that can generate sequential and random reads/writes with configurable block sizes. This is suitable for analysing disk throughput and I/O wait under different patterns. |
| `iperf3`    | Network-intensive    | A standard network performance testing tool that can measure TCP and UDP throughput between the server and workstation, helping me understand the impact of network traffic on CPU usage and latency. |
| `nginx`     | Server application (HTTP) | A lightweight, widely-used web server that lets me simulate realistic client-server interactions. It provides a real service to benchmark (HTTP requests) while monitoring CPU, memory, disk and network usage. |

## 2. Installation Documentation (SSH-based)

All installation was performed from the Windows workstation using SSH, in line with the requirement to administer the server remotely.
From the workstation:

```bash
ssh ashish@192.168.1.171
```
<img width="949" height="811" alt="image" src="https://github.com/user-attachments/assets/be231449-51e7-4d64-ad5d-0e6e3d07c65d" />

Once connected to the server, I installed the test applications using `apt`:

```bash
sudo apt update
sudo apt install -y stress-ng nginx fio iperf3
```
<img width="1269" height="993" alt="image" src="https://github.com/user-attachments/assets/d8356384-e350-4a14-822b-14bd5e890642" />

I then verified that each application was available:

```bash
stress-ng --version
```
<img width="999" height="58" alt="image" src="https://github.com/user-attachments/assets/9b8cbaa5-5af8-4905-a3b4-da3ff9d95159" />

```bash
nginx -v
```
<img width="563" height="54" alt="image" src="https://github.com/user-attachments/assets/4a08d3b8-4be1-40b9-bbf3-c2fc3d4870c0" />

```bash
fio --version
```
<img width="508" height="54" alt="image" src="https://github.com/user-attachments/assets/07d398c3-0423-4531-a559-2ba745585d92" />

```bash
iperf3 --version
```
<img width="1880" height="132" alt="image" src="https://github.com/user-attachments/assets/bf577a72-74ff-4056-915f-cea791f0b596" />

## 3. Expected Resource Profiles

Before running any detailed benchmarks, I defined expected resource profiles for each selected application. These expectations will later be compared with measured data.

| Application / Scenario          | CPU Usage (Expected)                          | Memory Usage (Expected)                          | Disk I/O (Expected)                                  | Network Usage (Expected)                            | Notes |
|---------------------------------|-----------------------------------------------|--------------------------------------------------|------------------------------------------------------|-----------------------------------------------------|------|
| `stress-ng` (CPU workers)       | Very high CPU utilisation on all used cores.  | Low to moderate (mainly for process overhead).   | Minimal disk activity once binaries are loaded.      | No network usage.                                   | Used to push CPU scheduling and thermal limits. |
| `stress-ng` (memory workers)    | Moderate CPU (handling memory operations).    | High RAM usage proportional to allocated memory; possible swap usage if allocation exceeds physical RAM. | Occasional disk I/O if swapping occurs.             | No network usage.                                   | Used to study memory pressure and swapping. |
| `fio` (sequential read/write)   | Low to moderate CPU (I/O-bound).              | Low memory footprint (buffers and metadata).     | High disk throughput with sustained sequential reads/writes. | No network usage.                                   | Helps characterise raw disk performance. |
| `iperf3` (TCP throughput tests) | Moderate CPU on both server and client during tests. | Low to moderate memory usage.                   | Minimal disk usage (only logging).                   | High network throughput (close to link capacity) during tests. | Used to evaluate network stack and NIC performance. |
| `nginx` under HTTP load         | CPU depends on request rate; moderate under realistic load, higher under stress. | Memory usage increases with worker processes and connection count, but should remain stable. | Disk I/O mainly for reading static content and logging; potentially higher if logs are verbose. | Moderate to high network usage depending on client request pattern. | Represents a realistic server workload for the system. |

## 4. Monitoring Strategy
For each application, I will reuse the SSH-based monitoring approach defined in Week 2. All monitoring commands will be executed from the workstation using SSH and later automated in the `monitor-server.sh` script.

### 4.1 General approach
- Start the target workload (e.g. `stress-ng`, `fio`, `iperf3`, or an HTTP load generator hitting nginx).
- From the workstation, run monitoring commands over SSH, logging output to timestamped files, for example:

```bash
ssh ashish@192.168.1.171 "vmstat 1 60" > logs/vmstat_stress-ng_cpu_60s.txt
```
<img width="1241" height="662" alt="image" src="https://github.com/user-attachments/assets/30a88efb-0851-48ab-b726-e9df08e0c8dd" />

```bash
ssh ashish@192.168.1.171 "iostat -xz 1 60" > logs/iostat_fio_seq_60s.txt
```
<img width="1882" height="635" alt="image" src="https://github.com/user-attachments/assets/3367c2ab-a2e0-437c-9e46-26fbeb1e455f" />
<img width="1897" height="706" alt="image" src="https://github.com/user-attachments/assets/abd52015-15b8-448b-ac56-14e5f0bdfd77" />


- Logs will be stored with descriptive filenames including the tool, scenario and duration.
- In Week 5, I will encapsulate these SSH commands into a monitor-server.sh script so that each test run uses a consistent monitoring procedure.

### 4.2 Application-specific monitoring
```bash
stress-ng (CPU mode)
```
- Metrics: CPU utilisation, run queue length, context switches.
- Tools: vmstat 1, mpstat 1, top/htop.
- Method: Run stress-ng with a fixed number of workers, capture vmstat output for the duration of the test, and verify that all cores are busy.
 
```bash
stress-ng (memory mode)
```
- Metrics: Memory usage, swap in/out.
- Tools: vmstat 1, free -h, dmesg (for OOM or swap messages).
- Method: Run stress-ng --vm with increasing memory sizes and watch for changes in free memory and swap usage.

```bash
fio (disk I/O)
```
- Metrics: Disk throughput (read/write), I/O wait time, latency.
- Tools: iostat -xz 1, vmstat 1, dstat or iotop (if needed).
- Method: Run a set of fio jobs (sequential and random, read and write) and capture iostat output to see how the disk behaves under load.
 
```bash
iperf3 (network)
```
- Metrics: Network throughput, CPU usage during transfers.
- Tools: iperf3 output itself, plus vmstat 1 or top for CPU.
- Method: Run iperf3 in server mode on the VM and client mode on the workstation, then record the reported bandwidth and CPU usage.

```bash
nginx (HTTP server)
```
- Metrics: CPU usage, memory usage, request rate, and possibly response times.
- Tools: vmstat 1, top/htop, nginx access logs, and a simple HTTP load tool (e.g. curl in a loop or a separate benchmark tool).
- Method: Generate HTTP requests to nginx from the workstation while logging system metrics and later correlate request volume with resource usage.

## 5. Reflection
During​‍​‌‍​‍‌ this week, I changed my focus from abstract performance goals to tangible instruments and applications. The decision to use `stress-ng`, `fio`, `iperf3`, and `nginx` made me understand that CPU, memory, disk, and network loads are different and that each one should be measured in a different way. It will be much easier to find that something is going wrong if it is written down that the resource profiles are expected, for example, in case a disk test is found to saturate the CPU unexpectedly. I felt that more administration should be done remotely and not via the VM console, which is in line with the coursework requirements, when I was documenting the installation commands over ​‍​‌‍​‍‌SSH.
