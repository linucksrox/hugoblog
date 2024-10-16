---
title: Setting Up Onlyoffice With Nextcloud And Docker
date: 2021-06-11
feature_image: /images/nextcloud-onlyoffice.jpg
draft: true
thumbnail: images/uploads/nextcloud-onlyoffice.jpg
tags:
  - docker
  - nextcloud
summary: A guide to setting up Onlyoffice in Docker with Nextcloud and securing
  it with a secret key
---
# Getting Started

This walkthrough assumes you are already familiar with docker-compose, reverse proxies and have an instance of Nextcloud running.

## Running OnlyOffice

You can use the official version of OnlyOffice which has certain limitations such as a limit of the number of concurrent users, and no mobile editing support. An alternative and the one that I use is [https://hub.docker.com/r/alehoho/oo-ce-docker-license](https://hub.docker.com/r/alehoho/oo-ce-docker-license "https://hub.docker.com/r/alehoho/oo-ce-docker-license")

This version includes mobile editing support and removes the concurrent user limitation. The only downside is that the repository is a little older and so you won't get timely updates (or you may never get updates, we'll see what happens with that repo over time).

1. Head over to [https://github.com/linucksrox/onlyoffice-docker-nextcloud-unlocked](https://github.com/linucksrox/onlyoffice-docker-nextcloud-unlocked "https://github.com/linucksrox/onlyoffice-docker-nextcloud-unlocked") and clone that repo to your docker machine.
2. If you're using docker-compose, edit the docker-compose.yml file. If you're using Docker Swarm then edit stack.yml instead. My example includes a reference Traefik v2 configuration which you can comment out or remove if you are using a different reverse proxy.
   1. The only change you need to make is setting a different secret for the JWT_SECRET environment variable.
3. Deploy the OnlyOffice container
   1. docker-compose method: `docker-compose up -d`
   2. Docker Swarm method: `docker stack deploy -c stack.yml`
4. Tell Nextcloud about the JWT secret by adding this to Nextcloud's settings.php file
```php
// add this array to Nextcloud's config.php to match up the JWT auhtorization header with what is defined in stack.yml
'onlyoffice' => 
  array (
    'jwt_header' => 'NCAuth',
  ),

// Add this if you get an error about host not connected because it violates local access rules
'allow_local_remote_servers' => true,
```
5. 