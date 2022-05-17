### Basic guide to Kernel modules 
how to write a hello-world module
how to build it
how to debug it with advanced Linux command

### miniFirewall based on Netfilter module
#### Netfilter hook design

#### version1: just logging and accepting the packet
#### version2: filter packets based on protocol and IP address
#### version3: pass filter rules from user space to modules

### more space to explore
#### stateful firewall

1 netfilter vs bpf
  netfiltr in year 1998. bpf in year 1997
  firewall只是netfilter的一个功能而已。

2 linux module 101
  makefile 解释 
    kbuild
    -C option
    obj-m vs obj-y
  相关commnds的解释 => lsmod, insmod, rmmod, modprobe, dmesg

3 netfilter 架构解释 
  hooks
  hook function相关参数解释：priority等
  kernel source code看hook的流程 => ip_rcv

4 实现firewall
  基于protocol，基于ip address => hardcode filter条件
  config filters manually => module和user space的通信
  一个hook多个callback => 

把自己实现的功能对应于某个iptables的command，比如下面这样
iptables -A OUTPUT -p tcp -d bigmart.com -j ACCEPT
iptables -A OUTPUT -p tcp -d bigmart-data.com -j ACCEPT
iptables -A OUTPUT -p tcp -d ubuntu.com -j ACCEPT
iptables -A OUTPUT -p tcp -d ca.archive.ubuntu.com -j ACCEPT
iptables -A OUTPUT -p tcp --dport 80 -j DROP
iptables -A OUTPUT -p tcp --dport 443 -j DROP
iptables -A INPUT -p tcp -s 10.0.3.1 --dport 22 -j ACCEPT
iptables -A INPUT -p tcp -s 0.0.0.0/0 --dport 22 -j DROP

background
防火linux墙自研
Firewalls are an important tool that can be configured to protect your servers and infrastructure. 
Netfilter is a framework provided by the Linux kernel that allows various networking-related operations to be implemented including firewall.
能学到的东西:
Linux kernel module development.
Linux kernel network programming. 
Linux kernel Netfilter module development.



Netfilter is a framework provided by the Linux kernel that allows various networking-related operations to be implemented in the form of customized handlers.

Netfilter provides hooks at the critical places on the packet traversal path inside the Linux kernel. These hooks allow packet to go through additional program logics that are installed by system administrators. These program logics need to be installed inside the kernel, and the loadable kernel module technology makes it convenient to achieve that. 

Iptables is a user-space utility prograsm that allows a system administrator to configure the IP packet fitler rules of the Linux kernel firewall, implemented as different Netfilter modules. Iptables superseded ipchains; and the successor of iptables is nftables.


Although Linux is a monolithic kernel, it can be extended using kernel modules. These are special objects that can be inserted into the kernel and removed on demand. In practical terms, kernel modules make it possible to add and remove drivers and interfaces that are not included in the kernel itself. 
