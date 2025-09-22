---
title: OOMKill in Kubernetes and Linux (Exit Code 137)
published: true
description: null
tags: 'linux,bash,kubernetes'
id: 2861019
date: '2025-09-22T13:23:25Z'
---

When a container(process) is terminated due to an [Out Of Memory (OOM) manager][out-of-mem-mngmnt] condition, Kubernetes marks it as **OOMKilled**. In this case , the Linux kernel’s OOM Killer terminates the process and the container exits with exit code [**137**][list-of-exit-codes]. This exit code is an important troubleshooting signal in Kubernetes.

### Why OOMKill Happens

Linux uses an over-commitment policy for memory management. Applications can request more memory than is physically available. When the actual demand exceeds system capacity, the kernel must decide which process to kill in order to reclaim memory. [How OOM Kill works ?][out-of-mem-mngmnt]

- The OOM Killer selects and terminates processes based on their OOM score.
- In Kubernetes, this usually happens when a container exceeds its memory limit.
- Kubernetes then records the container’s state as OOMKilled, and logs exit code 137. [OOM Killer Sends What Signal to the Process ?][kill-sig-in-linux]

[Exit codes][list-of-exit-codes] `1 - 2`, `126 - 165`, and `255` have special meanings, and should therefore be avoided for user-specified exit parameters. Ending a script with exit 127 would certainly cause confusion when troubleshooting (is the error code a "command not found" or a user-defined one?).

## Resources

- [Exit Status][exit-status]
- [Exit Codes][list-of-exit-codes]
- [Out Of Memory Management][out-of-mem-mngmnt]
- [Linux Memory Overcommitment and the OOM Killer][memory-overcommitment-oom-killer]
- [Kill signal linux kernel][kill-sig-in-linux]
- [OOM Dashboard][oom-grafana-dashboard]
- [Kubernetes OOMKilled Error][kubernetes-oomkilled-error]
- [Virtual Memory in Linux][virt-mem-in-linux]
- [Go Memory Management][go-mem-mngmnt]
- [Virtual Memory][virt-memory]
- [Memory Segments in C][mem-seg-in-c]
- Tasks
  - [Kill Process Group on OOM Event][kill-process-group-on-oom-event]
  - [Make the Container Exit When One of Its Processes Runs Out of Memory][kill-container-on-child-process-oom-event-docker]
  - [Make a Kubernetes Pod Survive an OOM Event Without Restarting][make-kubernetes-pod-outlive-oom-event]

[exit-status]: https://www.gnu.org/software/bash/manual/html_node/Exit-Status.html
[list-of-exit-codes]: https://tldp.org/LDP/abs/html/exitcodes.html
[out-of-mem-mngmnt]: https://www.kernel.org/doc/gorman/html/understand/understand016.html
[kill-sig-in-linux]: https://github.com/torvalds/linux/blob/master/mm/oom_kill.c#L948
[oom-grafana-dashboard]: https://grafana.com/grafana/dashboards/16718-oom-and-restarts/
[kubernetes-oomkilled-error]: https://lumigo.io/kubernetes-troubleshooting/kubernetes-oomkilled-error-how-to-fix-and-tips-for-preventing-it/
[memory-overcommitment-oom-killer]: https://www.baeldung.com/linux/memory-overcommitment-oom-killer
[virt-mem-in-linux]: https://www.youtube.com/watch?v=2bjuqRLFaHc
[go-mem-mngmnt]: https://povilasv.me/go-memory-management/
[kill-process-group-on-oom-event]: https://labs.iximiuz.com/challenges/kill-process-group-on-oom-event
[kill-container-on-child-process-oom-event-docker]: https://labs.iximiuz.com/challenges/kill-container-on-child-process-oom-event-docker
[make-kubernetes-pod-outlive-oom-event]: https://labs.iximiuz.com/challenges/make-kubernetes-pod-outlive-oom-event
[virt-memory]: https://www.cs.uic.edu/~jbell/CourseNotes/OperatingSystems/9_VirtualMemory.html
[mem-seg-in-c]: https://youtu.be/urU7UhF7D3Q
