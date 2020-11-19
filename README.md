# Sklinux (Skore Linux)

Independent GNU/Linux distribution made following the [Linux From Scratch](http://www.linuxfromscratch.org/lfs/view/stable/) 9.1 book.

![image](https://user-images.githubusercontent.com/24211835/86545683-c7bd4380-bf06-11ea-90ad-403c70afc777.png)

## Installation

### Important stuff

**WARNING:** This distribution was compiled for a specific hardware, and may not function properly in any computer. In fact, it's unlikely that Sklinux will run error-free in real hardware. It's advised to install it in a Virtual Machine.

**Tested Specs:** 8th gen Intel Core i3 CPU, 4GB of RAM. Also, it's recommended 2GB of free disk space. It may run in computers with similar specifications, although a 64-bit architechture is mandatory!

It's also mandatory that the machine has internet access or at least a copy of Sklinux tarball. If using a Virtual Machine, give it 2 cores and keep in mind Sklinux has no built-in DHCP client, so it's recommended a bridged network connection.

Sklinux can only be ran as is in Legacy Mode. If your system uses UEFI, it's still possible to run sklinux, but a custom Linux kernel must be compiled with support for UEFI.

### Host distribution

There's no live image of Sklinux, therefore a host distribution is needed to configure the pre-compiled system. Basically any linux distro live image is good enough, but here are the needed programs/tools anyway:

* fdisk/cfdisk/parted or similar
* chroot (Unix default tool)
* mkfs.ext4 formatting tool
* tar archive manager
* terminal emulator or tty access

So, with your favorite linux live image booted and your terminal opened, let's begin.

### Sklinux tarball

Start by downloading Sklinux tarball. This can be done through a web browser or with [gdown](https://pypi.org/project/gdown/) (which can be installed with pip). There are currently two versions:

[Sklinux Minimal](https://drive.google.com/uc?id=1zbmglmzHMjd0Kx51tWIjpAUvYQCgLkkx) (261MB): just the basic system, without any additional tool.

[Sklinux Web Ready](https://drive.google.com/uc?id=1RV9YAcJlDI7EgEY3oC-CGNaAU7sdKwEK) (343MB): CA, wget, cURL, git and Links Web Browser pre-installed.

From now on, it's recommended to run all the commands with root access (permanent or temporary with `sudo` or `doas`), otherwise some of them might not work properly, if at all.

### Partitioning and formatting

Run `lsblk` to see the name of the available disk, where Sklinux will be installed (e.g. `sda`).

Now, you must create a Linux type, bootable partition. Below are the instructions to do this with fdisk.

**IMPORTANT:** all the examples consider the available disk as being `sda`. Use the appropriate name for your situation.

In your terminal, run `fdisk /dev/sda`. Create a partition with `n`, format it as primary with `p` and then press `Enter` three times. Use `a` to make the partition bootable and `w` to write everything.

Having that done, `lsblk` to see the newly created partition (from now on, the examples will use `sda1`).

Format this partition with `mkfs.ext4 /dev/sda1` (switching `sda1` with your created partition).

### Mounting and untarring

The following commands create an environment variable, a folder and mount the partition:
```shell
export SKL=/mnt/sklinux
mkdir -pv $SKL
mount -vt ext4 /dev/sda1 $SKL
cd $SKL
```
Now, untar Sklinux tarball. Supposing it was downloaded inside `~/Downloads`, you'd do:
```shell
tar -xzpvf ~/Downloads/sklinux.tar.gz
```

### Chroot into Sklinux

By untarring all the files, you **literally** installed Sklinux, congratulations!

But it won't boot yet. There are some configurations that must be done, and a nice way to do it is by entering Sklinux temporarily from within the running live image.

First, there are more items to be mounted, so:

```shell
mount -v --bind /dev $SKL/dev
mount -vt devpts devpts $SKL/dev/pts -o gid=5,mode=620
mount -vt proc proc $SKL/proc
mount -vt sysfs sysfs $SKL/sys
mount -vt tmpfs tmpfs $SKL/run
if [ -h $SKL/dev/shm ]; then
  mkdir -pv $SKL/$(readlink $SKL/dev/shm)
fi
```

Now change root to Sklinux (**there aren't spaces after the backslashes**):

```shell
chroot "$SKL" /usr/bin/env -i              \
    HOME=/root TERM="$TERM" PS1='\u:\w\$ ' \
    PATH=/bin:/usr/bin:/sbin:/usr/sbin     \
    /bin/bash --login
```

If everything went right, you should see an ufetch (informations about OS, Kernel, Packages, etc). Also the shell prompt should have changed to something like `[root@OS /]# `.

From now on, if you close and open your terminal, you should `chroot` back into Sklinux.

### Network configuration

Run `ip a` and look for your main network adapter (e.g `eth0` or `enp0s3`). Notice that `lo` is irrelevant, since it's just a loopback. Consider `<NAME>` to be the name of your adapter.

If `<NAME>` isn't `enp0s3` run `rm /etc/sysconfig/ifconfig.enp0s3`. Otherwise, it's not necessary.

Let's set a static IP address for Sklinux. You must know your gateway and broadcast. Generally, your gateway will be something like `192.168.X.1` and your broadcast will be `192.168.X.255`. The `X` number is very relevant, because it's needed for the static IP address. You can get you broadcast by looking at `brd` in `ip a` output, and your gateway by running `ip r`. It's the chain of numbers right after *default via*.

Now, choose an IP address for your Sklinux install. A value like `192.168.X.30` should be safe (remember to replace `X` with your value). So let `<NAME>` be the name of your adapter, `<IP>` be your chosen IP address, `<BRD>` be your broadcast and `<GTW>` your gateway. Then run:

```shell
cat > /etc/sysconfig/ifconfig.<NAME> << "EOF"
ONBOOT=yes
IFACE=<NAME>
SERVICE=ipv4-static
IP=<IP>
GATEWAY=<GTW>
PREFIX=24
BROADCAST=<BRD>
EOF
```

### Bootloader configuration

Let's create the `fstab` file, which is needed to inform the mount points and their respective configurations. Remember the name of your partition and replace the `sda1` in the second line of the command below with the correct name.

```shell
cat > /etc/fstab << "EOF"
/dev/sda1      /            ext4     defaults            1     1
proc           /proc        proc     nosuid,noexec,nodev 0     0
sysfs          /sys         sysfs    nosuid,noexec,nodev 0     0
devpts         /dev/pts     devpts   gid=5,mode=620      0     0
tmpfs          /run         tmpfs    defaults            0     0
devtmpfs       /dev         devtmpfs mode=0755,nosuid    0     0
EOF
```
Having done that, let's configure GRUB. Install it by running `grub-install /dev/sda` (remember that sda is an example).

For this next part, a brief explanation of how GRUB's naming convention works is important.

A partition named `sdXY` in Linux would be called `(hdZ,Y)` in GRUB. A little conversion from `X` to `Z` is needed, but it's very simple: The letter `a` becomes `0`, the letter `b` becomes `1`, and so on. Thus, `sda1` would be `(hd0,1)` and `sdb2` would be `(hd1,2)`.

Now that you know your "Linux Partition Name - `<LPN>`" (e.g. `sda1`) and your "GRUB partition Name - `<GPN>`" (e.g. `(hd0,1)`), run:

```shell
cat > /boot/grub/grub.cfg << "EOF"
set default=0
set timeout=5

insmod ext2
set root=<GPN>

menuentry "GNU/Linux, Linux 5.5.3-sklinux-1.0" {
        linux   /boot/vmlinuz-5.5.3-sklinux-1.0 root=/dev/<LPN> ro
}
EOF
```

### Extra configurations

Sklinux was pre-configured taking into consideration that your locale is `pt_BR` and your clock is set to UTC. If that's not the case, you must configure the following files:

* `/etc/sysconfig/clock`
* `/etc/sysconfig/console`
* `/etc/profile`

I suggest reading [this page](https://web.archive.org/web/20200412113615/http://www.linuxfromscratch.org:80/lfs/view/stable/chapter07/usage.html) and [this page](https://web.archive.org/web/20200412113600/http://www.linuxfromscratch.org:80/lfs/view/stable/chapter07/profile.html), if those need to be changed.

### Booting into Sklinux

Before booting into the system, know that there are no pre-configured users in Sklinux. There is a root user, though. So when presented the login prompt, enter `root` as the login and `sklinux` as the password.

So, exit out of `chroot` with `logout` and reboot the system (don't forget to detach the ISO), and you are good to go.

Notice that this procedure should be safe in most host distributions, but if you are not sure, remember to `umount` everything that was mounted.

Thank you very much for trying out Sklinux GNU/Linux distribution! :D
