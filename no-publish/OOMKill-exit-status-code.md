---
title: OOMKill in Kubernetes
published: true
description:
tags: 'linux,bash,kubernetes'
---

When a container(process) is terminated due to an [Out Of Memory (OOM) manager][out-of-mem-mngmnt] condition, Kubernetes marks it as **OOMKilled**, and the exit code [**137**][list-of-exit-codes] is logged for troubleshooting.

Exit codes `1 - 2`, `126 - 165`, and `255` have special meanings, and should therefore be avoided for user-specified exit parameters. Ending a script with exit 127 would certainly cause confusion when troubleshooting (is the error code a "command not found" or a user-defined one?).

[How OOM Kill works ?][out-of-mem-mngmnt]

Linux operating systems have specific ways of managing memory. One of the policies is over-commitment, which allows applications to reserve as much memory as they want in advance. However, the promised memory may not be available for actual use. Then, the system must provide a remarkable means to avoid running out of memory.

## Resources

- [Exit Status][exit-status]
- [Exit Codes][list-of-exit-codes]
- [Out Of Memory Management][out-of-mem-mngmnt]
- [Linux Memory Overcommitment and the OOM Killer][memory-overcommitment-oom-killer]
- [Kill signal linux kernel][kill-sig-in-linux]
- [OOM Dashboard][oom-grafana-dashboard]
- [Kubernetes OOMKilled Error][kubernetes-oomkilled-error]

[exit-status]: https://www.gnu.org/software/bash/manual/html_node/Exit-Status.html
[list-of-exit-codes]: https://tldp.org/LDP/abs/html/exitcodes.html
[out-of-mem-mngmnt]: https://www.kernel.org/doc/gorman/html/understand/understand016.html
[kill-sig-in-linux]: https://github.com/torvalds/linux/blob/master/mm/oom_kill.c#L948
[oom-grafana-dashboard]: https://grafana.com/grafana/dashboards/16718-oom-and-restarts/
[kubernetes-oomkilled-error]: https://lumigo.io/kubernetes-troubleshooting/kubernetes-oomkilled-error-how-to-fix-and-tips-for-preventing-it/
[memory-overcommitment-oom-killer]: https://www.baeldung.com/linux/memory-overcommitment-oom-killer
