---
draft: false
title: Let's Make Lights Out for Android Part 1 - Creating the Layout
summary: A quick start to create a layout for a Lights Out clone
author: Eric Daly
date: 2018-02-02
thumbnail: images/uploads/tiger-lights-out.jpg
feature_image: /images/tiger-lights-out.jpg
tags:
  - android
---
I've always enjoyed the classic game Lights Out by Tiger Electronics. It's a fairly simple idea: there is a grid of 25 buttons which can be lit or not lit. When you press a button, it just toggles the light for the button you pressed plus any adjacent buttons. So you would start with a randomized board (or predetermined pattern), and the goal is to turn off all the lights while pressing the fewest buttons possible.

Ok, maybe I'm one of only 4 people who think this is cool, but either way it's a good starter project, so let's make it for Android!

The end goal here is to demonstrate how to create a simple game from scratch, add some features, refactor the code, and eventually release it on the Google Play store. I'll walk through every step start to finish, so you should be able to follow along even if you're new to Android or Java.

For reference, [here is the current state of the project on Github.](https://github.com/linucksrox/AndroidLogicPuzzle)

## 1. Create a new project

* Click Start a new Android Studio project. 


  ![](/images/letsmakeandroidlogicpuzzle1.png)
* Set up your project.

  ![](/images/letsmakeandroidlogicpuzzle2.png)

  1. Name your app. I'll go with Lights Out.
  2. Your personal/business domain (can be anything, but should be unique so that the app's name doesn't conflict with an app with the same name in places like the Play store).
  3. Where are you storing your project? I recommend creating a folder for Android projects.
  4. Click Next.
* Unless you have a specific use case, just use the default settings for Target Android Devices. Click Next.

  ![](/images/letsmakeandroidlogicpuzzle3.png)
* We just need an empty activity for now, so click Next.

  ![](/images/letsmakeandroidlogicpuzzle4.png)
* I'll use the defaults for now. As apps get more complicated we start thinking of better names to organize everything. For now, keep the defaults and click Finish.

  ![](/images/letsmakeandroidlogicpuzzle5.png)
* Wait while android builds the project.

  ![](/images/letsmakeandroidlogicpuzzle6.png)
* Expand app/java/com.domain.appname and app/res/layout to find the MainActivity.java and activity_main.xml files to get started.

  ![](/images/letsmakeandroidlogicpuzzle7.png)

## 2. Create a layout

Our layout is going to be very basic at this point, because we just want to show a grid of buttons that we can start interacting with. We'll focus on tweaking the layout later on after implementing some of the functionality. I like to focus on getting things functional before worrying too much about how it looks.

As you might know already, there are a bazillion ways to do layouts, and I'm just focused on showing something that's very simple for two reasons:  

1. It should be simple to understand.  
2. It shouldn't take too much effort to feel like you're making progress.

* Open the provided layout file activity_main.xml. They start you out with a single TextView inside a layout, but we'll completely replace that with our own layout.
  There are a few things to note here:

1. The outer RelativeLayout uses a vertical orientation, which means that each element can be placed relative to the outer edges and relative to each other. In this case, every inner LinearLayout represents one row of buttons from top to bottom.
2. The outer RelativeLayout has layout_gravity set to center_horizontal, meaning the entire grid of buttons will be centered on the screen horizontally.
3. The inner LinearLayouts (each row of buttons) have their orientation set to horizontal, meaning each element inside of them (each button) is placed left to right.
4. We set the layout_width and layout_height of every button to the same value - buttonWidth - so that every button is a square. We might change this later on so that buttons dynamically shrink or expand depending on screen size, but this method is a little easier for now.
5. The button background is set to whatever color we assign to colorLightOff, which will be light gray.
6. The button layout_margin is set to the value of buttonSpacing, which just separates each button by a little bit so they're not all right next to each other.

We'll define the dimensions and colors next, so don't worry about any errors you see after pasting this into your layout.

{% gist 2a5e598ba67558385718a27a2f8ab489 layout_main.xml %}

* You probably noticed that there are some values we just referenced that don't exist, so we need to create those. We need to define the buttonWidth and buttonSpacing dimensions and the colorLightOn/colorLightOff colors. Starting with the dimensions, right click on app->res->values, and click New->Values resource file. Name it dimensions and click OK.

  ![](/images/new-dimensions-file.png)

  ![](/images/new-values-resource.png)
* Open dimensions.xml (the one you just created), and add a dimension element named buttonWidth, setting the value to 60dp. Also add buttonSpacing, setting its value to 2dp. You can tweak these to your liking a little bit later.

{% gist 2787528b39b70cbda9bfe6d5685bef17 dimensions.xml %}

* Next, we need to define some colors for the lights when they're on or off. Open app->resources->values->colors.xml (this was already provided for you when you created your project). Add the two colors named colorLightOn and colorLightOff, leaving the existing colors which are used in other parts of the app.

{% gist ca46e1ae8aebbb28d833dd8191c31fab colors.xml %}

* Back in the layout preview or layout design mode, you should now see the grid pattern.

Congratulations! You made a button grid layout, and you can tweak the colors or dimensions however you want before continuing on to the next part where we'll hook up the logic and actually toggle the lights on and off when you press the buttons.

Leave a comment if you have any questions or want me to elaborate on anything.