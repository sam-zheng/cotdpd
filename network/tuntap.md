# How tun/tap works
https://www.kernel.org/doc/html/latest/networking/tuntap.html

### Bootstrap
when linux initializes itself, `tun_init`(`drivers/net/tun.c`) is called, which, among other things, registers `tun_miscdev`, a `miscdevice` by calling `misc_register` with the the system (in a global list `misc_list`). `miscdevice` is a kind of char device, see also `misc_init`, `misc_open`.

### How to use

	mkdir /dev/net (if it doesn't exist already)
	mknod /dev/net/tun c 10 200
	
tun/tap device is designated with major version 10(for char device), and minor version 200.

#### From the perspective of VPN software utilizing tun/tap

when `sys_open` is called on `/dev/net/tun`, `misc_open` gets called, which then calls `tun_chr_open` which allocates a `tun_file` containing a `socket` to be used (e.g. by the VPN software) for reading from/writing to (with (`tun_socket_ops`)) the application program which was bound to the associated `net_device`.
after `dev/net/tun` was opened, `ioctl` needs to be called with `TUNSETIFF` to attach it to the `net_device` denoted by the given `ifreq.ifr_name`, or a newly allocated `net_device` if `ifreq_ifr_name` is not provided(`tun_set_iff`), if a new `net_device` is allocated, `tun_net_init`, among other things, is called to associate a specific `net_device_ops` to the `net_device` depending on if it's a `IFF_TUN` device(`tun_netdev_ops`) or a `IFF_TAP` device(`tap_netdev_ops`). 

Writting (typically by VPN software) to this socket:
	
	tun_sendmsg() -> tun_get_user() which puts the sk_buffer into the rx queue(backlog?) which is then pushed up the networking stack to the user space application, see below.
	
Reading from this socket:
	
	tun_recvmsg() -> tun_do_read() which takes a sk_buffer from tun_file.tx_ring(by tun_net_xmit(), see below) and call tun_put_user() to put it into user space program(VPN).
	
#### From the perspective of applications using the VPN software

User space applications creates sockets bound to the `net_device` above as it does with any other normal `net_device`s. data going out of such sockets will eventually go through `tap_netdev_ops.ndo_start_xmit()`, i.e. `tun_net_xmit`, which then puts the `sk_buffer` on `tun_file.tx_ring` which will be taken by the VPN software, see above. reading from the sockets takes the `sk_buffer` from the rx queue of `net_device` which was put there by the VPN software by `tun_sendmsg()`, see above.

### Conclusion

TUN/TAP driver provides a way for user space programs to intercept the data sent out from the `net_device` and control the way data is received by the `net_device`, thus it's convenient to implement VPN with this mechanism.
