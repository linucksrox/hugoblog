---
author: "Eric Daly"
title: "Docker Versus Virtual Machines"
date: "2020-02-21 08:10:00 -0400"
feature_image: "/images/pexels-pixabay-262353.jpg"
tags: [
  "docker",
]
---

## About Virtual Machines ##
In the context of this article, virtual machines is in reference to running a VirtualBox VM or running a VMWare VM that runs a guest operating system. For example I run Linux on a laptop and run Windows in a VirtualBox VM as a guest. Virtual Machines in this context do not refer to the Java Virtual Machine or any other definitions other than a virtual host computer which runs a guest operating system.

## How Is Docker Different? ##
Docker is not a virtual machine like you might think of a virtualbox VM. It's a tool for running isolated applications within a container based on an image. Docker images are similar to ISO images in the sense that they contain a certain application state, but can't be changed. So you can build multiple identical containers based on a single image (or destroy and rebuild a container), just like you might use a single Windows ISO image to install multiple identical VirtualBox guests.

The difference is that Docker is designed to run a single application or service, while a virtual machine runs an entire operating system and potentially multiple applications. Sure, you can run a single application in a virtual machine like you might traditionally do with a web server. Install a Linux virtual machine and then install Apache/PHP. With a VM if you need a web server, you always start with a host OS such as Ubuntu server. Then you install updates, packages, dependencies, and finally environment config. Now you're ready to deploy.

What if you could build all of that extra update/install/config stuff into an image? You can do that with templates, but in order to maintain that template you have to go into the running system, make changes live, and then commit those new changes as a new version. Doing this stuff by hand is not easily reproducible and can lead to issues over time.

Docker can be used to solve a problem like this because a Docker image (kind of like a VM template) can be built using a Dockerfile. It can be reproduced or modified without making live changes on the system itself, but all defined in the Dockerfile that is used to build that image. Another benefit of this is that you can build someone else's image with nothing other than the Dockerfile (a human readable text file) and avoid having to transfer the actual image file.

Docker runs differently than virtual machines. A virtual machine is a complete virtual computer with its own kernel, operating system, etc. Docker is just an isolated environment, sharing the same kernel as the host machine, making it much faster and smaller to run. This is also important to keep in mind, as Docker is not always better than a VM, or the other way around. It always depends on what you need to do, and kernel sharing is not always ideal.
![](/postassets/docker-vm-diagram.png)
![](/postassets/docker-container-diagram.png)

## Basic Docker Components ##
**Image -** A Docker image is a snapshot of a container, kind of similar to a live Linux ISO image or a VM template. It is unchangeable, and if you need to make changes you need to build a new image.

**Container -** A container is a running instance of an image, kind of similar to a running instance of a live Linux ISO running inside a VM.

**Dockerfile -** This is a human readable text file that is used to build images. At its most basic, you start with a FROM command to start with a base image (a different existing Docker image), then you make any modifications to the image, and finally define which command to run when the container is started (this can be a shell script, starting the Apache service, etc.). You can use the `docker build` tool to build an image based on this Dockerfile.

**docker-compose -** While you can use `docker run ...` to run containers, that is tedious at best and hard to remember/reproduce especially during testing. With docker-compose, you can define all your runtime settings (volume mapping, exposed ports, environment variables, which image you want to use, etc) in a file docker-compose.yml and then simply run `docker-compose up` to get it all running, and `docker-compose down` to break it all down.

## Workflow ##
I like to differentiate between development and production workflow with Docker. Since Docker is supposed to be a tool to make my life easier, then how is it any easier in Docker to start with a base OS, build on top of that, create an image, then finally run that image in a container? Isn't that the same as running a VM?

This is where Docker is very different than VMs. Starting out, you probably don't need to worry about Dockerfiles at all, at least not during development. Start with [Docker Hub](https://hub.docker.com/), find an image that will do what you need, and adjust volumes, ports and environment variables from there as needed. In other words, with a VM you start with a blank operating system and have to build it up. With  Docker, you start with the application you need to run and all other dependencies are already in place.

For example, [Jellyfin](https://jellyfin.org/) is a free software media system (kind of like a personal Netflix). You could, if you wanted to, create a new VM, install Ubuntu server, install Apache, etc. and eventually download and set up Jellyfin after configuring a virtual host and all that good stuff. How long did it take you just to get that VM up and running, let alone get everything installed? Or you can take their Docker image and just run it. All the work has been done, so no point reinventing the wheel.

### Persistence ###
Unlike a VM, containers should be thought of as disposable. You should be ready to throw away a container and spin up a new one at any point in time. When a container is stopped and removed, everything within it is gone. So if there is any data you want to keep, that needs to be stored outside the container. That is done using volumes.

There are many ways to use volumes, but the simplest thing you can do starting out is just map a local directory as a volume so that changes to that directory are seen live in the container. In the VM world, this would be kind of like sharing a CIFS folder in the host operating system and mapping that in the guest operating system, except it's all set up automatically right when you run the container.

### Development Versus Production ###
Your development workflow probably looks different than your production workflow. For example, for a PHP web application, in production you are probably writing a Dockerfile that starts from  a PHP base image, copies your code into the container (maybe into /var/www/html) and you build a custom image from that Dockerfile. This is your dockerized application that can be distributed an deployed.

In contrast, during development you are making changes constantly and want to test those changes. It would seem ridiculous to have to rebuild the image every time you make a change, and then restart the container based on the updated image. So what you would probably do here is run the base PHP container and map a volume from your local src directory to the container's /var/www/html directory. Then any changes you make to your src directory are instantly updated in the container and you don't have to rebuild anything.

So why not just use this method for everything, instead of building custom Docker images? You should differentiate between state and data. Data should be mapped with a volume, and state should exist within the container. The reason for mapping volumes for data is obvious, you want to keep that data when the container gets destroyed. The reason for only keeping state inside the container is because that's part of the power of Docker. State exists within the container, and in order to change state (update application code, etc.) you only need to restart that container. The only changes to state are changes to application code, which should generally mean an updated image, and environment settings which are specified when you run the container, so all of that is reproducible.

## Summary ##
Hopefully this helps explain some of the ideas behind Docker and how it's different than virtual machines. To oversimplify, VMs virtualize an entire operating system, while Docker virtualizes an application.
