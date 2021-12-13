---
draft: false
title: How To Get Going With Jekyll and GitHub Pages
summary: How to set up Jekyll for a blog (outdated)
author: Eric Daly
date: 2016-07-29
thumbnail: https://holianh.github.io/assets/images/website-using-jekyll-github-pages.jpg
feature_image: /images/neonbrand-3GZNPBLImWc-unsplash.jpg
tags:
  - jekyll
---
I'm running [Xubuntu 16.04][xubuntu]. This method should be the same/similar in any Debian/Ubuntu distro. These are the steps I followed to get this thing working, using the terminal:
```
sudo apt install ruby ruby-dev ri bundler build-essential git
sudo gem install jekyll
sudo gem install minima
```

Note that if the Jekyll version is newer than the supported version on GitHub (see the [GitHub dependency versions][github-pages-versions] page) then you will need to install the latest supported version of jekyll instead, like this:
```
sudo gem install jekyll -v 3.1.6
```

Next, you need to create a special repo in GitHub for your site. In GitHub, create a new repo, and name it:
```
whatever-your-username-is.github.io
```

Finally, grab a local copy of the new repo:
```
git clone https://github.com/whatever-your-username-is/whatever-your-username-is.github.io
```

Create a brand new Jekyll site with the same name:
```
jekyll new whatever-your-username-is.github.io
```

Move to the directory:
```
cd whatever-your-username-is.github.io
```

Test your site locally by running this command, then go to the site it shows you in the terminal (default http://127.0.0.1:4000):
```
jekyll serve
```

Once you know it's working, or after you make whatever changes you want, throw it back up on GitHub!
```
git push
```

After you successfully push your newly created Jekyll site to GitHub, wait maybe 2-3 seconds (not really, it's just amazingly fast) and then go to:
```
http://whatever-your-username-is.github.io
```

Start blogging!

[xubuntu]: https://xubuntu.org/
[github-pages-versions]: https://pages.github.com/versions/
