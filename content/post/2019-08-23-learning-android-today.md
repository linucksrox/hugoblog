---
draft: false
title: Learning Android Today
summary: Thoughts about learning Android and how to approach it
author: Eric Daly
date: 2019-08-23
thumbnail: images/uploads/2021-06-07_15-20-18.png
feature_image: /images/2021-06-07_15-20-18.png
tags:
  - android
---

Back in 2010 when I got my first Android phone (the original HTC Evo 4g, before LTE was a thing) I was interested in learning Android development. I started doing a little bit of reading through the documentation, but was quickly overwhelmed with trying to understand all the new terminology like Activities and Intents. I built a few sample apps and played around a bit with it, but lost interest. I wish I would have kept going with it.

Fast forward to 2017, I decided to commit to learning Android development professionally (in my limited free time) and after two years of spending little bits of time here and there (maybe an hour at a time) I finally feel like I have a good grasp on Android in general. I don't feel stuck anymore when I need to figure out how to do something. Now that I'm at this point, I thought it would be interesting to go back and map out what would have been the most efficient way to get to this point. This will probably be a bunch of tips and advice that has already been given elsewhere, but hopefully this can help someone who is new to Android decide where to start and see a bigger picture of how all the pieces fit together with modern development and libraries.

Something I realized with Android architecture components is that often times you need to know one thing in order to use another thing, but the other thing can't be used by itself without knowing other concepts, and it was like running in circles sometimes trying to get a foothold on where to start. So this is the advice I would give my past self had I known what I know today about Android development.

My recommended Android learning path
--
### General Thoughts ###
One does not simply "learn Android." It takes a lot of time and persistence to become proficient and be able to make something useful. Android is a massive framework and I realize I'll never "know" Android. It's not like the good 'ol days where you could read a book on C and "know" the language. I did that after I graduated high school (C Primer Plus is a great book by the way), but even then it wasn't particularly useful. Lately my mindset is more of making continual progress by chipping away at these things. Even just writing this article I'm seeing the vast difference in knowledge I have now compared to when I started seriously learning Android a couple years ago.

Aside from having a realistic mindset, it's very important to work on your own project. You will learn exponentially more than just running through books and tutorials. It can seem counterintuitive because you don't know what you need to know to get started or proceed, but that's what highlights your knowledge gaps and makes it obvious what you don't know but need to know. Sample projects are great, but should not be the only source of learning, and should not be the primary source of learning. I would recommend trying to limit tutorials and app walkthroughs except when you're trying to learn something that you will then implement in your own project. A lot of times I felt like I was learning but ultimately I couldn't remember how or why things worked and had to redo the tutorial anyway, when I was ready to actually use it for my app.

### Kotlin ###
This is one of the most important decisions I've made, using Kotlin exclusively instead of Java. I was partial to Java since I had studied it extensively in the past and was (so I thought) very well versed in it. But using Kotlin once you start using it, it just makes things a lot easier and faster (less typing, less thinking/caring about the lower level details of how the language works and more flowing through what your app needs to do). It is just less in the way of what you want to get done.

### Activities ###
An activity is a screen in an app. It's a basic building block of Android apps, and you have to have at least 1 activity (nowadays a common practice is only having 1 Activity and switching out what you see on the screen within that Activity using Fragments).

### Fragments ###
Fragments are like Activities but can be swapped out within an Activity, or shown side by side in an Activity like in a master/detail type interface. I've decided generally to work with Fragments and forgo multiple Activities because of their flexibility and tools like Navigation that makes the overall development experience a lot easier to brain (for me).

### Constraint Layout ###
Design is not my strong suit, so generally Constraint layouts are now the best way to get flexible layouts that are easy to assemble and work well with drag and drop. It can help reduce a lot of complexity where you used to have to put more consideration into Relative layouts and/or combinations of Linear layouts for example. Basically you tend to end up with less nesting and more efficient layouts. Easier to use + more efficient = my preference.

### Lambdas ###
Not specific to Android, but relevant. I came from a Java background, but this was before Java had lambdas. It took me a minute (actually more) to understand what lambdas were and why I should use them. Now I can't imagine going back to manually instantiating inner classes or even just anonymous inner classes, when a lambda can be had in just a couple lines of code and is so much easier to read. This is one of those things where I struggled with understanding how it worked and had to stop worrying about the low level details and just figure out how useful they were and how to use them effectively. The more you use them the more sense it makes.

### Lifecycles ###
It's important to understand Activity and Fragment lifecycles (create, resume, start, pause, stop, destroy). You shouldn't necessarily have to handle everything lifecycle related all the time, but it's especially important to know in many situations. There might be something you want to save whenever the user leaves an app, so you need to know when to handle that operation. There also might be "weird" issues you come across and understanding the lifecycle will help you track down the issue.

### RecyclerView + ListAdapter ###
RecyclerView is something you will use in many apps, and it's important to get the hang of it sooner or later. It can seem overwhelming up front, but ultimately you have a list view + layout, list item + layout, data that goes into the list, and an adapter that ties the actual data to the actual view. Everything in between for the most part is a lot of copy and paste from another implementation. It used to be good, but you still had to worry about refreshing when data changed and taking the right approach which could affect performance. Adding in ListAdapter, this handles a bunch of things automatically like what changed when the data updates, so now you automatically get nice animations built in and it's more efficient when updating data because only refreshes what changed.

