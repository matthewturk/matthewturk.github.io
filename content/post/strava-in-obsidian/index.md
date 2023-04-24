---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Strava in Obsidian"
subtitle: ""
summary: "How I import my Strava runs into Obsidian"
authors: []
tags: []
categories: []
date: 2023-04-24T16:13:04-05:00
lastmod: 2023-04-24T16:13:04-05:00
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

I've been using [Obsidian](https://obsidian.md) for almost two years now, and I've been finding it more and more integrated into my daily life.
I take notes (using a variety of templates) in it, track my daily goals and some personal statistics, and I love the plugin ecosystem.
I've also really enjoyed reading [Eleanor Konik](https://www.eleanorkonik.com/)'s newsletter about it.
Obsidian hasn't really made me "10x better" or whatever, and I think there's a lot in the "Personal Knowledge Management" space that just isn't for me, but I have found it to be a very, very useful way to track what I'm doing.

The place where it wasn't totally working for me was with tracking my runs, though.
I've been running on and off for about fifteen years, but during the pandemic it really escalated; when I got injured back in February 2022, I was averaging about 50 miles a week, and I felt just amazing.
(After about six months of rest and recuperation, I'm at a much more manageable and reasonable weekly mileage, but I have still set myself a goal of an ultra after tenure.)
I really wanted to be able to control this information, though, and see it alongside my other daily info.

So I started looking at possibilities, and here's what I've come up with.
I spent a little time trying to figure out if I could build out a plugin, and while I'm *oh-kay* at typescript I'm not really that great at the "last 10%" problem, so instead I'm building it out of a couple other plugins.
Here's what I'm using:

 * [strava-offline](https://github.com/liskin/strava-offline) for downloading Strava data in both GPX and data record (via sqlite) forms
 * [Obsidian Execute Code](https://github.com/twibiral/obsidian-execute-code) for easily running my update scripts within my vault
 * [Obsidian Dataview](https://github.com/blacksmithgu/obsidian-dataview) for viewing the data in table form and querying embedding-page metadata
 * [Obsidian Leaflet](https://github.com/valentine195/obsidian-leaflet-plugin) for map views of the GPX files

It sounds like a lot!
But I promise, it's not so bad.

### Getting the Data

I use the Python package [Strava Offline](https://github.com/liskin/strava-offline) to download my data.
It's `pip`-installable, and has a very stable and straightforward command-line interface.
The trickiest part is getting the `_strava4_session` cookie, but the website explains exactly how to do that.

I use [Obsidian Execute Code](https://github.com/twibiral/obsidian-execute-code) to run code snippets in my notes.
I don't do a *lot* in there, as I'm really quite happy to use [Jupyterlab](https://jupyter.org/), but I have a tendency to put maintenance scripts and whatnot in my vault and run them with regularity.
Often I keep Python stuff in here (and I've used [py-obsidianmd](https://github.com/selimrbd/py-obsidianmd) in the past, but not here) but for this I use just straight up shell commands:

```
strava-offline sqlite
strava-offline gpx --strava4-session MY_SESSION_COOKIE_HERE
sqlite3 -header -csv ~/.local/share/strava_offline/strava.sqlite \
  "select id,name,start_date,moving_time,elapsed_time,distance,total_elevation_gain,type from activity" \
  > ~/Documents/Journal/projects/running/activities.csv
cp -vn ~/.local/share/strava_offline/activities/*.gz \
       ~/Documents/Journal/projects/running/gpx-files/
```

This executes the `strava-offline` package (note that if you're in a conda environment or something you'll need to activate it) to update the sqlite file *and* get the new GPX files.
Then it dumps from the sqlite file into a `.csv` file stored under my `projects/running/` directory.
Then I copy (with verbose mode on, and with no-clobber on, too) all the `gpx.gz` files to `projects/running/gpx-files/`.
**Note!** You can use macros like `@vault` and whatnot in `execute-code` to make this nicer looking.
I'd probably recommend that.

So now, the GPX files and CSV file are in your Vault!
If you use sync, and you have it syncing non-MD files, they should be picked up just fine.

### Displaying the Data

I have two different views set up for my activities.
I use [Obsidian Dataview](https://github.com/blacksmithgu/obsidian-dataview) for lots and lots of queries and stuff, and it makes sense to use it here as well for an overview display.
For detail, I use [Obsidian Leaflet](https://github.com/valentine195/obsidian-leaflet-plugin) and embedded dataview views of the data.

I should note that as of the writing of this blog entry, my [pull request to dataview](https://github.com/blacksmithgu/obsidian-dataview/pull/1882) hasn't been accepted, but it looks like it will be.
This enables the CSV parser to regard things as dates -- otherwise the dates just show up as `fp_incr`.
I also want to thank [Jeremy Valentine](https://github.com/javalent) for helping me out to get the `.gpx` parser in [obsidian-leaflet](https://github.com/javalent/obsidian-leaflet) to [handle `.gz`-compressed files](https://github.com/javalent/obsidian-leaflet/pull/319), as the file size really does make a big different for GPX files.

Anyway!  I have a top-level file called `Running Activities.md` which includes this `dataview` query:

```
table without id id, start_date, name, moving_time, elapsed_time, type, total_elevation_gain
from csv("activities.csv")
sort id desc
```

This is pretty basic, and would probably work a bit better if I were more careful about it.
But, it's a good start.

For individual runs, my setup is:

 1. A simple template for every run, and each run gets its own note.
 2. An embedded dataview "view" that generates the map for each template.
 3. Links to the daily notes from each of the runs, and daily notes have queries for backlinks.

My run "view" template looks like this, and I call it `run-activity.md`:

```
## `$= dv.current().name`
```leaflet
id: ACTIVITY_NAME
zoomFeatures: true
maxZoom: 20
gpx:
  - [[gpx-files/ACTIVITY_ID.gpx.gz]]
```

This is the view that is embedded; in the main `run.md` template, which is instantiated for each run, I have:

```
---
date: a
activityId: Whatever
aliases: []
tags: 
 - running
 - running/route
 - running/strava
---

## Run

\```dataviewjs
 dv.paragraph((await dv.io.load("templates/run-activity.md")).replace("ACTIVITY_ID", dv.current().activityId));
\```

## Notes
```

But, as I'll note below, there are some issues with this!
I want to make it easier to do.

### Future Ideas

In reality, there is a pretty straightforward way of making this much more streamlined.
The Obsidian plugin template is really nice, the Strava Javascript API is really nice, and I think it could be all brought in-house to a single plugin.
I just haven't figured out the best way to do that, or to convince myself to do it.

The other thing is that the actual creation of the instances of `run.md` is a little clunky.
I want to be able to instantiate templates in [Templater](https://silentvoid13.github.io/Templater/) from within the Obsidian API, but I can't see how to do that exactly.
It would be nice to be able to supply variables, a template input filename, and an output filename to a function, but [it isn't quite available as-is](https://github.com/SilentVoid13/Templater/issues/1004).
I think there might be a way to make [QuickAdd](https://github.com/chhoumann/quickadd) do it, but I haven't figured it out quite yet.
This remains the most irritating part -- my template will prompt me for the activity id, but that's really not the most efficient way to do it.
I'd instead much rather be able to parse the CSV file on demand (maybe with a [button](https://github.com/shabegom/buttons)) and any *new* entries get a new instance of the template.

So that's that!
I think that this can be smoother, and it probably will be pretty soon, but here's how I'm doing it right now, rough edges and all.
Drop me a line if you have any suggestions or questions!
