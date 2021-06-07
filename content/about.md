+++
title = "About Me"
aliases = ["about","contact"]
author = "Eric Daly"
#feature_image = "/images/hugo.png"
+++

I'm always looking for new technology to experiment with and use. Over time I've added more and more systems to my server that I use every day (see below). I would rather do something myself even if it's easier or might work a little better to use something else. I use open source software whenever possible, and try to give back by sharing my experience and knowledge with other people on community forums like Nextcloud and reddit for "fake internet points" as one commenter said when I questioned why people downvote without commenting, but I digress.

What's Happening Right Now
===

Software Development
---
Currently I'm focused on learning Android development. I've spent the most time learning and playing with C++ and Java, so Android is a natural progression of those skills. These are some projects that I've created:
- [Reminder List](https://github.com/linucksrox/ReminderList) - A to do list app that allows you to schedule recurring to do items (for example "Change furnace filter" has a 3 month schedule so 3 months from when you check it off, it comes back unchecked and notifies you to do it again). The idea is to work around the limitations of recurring calendar appointments which don't account for the delay in how long it takes you to actually complete the thing. So in the filter example, you would always get notified 3 months from when you actually do the thing and check it off the list, whereas if you just use a recurring calendar notification and you wait an extra month to change the filter, then it's going to remind you in 2 months which would be too soon.
- [Textecutor](https://github.com/linucksrox/Textecutor) - Send a text message to someone you need to call, turning their ringer to full max volume (they just need the app installed and your phone number needs to be on their authorized list of people). This is a work in progress, but is usable in its current state.
- [Lights Out](https://github.com/linucksrox/AndroidLogicPuzzle) - This is just a clone of the classic Tiger Electronics handheld game where you have to turn off all the lights. I'm particularly interested in creating an algorithm which can calculate the fewest number of possible moves to solve.

System Administration
---
Right now I manage two physical servers at home: FreeNAS for storing files over my network, and ESXi running about 10 VMs for various services. On the back burner I have Proxmox in mind and I'm looking at Docker and Ansible to create more of an Infrastructure As Code to make updates/deployment easier later on (like if I need to rebuild my entire setup from a backup).
- FreeNAS (16GB RAM, 6 2TB disks in RAIDZ2, ~7TB usable storage capacity) on an older desktop with no ECC RAM ([but I've done my homework and I understand the risks!](https://jrs-s.net/2015/02/03/will-zfs-and-non-ecc-ram-kill-your-data/))
- pfSense firewall
- bind DNS for local DNS resolution - wildcard resolution is nice for subdomains!
- Nextcloud using FreeNAS as the backend storage, supporting 6 users. Combine this with the Nextcloud app, and you have a nice picture backup system for free!
- Ampache music streaming server (using dsub on Android for a Pandora like experience with my personal music collection)
- restic backup software which grabs a majority of my data shared from FreeNAS and sends it to Backblaze B2 (and encrypts/deduplicates it!). I have a series of scripts which are scheduled via cron to backup, check the backup, and prune old "expired" snapshots. I'm doing just over 1TB to the cloud currently for less than $6/month, at the expense of some manual monitoring/intervention and initial work into the scripts. Not bad!
- Nginx reverse proxy - this sits in front of Nextcloud and Ampache
- OpenVPN for external access into my network and security on public Wi-Fi networks
- Wazo PBX combined with Voip.ms service, allowing me to use my phone as a SIP extension and make outgoing calls from my house. Without this it's difficult to make calls without going outside.
- Prosody as a chat server using the XMPP protocol (paired with Conversations for Android). This is far more reliable than SMS and especially MMS.
- Gitlab for keeping track of small projects that I don't want to put on github
- Docker in swarm mode for quickly spinning up new services.
  - Traefik is awesome and does dynamic reverse proxying by just specifying labels on any Docker services you deploy. This connects to Let's Encrypt, uses DNS Challenge, and automatically renews a wildcard cert for this domain with no manual intervention!
  - Bitwarden_rs for password management, self hosted (alternative to LastPass, Dashlane, etc.)
  - Jellyfin media server which is kind of like a personal Netflix, but can also be used with Kodi.

Music
---
Aside from programming and other technology interests, I play the piano (lately I'm mostly using my Yamaha KX8 controller with Waves Grand Rhapsody Red Piano preset) and really enjoy recording music. [Check out some piano music I've written on SoundCloud.](https://soundcloud.com/linucksrox)
