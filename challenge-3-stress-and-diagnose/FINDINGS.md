# Challenge 3 — Findings

## Root cause

The AWS DevOps Agent investigated the EC2 instance `challenge3-stress` and determined that the high CPU utilization was caused by an intentional stress workload launched during instance startup.

The EC2 UserData script created multiple infinite busy-loop processes, one for each available CPU thread. These processes continuously consumed CPU resources, resulting in sustained high CPU utilization and triggering the `challenge3-high-cpu` CloudWatch alarm.

## Fix applied

I connected to the EC2 instance using AWS Systems Manager Session Manager and terminated the runaway CPU-intensive processes.

```bash
pkill -f 'while true; do :; done'
```

After stopping the processes, CPU utilization dropped from nearly 100% to normal levels. The instance performance recovered and the CloudWatch alarm returned to a healthy state.

## Evidence

* [x] Screenshot 1: DevOps Agent diagnosis identifying the CPU saturation issue (`screenshots/ss1.png`)
* [x] Screenshot 2: EC2 recovery showing CPU utilization returned to normal (`screenshots/ss2.png`)
