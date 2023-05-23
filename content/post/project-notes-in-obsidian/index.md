---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Project Notes in Obsidian"
subtitle: ""
summary: "A quick setup for project notes in Obsidian"
authors: []
tags: []
categories: []
date: 2023-05-23T12:03:56-05:00
lastmod: 2023-05-23T12:03:56-05:00
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

As I noted in a [previous post]({{< ref "/post/strava-in-obsidian" >}}), I've been using [Obsidian](https://obsidian.md) for personal note taking and whatnot.
I swear by using the [Templater](https://github.com/SilentVoid13/Templater) plugin, and I have setups for note taking for meetings and check-ins with students.

I recently formalized something that I do in all of them, which makes it a lot easier to set up.
For instance, my `meeting-notes` template looks like this:

```
<%*
let projectName = await tp.user.helpfulPrompt(tp, "project", "#projects", "Project Name?");
let slugName = tp.user.slugger(projectName);
-%>
---
fileClass: meeting-notes
aliases: ["<% projectName %>-meeting-<% tp.date.now() %>"]
tags:
 - notes
 - meeting
 - projects/<% projectName %>
project: <% projectName %>
created: <% tp.date.now() %>
modified: <% tp.date.now() %>
date: <% tp.date.now() %>
attendees:
 - Matt Turk
 - ...
title: Notes - <% projectName %> - <% tp.date.now() %>
---

# Notes - <% projectName %> - <% tp.date.now() %>

## Agenda and Notes

1. start
2. ...

## Action Items

Use the metadata `assigned` like `(assigned:: Matt Turk)` in these.

 - [ ] 

## Other
<% await tp.user.safeRenamer(tp, "projects/" + projectName, slugName + " - notes - " + tp.date.now()) %>
```

and my `check-in` template looks like this:

```
<%*
let personName = await tp.user.helpfulPrompt(tp, "student", "#student", "Student Name?");
let slugName = tp.user.slugger(personName);
-%>
---
aliases:  []
student: <% personName %>
tags:
 - student/<% slugName %>
 - students/checkin 
modified: <% tp.date.now() %>
created: <% tp.date.now() %>
title: <% personName %> - Checkin - <% tp.date.now() %>
---

# <% personName %> - Checkin - <% tp.date.now() %>

## Discussion

## Notes

## Action Items

<% await tp.file.move("projects/students/" + slugName + "/"
        + personName + " - Checkin - " + tp.date.now()) %>
```

I've set up a couple functions in [user scripts](https://silentvoid13.github.io/Templater/user-functions/script-user-functions.html) that make things a lot easier for me.
(I just now noticed I don't use `safeRenamer` in my `check-in` template, though, and I'll have to fix that.)

Because I have metadata tags for `project:` and `student:` in the frontmatter of these templates, I can do some searching to provide an autocomplete for *existing* values, and also allow for *new* values to be added.

Here's my `safeRenamer` script, which honestly I'm not sure is exactly the right one to use, but which is working for me:

```javascript
async function safeRenamer(tp, newFolder, newFilename) {
    if (!(await app.vault.exists(newFolder))) {
        await app.vault.createFolder(newFolder);
    }
    var newFullpath = `${newFolder}/${newFilename}.md`;
    var exists = await tp.file.exists(newFullpath);
    if (exists) {
        for (let i =1; i <= 100; i++) {
            newFullpath = `${newFolder}/${newFilename} - ${i}.md`;
            exists = await tp.file.exists(newFullpath);
            if (!exists) break;
        }
    }
    // We now have a newFullpath that probably includes the .md extension.
    // But, we don't want obsidian to use that.
    if (newFullpath.endsWith(".md")) {
        newFullpath = newFullpath.slice(0, -3);
    }
    return tp.file.move(newFullpath);
}

module.exports = safeRenamer;
```

And, this set of functions helps me figure out what *already* exists to provide a set of options, as well as allowing for a new value to be added.
Here is `frontmatterKeyValues.js`:

```javascript
// https://stackoverflow.com/questions/1960473/get-all-unique-values-in-a-javascript-array-remove-duplicates
function onlyUnique(value, index, array) {
    return array.indexOf(value) === index;
}

async function frontmatterKeyValues(key, selectionTag) {
    const dv = app.plugins.plugins["dataview"].api;
    if (!selectionTag.startsWith("#")) selectionTag = `#${selectionTag}`;
    keyValues = dv.pages(selectionTag).filter( p => p[key] ).map( p => p[key] ).filter(onlyUnique);
    return keyValues.array();
}

module.exports = frontmatterKeyValues;
```

and, finally, `helpfulPrompt.js`:

```javascript
const OTHER = "Other";

async function helpfulPrompt(tp, key, tag, header) {
    let knownValues = await tp.user.frontmatterKeyValues(key, tag);
    knownValues.push(OTHER);
    let selectedValue = await tp.system.suggester(knownValues, knownValues, true);
    if (selectedValue === OTHER) {
        selectedValue = await tp.system.prompt("Use which value?", "", true);
    }
    return selectedValue;
}

module.exports = helpfulPrompt;
```

I also wrote this up (`tagLeafFinder.js`), but have not yet used it, for identifying stem values in keys:

```javascript
async function tagLeafFinder(stemTag) {
    if (!stemTag.startsWith("#")) stemTag = `#${stemTag}`;
    if (!stemTag.endsWith("/")) stemTag = `${stemTag}/`;
    const tags = Object.keys( app.metadataCache.getTags() ).filter( t => t.startsWith(stemTag) );
    return tags;
}

module.exports = tagLeafFinder
```

Now, whenever I start a project meeting, I hit `Alt-n` and choose the project from the (on-the-fly-generated) list, and I get my meeting notes.
And, dataview query in my overview page tells me all the todo items assigned to me across all meeting notes.
