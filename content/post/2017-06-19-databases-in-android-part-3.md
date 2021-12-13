---
draft: false
title: Databases In Android - Part 3
summary: Part 3 of a mostly code look at using SQLite in Android (not using Room)
author: Eric Daly
date: 2017-06-19
thumbnail: images/uploads/pexels-sergei-starostin-6466141.jpg
feature_image: /images/pexels-sergei-starostin-6466141.jpg
tags:
  - android
---

Before I get into ContentProviders, I just want to mention how difficult it is to blog consistently. I mean, it's not too bad. You just have to commit to a schedule and stick with it. You just have to be motivated to write instead of work on your app that is probably way more interesting to work on. It's not a complaint though, more of a rant. There's just not much time available to work on side projects when you have a full time job and family to spend time with, assuming you don't work for Apple.

<br>
![](/assets/images/apple-rarely-get-to-see-my-kids.jpg)
<br>
<br>
That's not me. It's the other way around:
> I have to limit how much I work, or I risk not seeing my kids.

<br>

ContentProviders
----------------

[ContentProviders](https://developer.android.com/guide/topics/providers/content-providers.html) manage access to data, and provide a layer of abstraction between raw data and the application (widget, other apps, your own app, etc.). At the expense of a little more complexity, it adds the value of centralized data security (instead of making outside applications sanitize data, it can all be done in the provider). You can also change the data backend without making any changes to the provider API, and so no changes are necessary in any applications accessing the provider. For example you could change from using a flat file data store to using a SQLite database.

<br>
<br>
Well I ran out of time today, so I'll add to this next time.
