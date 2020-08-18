# What exactly is tty and how it works
tty has been puzzling me for quite a long time. despite all the excellent explanations online, still it felt mysterous.

### 1. tty data structures
Some *tty* related data structures in the Linux kernel:
* `tty_driver` - manages a set of *tty*s of a specific type
* `tty_struct` - stores (control?) information about a specific *tty*
* `tty_port` - stores/buffers the data from input device(keyboard)
* `tty_ldisc` - tty line displine, e.g. line oriented input handling
* `cdev` - represents char device, has a pointer to `file_operations` which implements the logic, among other things, to open the char device
...

### 2. Virtual console(vt)
Generally, a device is identified by a version number(`dev_t`) which includes a major part and a minor part. devices are special files in Linux, you could create a device file by specifying the type and version number, e.g.

	mknod mytty c 4 1

this will create a special device file "mytty" with major version 4 and minor 1, which is the first virtual console. major version 4 and minor version 1-63 are for virtual consoles(vt.c: `vty_init()`).
when system call sys\_open() is called with a path pointing to a tty special file like "mytty" created above, the file is looked up and first populated with a special default `file_operations`: `def_chr_fops` (inode.c: `init_special_inode()`) whose `open()` function will look up the real char dev from `cdev_map` and replace the default `file_operations.open()` with the real one. `cdev_map` contains all the char devices registered with the system. see `tty_init` for example.

To sum up a bit, `cdev`s are registered with the system(e.g. on system initialization), char device files are created with special major & minor version, char device files are opened by calling `cdev.ops.open()`, e.g. `tty_open()`, which look up the `tty_driver` using the device number, and allocate a `tty_struct` and install it with the `tty_driver`, which then calls `tty_operations.install()` to do the low level installation, e.g. vt.c: `con_install()` which associates the `tty_port` residing in `vc_data`.

flow of data from keyboard to tty and finally display in the case of vt:
>key stroke => keyboard interrupt handler => vt keyboard input handler (`kbd_event()`) => vt `kbd_keycode()` => char inserted into `vc_data.port` => `vc_data.port->buf->work` is scheduled to push the character into `tty_ldisc` => display (echo) the char when necessary and wake up the process waiting on the input when appropriate

### 3. pty
Pseudo tty, kind of like pipe, it has a master/slave pair when opened, output to one end becomes the input of the other, see code `pty_init`...

### Conclusion
It has become a bit clearer after looking into the source code, still more details to be studied though...
