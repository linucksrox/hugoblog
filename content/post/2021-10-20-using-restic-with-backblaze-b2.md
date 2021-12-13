---
title: Using Restic with Backblaze B2 for Off-Site Backup
date: 2021-10-20
feature_image: /images/restic_logo.png
draft: false
thumbnail: /static/images/uploads/restic_logo.png
tags:
  - restic
  - backup
summary: How to use restic with Backblaze B2 and some shell scripts to help with
  scheduling/automation/monitoring
---

[Checkout the Github repo here](https://github.com/linucksrox/restic-scripts)

# My Use Case
I have been using restic for just over 3 years with Backblaze B2 and have saved money at the expense of my time. Before that I was running Crashplan home edition until they increased the price to $10/month (at the time I was only storing about 600GB off-site). Backblaze B2 was offering pay as you go pricing for $0.005/GB so I would be saving about $5/month. Fast forward to today, I'm still spending less than $10/month (but slowly reaching that threshold), while retaining encrypted, deduplicated snapshots going back to over 3 years ago. Today my total off-site backup size is around 1.6TB.

# Why Restic?
These are the main benefits of pairing restic with B2 in my opinion:

- restic encrypts data before sending over the wire, so I don't have to worry about which data storage provider I use
- restic deduplicates data which means it stores less data overall, decreasing my monthly backup cost but this is at the expense of using more computing power and taking longer to complete each snapshot
- Backblaze B2 has been 100% reliable in my experience, and their pricing is about as cheap as you can find. When I decided on Backblaze, Wasabi was also offering a similar solution for $5/month per TB, but their prices have increased slightly since then. They also do not charge egress or transaction fees based on transaction types like Backblaze does, so you can more easily wind up with additional costs using Backblaze B2 depending on how you run your backups. But in my experience, these costs have been very minimal (well under $1 per month even when I use class B transactions heavily).
- restic snapshots allow me to keep many versions of my data over time without really worrying about the overall storage usage.
- restic features an easy way to prune old snapshots based on a retention policy that can be easily scripted (which you can see in the restic-scripts repository, `restic_forgetandprune_bypolicy.sh`)

# Getting Started
Assuming you have already thought about your backup retention including how much data you are backing up, how long you want to keep snapshots, etc. we can start with getting a machine set up to actually run the backups, checks, logging/monitoring and restores.

I'm not going into specifics about setting up Backblaze B2 so that is an excercise for you (or any other destination you want to use).

1. Make sure your machine is ready to run restic jobs. Since restic does deduplication, it can be heavy on resources like CPU/RAM and also cache storage. In case it gives you any idea, I have been working with a ~1.6TB backup repository, adding somewhere around 200MB-500MB twice daily, and it's using almost 8GB of cache data, consumes almost all 8GB of RAM, and a fair amount of the 8 CPU cores I allocated to that VM.
1. Mount any network shares you will be using as a source. You will most likely want to update `/etc/fstab` so that these will mount automatically on reboot.
1. Install restic. You might be able to use your distro's package manager, but I would recommend just downloading the latest stable binary from Github: [https://github.com/restic/restic/releases](https://github.com/restic/restic/releases)
1. Clone the restic-scripts repo: [https://github.com/linucksrox/restic-scripts](https://github.com/linucksrox/restic-scripts)
1. Edit `restic_env.sh` to point to your Backblaze B2 bucket and configure your backups.
1. Assuming you're creating a brand new restic repository, you'll need to start by initializing the repo before you can run backups. Run `restic_initialize_repo.sh` or follow the restic documentation for that: [https://restic.readthedocs.io/en/latest/030_preparing_a_new_repo.html](https://restic.readthedocs.io/en/latest/030_preparing_a_new_repo.html)
1. Follow the instructions in the README there, and open an issue if you have any questions or something does not work.

# Use Cases
## Initializing A New Restic Repository
Run `restic_initialize_repo.sh` which will create a brand new repository if data does not already exist there. If you're familiar with git, this is kind of similar to git init. No data is actually sent, but a config is created with your repository details, ready to track snapshots.

## Backing Up Data
Run `restic_backup.sh` for this which will rely on `restic_unlock.sh`, `restic_env.sh` and `restic_excludelist.txt`. This just takes a backup snapshot using all the environment variables you already have configured. This can be scheduled with cron (see the README for a cron example and how to pipe the output to a log file).

## Listing Snapshots
Run `restic_snapshots.sh`

## Checking Your Backup Repo
Run `restic_check.sh`. I recommend scheduling this with cron (see the README for an example of this).

## Prune Old Snaphsots
You can define a backup policy by number of snapshots, number of weeks/months/years and other options like always keeping the latest snapshot. `restic_forgetandprune_bypolicy.sh` contains the policy I use and will automatically remove old snapshots and associated data blobs, keeping the 4 most recent, 30 most recent days, 8 most recent weeks, 12 most recent months, and 100 years (basically don't ever delete old backups but only retain yearly intervals beyond the most recent 12 months). You can easily adjust this by modifying the options in the file.

There's also `restic_forgetandprune_2weeks.sh` which removes everything except the past 2 weeks worth of snapshots if that's more your speed.

## Forget A Specific Snapshot
You may need this in a situation where a backup fails to complete or you are getting errors after running `restic check`. If you don't already know the snapshot id you need to remove, run `restic_snapshots.sh` to find the snapshot id of the backup you want to remove, then run `restic_forget_byid.sh` and pass in the snapshot id to have it removed. This script doesn't automatically prune (so it removes the snapshot from the index but doesn't actually remove data blobs). Run `restic_prune.sh` to prune, obviously.

## Manage Backup Repository Keys
Restic uses keys to encrypt/decrypt repository data. You will need to add at least 1 key, but you can use these scripts to add and remove keys if you need to rotate them out for example.
- `restic_key_list.sh` - list current keys
- `restic_key_add_password.sh` - add a password to a key
- `restic_key_change_password.sh` - change the password for an existing key
- `restic_key_remove_password.sh` - remove password from existing key

## Mount All Snapshots
Using `restic_mount.sh` you can mount snapshots in order to browse/search for files across all snapshots, restore data, or just test your backups. This enables you to use a file manager or CLI, whichever fits your needs. By default, it expects there to be a `mount` directory in the same location as this script (`./mount`), but you can obviously change this to fit your needs in the script.

## Get Backup Stats
Run `restic_stats.sh` to get some stats about the total size of your repo and the total deduplicated data size for comparison.
