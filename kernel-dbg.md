# Linux Kernel Debugging

Build a linux kernel for debugging run and connect to it via GDB.
Mostly copy and pasted from the [Refs](#refs) as a self reference.

Using archlinux as the host system.


## Setup

Check for missing packages and install them.
Since most of them may be installed anyway the step can be sipped.
If sometime later an error occurs because of a missing package, it can be installed just then.
```
$ sudo pacman -S base-devel bc coreutils cpio gettext initramfs kmod libelf ncurses pahole perl python rsync tar wget xz
```

**QEMU**  
minimal  
```
$ pacman -S qemu qemu-system qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils
$ pacman -S qemu-base
```

full  
```
$ pacman -S install qemu-full
```


## Get source

Get source from and sig from 
[kernel.org](kernel.org) 
or any other source you like.


### Decompress 1
```
$ xz --decompress linux-*.tar.xz
```

### Check sig
$ gpg --search-keys ABAF11C65A2970B130ABE3C479BE3E4300411886
$ gpg --search-keys 647F28654894E3BD457199BE38DBBDC86092693E
$ gpg --verify linux-*.tar.sign

### Decompress 2
```
$ tar -xf linux-*.tar
```

## Config file

Copy existing config file from host system
```
$ cd linux-*/
$ zcat /proc/config.gz > .config
```

### Debian and Fedora's derivatives:
$ cp /boot/config-"$(uname -r)" .config


## Init config file

Backup old default config
```
$ make olddefconfig
```

Make a new default config file
and open gui config menu and make changes for debugging purposes.
```
$ make defconfig 
$ make menuconfig
```

Enable (by pressing enter ("x") or y ("*")).  
The "kernel hacking" stuff is relevant.  
The VirtIo stuff is optional.  

- device drivers
    - Block devices
        - <x> VirtIo Block device/driver
    - Network device support
        - <x> VirtIo Network driver
- kernel hacking 
    - [*] Kernel debugging 
      - Compile-time checks and compiler options
        - Debug information
            - [x] Rely on the toolchain's implicit default DWARF version
        - [*] Provide GDB scripts for kernel debugging
      - Generic Kernel Debugging Intstruments
          - [*] KGDB
            
exit & save

This leads to  
```
DEBUG_KERNEL=y
CONFIG_GDB_SCRIPTS=y

CONFIG_HAVE_ARCH_KGDB=y
CONFIG_KGDB=y
CONFIG_KGDB_HONOUR_BLOCKLIST=y
CONFIG_KGDB_SERIAL_CONSOLE=y

...
[to be filled]
```
in the .config file.

You can permanently disable  
`CONFIG_RANDOMIZE_BASE`
aswell.  
Or pass it dynamically as a kernel runtime parameter (like done in this tutorial) when the kernel is started.
If the kernel is built just for debugging purposes, 
  it maybe more convinient to permanently disable it in the config.
 

### (Optional) tag your build
```
$ ./scripts/config --file .config --set-str LOCALVERSION "-<tagname>"
```
Will result in a build file named
vmlinuz-<version>-<tagname>

i.e. 
```
CONFIG_LOCALVERSION="-<tagname>"
```
in the .config

### Do it manually 
This all can be done as well by using the `scripts/config` if you know the parameters
```
$ ./scripts/config -e DEBUG_KERNEL -e GDB_SCRIPTS -e KGDB ...
```

## Build the kernel
If the .config is setup, the kernel can be built.
This make take some couple minutes.
```
$ make -j$(nproc) 2>&1 | tee log
```

Will build an x86 image, which doesn't mean it's 32-bit.
If x86_64_defconf has been set, the `x86_64/boot/bzImage` will only be a link to `x86/boot/bzImage`  

Usefull file locations
```
./arch/x86_64/boot/bzImage    # kernel binary image
./vmlinux                     # target for GDB
./vmlinux-gdb.py              # pre-defined GDB helpers
```

If ./vmlinux-gdb.py is not present (GDB_SCRIPTS not set in .config)
```
$ make scripts_gdb
```
will create it.

Optionally add the script file to GDB’s auto load path so that you can later use the lx-* helper commands.
Otherwise it has to be manually added as a parameter each time when calling gdb.
```
$ echo "add-auto-load-safe-path path/to/linux-v.x.y/vmlinux-gdb.py" >> ~/.gdbinit
```

Check for debug symbols
```
$ file vmlinux
"... with debug_info, not stripped" 
```
should appear at the end of the output.
If not, something went wrong with the .config file and it should be checked and built again.


## Create root file system

Info:  
There is the bzImage, the kernel as an executable binary. 
In order for the kernel to boot it needs an initial root filesystem initramfs to get stuff going and setup correctly initial binaries for the real system to work, 
  like the init process aka the parent of all processes, 
  and a bunch of other important binaries in /sbin.
When the initramfs is done executing crucial binaries as root, 
  it then looks for the REAL root filesystem to mount and pivot the root to it, 
  free itself from memory, and the full system boots. 
If it did not find the REAL root filesystem, 
  it will boot the kernel and throw you in a recovery shell to boot it manually, 
  or figure out what went wrong.


### Using buildroot

Move into parent dir and clone
```
$ git clone git://git.buildroot.net/buildroot
```

The layout should be
```
$ tree -d -L1
.
├── buildroot
└── linux-6.11

```
$ cd buildroot
$ make menuconfig [qemu_x86_64_defconfig]
```

The required options you need to set are:

- Target options 
    - Target architecture
        - [*] x86_64
- Toolchain 
    - [*] C++ support
- Target packages 
    - Network applications 
        - [*] openssh [*]; for sending files into the QEMU guest through SSH 
- Filesystem images 
     - ext2/3/4 root filesystem; 
        - [*] ext4

```
$ make -j$(nproc)
```
If some required modules (on your host system) are not installed, install them and try again.

This root file system can be used for different kernels.

### Using an initial ram disk
Alternativ (not tested yet):  
Just make an init ram disk

```
$ mkinitramfs -o ramdisk.img
$ file ramdisk.img
```


## Running QEMU

### buildroot  

Docs: https://www.qemu.org/docs/master/system/invocation.html

```
$ sudo qemu-system-x86_64 \
  -kernel linux-6.11/arch/x86_64/boot/bzImage \
  -nographic \
  -drive format=raw,file=buildroot/output/images/rootfs.ext4,if=virtio \
  -append "root=/dev/vda console=ttyS0 nokaslr" \
  -m 4G \
  -enable-kvm \
  -cpu host \
  -smp $(nproc) \
  -net nic,model=virtio \
  -net user,hostfwd=tcp::10022-:22 \
  -s -S

# one line
$ sudo qemu-system-x86_64 -kernel linux-6.11/arch/x86_64/boot/bzImage -nographic -drive format=raw,file=buildroot/output/images/rootfs.ext4,if=virtio -append "root=/dev/vda console=ttyS0 nokaslr" -m 4G -enable-kvm -cpu host -smp $(nproc) -net nic,model=virtio -net user,hostfwd=tcp::10022-:22 -s -S
```

Options:
- -kernel the kernel bzImage 
- -nographic No QEMU gui
- -drive Define a new drive. This includes creating a block driver node (the backend) as well as a guest device, and is mostly a shortcut for defining the corresponding -blockdev and -device options.
    - file Disk image
    - format Disk format
    - if Interface type the drive is connected. Available types are: ide, scsi, sd, mtd, floppy, pflash, virtio, none
- -append kernel command line
- -m Startup RAM size
- -enable-kvm KVM full virtualization support
- -cpu
- -smp CPU number
- -net nic Legacy option to configure or create an on-board (or machine default) Network Interface
- -net user Configure a host network backend
    - hostfwd This will forward port 22 from the virtual machine to port 10022 on your host, making the VM accessible over SSH.
- -s Shorthand for -gdb tcp::1234, i.e. open a gdbserver on TCP port 1234
- -S Do not start CPU at startup (you must type ‘c’ in the monitor).
    
Using the `-s -S` combo holds QEMU from booting the kernel until a GDB instance is attached.
So if nothing seems to happen after pressing ENTER, this is just expected.

The relevant pathes are set in `-kernel` parameter, where the bzImage is set and in `-drive format=raw,file=...` the path to the root file system is set.


### init ram disk
```
$ qemu-system-x86_64 -kernel <kernel-dir>/arch/x86_64/boot/bzImage \
  -initrd <path-to>/ramdisk.img \
  - m 512 -s -S \ 
  -append "console=ttyS0 nokaslr"
```

## Attaching GDB in a separate shell window
```
$ sudo gdb linux-6.11/vmlinux [-iex "set auto-load safe-path $(pwd)/linux-6.11/."] [-tui]

(gdb) target remote :1234   # Attach to QEMU
(gdb) lx-symbols # load kernel symbols
(gdb) [hbreak start_kernel] # break point on start_kernel
(gdb) [b mm_alloc] # break point on mm_alloc
(gdb) c
(gdb) lx-dmesg  # Display kernel dmesg log in GDB shell
(gdb) ...
```

GDB is an interactive debugger, so when we say continue, the QEMU window will continue its execution just like an OS running normally, 
  until it hits the next break point or you can break with CTRL+C. 

To exit out of the QEMU nographics mode, in the QEMU window, type ctrl + a, release, then type x.

In case of  
`warning: File "<path>/linux-6.11/scripts/gdb/vmlinux-gdb.py" auto-loading has been declined by your `auto-load safe-path' set to "$debugdir:$datadir/auto-load".`
check https://sourceware.org/gdb/current/onlinedocs/gdb.html/Auto_002dloading-safe-path.html

To add a trusted path 
(gdb) set auto-load safe-path [directories]

Or to enable it one time add the `-iex ...` option to the gdb invoking command line.

## Login to your kernel

buildroot login:  
name: `root`  
no password  


## Sending Files to the QEMU Guest

Check if eth0 network interface is available:
```
(in-guest) ifconfig -a
```

Add eth0 to the list of interfaces and enable this interface by:
```
(in-guest) echo "iface eth0 inet dhcp" >> /etc/network/interfaces
```

Bring it to life.
You might need this step every time you boot the guest.
```
(in-guest) ifup eth0
```

Allow OpenSSH to accept logins as root and empty password:

```
(in-guest) vi /etc/ssh/sshd_config
    # Uncomment "PermitRootLogin" and set "yes";
    # Uncomment "PermitEmptyPasswords" and set "yes"

(in-guest) /etc/init.d/S50sshd restart
```

ssh into the guest from the host through:
```
ssh -p 10022 root@localhost
    # Lowercase `p` for ssh
```

send files:
```
scp -P 10022 file root@localhost:/root/some/path
    # uppercase `P` for scp

# or sftp
sftp -P 10022 root@localhost
```


## Compiling custom modules for that kernel

Install modules (optional, can be skipped)
```
$ make modules_install -j$(nproc) INSTALL_MOD_PATH=../
```
Default location is `/lib/modules/<kernel_release>-<localversion>` (sudo required).
Since it's only a development version, you should change the directory, like done in the previous command.
This creates a `lib/modules/<kernel_release>-<localversion>` structure in the parent directory of your current one.
This current one ought to be `<some_path>/linux-xxxx` .
The resulting path `<result_path>/build` is the Makefile value for the `-C` param. 

**This build directory is just a link: `build -> <some_path>/linux-6.11`, so giving `<some_path>/linux-6.11` as the value for the `-C` param works as well.
No need for the `modules_install` command if just compiling custom modules is wanted.**

Best compile modules with symbols!

https://www.linux.com/training-tutorials/kernel-newbie-corner-kernel-and-module-debugging-gdb/

## Add symbols of your loaded module to gdb
```
vm$ ls /sys/module/<moduleName>/sections -A1
vm$ cat /sys/module/<moduleName>/sections/.text
vm$ cat /sys/module/<moduleName>/sections/.data
vm$ cat /sys/module/<moduleName>/sections/.bss
vm$ cat ...

(gdb) add-symbol-file .../<moduleName>.ko <text_address> 
 -s .data <data_address>  
 -s .bss <bss_address> 
 ...
```

## Breakpoints
The Linux kernel sets the text segment as read-only. 
A debugger would be unable to add a SW breakpoint, so only HW breakpoints can be used.

But software breakpoints in intern modules worked as well.
```
b ...
hbreak ...
```

Seems to depend on the kernel .config file
```
# Write protection for kernel read-only data structures, which is used to catch potential kernel bugs.
CONFIG_DEBUG_RODATA

# Set loadable kernel module data sections as NX, and text section as read-only in order to catch potential kernel bugs.
CONFIG_DEBUG_SET_MODULE_RONX
```


## Refs
- https://itsfoss.com/compile-linux-kernel/  
- https://www.josehu.com/technical/2021/01/02/linux-kernel-build-debug.html  
- https://sergioprado.blog/debugging-the-linux-kernel-with-gdb/  
- https://www.kernel.org/doc/html/v6.12-rc2/dev-tools/kgdb.html  
- https://kernelnewbies.org/KernelBuild  
- https://ghostinthehive.gitbook.io/ghostinthehive-as-a-ghost-in-the-hive/debugging-the-linux-kernel-with-qemu-and-gdb
- https://kernelmethod.org/notes/linux_kernel_debugging/

