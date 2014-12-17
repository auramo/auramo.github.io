---
layout: post
title: Vagrant performance tuning
date: 2014-12-11 15:27:31
---

The Vagrant setup I talk about in this post: OSX as the host system, Ubuntu as
guest and VirtualBox as the provider.

I use a Macbook Pro as my work machine, but the
environment requires pretty hard-core Linux-stuff related to building C++/Linux
distros etc; hence I'm using multiple Linux virtual machines.

I used to run Linux VMs on VirtualBox directly. Hunting down and hand-configuring
images was a pain. I found out that Vagrant simplified things a lot for me.
There are however some things which will bite you when you start using it for serious stuff.

I ran into two pretty bad performance issues. The first one was that my network speed
was pretty miserable, about 70 times slower than the host system.

It turns out that the problem was with the VirtualBox NAT-interface and
the default Network Adapter type. Adding this line to my Vagrant configuration
made all the difference:

{% highlight ruby %}
config.vm.provider "virtualbox" do |v|
    v.customize ["modifyvm", :id, "--nictype1", "virtio"]
end
{% endhighlight %}

Now the download speed improved to an acceptable rate, about 15x more than I had with
the default settings. If you’re using plain VirtualBox without Vagrant, switch
the Adapter Type of your NAT interface from the GUI to [Paravirtualized network adapter](https://www.virtualbox.org/manual/ch06.html):

*
Paravirtualized network adapter (virtio-net)
*

Or just use the bridged mode which seems to be faster anyway. This is not an
option in Vagrant, it requires eth0 to be a NAT interface. You can add another
bridged interface but eth0 still has to be NAT.

I also ended up [asking and answering my own question](http://superuser.com/questions/850357/how-to-fix-extremely-slow-virtualbox-network-download-speed/850389#850389) on Superuser about this.

The second performance issue was with the shared folder which is accessible
via */vagrant* on guest. If you have any large project or any builds running from the
shared folder, you’ll bump into this issue. I noticed it when the my code’s
tab completion appeared to be super slow. By default vagrant uses CIFS for the shared
folder. To speed this up we can change this to use NFS instead. The solution was adding these
lines to the Vagrantfile:

{% highlight ruby %}
config.vm.synced_folder ".", "/vagrant", type: "nfs"
config.vm.network "private_network", ip: "10.9.8.5"
{% endhighlight %}

The latter line is required by NFS, you’ll get an error without it. It creates
a host-only network. This second interface will be handy anyway, you can e.g. access your HTTP server
via that IP and skip port forwarding.
