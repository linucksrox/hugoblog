---
title: "About Me"
date: 2021-06-12
---

I enjoy working with technology and learning about new tools. Over time I've added more and more systems to my home lab that I use daily. I prefer open source software when possible over proprietary solutions. I also try to give back by sharing my experience and knowledge with other people on community forums like [Nextcloud](https://help.nextcloud.com) and [Reddit](https://reddit.com).

My Home Lab
---
Right now I manage 1 physical server: HP DL380P Gen 8 running Proxmox with a separate 1U rack PC running PFsense for my firewall. Here are some of the services I run:
- piHole for DNS ad blocking at the network level
- bind DNS for local DNS resolution - wildcard resolution is nice for subdomains!
- Nextcloud, supporting 6 users. Combine this with the Nextcloud Android app, and you have a nice picture backup system for free!
- restic backup software which grabs a majority of my data shared from FreeNAS and sends it to Backblaze B2 (and encrypts/deduplicates it!). I have a series of scripts which are scheduled via cron to backup, check the backup, and prune old "expired" snapshots. I'm doing just over 1TB to the cloud currently for less than $6/month, at the expense of some manual monitoring/intervention and initial work into the scripts. Not bad!
- Traefik reverse proxy
- Wireguard VPN
- VitalPBX combined with Voip.ms service, allowing me to use my phone as a SIP extension and make outgoing calls from my house. Without this it's difficult to make calls without going outside.
- Prosody as a chat server using the XMPP protocol (paired with Conversations for Android). This is far more reliable than SMS and especially MMS.
- Gitlab for keeping track of small projects that I don't want to put on github
- Docker in swarm mode for quickly spinning up new services.
  - Vaultwarden for password management, self hosted (alternative to LastPass, Dashlane, etc.)
  - Jellyfin media server which is kind of like a personal Netflix, but can also be used with Kodi.
  - Ampache music streaming server (using dsub on Android for a Pandora like experience with my personal music collection)
  - Unifi Access Point controller

Kubernetes
---
I have been working with Docker for a while and wanted to take the next step by learning Kubernetes and running my own cluster at home. This consists of multiple master and worker nodes each in their own VMs on my Proxmox server. Currently I only have 2 instances of Traefik running as internal and external ingress controllers, and have Cert Manager configured to get certificates easily using DNS challenge. Eventually I will migrate my Nextcloud instance to Kubernetes and from there I plan on migrating most of my other services over as well.

Software Development
---
Here are some Android projects I have developed personally.
- [Reminder List](https://github.com/linucksrox/ReminderList) - A to do list app that allows you to schedule recurring to do items (for example "Change furnace filter" has a 3 month schedule so 3 months from when you check it off, it comes back unchecked and notifies you to do it again). The idea is to work around the limitations of recurring calendar appointments which don't account for the delay in how long it takes you to actually complete the thing. So in the filter example, you would always get notified 3 months from when you actually do the thing and check it off the list, whereas if you just use a recurring calendar notification and you wait an extra month to change the filter, then it's going to remind you in 2 months which would be too soon.
- [Textecutor](https://github.com/linucksrox/Textecutor) - Send a text message to someone you need to call, turning their ringer to full max volume (they just need the app installed and your phone number needs to be on their authorized list of people). This is a work in progress, but is usable in its current state.
- [Lights Out](https://github.com/linucksrox/AndroidLogicPuzzle) - This is just a clone of the classic Tiger Electronics handheld game where you have to turn off all the lights. I'm particularly interested in creating an algorithm which can calculate the fewest number of possible moves to solve.

Music
---
Aside from programming and other technology interests, I play the piano (lately I'm mostly using my Yamaha KX8 controller with Waves Grand Rhapsody Red Piano preset) and really enjoy recording music. [Check out some piano music I've written on SoundCloud.](https://soundcloud.com/linucksrox)
