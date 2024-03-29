---
title: "Write a Linux packet sniffer from scratch: part two- BPF"
date: 2022-03-28 14:15:15
tags: BPF, JIT, virtual machine
---

### Introduction

In the [previous article](https://organicprogrammer.com/2022/02/22/how-to-implement-libpcap-on-linux-with-raw-socket-part1/), we examined how to develop a network sniffer with `PF_SOCKET` socket in Linux platform. The sniffer developed in the last article captures all the network packets. But a powerful network sniffer like `tcpdump` should provide the packet filtering functionality. For instance, the sniffer can only capture the `TCP` segment(and skip the UPD), or it can only capture the packets from a specific source IP address. In this article, let's continue to explore how to do that. 

### Background of BPF

`Berkeley Packet Filter(BPF)` is the essential underlying technology for packet capture in Unix-like operating systems.  
Search BPF as the keyword online, and the result is very confusing. It turns out that `BPF` keeps evolving, and there are several associated concepts such as `BPF` `cBPF` `eBPF` and `LSF`. So let us examine those concepts along the timeline:

- In **1992**, `BPF` was first introduced to the BSD Unix system for filtering unwanted network packets. The proposal of BPF was from researchers in Lawrence Berkeley Laboratory, who also developed the `libpcap` and `tcpdump`. 

- In **1997**, Linux Socket Filter(LSF) was developed based on BPF and introduced in Linux kernel version 2.1.75. Note that `LSF` and `BPF` have some distinct differences, but in the Linux context, when we speak of BPF or LSF, we mean the same packet filtering mechanism in the Linux kernel. We'll examine the detailed theory and design of BPF in the following sections. 

- Originally, BPF was designed as a network packet filter. But in **2013**, BPF was widely extended, and it can be used for non-networking purposes such as performance analysis and troubleshooting. Nowadays, the extended BPF is called `eBPF`, and the original and obsolete version is renamed to classic BPF (`cBPF`). **Note that what we examine in this article is cBPF, and eBPF is not inside the scope of this article**. `eBPF` is the hottest technology in today's software world, and I'll talk about it in the future. 

### Where to place BPF

The first question to answer is where should we place the filter. The last article examines the path of a received packet  as follows: 

<img src="/images/pf-packet-socket.png" title="PF_PACKET socket" width="400px" height="300px">

The best solution to this question is to put the filter as early as possible in the path. Since copying a large amount of data from kernel space to the user space produces a huge overhead, which can influence the system performance a lot. So BPF is a kernel feature. The filter should be triggered immediately when a packet is received at the network interface.As the original BPF [paper](https://www.tcpdump.org/papers/bpf-usenix93.pdf) said **To minimize memory traffic, the major bottleneck in most modern system, the packet should be filtered 'in place' (e.g., where the network interface DMA engine put it) rather than copied to some other kernel buffer before filtering.**
Let's verify this behavior by examining the kernel source code as follows (**Note** the kernel code shown in this article is based on version 2.6, which contains the `cBPF` implementation.): 

```c
/* source code file of net/packet/af_packet.c */
/* packet_create: create socket */
static int packet_create(struct net *net, struct socket *sock, int protocol)
{
    /* some code omitted ... */
	po = pkt_sk(sk);
	sk->sk_family = PF_PACKET;
	po->num = proto;

	spin_lock_init(&po->bind_lock);
	po->prot_hook.func = packet_rcv; // attach hook function to socket

	if (sock->type == SOCK_PACKET)
		po->prot_hook.func = packet_rcv_spkt; // attach hook function to socket

	if (proto) {
		po->prot_hook.type = proto;
		dev_add_pack(&po->prot_hook);
		sock_hold(sk);
		po->running = 1;
	}
}
```
`packet_create` function handles the socket creation when the application calls the `socket` system call. In lines 11 and 14, it attaches the hook function to the socket. The hook function executes when the packet is received.

The following code block shows the hook function `packet_rcv`:  

```c
/* hook function packet_rcv is triggered, when the packet is received */
static int packet_rcv(struct sk_buff *skb, struct net_device *dev, struct packet_type *pt, struct net_device *orig_dev)
{
    /* some code omitted ... */
    sk = pt->af_packet_priv;
    snaplen = skb->len;
    res = run_filter(skb, sk, snaplen); // filter logic
    if (!res)
	    goto drop_n_restore; // drop the packet

    __skb_queue_tail(&sk->sk_receive_queue, skb); // put the packet into the queue
}
```
`packet_rcv` function calls `run_filter`, which is just the BPF logic part(Currently, you can regard it as a black box. In the next section, we'll examine the details). Based on the return value of `run_filter` the packet can be filtered out or put into the queue. 

So far, you can understand BPF(or the packet filtering) is working inside kernel space. But the packet sniffer is a user-space application. The next question is how to link the filtering rules in user space to the filtering handler in kernel space. 

To answer this question, we have to understand BPF itself. It's right time to understand this great piece of work. 

### BPF machine

As I mentioned above, `BPF` was introduced in this original [paper](https://www.tcpdump.org/papers/bpf-usenix93.pdf) written by researchers from Berkeley. I strongly recommend you read this great paper based on my own experience. In the beginning, I felt crazy to read it, so I read other related documents and tried to understand BPF. But most documents only cover one portion of the entire system, so it is difficult to piece all the information together. Finally, I read the original paper and connected all parts together. **As the saying goes, sometimes taking time is actually a shortcut.**

##### Virtual CPU

A packet filter is simply a boolean-valued function on a packet. If the value of the function is true the kernel copies the packet for the application; if it is false the packet is ignored. 

In order to be as flexible as possible and not to limit the application to a set of predefined conditions, the `BPF` is actually implemented as a `register-based virtual machine` (for the difference between stack-based and register-based virtual machine, you can refer to [this article](http://troubles.md/wasm-is-not-a-stack-machine/)) running a user-defined program.  

You can regard the `BPF` as a `virtual CPU`. And it consists of an `accumulator`, an `index register(x)`, a scratch memory store, and an implicit `program counter`. If you're not familiar with these concepts, I add some simple illustrations as follows:

- An `accumulator` is a type of register included in a CPU. It acts as a temporary storage location holding an intermediate value in mathematical and logical calculations. For example, in the operation of "1+2+3", the accumulator would hold the value 1, then the value 3, then the value 6. The benefit of an accumulator is that it does not need to be explicitly referenced.
- An `index register` in a computer's CPU is a processor register or assigned memory location used for modifying operand addresses during the run of a program. 
- A `program counter` is a CPU register in the computer processor which has the address of the next instruction to be executed from memory. 

In the BPF machine, the accumulator is used for arithmetic operations, while the index register provides offsets into the packet or the scratch memory areas.  

##### Instructions set and addressing mode
Same as the physical CPU, the `BPF` provides a small set of arithmetic, logical and jump instructions as follows, these instructions run on the BPF virtual machine(or CPU): 

<img src="/images/bpf-instructions.png" title="BPF instructions" width="400px" height="300px">

The first column *opcodes* lists the BPF instructions written in an assembly language style. For example, **ld**, **ldh** and **ldb** means to copy the indicated value into the `accumulator`. **ldx** means to copy the indicated value into the `index register`. **jeq** means jump to the target instruction if the `accumulator` equals the indicated value. **ret** means return the indicated value. You can check the functionality of the instructions set in detail in the paper. 

This kind of assembly-like style is more readable to humans. But when we develop an application (like the sniffer written in this article), we use binary code directly as the BPF instruction. This kind of binary format is called `BPF Bytecode`. I'll examine the way to convert this assembly language to bytecode later. 

The second column *addr modes* lists the addressing modes allowed for each instruction. The semantics of the addressing modes are listed in the following table: 

<img src="/images/address-mode.png" title="BPF instructions address mode" width="400px" height="300px">

For instance, **[k]** means the data at byte offset k in the packet. **#k** means the literal value stored in k. You can read the paper in detail to check the meaning of other address modes.  

##### Example BPF program

Now let's try to understand the following small BPF program based on the knowledge above: 

```asm
(000) ldh      [12]
(001) jeq      #0x800           jt 2    jf 3
(002) ret      #262144
(003) ret      #0
```
The BPF program consists of an array of BPF instructions. For example, the above BPF program contains four instructions. 

The first instruction **ldh** loads a half-word(16-bit) value into the accumulator from offset 12 in the Ethernet packet. According to the Ethernet frame format shown below, the value is just the `Ethernet type` field. The Ethernet type is used to indicate which protocol is encapsulated in the frame's payload (for example,  0x0806 for ARP, **0x0800** for IPv4, and 0x86DD for IPv6).

<img src="/images/ethernet-frame-format.png" title="Ethernet frame fromat" width="600px" height="400px">

The second instruction **jeq** compares the accumulator (currently stores `Ethernet type` field) to `0x800`(stands for IPv4). If the comparison fails, zero is returned, and the packet is rejected. If it is successful, a non-zero value is returned, and the packet is accepted. **So the small BPF program filters and accepts all IP packets**. You can find other BPF programs in the original paper. Go to read it, and you can feel the flexibility of BPF as well as the beauty of the design. 

##### Kernel implementation of BPF

Next, let's examine how kernel implements BPF. As mentioned above, the hook function `packet_rcv` calls `run_filter` to handle the filtering logic. `run_filter` is defined as follows: 
```c
/* Copied from net/packet/af_packet.c */
/* function run_filter is called in packet_rcv*/
static inline unsigned int run_filter(struct sk_buff *skb, struct sock *sk,
				      unsigned int res)
{
	struct sk_filter *filter;

	rcu_read_lock_bh();
	filter = rcu_dereference(sk->sk_filter); // get the filter bound to the socket
	if (filter != NULL)
		res = sk_run_filter(skb, filter->insns, filter->len); // the filtering is inside sk_run_filter function
	rcu_read_unlock_bh();

	return res;
}
```

You can find that the real filtering logic is inside `sk_run_filter`:  

```c
unsigned int sk_run_filter(struct sk_buff *skb, struct sock_filter *filter, int flen)
{
	struct sock_filter *fentry;	/* We walk down these */
	void *ptr;
	u32 A = 0;			/* Accumulator */
	u32 X = 0;			/* Index Register */
	u32 mem[BPF_MEMWORDS];		/* Scratch Memory Store */
	u32 tmp;
	int k;
	int pc;

	/*
	 * Process array of filter instructions.
	 */
	for (pc = 0; pc < flen; pc++) {
		fentry = &filter[pc];

		switch (fentry->code) {
		case BPF_ALU|BPF_ADD|BPF_X:
			A += X;
			continue;
		case BPF_ALU|BPF_ADD|BPF_K:
			A += fentry->k;
			continue;
		case BPF_ALU|BPF_SUB|BPF_X:
			A -= X;
			continue;
		case BPF_ALU|BPF_SUB|BPF_K:
			A -= fentry->k;
			continue;
		case BPF_ALU|BPF_MUL|BPF_X:
			A *= X;
			continue;
		/* some code omitted ... */
		case BPF_RET|BPF_K:
			return fentry->k;
		case BPF_RET|BPF_A:
			return A;
		case BPF_ST:
			mem[fentry->k] = A;
			continue;
		case BPF_STX:
			mem[fentry->k] = X;
			continue;
		default:
			WARN_ON(1);
			return 0;
		}
	}

	return 0;
}
```
Same as we mentioned, `sk_run_filter` is simply a boolean-valued function on a packet. It maintains the accumulator, the index register, etc. as local variables. And process the array of BPF filter instructions in a `for` loop. Each instruction will update the value of local variables. In this way, it simulates a virtual CPU. Interesting, right? 

##### BPF JIT

Since each network packet must go through the filtering function, it becomes the performance bottleneck of the entire system. 

A `just-in-time (JIT)` compiler was introduced into the kernel in **2011** to speed up BPF bytecode execution. 

- What is a `JIT` compiler? A `JIT` compiler runs **after** the program has started and compiles the code(usually bytecode or some type of VM instructions) on the fly(or just in time) into a form that's usually faster, typically the host CPU's native instruction set. This is in contrast to a `traditional compiler` that compiles all the code to machine language **before** the program is first run. 

In the `BPF` case, the `JIT` compiler translates BPF bytecode into a host system's assembly code directly, which can optimize the performance a lot. I'll not show details about JIT in this article. You can refer to the [kernel code](https://elixir.bootlin.com/linux/v3.19.8/source/arch/arm/net/bpf_jit_32.c#L868).  

### Set BPF in sniffer

Next, let's add BPF into our packet sniffer. As we mentioned above in the application level, the BPF instructions should use bytecode format with the following data structure:

```c
struct sock_filter {    /* Filter block */
        __u16   code;   /* Actual filter code */
        __u8    jt;     /* Jump true */
        __u8    jf;     /* Jump false */
        __u32   k;      /* Generic multiuse field */
};
```

How can we convert the BPF assembly language into bytecode? There are two solutions. First, there is a small helper tool called `bpf_asm`(which is provided along with the Linux kernel), and you can regard it as the BPF assembly language interpreter. But it is not recommended to application developers. 

Second, we can use `tcpdump`, which provides the converting functionality. You can find the following information from the tcpdump man page: 

- -d:   Dump the compiled packet-matching code in a human-readable form to standard output and stop.

- -dd:  Dump packet-matching code as a C program fragment.

- -ddd: Dump packet-matching code as decimal numbers (preceded with a count).

`tcpdump ip` means we want to capture all the IP packets. With options **-d**, **-dd** and **-ddd**, the output goes as follows: 

```bash
baoqger@ubuntu:~$ sudo tcpdump -d ip
[sudo] password for baoqger:
(000) ldh      [12]
(001) jeq      #0x800           jt 2    jf 3
(002) ret      #262144
(003) ret      #0

baoqger@SLB-C8JWZH3:~$ sudo tcpdump -dd ip
{ 0x28, 0, 0, 0x0000000c },
{ 0x15, 0, 1, 0x00000800 },
{ 0x6, 0, 0, 0x00040000 },
{ 0x6, 0, 0, 0x00000000 },

baoqger@SLB-C8JWZH3:~$ sudo tcpdump -ddd ip
4
40 0 0 12
21 0 1 2048
6 0 0 262144
6 0 0 0
```
Option **-d** prints the BPF instructions in assembly language (same as the example BPF program shown above). Options **-dd** prints the bytecode as a C program fragment. **So tcpdump is the most convenient tool when you want to get the BPF bytecode**.

The BPF filter bytecode (wrapped in the structure `sock_fprog`) can be passed to the kernel through `setsockopt` system call as follows: 

```c
// attach the filter to the socket
// the filter code is generated by running: tcpdump tcp
struct sock_filter BPF_code[] = {
	{ 0x28, 0, 0, 0x0000000c },
	{ 0x15, 0, 1, 0x00000800 },
	{ 0x6, 0, 0, 0x00040000 },
	{ 0x6, 0, 0, 0x00000000 }
};    
struct sock_fprog Filter;
// error prone code, .len field should be consistent with the real length of the filter code array
Filter.len = sizeof(BPF_code)/sizeof(BPF_code[0]); 
Filter.filter = BPF_code;


if (setsockopt(sock, SOL_SOCKET, SO_ATTACH_FILTER, &Filter, sizeof(Filter)) < 0) {
	perror("setsockopt attach filter");
	close(sock);
	exit(1);
} 
```

`setsockopt` system call triggers two kernel functions: `sock_setsockopt` and `sk_attach_filter` (I'll not show the details for these two functions), which **binds the filters to the socket**. And in `run_filter` kernel function (mentioned above), it can **get the filters from the socket** and **execute the filters on the packet**. 

So far, every piece is connected. The puzzle of BPF is solved. The `BPF` machine allows the user-space applications to inject customized BPF programs straight into a kernel. Once loaded and verified, BPF programs execute in kernel context. These BPF programs operate inside kernel memory space with access to all the internal kernel states available to it. For example, the `cBPF` machine which uses the network packet data. But this power can be extended as `eBPF`, which can be used in many other varied applications. As someone [said](https://www.brendangregg.com/bpf-performance-tools-book.html) **In some way, eBPF does to the kernel what Javascript does to the websites: it allows all sorts of new application to be created.**  In the future, I plan to examine eBPF in depth. 

<img src="/images/bpf-run-instructions.png" title="BPF Run Instructions" width="600px" height="400px">

### Process the packet

We examined the `BPF` filtering theory on the kernel level a lot in the above section. But for our tiny sniffer, the last step we need to do is process the network packet. 

- First, the `recvfrom` system call reads the packet from the socket. And we put the system call in a `while` loop to keep reading the incoming packets. 

- Then, we print the source and destination `MAC` address in the packet(the packet we got is a raw Ethernet frame in Layer 2, right?). And if what this Ethernet frame contains is an `IP4` packet, then we print out the source and destination `IP` address. To understand more about it, you can study the header format of various network protocols. I will not cover in details here.

```c
while(1) {
	printf("-----------\n");
	n = recvfrom(sock, buffer, 2048, 0, NULL, NULL);
	printf("%d bytes read\n", n);

	/* Check to see if the packet contains at least
	* complete Ethernet (14), IP (20) and TCP/UDP
	* (8) headers.
	*/
	if (n < 42) {
		perror("recvfrom():");
		printf("Incomplete packet (errno is %d)\n", errno);
		close(sock);
		exit(0);
	}

	ethhead = buffer;
	printf("Source MAC address: %.2x:%.2x:%.2x:%.2x:%.2x:%.2x\n",
		ethhead[0], ethhead[1], ethhead[2], ethhead[3], ethhead[4], ethhead[5]
	);
	printf("Destination MAC address: %.2x:%.2x:%.2x:%.2x:%.2x:%.2x\n",
		ethhead[6], ethhead[7], ethhead[8], ethhead[9], ethhead[10], ethhead[11]
	);

	iphead = buffer + 14; 

	if (*iphead==0x45) { /* Double check for IPv4
						* and no options present */
		printf("Source host %d.%d.%d.%d\n",
				iphead[12],iphead[13],
				iphead[14],iphead[15]);
		printf("Dest host %d.%d.%d.%d\n",
				iphead[16],iphead[17],
				iphead[18],iphead[19]);
		printf("Source,Dest ports %d,%d\n",
				(iphead[20]<<8)+iphead[21],
				(iphead[22]<<8)+iphead[23]);
		printf("Layer-4 protocol %s\n", transport_protocol(iphead[9]));
	}
}
```

You can find the complete source code of the sniffer in this Github [repo](https://github.com/baoqger/raw-socket-packet-capture-/blob/master/raw_socket.c).

### Summary
In this article, we examine how to add filters to our sniffer. First, we analyze why the filter should be running inside kernel space instead of the application space. Then, this article examines the `BPF` machine design and implementation in detail based on the paper. We reviewed the kernel source code to understand how to implement the `BPF` virtual machine. As I mentioned above, the original `BPF`(`cBPF`) was extended to `eBPF` now. But the understanding of the BPF virtual machine is very helpful to `eBPF` as well.   