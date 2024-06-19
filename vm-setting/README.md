# Changing Config `allow_mmap`

During this exercise, I came accross the `node.store.allow_mmap: false` setting from the EKS documentation page.
It brought me to the [Setting Virtual Memory with Daemonset](https://www.elastic.co/guide/en/cloud-on-k8s/2.7/k8s-virtual-memory.html) page.

The problem is that if I chose `mmapfs` I would need to change kernel level settings. This could lead to unexpected application behavior of the apps running within the cluster. Also, in my humble opinion, changing Kernel level settings are far beyond the responsibilities of the platform team since it seems more on infrastructure side. But this is ONLY if the `max_map_count`is too low.

Luckily, `niofs` is an option available and seems to be a good fit for production workloads, due to the fact it is optimized for concurrency. However it has some problems in Windows environments.

# Decision

For the sake of the exercise, I decided:
    - check the `max_map_count` current value
    - check if the cluster is not Windows-based so that `niofs`ß

# Solution

I will apply the deamonset on this folder (`daemonset.yaml`), and run commands as root on the nodes to verify I can opt for the `niofs` or `mmap` setting:

```
$ chroot /host /bin/bash
[root@ip-10-0-1-209 /]# cat /etc/os-release
NAME="Amazon Linux"
VERSION="2"
ID="amzn"
ID_LIKE="centos rhel fedora"
VERSION_ID="2"
PRETTY_NAME="Amazon Linux 2"
ANSI_COLOR="0;33"
CPE_NAME="cpe:2.3:o:amazon:amazon_linux:2"
HOME_URL="https://amazonlinux.com/"
SUPPORT_END="2025-06-30"

#Also
[root@ip-10-0-3-51 /]# cat /proc/sys/vm/max_map_count
524288

```

This definitely verifies that the cluster is Linux based; and that the `max_map_count`is a high number. Thus `niofs` and `mmap` both are available options for `index.store.type`. By explicit declaration and not arbitrary, like `hybridfs`, I will select `niofs` as my store type.

# References

[ECK Virtual Memory](https://www.elastic.co/guide/en/cloud-on-k8s/master/k8s-virtual-memory.html#k8s-virtual-memory)
[Setting Virtual Memory with Daemonset](https://www.elastic.co/guide/en/cloud-on-k8s/2.7/k8s-virtual-memory.html)
