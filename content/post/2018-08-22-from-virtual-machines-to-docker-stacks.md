---
title: "From Virtual Machines to Docker Stacks"
date: 2018-08-22
author: "Eric Daly"
feature_image: "/images/pexels-skitterphoto-730922.jpg"
tags: [
  "docker",
]
---

I've been attempting to learn Docker with the goal of using it in production at home and at work. I see several benefits to doing this:

- Containers are less tied to the host OS compared to packages available in the repository. For example, in Ubuntu 16.04 you may not be able to get an old enough or new enough version of Redis to satisfy a requirement for your web application, leading to changing the host OS or compiling yourself (time consuming and difficult to update).
- Containers don't (shouldn't) hold state. They are disposable which is good for updating and if something goes wrong.
- Containers should be easy to work with. I can write a .yml file defining containers and parameters, and then run or "deploy" the stack using a single command. This is light years ahead of manually installing packages and configuring them manually.
- Using containers should save a lot of time, including the time it takes to define the .yml file and get everything set up.
- Once a .yml file is defined, it's easy to reuse this in another environment, and easy to alter requirements.

I'm sure there are more things, but that's just off the top of my head.

Now in my experience, what I found was that there is a plethora of Docker tutorials that get you going from scratch. It's easy to run individual containers, and sometimes they tell you about mapping volumes or mapping ports so you can access the container from the host. From there it seems like they go directly to Docker Swarm and Kubernetes, along with deploying to a cloud platform like AWS.

What about me? I want to deploy to my own ESXi environment, but I'm not interested in hosting my own Kubernetes platform because of the maintenance burden. I also want to take advantage of all the features Docker provides, and if I have to run individual containers one at a time using `docker run` then it's a no go for me. Often times, especially with web applications, you'll have several different pieces of software working together. You could build a custom Docker image with exactly the pieces you need, but this would be difficult to maintain and only makes sense if that software is something you are developing. If you just want to run other people's software, such as Nextcloud with a MariaDB backend and Redis caching, you're gonna want to run these as individual "services."

I started looking at docker-compose because it seems to be capable of doing exactly what I want: define a stack of services in docker-compose.yml, then deploy it using a single command. However, in the documentation they kind of gloss over docker-compose and suggest going directly into Docker Swarm. Hold on a minute, I don't care about Docker Swarm or clustering. Or do I?

It turns out Docker Swarm is exactly as easy as docker-compose. I guess docker-compose is more geared towards local development and testing, but we should all be using Docker Swarm in production. Basically we write the same docker-compose.yml file, but run a different command. Instead of `docker-compose up -d` we just run `docker stack deploy`.

Ok, but why do I care about that? Docker Swarm makes it possible to scale up your service if you ever need to, so all of a sudden if one front end web server isn't enough, just tell it to run 5 replicas and it will automatically spin them up and load balance for you. You can also have this happen automatically. The other thing that caught my attention was that with Docker Swarm, you have a master and then you can have workers. I can install an Ubuntu VM just for running Docker stacks on one physical host, then install another Ubuntu VM on another host, and let's say one more for good measure. The second and third hosts can join the Docker swarm, and all of a sudden not only do I have load balancing, but I have physical redundancy failover, kind of like live vMotion if you come from a VMWare world. For free.
