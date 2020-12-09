# rtnetlink
The command `ip route` calls rtnetlink in the kernel to do it's work. `rtnetlink_net_init` is called once per net namespace, where a `sock` is created for receiving commands. the `sock` is an instance of `netlink_sock` whose `netlink_rcv` is set to `rtnetlink_rcv`, which gets called from `netlink_unicast_kernel` which is called by `netlink_sendmsg` which is then called by the system call `__sys_sendmsg`. user space program `ip` creates a netlink socket to talk to the per-net socket in the kernel space(`net->rtnl`, see `rtnetlink_net_init`).

### iproute2/lib/libnetlink.c
`__rtnl_talk_iov` funciton is where the talk with kernel(clib) happens. a snapshot of the kernel call stack when executing `ip a show` to dump the ip addresses of the devices.

	inet_fill_ifaddr() at devinet.c:1,656 0xffffffff819e94c4	
	in_dev_dump_addr() at devinet.c:1,786 0xffffffff819ea30c	
	inet_dump_ifaddr() at devinet.c:1,863 0xffffffff819ea4a3	
	rtnl_dump_all() at rtnetlink.c:3,785 0xffffffff81933ddb	
	netlink_dump() at af_netlink.c:2,246 0xffffffff8196fdee	
	__netlink_dump_start() at af_netlink.c:2,354 0xffffffff819715b6	
	netlink_dump_start() at netlink.h:246 0xffffffff81935361	
	rtnetlink_rcv_msg() at rtnetlink.c:5,526 0xffffffff81935361	
	netlink_rcv_skb() at af_netlink.c:2,470 0xffffffff819735c5	
	netlink_unicast_kernel() at af_netlink.c:1,304 0xffffffff81972da8	
	netlink_unicast() at af_netlink.c:1,330 0xffffffff81972da8	
	netlink_sendmsg() at af_netlink.c:1,919 0xffffffff8197318a	
	sock_sendmsg_nosec() at socket.c:651 0xffffffff81901286	
	sock_sendmsg() at socket.c:671 0xffffffff81901286	
	__sys_sendto() at socket.c:1,992 0xffffffff819022e9	
	__do_sys_sendto() at socket.c:2,004 0xffffffff8190233f	
	__se_sys_sendto() at socket.c:2,000 0xffffffff8190233f	
	__x64_sys_sendto() at socket.c:2,000 0xffffffff8190233f	
	do_syscall_64() at common.c:46 0xffffffff81ba95b3	
	entry_SYSCALL_64() at entry_64.S:118 0xffffffff81c0007c	
