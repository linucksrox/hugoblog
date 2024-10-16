---
layout: blog
draft: false
title: Why I Recommend Joplin and Nextcloud
date: 2022-06-10T14:37:10.670Z
feature_image: https://raw.githubusercontent.com/laurent22/joplin/dev/Assets/WebsiteAssets/images/home-top-img.png
tags:
  - notes
summary: Joplin is a Markdown based note taking app that includes features like
  checklists, embedding images, and more.
---
In my quest to replace paper notes and paper task lists over the years, I have been trying out different web based solutions. I am mostly opposed to desktop applications you need to install, although I did use VSCode for a while and store .txt files in a folder synced with Nextcloud. But I still wind up not being able to find notes I know I wrote, and it's possible I didn't store them in the same folder for some reason.

Combine that with the fact that I have also been using Nextcloud Deck primarily for task lists (but also for some notes if they are related), maybe the notes I'm looking for are there. But it's a little difficult to search there as well. And the interface for Nextcloud Deck was limited because the editor was only available in the right hand column. So that means if you take a lot of notes, you have to do a lot of scrolling. Apparently this just changed in a recent update, but it's about a week too late since I just switched to Joplin...

I started investigating fully web based solutions, keeping in mind some key deal breakers:

* I need to be able to self host this for privacy/security.
* This needs to be free. I may be willing to pay for a good solution some day, but today I need to know what all the open source, self hosted options are.
* It needs to be better than Nextcloud Deck and VSCode .txt files.

What I found was that although there does seem to be a wide variety of options, they are not all the equal. Some of the more promising looking web based options wound up being wrong for me for a variety of reasons:

* git based notes are cool but I don't think that is right for my use case
* Some options don't seem to have a good way to browse/search/link older notes which makes it bad for historical reference
* Some projects did not have any recent commits, so I'm hesitant to start using a project that may no longer be maintained.

One project that stands out is called Standard Notes. This looked the most promising, but it's not as simple to deploy as almost anything else out there. Because they are very micro services oriented, it's a real chore to deploy this one and for me, just for notes, I wasn't willing to invest that much time researching all the options I needed to understand in order to deploy this in production. I may come back to this project, but for now I felt the barrier to entry was too high and I moved on after a short amount of experimentation.

The last one I tried was Joplin. I read a lot of praise about Joplin and wanted to give it a shot. I have avoided it only because it requires you to install an app locally and configure sync. However, the app is quick to install and sync is very easy to set up, especially if you already run Nextcloud.

Joplin ticks all the boxes for me: it takes notes, I can do checklists, it syncs with a private server, I can easily find things with search, and the desktop editor is really nice while keeping everything in one place.

Here's a tip I didn't realize at first due to my lack of WebDAV experience. When syncing Joplin with Nextcloud, the server URL is something like `https://cloud.example.com/remote.php/webdav/` but I would **strongly** recommend creating a folder in your account first that Joplin will sync to. For me, that's just a folder called "joplin" which makes that WebDAV URL `https://cloud.example.com/remote.php/webdav/joplin`

Go forth and take better notes!