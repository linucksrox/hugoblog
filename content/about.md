---
title: "About Me"
date: 2026-03-03
---

I'm always seeking out new technology and trying out new systems on my homelab. Over time I've added more and more systems to my homelab that I use daily. I prefer open source software when possible over proprietary solutions, but also understand the pros/cons of when it makes sense to use paid (supported) solutions. I also try to give back by sharing my experience and knowledge with other people on community forums like [Nextcloud](https://help.nextcloud.com) and [Reddit](https://reddit.com), along with sharing detailed documention right here on my blog to help fill some gaps.

My Home Lab
---
My short term goal is to replace my old 2U HP DL380p Gen8 server with 3 Minisforum MS-01 units. I currently have 3 of those clustered with the old server, for a total of 4 hosts. This is great for learning about Proxmox clustering, high availability and practicing managing a more enterprise-y type of setup. It also makes it possible (along with Kubernetes and other systems) to perform hardware maintenance with no downtime. Overkill for a homelab, but this is what keeps me sharp!

I recently wired these up using a Unifi Aggregation switch (8 10G networking ports) and split out a bunch of stuff on different VLANs. This really helped with both Proxmox cluster performance but more importantly Longhorn storage performance for the Talos cluster. I use the v2 data engine for that BTW, and it has been for the most part rock-solid.

I have a separate OPNSense box for my firewall, allowing me to take full advantage of my 1gig fiber internet connection.

For my NAS, I'm using a RAIDZ2 on Proxmox directly and sharing NFS/CIFS from there.

Here are some of the services I run:
- piHole for DNS ad blocking at the network level
- bind DNS for local DNS resolution - wildcard resolution is nice for subdomains
- Nextcloud, supporting 6 users along with desktop and mobile apps
- Immich for photo backup and sharing - a vast improvement over Nextcloud when it comes to searching and reliability of backups
- restic backup software which grabs a majority of my data shared from the NAS and sends it to Backblaze B2 (and encrypts/deduplicates it!). I have a series of scripts which are scheduled via cron to backup, check the backup, and prune old "expired" snapshots. I'm doing close to 2TB to the cloud currently for around $16/month, at the expense of some manual monitoring/intervention and initial work into the scripts. Not bad!
- Traefik reverse proxy (both in Docker and as a Kubernetes ingress controller)
- Netbird - this is an awesome VPN to access my homelab and services when I'm out and about
- Wireguard VPN - this is the old, reliable VPN backup for when Netbird is funky
- VitalPBX combined with Voip.ms service, allowing me to use my phone as a SIP extension and make outgoing calls from my house. Without this it's difficult to make calls without going outside due to flaky cell service in my area.
- Gitlab (running on Docker) for keeping track of small projects that I don't want to put on github
- Docker in swarm mode for quickly spinning up new services.
  - Vaultwarden for password management, self hosted (alternative to LastPass, Dashlane, etc.)
  - Jellyfin media server which is kind of like a personal Netflix, but can also be used with Kodi.
  - Ampache music streaming server (using dsub on Android for a Pandora like experience with my personal music collection)
  - Unifi Access Point controller
- Prometheus/Grafana for monitoring - I want to expand this to include more logging and potentially alerts (not too many alerts though)
- UptimeKuma - excellent simplified monitoring dashboard, especially for homelabs
- Lots of other things I've tried or currently run but are less noteworthy

Kubernetes
---
I have been working with Docker for many years now, personally and professionally. I earned my CKA (Certified Kubernetes Administrator) certification in July, 2025. Kodekloud was amazing for the course content and the labs to prep and really learn Kubernetes inside and out.

I now run a Talos Linux cluster consisting of 3 control plane nodes and 6 worker nodes (3 dedicated to storage with Longhorn). I'm in the process of migrating my old Docker Swarm based services to the Kubernetes cluster for ease of management and durability.

Essential Kubernetes components I run are MetalLB, cert-manager, Treafik ingress controller, Longhorn (after trying democratic-csi and OpenEBS Mayastor), and Velero for backups.

Software Development
---
Here are some Android projects I have developed personally. They are not great, but something I did accomplish. I tend toward systems administration, but have also worked in Java, PHP and Kotlin building maintainable code that adheres to best practices. I am very familiar with reading source code and understanding pretty quickly what it's doing, especially when there is a problem (for example why SSO/SAML redirection was using HTTP vs. HTTPS).
- [Reminder List](https://github.com/linucksrox/ReminderList) - A to do list app that allows you to schedule recurring to do items (for example "Change furnace filter" has a 3 month schedule so 3 months from when you check it off, it comes back unchecked and notifies you to do it again). The idea is to work around the limitations of recurring calendar appointments which don't account for the delay in how long it takes you to actually complete the thing. So in the filter example, you would always get notified 3 months from when you actually do the thing and check it off the list, whereas if you just use a recurring calendar notification and you wait an extra month to change the filter, then it's going to remind you in 2 months which would be too soon.
- [Textecutor](https://github.com/linucksrox/Textecutor) - Send a text message to someone you need to call, turning their ringer to full max volume (they just need the app installed and your phone number needs to be on their authorized list of people). This is a work in progress, but is usable in its current state.
- [Lights Out](https://github.com/linucksrox/AndroidLogicPuzzle) - This is just a clone of the classic Tiger Electronics handheld game where you have to turn off all the lights. I'm particularly interested in creating an algorithm which can calculate the fewest number of possible moves to solve.

Music
---
Aside from programming and other technology interests, I play the piano (lately I'm mostly using my Yamaha KX8 controller with Waves Grand Rhapsody Red Piano preset or the amazing Alicia's Keys) and really enjoy recording music. [Check out some piano music I've written on SoundCloud.](https://soundcloud.com/linucksrox)
