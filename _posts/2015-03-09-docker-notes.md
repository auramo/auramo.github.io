---
layout: post
title: Docker image as a development environment
date: 2015-03-09 21:20:00
---

I've been setting up this Docker Image which should becomeme a shared development
environment. At least for tests, I hope. I haven't been using Docker before, and
I have been expecting it to be like a lightweight virtual machine system. Sometimes
that expectation doesn't really match reality.

Docker has pretty good documentation. What I'm trying to get across here is how you
should approach a situation similar to mine: create a shareable, lightweight environment
to be used by multiple developers.

## Starting an interactive shell


* How to
* Beware, you'll have to freeze it as image to use the same situation again. 

## Containers vs Images

Talk about containers getting created on every command from outside.

When you make some changes, either interactively via shell, or by running commands,
you'll get a new container when the command or shell session ends. As a result,
you'll get a container. You'll have to commit that container into an image to use it later. 

To do this, first, show the last container:

    sudo docker ps -l

From the output, you'll get a container ID (here 99a2ac75bbc3), then run commit:

    sudo docker commit 99a2ac75bbc3 new-image-with-my-changes

## Where does it all go?

* Talk about /usr/lib/docker and how it fills up fast
* Refer to using volumes later

Trick on how to clean up containers and images with list -q | grep | xargs

## Using volumes

* Should be performant
* Note, you can use multiple -v params
* Host path must be absolute. Expand shell trick.


