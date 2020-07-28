---
layout: post
title:  "Bare bones Ubuntu install for qemu"
date:   2020-07-27 14:08:44 -0700
categories: linux
---
# Install the required packages

{% highlight console %}
$ sudo apt-get install qemu-system-x86 debootstrap
{% endhighlight %}

# Create a qemu disk image and add it as an nbd device
{% highlight console %}
$ qemu-img create -f qcow2 -o size=10G debian_minimal.img
$ sudo modprobe nbd
$ sudo qemu-nbd -c /dev/nbd0 debian_minimal.img
{% endhighlight %}

# Partition the virtual disk
{% highlight console %}
$ sudo fdisk /dev/nbd0
{% endhighlight %}

{% highlight plaintext %}
Welcome to fdisk (util-linux 2.34).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Command (m for help): g
Created a new GPT disklabel (GUID: A70C1727-8216-0A4D-AA28-00ECBF4EF13D).

Command (m for help): n
Partition number (1-128, default 1): 1
First sector (34-4194270, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-4194270, default 4194270): +1M

Created a new partition 1 of type 'Linux filesystem' and of size 1 MiB.

Command (m for help): t
Selected partition 1
Partition type (type L to list all types): 4
Changed type of partition 'Linux filesystem' to 'BIOS boot'.

Command (m for help): n
Partition number (2-128, default 2): 2
First sector (4096-4194270, default 4096): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (4096-4194270, default 4194270):

Created a new partition 2 of type 'Linux filesystem' and of size 2 GiB.

Command (m for help): p
Disk /dev/nbd0: 10 GiB, 2147483648 bytes, 4194304 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 24DC1A1E-9F65-42EB-8BE7-04CFFACB7E76

Device      Start     End Sectors Size Type
/dev/nbd0p1  2048    4095    2048   1M BIOS boot
/dev/nbd0p2  4096 4194270 4190175   10G Linux filesystem

Command (m for help): 

{% endhighlight %}

# Format the disk and mount it
{% highlight console %}
$ sudo mkfs.ext4 /dev/nbd0p2
$ sudo mkdir /mnt/qemu
$ sudo mount /dev/nbd0p2 /mnt/qemu
{% endhighlight %}

# Bootstrap a minimal root filesystem
{% highlight console %}
$ sudo debootstrap --include=less,locales-all,vim,sudo \
    --arch amd64 focal /mnt/qemu http://archive.ubuntu.com/ubuntu/
{% endhighlight %}

# Chroot into new root fs and install some more packages
{% highlight console %}
$ sudo cp /etc/apt/sources.list /mnt/qemu/etc/apt
$ sudo mount --bind /dev /mnt/qemu
$ sudo mount --bind /run /mnt/qemu
$ sudo chroot /mnt/qemu
{% raw %}#{% endraw %} mount none -t proc /proc
{% raw %}#{% endraw %} mount none -t sysfs /sys
{% raw %}#{% endraw %} mount none -t devpts /dev/pts
{% raw %}#{% endraw %} export HOME=/root
{% raw %}#{% endraw %} export LC_ALL=C
{% raw %}#{% endraw %} apt update
{% raw %}#{% endraw %} apt-get install systemd-sysv
{% raw %}#{% endraw %} apt-get install -y \
	ubuntu-standard \
	discover \
	os-prober \
	network-manager \
	resolvconf \
	net-tools \
	locales \
	linux-generic \
	vim \
	ifupdown
{% endhighlight %}

# Make root fs read/write

Add a line to /etc/fstab as follows
{% highlight plaintext %}
/dev/sda2 / ext4
{% endhighlight %}

# Install grub and exit chroot

{% highlight console %}
{% raw %}#{% endraw %} grub-install /dev/nbd0
{% raw %}#{% endraw %} update-grub
{% raw %}#{% endraw %} passwd root
{% raw %}#{% endraw %} umount /proc/ /sys/ /dev/pts/ /dev/ /run/ 
{% raw %}#{% endraw %} exit
{% endhighlight %}

# Fire up qemu
{% highlight console %}
$ qemu-system-x86_64 -hda debian_tiny.img -m 2048
{% endhighlight %}

Grub will likely be broken, press c to get a command prompt then enter
{% highlight plaintext %}
set root = (hd0,gpt2)
linux /boot/vmlinux-... root=/dev/sda2 # very important, specify your rootfs partition here
initrd boot/initrd-...
boot
{% endhighlight %}

Once logged in run update-grub to fix

# Fix networking
For some reason my /etc/resolv.conf wound up with a broken symlink. To fix:

{% highlight console %}
$ echo "nameserver 127.0.0.53" > /etc/resolv.conf
{% endhighlight %}

Also replace the contents of /etc/network/interfaces with:
{% highlight plaintext %}
auto lo
iface lo inet loopback
auto ens3
iface ens3 inet dhcp
{% endhighlight %}

# Links that I used for reference
<http://tic-le-polard.blogspot.com/2015/04/qemu-create-complete-system-image.html>  
<https://itnext.io/how-to-create-a-custom-ubuntu-live-from-scratch-dd3b3f213f81>
