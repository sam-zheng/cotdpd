# How network bridge(software) works
network bridge route/forward packets between different networks

### 1. Some commands

	brctl addbr br0
	ip addr add dev br0 172.16.0.3/24
	ip link set br0 up
	
add network interface to the bridge

	// prepare virtual ethernet interfaces
	ip link add veth0 type veth peer name veth1
	ip link set veth0 up
	brctl addif br0 veth0 // add veth0 to the bridge
	
### 2. Implementation in Linux kernel

`br_add_if`->`netdev_rx_handler_register` stores `br_handle_frame` in `net_device.rx_handler` and the `net_bridge_port`
at which the given `net_device` is to be added to the bridge to `net_device.rx_handler_data`.
when the `net_device` which is attached to a port of the bridge receives data with `__netif_receive_skb_core`, it goes to `br_handle_frame` were
the bridge device takes control and does forwarding of the data or handles locally(software implemented bridge can be configured with an ip address and could
work as a normal (virtual) network device)

### 3. Some tests

	sz@sz-mac:~$ sudo ip addr add dev br0 172.16.0.1/24
	sz@sz-mac:~$ sudo ip link set br0 up
	sz@sz-mac:~$ sudo ip link add veth0 type veth peer name veth1
	sz@sz-mac:~$ sudo ip link set veth0 up
	sz@sz-mac:~$ sudo brctl addif br0 veth0
	sz@sz-mac:~$ sudo ip addr add dev veth0 192.168.100.1/24
	sz@sz-mac:~$ sudo ip addr add dev veth1 172.16.0.2/24
	sz@sz-mac:~$ sudo ip ink set veth1 up

### 4. A C program to create a container with separate namespace demonstrating communicating between different namespaces (busybox in the rootfs folder).

	#include <stdio.h>
	#include <unistd.h>
	#include <linux/sched.h>
	#include <sys/wait.h>
	#include <errno.h>
	#include <sys/mount.h>
	#include <syscall.h>
	
	static char child_stack[1024 * 1024];
	
	int child_main(void *arg) {
		/* Unmount procfs */
		umount2("/proc", MNT_DETACH);
		/* Pivot root */
		mount("./rootfs", "./rootfs", "bind", MS_BIND | MS_REC, "");
		mkdir("./rootfs/oldrootfs", 0755);
		syscall(SYS_pivot_root, "./rootfs", "./rootfs/oldrootfs");
		chdir("/");
		umount2("/oldrootfs", MNT_DETACH);
		rmdir("/oldrootfs");
		/* Re-mount procfs */
		mount("proc", "/proc", "proc", 0, NULL);
	
		sethostname("example", 7);
		system("ip link set veth1 up");
	
		char ip_addr_add[4096];
		snprintf(ip_addr_add, sizeof(ip_addr_add),
				"ip addr add 172.16.0.101/24 dev veth1");
		system(ip_addr_add);
		system("route add default gw 172.16.0.100 veth1");
	
		/* Run the process */
		char **argv = (char **) arg;
		execvp(argv[0], argv);
		return 0;
	}
	
	int main(int argc, char *argv[]) {
	
	//	brctl addbr br0
	//	ip addr add dev br0 172.16.0.100/24
	//	ip link set br0 up
		system("ip link add veth0 type veth peer name veth1");
		system("ip link set veth0 up");
		system("brctl addif br0 veth0");
		int flags =
		CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWPID | CLONE_NEWIPC | CLONE_NEWNET;
		int pid = clone(child_main, child_stack + sizeof(child_stack),
				flags | SIGCHLD, argv + 1);
		if (pid < 0) {
			fprintf(stderr, "clone failed: %d\n", errno);
			return 1;
		}
		char ip_link_set[4096];
		snprintf(ip_link_set, sizeof(ip_link_set) - 1, "ip link set veth1 netns %d",
				pid);
		system(ip_link_set);
		waitpid(pid, NULL, 0);
		return 0;
	
	}
		