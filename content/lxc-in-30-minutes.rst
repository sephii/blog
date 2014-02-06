Setting up LXC containers in 30 minutes (Debian Wheezy)
=======================================================

:date: 2013-01-11
:category: Sysadmin

Why LXC?
--------

So I'm doing web development, and I'm using Debian Wheezy as my development
environment, which doesn't have the same version of software than stable, which
is what we usually use as target servers. I used to use chroots for this, but I
found them painful to manage, especially when running daemons on the same ports
than on the host machine.

People like to use virtualization for this, such as !VirtualBox (esp. with
Vagrant) but I didn't want that since it forces you to start a whole virtual
machine every time you want to develop. Running a virtual machine is quite a
heavy process and they constantly use resources even if they don't do anything.
The big plus of using containers is that you nearly don't have any performance
hit, and your container doesn't take any resource to run. Only processes that
run will use some resources.

The main drawback of using LXC is that you can only run systems that support the
same kernel as your host. Basically it means that you won't be able to create a
BSD, Solaris or Windows container on your Debian host. But you'll still be able
to create any Linux container, whatever distribution you want to install.

I won't go in detail about what LXC is. If you want more information, check out
this Wikipedia page: http://en.wikipedia.org/wiki/LXC.

Cgroups configuration
---------------------

Cgroups are a kernel feature that's needed for lxc to run properly. Start by
adding the following to your ``/etc/fstab`` file::

    cgroup  /sys/fs/cgroup  cgroup  defaults  0   0

Mount it::

    sudo mount /sys/fs/cgroup

Installing the packages
-----------------------

We'll be using lxc to create and manage the containers, and libvirt to manage
the network::

    sudo apt-get install lxc libvirt-bin dnsmasq-base

First container creation
------------------------

Do this for each container you want to create. Let's create our first container::

    sudo lxc-create -n myfirstcontainer -t debian

Answer to the different questions (just go with the defaults, except when it
asks you for a root password if you want to put a custom password).

Once the container is created (this takes some time because it needs to download
all the base packages and install a base system), chroot in it and create
some TTY devices. I found that the ``lxc-console`` command didn't work if I didn't
create these devices first::

    sudo chroot /var/lib/lxc/myfirstcontainer/rootfs
    mknod -m 666 /dev/tty1 c 4 1
    mknod -m 666 /dev/tty2 c 4 2
    mknod -m 666 /dev/tty3 c 4 3
    mknod -m 666 /dev/tty4 c 4 4
    mknod -m 666 /dev/tty5 c 4 5
    mknod -m 666 /dev/tty6 c 4 6

Exit the chroot (by pressing Ctrl-D or by typing exit) and try to start it::

    sudo lxc-start -n myfirstcontainer

You'll get some warnings and permission errors. You can safely ignore them. Now
open a new shell and try to open a console to the container::

    sudo lxc-console -n myfirstcontainer

You should be brought to a login prompt. Log in as root using the password you
provided during the container creation phase. At this point you already have a
fully functional container. The next step is to setup an ip address for your
container so you can ssh into it, make http requests, etc. To shut down the
container cleanly, use the ``halt`` command like on your host machine (if it's
stuck and the ``halt`` command doesn't work, use the ``stop`` command)::

    sudo lxc-halt -n myfirstcontainer

Network configuration
---------------------

We'll be using libvirt to manage the network bridge that we'll create for our
containers. Start by enabling the default network configuration that comes with
libvirt (you can skip the ``-c`` option if you don't have any other
virtualization solution installed, such as !VirtualBox)::

    sudo virsh -c lxc:/// net-define /etc/libvirt/qemu/networks/default.xml
    sudo virsh -c lxc:/// net-start default
    sudo virsh -c lxc:/// net-autostart default

The default network libvirt uses is ``192.168.122.0/24``. If you want to change
it, run ``virsh -c lxc:/// net-edit default`` and adapt the settings
accordingly.

The last line will allow the interface to be automatically started on boot, so
you don't have to do it manually. Now if you run ``ifconfig`` you'll see you
have a new interface named ``virbr0``.

Now that our bridge is defined, we need to set the network interface of our
container. To do this, add the following lines to
``/var/lib/lxc/myfirstcontainer/config`` (feel free to replace the ip address
to match your bridge configuration)::

    lxc.network.type = veth
    lxc.network.flags = up
    lxc.network.link = virbr0
    lxc.network.ipv4 = 192.168.122.2/24
    lxc.network.ipv4.gateway = 192.168.122.1

Start your machine with the ``lxc-start`` command, ``lxc-console`` into it, run
``ifconfig`` and check that you have an ``eth0`` interface defined.

SSH
---

I don't know why but the base ssh install seems to have a problem with the keys
generation, making it unusable. To fix it, run ``lxc-console`` to go in your
container and reinstall ssh::

    apt-get install --reinstall openssh-server

Now you should be able to ssh to your container from your host:

    ssh root@192.168.122.2

Mount points
------------

You'll probably want to mount some of your host directories in your container.
Here's an example to mount the directory ``/home/you/directory_to_mount`` to
``/srv/mountpoint``. Add the following lines to
``/var/lib/lxc/myfirstcontainer/config``::

    lxc.mount.entry = /home/you/directory_to_mount /var/lib/lxc/myfirstcontainer/rootfs/srv/mountpoint none defaults,bind 0 0

You must also create the mountpoint manually::

    sudo mkdir /var/lib/lxc/myfirstcontainer/rootfs/srv/mountpoint

Automatically start your containers
-----------------------------------

To automatically start your containers at boot, all you have to do is to put
symlinks to your containers config files in the ``/etc/lxc/auto/`` directory.
For example for your previously created container (it's important that your
symlink has the same name as your container)::

    sudo ln -s /var/lib/lxc/myfirstcontainer/config /etc/lxc/auto/myfirstcontainer

What's next?
------------

* Automate the installation of your containers using salt/ansible/whatever you
  like.

Sources
-------

* http://wiki.debian.org/LXC
* https://wiki.archlinux.org/index.php/Linux_Containers#Terminal_settings
