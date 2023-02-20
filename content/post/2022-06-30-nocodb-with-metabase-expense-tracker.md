---
layout: blog
draft: true
title: Expense Tracker with Firefly III and Metabase
date: 2023-02-20T13:40:56.599Z
feature_image: /images/screenshot-2023-02-20-at-8.55.58-am.png
tags:
  - metabase
  - fireflyiii
  - expense
summary: Quickly set up an easy to use, web based expense tracker with nice
  analytics using Firefly III and Metabase, with a simple docker-compose stack.
---
# What is Firefly III?
[Firefly III](https://www.firefly-iii.org/) is "A free and open source personal finance manager" that is web based and can be self hosted easily in Docker/podman/whatever container. I switched to this after many years of using GnuCash because I wanted to move to a more modern platform, and specifically wanted the ability to access a database instead of using the flat file database format that GnuCash uses. This would allow me to integrate easily with Metabase. My use case is to hand enter every transaction that appears on my bank account or credit card account, and specify what category they belong to with the goal of using that data to track budgets (which I might talk about in the future) and get an easy high level view of how much is being spent per week/month/whatever by category. I should also be able to quickly dive into a problematic spending category and get a breakdown of what transactions that is made up of. This is where Metabase will be a great help. Don't even bother looking at the Firefly III built in dashboards. I'm sure they're nice, but there's just no comparison to the power and flexibility you will get using Metabase to analyze the data.

# What is Metabase?
[﻿Metabase](https://www.metabase.com/) is a free open source BI (Business Intelligence) dashboarding tool, which also happens to run easily in a Docker/podman/whatever container. This means I can create charts, graphs, dashboards, (dynamic) pivot tables, or whatever else I want based on a set of data. You can easily add filters, and one feature I really like about Metabase is how easy date filters are to work with. The feature image of this post shows an example where the blue is the number of transactions in that category in the pats 12 months, and the green is the average transaction cost. This is kind of a weird chart, but also kind of interesting.