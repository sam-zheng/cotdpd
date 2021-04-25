# Notes on the `ping` program


### call stack of the `sendto` syscall of ping (until `arp_constructor`)
	
	arp_constructor() at arp.c:221 0xffffffff819e6090	
	___neigh_create() at neighbour.c:595 0xffffffff8193215d	
	__neigh_create() at neighbour.c:671 0xffffffff8193279b	
	ip_neigh_gw4() at route.h:374 0xffffffff819ae015	
	ip_neigh_for_gw() at route.h:392 0xffffffff819ae015	
	ip_finish_output2() at ip_output.c:223 0xffffffff819ae015	
	ip_finish_output() at ip_output.c:317 0xffffffff819b0528	
	NF_HOOK_COND() at netfilter.h:290 0xffffffff819b0528	
	ip_output() at ip_output.c:431 0xffffffff819b0528	
	ip_send_skb() at ip_output.c:1,567 0xffffffff819b0e40	
	ip_push_pending_frames() at ip_output.c:1,587 0xffffffff819b0e96	
	raw_sendmsg() at raw.c:672 0xffffffff819dd20b	
	sock_sendmsg_nosec() at socket.c:651 0xffffffff8190127f	
	sock_sendmsg() at socket.c:671 0xffffffff8190127f	
	__sys_sendto() at socket.c:1,992 0xffffffff819022e9	
	__do_sys_sendto() at socket.c:2,004 0xffffffff8190233f	
	__se_sys_sendto() at socket.c:2,000 0xffffffff8190233f	
	__x64_sys_sendto() at socket.c:2,000 0xffffffff8190233f	
	do_syscall_64() at common.c:46 0xffffffff81bb65b3	
	entry_SYSCALL_64() at entry_64.S:118 0xffffffff81c0007c	
	0x0	
	
### more call stacks

	icmp_echo() at icmp.c:966 0xffffffff819e7990	
	icmp_rcv() at icmp.c:1,102 0xffffffff819e8157	
	ip_protocol_deliver_rcu() at ip_input.c:204 0xffffffff819aab11	
	ip_local_deliver_finish() at ip_input.c:231 0xffffffff819aab6f	
	NF_HOOK() at netfilter.h:301 0xffffffff819aabf4	
	ip_local_deliver() at ip_input.c:252 0xffffffff819aabf4	
	NF_HOOK() at netfilter.h:301 0xffffffff819aad17	
	ip_rcv() at ip_input.c:539 0xffffffff819aad17	
	__netif_receive_skb_one_core() at dev.c:5,286 0xffffffff81926650	
	process_backlog() at dev.c:6,242 0xffffffff8192688a	
	napi_poll() at dev.c:6,688 0xffffffff819280a8	
	net_rx_action() at dev.c:6,758 0xffffffff819280a8	
	__do_softirq() at softirq.c:298 0xffffffff81e000d8	
	asm_call_on_stack() at entry_64.S:708 0xffffffff81c00f72	
	__run_on_irqstack() at irq_stack.h:26 0xffffffff81098142	
	run_on_irqstack_cond() at irq_stack.h:77 0xffffffff81098142	
	do_softirq_own_stack() at irq_64.c:77 0xffffffff81098142	
	do_softirq() at softirq.c:343 0xffffffff810e3c56	
	do_softirq() at softirq.c:190 0xffffffff810e3ca6	
	__local_bh_enable_ip() at softirq.c:195 0xffffffff810e3ca6	
	local_bh_enable() at bottom_half.h:32 0xffffffff819add8b	
	rcu_read_unlock_bh() at rcupdate.h:723 0xffffffff819add8b	
	ip_finish_output2() at ip_output.c:230 0xffffffff819add8b	
	ip_finish_output() at ip_output.c:317 0xffffffff819b0528	
	NF_HOOK_COND() at netfilter.h:290 0xffffffff819b0528	
	ip_output() at ip_output.c:431 0xffffffff819b0528	
	ip_send_skb() at ip_output.c:1,567 0xffffffff819b0e40	
	ip_push_pending_frames() at ip_output.c:1,587 0xffffffff819b0e96	
	raw_sendmsg() at raw.c:672 0xffffffff819dd20b	
	sock_sendmsg_nosec() at socket.c:651 0xffffffff8190127f	
	sock_sendmsg() at socket.c:671 0xffffffff8190127f	
	__sys_sendto() at socket.c:1,992 0xffffffff819022e9	
	__do_sys_sendto() at socket.c:2,004 0xffffffff8190233f	
	__se_sys_sendto() at socket.c:2,000 0xffffffff8190233f	
	__x64_sys_sendto() at socket.c:2,000 0xffffffff8190233f	
	do_syscall_64() at common.c:46 0xffffffff81bb65b3	
	entry_SYSCALL_64() at entry_64.S:118 0xffffffff81c0007c	
	0x0	
	
or

	__ip_local_out() at ip_output.c:113 0xffffffff819afc22	
	ip_local_out() at ip_output.c:124 0xffffffff819afcc2	
	ip_send_skb() at ip_output.c:1,567 0xffffffff819b0e40	
	ip_push_pending_frames() at ip_output.c:1,587 0xffffffff819b0e96	
	icmp_push_reply() at icmp.c:390 0xffffffff819e6935	
	icmp_reply() at icmp.c:452 0xffffffff819e790e	
	icmp_echo() at icmp.c:976 0xffffffff819e7976	
	icmp_echo() at icmp.c:967 0xffffffff819e79b1	
	icmp_rcv() at icmp.c:1,102 0xffffffff819e8157	
	ip_protocol_deliver_rcu() at ip_input.c:204 0xffffffff819aab11	
	ip_local_deliver_finish() at ip_input.c:231 0xffffffff819aab6f	
	NF_HOOK() at netfilter.h:301 0xffffffff819aabf4	
	ip_local_deliver() at ip_input.c:252 0xffffffff819aabf4	
	NF_HOOK() at netfilter.h:301 0xffffffff819aad17	
	ip_rcv() at ip_input.c:539 0xffffffff819aad17	
	__netif_receive_skb_one_core() at dev.c:5,286 0xffffffff81926650	
	process_backlog() at dev.c:6,242 0xffffffff8192688a	
	napi_poll() at dev.c:6,688 0xffffffff819280a8	
	net_rx_action() at dev.c:6,758 0xffffffff819280a8	
	__do_softirq() at softirq.c:298 0xffffffff81e000d8	
	asm_call_on_stack() at entry_64.S:708 0xffffffff81c00f72	
	__run_on_irqstack() at irq_stack.h:26 0xffffffff81098142	
	run_on_irqstack_cond() at irq_stack.h:77 0xffffffff81098142	
	do_softirq_own_stack() at irq_64.c:77 0xffffffff81098142	
	invoke_softirq() at softirq.c:393 0xffffffff810e42e3	
	__irq_exit_rcu() at softirq.c:423 0xffffffff810e42e3	
	irq_exit_rcu() at softirq.c:435 0xffffffff810e42e3	
	sysvec_apic_timer_interrupt() at apic.c:1,091 0xffffffff81bb9026	
	asm_sysvec_apic_timer_interrupt() at idtentry.h:581 0xffffffff81c00c42	
	0x0	
	
