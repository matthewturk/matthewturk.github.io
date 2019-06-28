+++
title = "Whole Tale: Exploration, Analysis and Reproducibility"
subtitle = ""

# Add a summary to display on homepage (optional).
summary = "What is this Whole Tale thing?"

date = 2019-06-28T10:32:44-05:00
draft = false

# Authors. Comma separated list, e.g. `["Bob Smith", "David Jones"]`.
authors = []

# Is this a featured post? (true/false)
featured = false

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = []
categories = []

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["deep-learning"]` references 
#   `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects = ["whole-tale"]

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder. 
[image]
  # Caption (optional)
  caption = ""

  # Focal point (optional)
  # Options: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight
  focal_point = ""
+++

<iframe width="560" height="315" src="https://www.youtube.com/embed/5PNv7FCUIhU" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## tl;dr

The [Whole Tale](https://wholetale.org/) leverages neat stuff like Jupyter, [repo2docker](https://github.com/jupyter/repo2docker/) and RStudio, building out a research environment that gives you access to any data, anywhere -- and when you stop you can pick right up where you left off, with your data, your research, and even any customizations or preferences you've set.

You can even try out the new "give me a dataset URL" page that I made that isn't *quite* amazing but is still fun.  (It's over [here](https://matthewturk.github.io/launch-wt/) for now but hopefully will move some day.)

## Whole Tale v0.7

Last week at [NCSA](https://ncsa.illinois.edu/) we had our twice-annual "all-hands" meeting for the [Whole Tale](https://wholetale.org/) project, which we've been working on for a few years.  It's a collaboration between University of Illinois, University of Chicago, University of Texas at Austin, Notre Dame, and UC Santa Barbara.

And this week we finally, *finally*, **finally** decided: we've been way too cautious in asking people to use the system, and because of that, we haven't really talked about some of the really awesome stuff that's been developed for it.  Everything is developed fully in the open on our [whole-tale org](https://github.com/whole-tale/) and wherever we can, we push changes upstream and coordinate with other projects.

So, here we go: what is in the hot-off-the-presses Whole Tale v0.7?  (We have a [release video](https://www.youtube.com/watch?v=5PNv7FCUIhU), too!  [Also, a much longer one from our previous release](https://www.youtube.com/watch?v=P1WDm_H1234), if that interests you.)

But first, you might be asking, what is the Whole Tale project, anyway?  (Of course there is a [paper out there](https://www.sciencedirect.com/science/article/pii/S0167739X17310695), but it's a lot longer than this blog post.)

## "Whole Tale," really?

It's a pun!  See, it's supposed to both tell the "whole story" and it's supposed to work for researchers at both ends of the distribution of data sizes -- both the "long tail" and the other bit.

The Whole Tale (hereafter just 'WT') is designed to be a way to -- first and foremost -- do your research, at any stage in the research lifecycle.  That means it should work for exploring data, for refining your processing of data, and all the way up to the point where you want to package it up and publish it.  And then, when someone wants to re-explore your work in a different way, or to apply your work to a different dataset, they can do so within this same environment.

I must confess, writing it like this sounds a bit underwhelming!  But there are a couple big hangups with that -- for starters, everybody has a different environment they like using.  And, folks usually want to apply their work to some kind of *data*, which might live somewhere else and might even be lots of bits from lots of places.  The final thing that is usually really tricky is that folks usually want to be able to have some stuff that hangs around in between sessions, or in between different research projects.

WT tries pretty hard to solve these different problems to make a really nice place to work on (and ultimately, publish!) research.  And by "solve," the idea isn't to *just* build stuff from scratch, but to find things that exist, glue them together, and make the experience as near-seamless as possible.

For these examples, let's imagine that our researcher is ... me, I guess.  And let's say that I want to analyze some data from Illinois, where I live.  Usually I analyze data in Jupyter, so let's start with that.

## Data in there

The first pain point that WT aims to address is that of *data*.  Many research workflows -- from the very start all the way up to the very end -- require having access to datasets.  Sometimes those datasets are created by the individual researcher, but in many cases, they aren't.  And so starting a research project often means answering (even just implicitly!) a bunch of questions about that data.

Where is the data?

How do I get the data into my working environment?

Can I grab a bunch of data from lots of other places?

WT handles data through the notion of "registration."  When a URL is ingested in the system, it stores some information about that data, and it figures out the best way to get at that data.  In the default case, WT will access it via HTTP.  It's particularly good at registering and accessing data through [Dataverse](https://dataverse.org/), [DataONE](https://dataone.org/) and [Globus](https://globus.org).

Under the hood, WT uses a system called [Girder](https://girder.readthedocs.io/en/stable/) for storing metadata about registered items, where they are located (as a hinting system for where to launch workers), what the access controls are, and what "collections" they belong to.

Here's the part that I like about this the *most*: WT does not just slurp up the data and make a copy locally.  Instead, it references the remote data and fetches subsets of it *on-demand*.  And more to the point, it does this by creating a [virtual filesystem](https://github.com/whole-tale/girderfs) that makes everything look like they're files there on disk.

So I'll use some open data from the State -- I'm curious about the distribution of rest areas in the state, so let's use [that data set](https://data.illinois.gov/dataset/rest_areas).  I grabbed the URL for the CSV by right-clicking on "Download" and copying the URL.  So now when I "register" this in Whole Tale, I'll see it as a CSV file right there in the filesystem.

(And I haven't had to download anything to my laptop yet.)

There are a handful of special things about this, and a few drawbacks.  WT knows how to register data from DataVerse and DataONE, and it can access anything over HTTP as long as the server sends the size back in the HTTP headers.  And you can access the file just like it really lived there.

## A Welcome Mat

When I open up my laptop, it's basically exactly where I left it the last time I closed it.  I've got tabs open in my browser, in-progress proposals in text editors, and all of my preferences are right there.  When I open up [vim](https://www.vim.org) it's got all my [plugins](https://vimawesome.com) and keybindings still set up.

Whole Tale wants to feel as much like home as your laptop.  So in between sessions, in between different setups, you get to keep a [home directory](https://github.com/whole-tale/wt_home_dirs).  And when you collaborate with somebody else, the "Tale" you work on will have a shared workspace directory.

When you use an environment in WT, there'll be three directories that you will see:  home, workspace and data.  Home is *yours*, and it'll be there when you quit and come back.  If you run this time in RStudio and stick stuff in home, it'll be there when you run again in Jupyterlab.  In workspace, stuff that gets saved shows up the next time somebody runs that Tale -- it isn't quite involatile data, but it's also not really *personal* data.  So this is where you might put intermediate results, or helpful scripts.

Everything under data, of course, is data that exists ... elsewhere.  It's all read only, but you can dynamically add new things to the running Tale -- not just things that you've registered, but things that show up in the WT catalog.

One of my favorite parts of WT, though, is the way that the "home" directory is made available.  Not only can you access it through the WT user interface, but you can even mount it as a directory on your laptop or desktop -- it is exposed as a WebDAV filesystem!  OSX, Windows and Linux all have native support for this.  So if you're working on getting some little scripts or maybe some supplemental data in, you can just drag it over to the WT folder on your desktop.

So you've got data, maybe from lots of places, you've got an environment, and every time you log in it feels like "home" -- now let's imagine you've done some work and you decide, eventually, that you are *done*.  Your work is ready to be published!

## Collaborating and Remixing

This part is the most fun -- within a given Tale, we've got the ability to share files across *instances* of that Tale (through the "workspace" folder) and you can also grab somebody else's published tale and mess about with it -- changing the environment, doing different things, and so on and so on.

## Try it out!

Give it a shot!  There are still some rough edges, but it should *mostly* work -- and I've personally been using it already to collaborate with students on projects and papers.

You can find out more about the project at [the Whole Tale Website](https://wholetale.org/) and all of the source is in the [whole-tale GitHub org](https://github.com/whole-tale/).  And if you want to try it out on your own resources, all of our development and deployment scripts are in our [deploy-dev](https://github.com/whole-tale/deploy-dev) repository.

Plus, we'd love to work with you to add integrations for new data stores, hear about cool ideas you might have, and if you do something fun with it, we *definitely* want to hear about that!