### Navigation ###
Now that you're working with Activities and Fragments, learn Navigation because it takes away much of the complexity of manually loading up intents and passing data around. You can have a graphical representation of your screens and how they connect to get a visual overview of the app flow.

### Data Binding ###
Tie your data to your views directly in the XML layout file and stop worrying about manually handling data changes/updates/refreshes at different varying points in time (or forgetting to code in another refresh when you make changes to your app later on).

### Live Data ###
Live data is a way to make your data "observable" meaning you can listen for changes in the data and take action, without having to write a lot of code to get it all working. You basically just wrap normal data inside LiveData and it allows you to set up an observer (like a listener) so whenever the data changes that listener gets notified. This also works nicely with data binding so your screen gets updated when the data changes regardless of if the user caused the change or something else on the back end changed it.

### Coroutines ###
This is something that will "unlock" your ability to move forward with other things such as databases and networking. Without background work off the Main thread, you will be unable to do many things, and even if you can do some things your app's performance can really suffer. From my experience, coroutines are a very straightforward way of doing background work which is both useful and necessary for different situations. Forget about AsyncTasks and loaders and everything else, coroutines are very easy to learn and you can get more complex over time as needed.

### Room ###
Make sure to learn coroutines before attempting Room, otherwise you'll end up doing very bad things like forcing Android to run queries on the main thread which you should never do unless you like living on the edge. It can be useful, but it's usually better not to play with fire. I started blogging about the old way of doing SQLite databases in Android. Basically any time you need to store data for your app (such as for a to do list), you'll want a SQLite database. There are plenty of ORMs out there (a library to make it easier to save and retrieve data from a local database), but Room is the official Android one and it is very nice to use compared to SQLiteOpenHelper, CursorAdapters, and ContentProviders. You lost me. With Room, you just make your model objects (entities), make a Dao interface which defines the database CRUD operations, and set up a Room database object (mostly copy and paste to get a singleton object) and you're good to go.

### Architecture ###
This can seem complex, but once you start using an architecture it makes a lot more sense. I would recommend starting with MVVM since that's what Google seems to be pushing, it works well with data binding, and makes handling lifecycle changes a lot easier. However, I would recommend **not** getting into architecture until you are comfortably getting apps running just using Activities and/or Fragments. But you'll want to be doing this when you start building "real" apps that you are going to have to support beyond the initial creation. This is one of those things that separates the noobs from the pros. More generally, start looking at program patterns and following best practices. This takes time and experience to get good at, but ultimately most computer science problems have been solved already and it's just a matter of recognizing the problem and applying an appropriate pattern as a solution. I would say watch out for anti-patterns, but that's a deeper topic and requires some understanding of **regular** patterns first. This is a rabbit hole to watch out for.

### ViewModel ###
This is actually used as part of the MVVM architecture, but on its own is useful to understand what it is and what benefits it provides. When used correctly, you no longer have to worry about things like screen rotation and saving/reloading data. However, I would say this is something that is more difficult to learn up front so don't worry about using ViewModels or MVVM architecture until you are comfortably running more simplistic apps.

Other important concepts that I need to start practicing
--
### Testing ###
Testing is extremely important in production apps that need to be maintained and expanded over time. While there's some debate, test driven development is an excellent workflow for developing any software because of many side effects such as catching bugs early (make a change, test, test fails, you know what you just changed so it's easy to find the problem), catching regressions, and having complete and evolving documentation that always matches your production code (tests are good examples of how to call your functions). This isn't something I do at this point, but my goal is to use test driven development for any production grade development I do in the future.

### Networking ###
Networking is something that is used constantly. Try to think of an app that is 100% functional without an internet connection. Just like RecyclerView, networking is used in many apps. It's also used for storing online copies of data (Firebase) or other things like cloud sync. This is another concept though that requires knowledge of doing background work, so learn coroutines before getting too far into networking, or else your app is going to be uninstalled.

### JSON Library ###
This kind of goes along with networking, although you can certainly work with JSON data from a local database or flat file. But JSON tends to be the format of choice for consuming APIs, and they map nicely to data objects. Actually, you don't need to work directly with JSON, there are a bunch of libraries to make JSON mapping easier such as GSON and moshi. I think Google has been pointing people toward moshi lately, so that's what I'll be using when I get more into the API side of things.

### Image Loading Library ###
A couple of big names are Picasso and Glide. Basically an image loading library handles all the technical details of decoding media, caching, and other things and makes it easy to do something like use the URL of an image from JSON and display it in an ImageView. There's a lot that goes on behind the scenes, but unless you're interested in developing/maintaining one of these libraries yourself, don't worry about the details and just use the tools provided.

### Design ###
Material design is a library that you can use to make things look good, but it's also a set of guidelines for designing apps that are easy to use, look good, and have a good flow. Design is one of my weaknesses, so I really struggle in this area (I know when something doesn't look good, but I struggle to make it better).

Conclusion
--
So I'm probably missing some things that people would find important, and including things that people would omit from their lists. The Android ecosystem is massive, and there are limitless rabbit holes to go down. In my opinion this is pretty much the core list of things to know as a general Android developer, and you can pick up what you need as you go from here. I hope this information is helpful to someone, and I hope to dig into more details by going through the process from start to finish with my app that I'm close to finishing. The app uses modern Android development practices and is a to do list that allows you to schedule when a checked item will return to the unchecked list and notify the user that it needs to be done again at that time. It seems simple enough, but there is a lot that goes into it, and this is my first "real" app that will have value to me personally and I plan to maintain in the future.
