---
title: How To Hot Swap ZFS Disks In Proxmox
date: 2021-10-13
feature_image: /images/zfs-hot-swap.jpg
draft: false
thumbnail: images/uploads/zfs-hot-swap.jpg
tags:
  - zfs
  - proxmox
summary: A guide to replacing a failed disk in a ZFS pool without shutting down
  or rebooting
---

Proxmox And ZFS
---
ZFS in Proxmox works really well and since I moved away from FreeNAS to Proxmox I have never had any issues. My only complaint is that replacing a disk is a bit more of a manual process than it was in FreeNAS, but you can still hot swap disks with no rebooting or downtime. You might consider that a good thing because it requires you to understand ZFS and storage a little better which can only help you down the road.

Hot swapping disks is not really complicated, but just involves a few steps that you have to follow in a particular order. So here is a detailed breakdown of what to do.

Checking The Status Of ZFS Pools And Disks
===

Is A Disk Failing?
---
- You may be getting email alerts from the SMART monitoring daemon
- Run `smartctl -a /dev/sdc` for the device you want to inspect
- Check the zpool status (CLI, proxmox UI)
	- `zpool status` OR `zpool status [poolname]`

Determining Which Disk Corresponds To Which Device Letter
---
- Go into Proxmox, click the node, then click Disks. This lists out device names and disk info including serial numbers.
- Run `zpool status [poolname]` to get a breakout of which devices are in the pool. If you added them by device-id (strongly recommended) then you will see that info including the serial numbers.
- Determining which disk corresponds to a physical slot on the server is a little more difficult. The best approach is to physically label disks on the outside of the bays and/or keep a spreadsheet of some sort that tracks at a minimum the serial numbers and physical bay locations. A less scientific/reliable method on my HP DL380P to find the bay after things are running and I don't want to shut down:
    - Offline the disk in question (see below), then check for the disk that doesn't light up when there's other activity. It helps to know which disks are in which ZFS pool.

Replacing A Failed Disk
===

Offline The Failed Disk
---
- Make a note of the existing disk-id (or whatever name is assigned to the disk as far as the zpool is concerned)
    - In the Proxmox UI, click the node, then Disks - ZFS and double click the pool to view the device breakdown
    - Or on the CLI: `zpool status [poolname]`
- If a disk is not already offline and zpool status is degraded (for example you want to proactively replace a drive that keeps throwing SMART errors but hasn't completely failed), you can set the disk offline manually:
	- `zpool offline [poolname] [disk-id]`

Replace The Physical Disk
---
- If you are more comfortable powering off the server before swapping disks, this is the time to power off.
- Pull offline disk out of the bay
- Install new disk into the bay
- Update spreadsheet/documentation with new disk information and bay location info.
- If you powered off the server to replace the disks, now is the time to power back on.

Initiate Resilvering
---
- Confirm which drive letter the new disk is assigned to (should reuse the existing letter from the old disk):
    - In Proxmox under the Node, then Disks, reload and check the device has the new serial number
    - `smartctl -a /dev/sdc`
    - If the new disk is assigned a new device letter like /dev/sdx then this would need to be modified in /etc/smartd.conf before reloading the daemon (see below)
- Get the new disk-id
    - `ls -a /dev/disk/by-id`
- Import the new disk into pool
	- `zpool replace -f [poolname] [old-disk-id] [new-disk-id]`

Reload SMART Monitoring
---
- Reload the SMART monititoring daemon so it doesn't attribute new SMART values to the old disk model, and confirm it picked up the correct disks
    - Update /etc/smartd.conf if the new disk was assigned to a new device letter
	- `service smartmontools restart`
	- `tail -n 100 /var/log/syslog`

Monitoring Resilvering Process
---
- Now that resilvering is started (or it should be), you can monitor the progress either through the Proxmox UI or on the CLI. The initial time estimate is not necessarily accurate, but be patient and allow it to complete before doing any other hardware maintenance.
    - In Proxmox, click on the node, then Disks - ZFS, and double click on the storage pool. This displays the current resilvering progress and should show which disk is being replaced.
    - Or from the CLI: `zpool status [poolname]`

Summary
---
I've done this procedure several times now without any issues. If you're not using the SMART daemon (but why not?) you could omit those steps.