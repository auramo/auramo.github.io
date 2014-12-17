---
layout: post
title: Vagrant performance tuning
date: 2014-12-11 15:27:31
---

This post is about this setup: OSX as the host system and VirtualBox as the
provider.

I used to run VirtualBox directly. Hunting down and hand-configuring VMs was a
pain. Vagrant simplifies things a lot. There is however some things which will
bite you when you really start using it.

I'm using multiple Vagrant machines at my current job. I use OSX, but the
environment has pretty hard-core Linux-stuff related to building C++/Linux
distros etc.

I had two pretty bad performance issues. The first one was that my network speed
was pretty miserable, about 70 times slower than the host system.

It turns out that the problem was with the VirtualBox NAT-interface and
the default Network Adapter type. Adding this line to my Vagrant configuration
made all the difference:

{% highlight ruby %}
    v.customize ["modifyvm", :id, "--nictype1", "virtio"]
{% endhighlight %}

Now the download speed is at acceptable rate, about 15x more than I had with
the default settings. If you’re using plain VirtualBox without Vagrant, switch
the Adapter Type of your NAT interface from the GUI to:

{% highlight ruby %}
"Paravirtualized network adapter (virtio-net)"
{% endhighlight %}

Or just use the bridged mode which seems to be faster anyway. This is not an
option in Vagrant, it requires eth0 to be a NAT interface.

More details about paravirtualized network adapter: https://www.virtualbox.org/manual/ch06.html

http://superuser.com/questions/850357/how-to-fix-extremely-slow-virtualbox-network-download-speed/850389#850389

The second performance issue hit me with the shared folder which is accessible
via /vagrant. If you have any large project or any builds running from the
shared folder, you’ll bump into this issue. I noticed it when the my code’s
tab completion was super slow. By default vagrant uses CIFS for the shared
folder. To speed this up we can change this to use NFS instead. Add these
lines to your Vagrantfile:

{% highlight ruby %}
config.vm.synced_folder ".", "/vagrant", type: "nfs"
config.vm.network "private_network", ip: "10.9.8.5"
{% endhighlight %}

The latter line is required by NFS, you’ll get an error without it. It creates
a host-only network which is handy anyway, you can e.g. access your HTTP server
via that IP and skip port forwarding.
