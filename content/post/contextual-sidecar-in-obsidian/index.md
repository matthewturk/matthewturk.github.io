---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Contextual Sidecar in Obsidian"
subtitle: ""
summary: "A plugin to keep helpful stuff available in a sidecar panel."
authors: []
tags: []
categories: []
date: 2024-03-03T12:00:00-05:00
lastmod: 2023-03-03T12:00:00-05:00
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

As I've mentioned on here before, I use [Obsidian](https://obsidian.md/) a lot.  One thing that I've really come to like as of late is the [meta-bind](https://github.com/mProjectsCode/obsidian-meta-bind-plugin) plugin.  This let's you add items to your notes that edit the frontmatter metadata, as well as buttons to execute actions, and special-purpose "views" of the data in your notes.

One of the most common types of notes I use is a [`meeting-notes` template](https://github.com/matthewturk/obsidian-helpful-bits/blob/dfe324f56587aace40a1adf4b8be83af4e7fb8df/meeting-notes.md).  (This is a link to the current version, but I try to keep that repo sorta up to date.)  I have places for attendees as well as TODOs.  What I was finding was that I often wanted to update the attendees from within the page, but my notes got really long and I had to scroll around etc.  Another thing that I've been doing is adding "person" entries, so that the meeting attendees link to pages of notes I keep on those people.

What I realized I needed was a way to keep the `meta-bind` inputs and actions visible at all times, even when I'm down in the page.  So I decided to write a plugin that would find a contextually-appropriate sidecar file and keep it open in the panel to the right.  After a couple weeks of iterating on the specifics, it's now in the Obsidian Community Plugin list, and it's available in source code at [matthewturk/obsidian-sidecar-panel](https://github.com/matthewturk/obsidian-sidecar-panel/).

This lets me add a panel that is queried from a `sidecar-panel` entry in the frontmatter, a set of tag mappings, or a default sidecar file.  For my meetings, I now have a button to spin up a [modal-form](https://github.com/danielo515/obsidian-modal-form) to make a new person entry, a button for a new todo, a list of the attendees (which I can add to right there!) and a view of all the previous TODOs for the people currently attending the meeting.  You can see that panel [here](https://github.com/matthewturk/obsidian-helpful-bits/blob/dfe324f56587aace40a1adf4b8be83af4e7fb8df/meeting-notes-editor.md).

Give it a shot, if you like!
