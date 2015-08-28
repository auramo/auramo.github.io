---
layout: post
title: Docker image as a development environment
date: 2015-03-09 21:20:00
---

I've been playing with Docker lately. According to documentation It's
most commonly used as a container for a single server-side process. My
use-case is a bit different: trying to get a development environment
running. Usually I'd use Vagrant for shared development environment
configuration and implementation, but I ran into a case where it
wasn't an option.

My requirements were basically these:

* The end-result should be an interactive shell for compiling and running the software
* I need be able to install various .deb packages into the Docker image
* I need to copy files to the image when it's being built
* I need to run various commands during installation
* All these steps should be automated (no manually created massive image-files)

## How to automate image creation

You can create an easily shareable text-file called
[Dockerfile](https://docs.docker.com/reference/builder/) for
your image. Let's say I want to share an Ubuntu 14.04 image with:

* a HELLO.TXT inside root's home directory which is echoed to root user
when he logs in.
* Emacs installed

This is what it would look like:

```
FROM ubuntu:14.04
RUN sudo apt-get install -y emacs
ADD HELLO.TXT /root/HELLO.TXT
RUN echo "cat /root/HELLO.TXT" >> /root/.bashrc
```

Store the above snippet to a directory as Dockerfile. Also store
HELLO.TXT with some text to the same directory. Then run command:

```
sudo docker build .
```

Now it downloads Ubuntu as the base image, and applies our
instructions. As a result we get an image with an ID:

```
Successfully built 1e49d046eb83
```

We can use that ID to run commands. Let's start an interactive shell.

## Starting an interactive shell


To start an interactive shell in our new image, we tell docker to run
/bin/bash:

```
sudo docker run -e 'HOME=/root' -i -t 1e49d046eb83
```

Now you should be greeted with "hello" and should be able to start
the installed Emacs.

The parameters to run-command:

* -i means that we are running an interactive command
* -t is the ID or name/tag of the image
* -e allows us to set environment parameters. For some weird reason
   $HOME doesn't work properly in Docker, so we'll have to set it
   explicitly here to get our .bashrc evaluated.

Now you have a working system, and with those basic ADD and
RUN-commands you can install or alter almost anything. Just to make our
dent in the universe, let's store a file to our image while we are in it:

```
echo "foo" > /bar.txt
```

## Clean rebuild

Docker uses it's cache to see what commands needs to be run when you
run build. This is faster and usually works ok. Sometimes you'll just
want to do a clean build though. This can be done with this parameter:

```
--no-cache=true
```

## Getting back

Let's get outta here! Press ctrl-D and you're back in your host
operating system. Nice, let's go back to our docker image by running
that same run command (above) again. Looks similar, but where's
/bar.txt?

```ls: cannot access /bar.txt: No such file or directory```

It turns out that every time you run a command, you get a new
_container_ based on the image ID we give the run command. So if we
run just one command like this:

```
docker run learn/tutorial echo "hello world"
```

It will create a new container for the command echo, run the command
and return. This container still exists, but running the same command
with the same image again will create another container based on the image
"learn/tutorial".

Since we're building a development environment, surely we'd like to
have some state there and not just start from scratch every time. You
can list all containers you have created by running commands with:

```sudo docker ps -a```

Or if you want to see just the last container you created by your last
command:

```sudo docker ps -l```

It will show when the container was created, with what command etc.

So the question remains, how do we get back to the container which had
our /bar.txt?

We'll have to create a new image based on the /bin/bash command we
ran. So with let's run that _ps -a_ command, check the id of the
container created by our /bin/bash -command and commit it as a new
image:

```
sudo docker ps -a
<check the id of /bin/bash container, happens to be 2ab151606f4c>
sudo docker commit 2ab151606f4c image-with-bar-txt
sudo docker run -i -t image-with-bar-txt /bin/bash
```

Now we can see our precious bar.txt again! This doesn't seem to be
what I want to do every time I go back to my development machine
though.

## Volumes

What if I could just always run the an image based on a Dockerfile
which is in version control? What if I wanted to see and edit all my files
on the host system? I can't really do these things if I start editing
my source files inside the Docker container's [Union File
System](https://docs.docker.com/terms/layer/#union-file-system).

This is why Docker has
[volumes](https://docs.docker.com/userguide/dockervolumes/#data-volumes). They
are just shared directories
between the host and Docker container. They are lighweight since there
is no NFS or CIFS, they introduce a a direct disk access between host
and the container.

Let's share our host's directory /home/clarence/work as /work
inside the Docker container. We'll just start bash again, but with a
-v flag:

```
sudo docker run -v /home/clarence/work:/work -i -t image-with-bar-txt /bin/bash
```

Now when you're in you can try:

```
touch /work/yeah
```

Then get out of the container, and check the directory (in the example
/home/clarence/work):

```
ls /home/clarence/work
-> yeah
```

Just adding multiple -v (or --volume) parameters to the run command we
can add as many volumes as we like. Note that the host path must be
absolute. So if you want to use relative paths, you'll have to expand
the path in beforehand e.g. in a shell script (EXAMPLE HERE)

## Where does it all go?

All the files in the Union File System, the stuff you install and work
on which is not under a volume directory go to /usr/lib/docker on your
host. This can grow quite large so if your disks gets full, check the
directory.

To free some space you can nuke all the containers with this:

```sudo docker ps -aq | xargs sudo docker rm```

And all the images with this:

```sudo docker images -q | xargs sudo docker rmi```

Of course you should only do that if all the state you need is in
either in the Dockerfile or volumes. Otherwise do some more
fine-grained deletions via checking the listings of the _ps_ and _images_ subcommands.






