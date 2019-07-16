+++
title = "yt: Internal and External Ecosystems"
subtitle = ""

# Add a summary to display on homepage (optional).
summary = ""

date = 2019-07-16T14:28:36-05:00
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
# projects = ["yt"]

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder. 
[image]
  # Caption (optional)
  caption = ""

  # Focal point (optional)
  # Options: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight
  focal_point = ""
+++

I think I've talked myself into proposing a big change in yt.  I'm not the "boss" of yt, so it might not happen, but I've kind of worked up my courage to make a serious suggestion.

This last week I have been at [SciPy 2019](https://scipy2019.scipy.org/) and I had the opportunity to see a lot of talks.

There were a few that really stuck with me, but for the purposes of this rather technically-focused blog post, I'm going to stick to just one in particular.

[Matt Rocklin](http://matthewrocklin.com/) gave a talk about [refactoring the ecosystem to prepare for heterogeneous computing](https://www.youtube.com/watch?v=Q0DsdiY-jiw)  (you should go watch it!).  More specifically, though, what it seemed to me was that it was a talk more about an opportunity to avoid fragmentation and think more carefully about how arrays and APIs are thought of and used.  That got me thinking about something I've kind of touched on in previous posts ([here](https://matthewturk.github.io/post/refactoring-yt-frontends-part1/), [here](https://matthewturk.github.io/post/refactoring-yt-frontends-part2/) and [here](https://matthewturk.github.io/post/refactoring-yt-frontends-part3/)) -- basically, that yt is pretty monolithic, and that's not really the best way to evolve with the ecosystem.

I'll be using [findimports](https://github.com/mgedmin/findimports) for exploring how monolithic it *is* versus how monolithic it *appears to be*.  Basically, I want to see: is it one repo with lots of interconnections, or is it essentially a couple repos?

(Also at the end I'll give a pitch for why this is relevant, so if you're even remotely intrigued, at *least* scroll down to the section labeled "OK, the boring stuff is over.")


```python
import pickle
import findimports
yt_imports = pickle.load(open("yt/yt/import_output.pkl", "rb"))
```

The structure of this is a set of keys that are strings of the filename/modulename, with values that are the objects in question.  The `findimports` objects have an attribute `imports` which is what we're going to look at first, but they also have an `imported_names` attribute which is the list of names that get imported, in the form of `ImportInfo` objects.  These have `name`, `filename`, `level` and `lineno` to show where and what they are. 


```python
yt_imports['yt.visualization.plot_window'].imports
```




    {'collections',
     'distutils.version',
     'matplotlib',
     'matplotlib.mathtext',
     'matplotlib.ticker',
     'numbers',
     'numpy',
     'pyparsing',
     'sys',
     'types',
     'unyt.exceptions',
     'yt',
     'yt.data_objects.image_array',
     'yt.frontends.ytdata.data_structures',
     'yt.funcs',
     'yt.units.unit_object',
     'yt.units.unit_registry',
     'yt.units.yt_array',
     'yt.utilities.exceptions',
     'yt.utilities.math_utils',
     'yt.utilities.orientation',
     'yt.visualization.base_plot_types',
     'yt.visualization.fixed_resolution',
     'yt.visualization.geo_plot_utils',
     'yt.visualization.plot_container',
     'yt.visualization.plot_modifications'}



There happen to be a fair number of things in here that are external to yt!  So, let's set up a filtering process for those.  We'll filter the `name` that is imported.

One thing I should note is that yt does many, but not *all*, of its imports in absolute form, which maybe isn't ... so great ... but which lets us do this more easily.


```python
filter_imports = lambda a: [_ for _ in sorted(a, key=lambda b: b.name) if _.name.startswith("yt.")]
```

We'll apply it to the `imported_names` attribute, since we're interested in characterizing how things are related and interweaved.


```python
import_lists = {_ : filter_imports(yt_imports[_].imported_names) for _ in yt_imports}
```


```python
import_lists['yt.visualization.plot_window']
```




    [ImportInfo('yt.data_objects.image_array.ImageArray', 'yt/visualization/plot_window.py', 40, 0),
     ImportInfo('yt.frontends.ytdata.data_structures.YTSpatialPlotDataset', 'yt/visualization/plot_window.py', 42, 0),
     ImportInfo('yt.funcs.ensure_list', 'yt/visualization/plot_window.py', 44, 0),
     ImportInfo('yt.funcs.fix_axis', 'yt/visualization/plot_window.py', 45, 0),
     ImportInfo('yt.funcs.fix_unitary', 'yt/visualization/plot_window.py', 45, 0),
     ImportInfo('yt.funcs.iterable', 'yt/visualization/plot_window.py', 44, 0),
     ImportInfo('yt.funcs.mylog', 'yt/visualization/plot_window.py', 44, 0),
     ImportInfo('yt.funcs.obj_length', 'yt/visualization/plot_window.py', 45, 0),
     ImportInfo('yt.load', 'yt/visualization/plot_window.py', 737, 0),
     ImportInfo('yt.load', 'yt/visualization/plot_window.py', 1373, 0),
     ImportInfo('yt.load', 'yt/visualization/plot_window.py', 1557, 0),
     ImportInfo('yt.load', 'yt/visualization/plot_window.py', 2067, 0),
     ImportInfo('yt.units.unit_object.Unit', 'yt/visualization/plot_window.py', 47, 0),
     ImportInfo('yt.units.unit_registry.UnitParseError', 'yt/visualization/plot_window.py', 49, 0),
     ImportInfo('yt.units.yt_array.YTArray', 'yt/visualization/plot_window.py', 51, 0),
     ImportInfo('yt.units.yt_array.YTQuantity', 'yt/visualization/plot_window.py', 51, 0),
     ImportInfo('yt.utilities.exceptions.YTCannotParseUnitDisplayName', 'yt/visualization/plot_window.py', 57, 0),
     ImportInfo('yt.utilities.exceptions.YTDataTypeUnsupported', 'yt/visualization/plot_window.py', 59, 0),
     ImportInfo('yt.utilities.exceptions.YTInvalidFieldType', 'yt/visualization/plot_window.py', 60, 0),
     ImportInfo('yt.utilities.exceptions.YTPlotCallbackError', 'yt/visualization/plot_window.py', 58, 0),
     ImportInfo('yt.utilities.exceptions.YTUnitNotRecognized', 'yt/visualization/plot_window.py', 61, 0),
     ImportInfo('yt.utilities.math_utils.ortho_find', 'yt/visualization/plot_window.py', 53, 0),
     ImportInfo('yt.utilities.orientation.Orientation', 'yt/visualization/plot_window.py', 55, 0)]



This still isn't *incredibly* useful, since we kind of want to look at imports at a higher level.  For instance, I want to know what `yt.visualization.plot_window` imports from in the broad cross-section of the code base.  So let's write something to collapse the package *under* yt that we import from.  We used `startswith(".yt")` earlier, so it'll be safe to do a split here.


```python
collapse_subpackage = lambda a: set(_.name.split(".")[1] for _ in a)
```


```python
collapse_subpackage(import_lists['yt.visualization.plot_window'])
```




    {'data_objects', 'frontends', 'funcs', 'load', 'units', 'utilities'}



Interesting.  We import from frontends?!  I guess I kind of missed that earlier.  Let's see if we can figure out the connections between different modules to see if anything stands out.


```python
from collections import defaultdict
```


```python
subpackage_imports = defaultdict(set)
for fn, v in import_lists.items():
    if not fn.startswith("yt."): continue # Get rid of our tests, etc.
    subpackage = fn.split(".")[1]
    subpackage_imports[subpackage].update(collapse_subpackage(v))
```

Let's break this down before we go any further -- for starters, not *everything* is an absolute import.  So that makes things a bit tricky!  But we can deal with that later.  Let's first see what all we have:


```python
subpackage_imports.keys()
```




    dict_keys(['__init__', 'api', 'arraytypes', 'config', 'convenience', 'exthook', 'funcs', 'mods', 'pmods', 'startup_tasks', 'testing', 'analysis_modules', 'data_objects', 'extensions', 'extern', 'fields', 'frontends', 'geometry', 'tests', 'units', 'utilities', 'visualization'])



A few things stand out right away.  Some of these we can immediately get rid of and not consider.  For instance, `pmods` is an MPI-aware importer, `mods` is a pretty old-school approach to yt importing, and we will just ignore `testing`, `analysis_modules`, `extensions` and `extern` since they're (in order) testing utilities, gone, a fake hook system, and "vendored" libraries that we should probably get rid of and just make requirements anyway.  `units` is now part of [`unyt`](https://github.com/yt-project/unyt) and some of the others are by-design grabbing lots of stuff.


```python
blacklist = ["testing", "analysis_modules", "extensions", "extern", "pmods",
             "mods", "__init__", "api", "arraytypes", "config", "convenience",
            "exthook", "funcs", "tests", "units", "startup_tasks"]

list(subpackage_imports.pop(_, None) for _ in blacklist);
```

We just want to see the interrelationships, so we'll look for N-by-N collisions, where N is just the values that show up as keys.


```python
collide_with = set(subpackage_imports.keys())
```


```python
collisions = {_: collide_with.intersection(subpackage_imports[_]) for _ in subpackage_imports}
```

And here we have it, the moment of truth!  What do we see ...


```python
print({_:len(__) for _, __ in collisions.items()})
```

    {'data_objects': 6, 'fields': 5, 'frontends': 6, 'geometry': 5, 'utilities': 6, 'visualization': 6}


Huh.  Well, that was not the dramatic, amazing reveal I'd hoped for.


```python
subpackage_imports = defaultdict(set)
for fn, v in import_lists.items():
    if not fn.startswith("yt.") or "tests" in fn: continue # Get rid of our tests, etc.
    subpackage = fn.split(".")[1]
    subpackage_imports[subpackage].update(collapse_subpackage(v))
list(subpackage_imports.pop(_, None) for _ in blacklist);
collisions = {_: collide_with.intersection(subpackage_imports[_]) for _ in subpackage_imports}
print({_:len(__) for _, __ in collisions.items()})
```

    {'data_objects': 6, 'fields': 4, 'frontends': 6, 'geometry': 4, 'utilities': 5, 'visualization': 6}


It gets a little bit better, but honestly, not much.  Our most isolated package -- by this (likely flawed) method -- are the `geometry` and `fields` packages.  So let's break down a bit more what we're seeing, by not filtering quite as much, and by setting up a reverse mapping.  And let's do it for both the collapsed name and the non-collapsed name.


```python
subpackage_imports = defaultdict(set)
imported_by = defaultdict(list)
for fn, v in import_lists.items():
    if not fn.startswith("yt.") or "tests" in fn: continue # Get rid of our tests, etc.
    subpackage = fn.split(".")[1]
    subpackage_imports[subpackage].update(set(_.name for _ in v))
    [imported_by[_.name].append(fn) for _ in v]
    [imported_by[_].append(fn) for _ in collapse_subpackage(v)]
```

And now we might be getting somewhere.  So now we can look up for any given import which files have imported it.  Let's see what imports the progress bar:


```python
imported_by["yt.funcs.get_pbar"]
```




    ['yt.__init__',
     'yt.data_objects.particle_trajectories',
     'yt.data_objects.level_sets.contour_finder',
     'yt.frontends.athena_pp.data_structures',
     'yt.frontends.enzo.data_structures',
     'yt.frontends.enzo_p.data_structures',
     'yt.geometry.particle_geometry_handler',
     'yt.utilities.minimal_representation',
     'yt.utilities.particle_generator',
     'yt.utilities.answer_testing.framework',
     'yt.visualization.streamlines',
     'yt.visualization.volume_rendering.old_camera']



Nice.  Now, let's look at visualization.


```python
imported_by["yt.visualization.api.SlicePlot"], imported_by["yt.visualization.plot_window.SlicePlot"]
```




    (['yt.__init__', 'yt.data_objects.analyzer_objects'],
     ['yt.utilities.command_line'])



We're starting to see that things might not be quite as clear-cut as we thought.  Let's look at geometry.  And I'm going to set up a filtering method so that we can avoid lots of redundant pieces of info -- for instance, I don't care about things importing themselves.


```python
filter_self_imports = lambda a: [_ for _ in imported_by[a] if not _.startswith("yt.{}".format(a))]
```

We'll only look at the first ten, because it's really long...


```python
filter_self_imports("geometry")[:10]
```




    ['yt.data_objects.construction_data_containers',
     'yt.data_objects.data_containers',
     'yt.data_objects.grid_patch',
     'yt.data_objects.octree_subset',
     'yt.data_objects.selection_data_containers',
     'yt.data_objects.static_output',
     'yt.data_objects.unstructured_mesh',
     'yt.frontends._skeleton.data_structures',
     'yt.frontends.ahf.data_structures',
     'yt.frontends.art.data_structures']



Here things are *much* clearer.  We import geometry *once* in the visualization subsystem, under `plot_modifications`.  I looked it up, and here's what it is:

```python
if not issubclass(type(index), UnstructuredIndex):
    raise RuntimeError("Mesh line annotations only work for "
                        "unstructured or semi-structured mesh data.")
```

This is probably an anti-pattern, but even if we wanted to retain this specific behavior, we could remedy it without too much trouble by having an attribute check, or some kind of string-key check.

As for all the `frontends` imports, those are all because they subclass `Index`!  And many of the places importing it in `data_objects` are just due to a lack of organization in the geometry/utilities/indexing code.

**Historical Sidenote**: As I was doing this, I read the header for `grid_patch.py` and it reads: `"Python-based grid handler, not to be confused with the SWIG-handler"`.  I am reasonably certain that it has been *years* since I thought about the proto-SWIG system I'd written to wrap around the Enzo C++ code.  Kinda supports the point I intend to make when I end this post, I think.

Back to the task at hand, let's look at some of the other top-level packages and how they related.  I'm now specifically interested in the `visualization` and `data_objects` ones.


```python
filter_self_imports("visualization")
```




    ['yt.__init__',
     'yt.data_objects.analyzer_objects',
     'yt.data_objects.construction_data_containers',
     'yt.data_objects.data_containers',
     'yt.data_objects.image_array',
     'yt.data_objects.profiles',
     'yt.data_objects.region_expression',
     'yt.data_objects.selection_data_containers',
     'yt.frontends.fits.misc',
     'yt.utilities.command_line',
     'yt.utilities.fits_image',
     'yt.utilities.answer_testing.framework']



Well, that's interesting.  A quick skim of the code suggests that `analyzer_objects` is probably dead code to be removed, `construction_data_containers` imports viz so that `.plot()` will work on projections, and there are a handful of other data-object-to-viz things that get done here.

In short, I'm pretty sure that `visualization` is a mostly independent subpackage.  And the same kind of goes for `geometry`.  The story isn't quite as clear for the others.

## OK, the boring stuff is over.

Here's where I wanted to get this whole time: yt is a monolithic package in packaging, but it also has a couple reasonably independent sub-packages within it.  It *would* be a target for breaking up, *if* those subpackages were independently useful.  But as it stands, they probably aren't, since they're all very tightly coupled.

They're coupled because they were written that way, without too much thought given to a public, externally-useful API.  This is something that's probably not surprising, since yt was a big package that evolved, rather than many packages that interoperated.  Lots of stuff we wanted didn't exist, and there was (we thought) really only one obvious way to do things, so why not just do it?

(**Historical sidenote**: It was a few years ago that I remember asking at a panel what middleware developers, like I saw myself, were supposed to do in a rapidly evolving ecosystem of the array container.  (I probably didn't phrase it very well.)  What I took away was: stick with numpy.  And we did stick with numpy, and more importantly, we *stuck with the Cython interface to numpy*.)

## OK, no more rambling, get to the point

yt did a lot of stuff on its own.  It doesn't need to anymore.  It's effectively a medium-ly coupled system, but one that has relied on the ability to change non-public APIs all the time.

I'm starting to convince myself it's time to mature as a project, and to do some combination of *jettisoning* and *liberating* things in yt.  I'll make it clear that I could be wrong on this, and seeing this through will probably take more thought, time and energy than I can reasonably personally commit, but I think I've started to see what would be a productive path forward.

# Steps to Integrate Better

Here are some things that I think yt could do.  These are my thoughts, which I have *not* presented to the steering council, written up in a YTEP, or even made any efforts toward.  In fact, the one member of the steering committee to whom I said, "I think I might blog about this" explicitly suggested I not do that thing (blog)!  But here I am a couple pages deep and I've always been a bit of a hoarder.

### Inventory

The first step would be to take a real inventory of what is in yt, and what ought to be in yt.  I have some thoughts on things that could be gotten rid of (and I will happily note that I intend to "kill my own darlings" first, before anyone else's) but that can come later.

Not everything yt does needs to be done *by yt*.  But before we can figure that out, let's figure out what yt is doing, and where it gets relied on.

### Indexing

The thing I think we absolutely need to think about is how tightly coupled the indexing system is with everything else.  This will almost certainly be a full blog post at a later time, but the way the indexing is coupled so tightly with the frontends makes me increasingly leery.

I personally think that the way the grid indexing is set up is too fixed on a pre-parsing step and a finalization step, with lots of little bits in between that need to be handled, and that it could much more easily be set up with a different procedure.

For the particle frontends, the particle bitmap indexing has the filenames and the particle types and the coordinates and *all* of that all wound together.  This should be decoupled, so that the notion of the particle locations is held at a higher level than the bitmap indexing.

### Consider Splitting the Package

This one ... I want to emphasize that what I think we should do is the process of considering it.  I'm not sure that it should be split.  But I think that evaluating the pros and cons will lead us to think about *how* our packages interact.

If two objects call weird methods on each other, why?  Does that need to happen?  Is it a micro-optimization that makes no sense other than obfuscation?  I'm not sure, but we can't figure it out unless we examine.

# Big Picture

yt may stay a big repository, but if we reduce the dependence on artisanal objects (why not just subclass `xarray.Dataset` instead of have a `GridPatch` object?  Well, okay, that one is pretty complicated, but we can talk about it later...) and we think about the specifics of why we use public APIs versus private APIs, it can lead to a more robust, lightweight package.

And anything that keeps us more lightweight, reduces maintenance burdens, and enables us to take advantage of the astounding advances in the ecosystem is probably going to be better in the long-run.
