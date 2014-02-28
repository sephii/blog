Managing Development Environments In An Heterogeneous Team
==========================================================

:date: 2014-02-25
:category: Sysadmin
:tags: debian, lxc
:slug: development-environments-in-heterogeneous-team
:status: draft

So you've followed my previous post and you now have a nice container that
allows you to develop in a blazingly fast, isolated environment, which
configuration is just like your target server. But now you've seen the light,
you want your team to be enlightened too, so that everyone can work on the same
environment and everyone can enjoy a blazingly fast and isolated environment.

Here comes your first issue: some people use GNU/Linux, some other people use
OS X,  and some other people use Windows. So many platforms! LXC stands for
"Linux containers", which mean they won't run on anything else than on a Linux
kernel.

Now you might be tempted to say to yourself "man, that was too good to be
true", run lxc-destroy, drop all the LXC stuff and go home. Goodbye awesome
performance. "I should just have used virtual machines" I hear you say.

Don't.

As i said in an update to my previous post, Vagrant allows you to run Linux
containers. But Vagrant also allows you to run virtual boxes. Why not use
containers on systems that can run them, and virtual boxes on systems that
can't?

Of virtual machines and containers
----------------------------------

Vagrant allows you to define a different configuration depending on the
provider you re using:

.. code-block:: ruby

    Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
      config.vm.box = "wheezy64"
      config.vm.box_url = "http://vagrantbox-public.liip.ch/liip-wheezy64.box"

      config.vm.provider "lxc" do |lxc, override|
        override.vm.box_url = "http://vagrantbox-public.liip.ch/liip-wheezy64-lxc.box"
      end
    end
