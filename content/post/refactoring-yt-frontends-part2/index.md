+++
title = "Refactoring yt Frontends - Part 2"
subtitle = ""

# Add a summary to display on homepage (optional).
summary = ""

date = 2019-06-10T12:59:33-05:00
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
projects = ["yt"]

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder. 
[image]
  # Caption (optional)
  caption = ""

  # Focal point (optional)
  # Options: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight
  focal_point = ""
+++

**SIDE NOTE**: I intended for this blog post to be a bit shorter than it turned out, and for it to cover some things it ... didn't!  So it looks like there'll be a part three in the series.

## Operations on Data Objects

In my previous post, I walked through a few aspects of how the chunking system in yt works, mostly focusing on the `"io"` style of chunking, where the order in which data arrives is not important.  This style of chunking lends itself very easily to parallelism, as well as dynamic chunk-sizing; we can see this in how operations such as `.max()` operate on a data object in yt.


```python
import yt
ds = yt.load("data/IsolatedGalaxy/galaxy0030/galaxy0030")
dd = ds.r[:,:,:]
```

    yt : [INFO     ] 2019-06-10 12:59:13,433 Parameters: current_time              = 0.0060000200028298
    yt : [INFO     ] 2019-06-10 12:59:13,434 Parameters: domain_dimensions         = [32 32 32]
    yt : [INFO     ] 2019-06-10 12:59:13,435 Parameters: domain_left_edge          = [0. 0. 0.]
    yt : [INFO     ] 2019-06-10 12:59:13,437 Parameters: domain_right_edge         = [1. 1. 1.]
    yt : [INFO     ] 2019-06-10 12:59:13,439 Parameters: cosmological_simulation   = 0.0
    Parsing Hierarchy : 100%|██████████| 173/173 [00:00<00:00, 5931.42it/s]
    yt : [INFO     ] 2019-06-10 12:59:13,487 Gathering a field list (this may take a moment.)



```python
max_vals = dd.max(["density", "temperature", "velocity_magnitude"])
print(max_vals)
```

    (7.73426503924e-24 g/cm**3, 24826104.0 K, 86290042.8768639 cm/s)


I want to highlight something here which sometimes slips by -- if you were to access the array hanging off an object like `dd`, like this:


```python
dd["density"]
```




    YTArray([4.92775113e-31, 4.94005233e-31, 4.93824694e-31, ...,
             1.12879234e-25, 1.59561490e-25, 1.09824903e-24]) g/cm**3



The entire array is loaded into memory.  This is done through a different chunk style, `"all"`, which pre-allocates an array and then loads the whole thing into memory in a single go.  I should note that there are a few reasons that this needs to always be provided in the same order as the `"io"` chunking style, which has presented some fun struggles in refactoring that I may mention later on.

But above, we aren't accessing `dd["density"].max()` and instead are accessing `dd.max("density")` (along with two other fields, temperature and the magnitude of velocity.)  This operation, inspired by the numpy / xarray syntax, iterates over chunks and computes the running max.

There are a few other fun operations that I mentioned last time like `argmax` and whatnot, but for now let's just look at `max`.  While the syntactic sugar for calling `dd.max()` is reasonably recent, the underpinning functionality dates back about a decade and has, from the start, been MPI-parallel.  It hasn't always been elegant, but it's been parallel and memory-conservative.

So how does `.max()` (and, its older form, `.quantities["MaximumValue"]()`) work?  Let's take a look at the source.


```python
dd.max??
```


    Signature: dd.max(field, axis=None)
    Source:   
        def max(self, field, axis=None):
            r"""Compute the maximum of a field, optionally along an axis.
    
            This will, in a parallel-aware fashion, compute the maximum of the
            given field.  Supplying an axis will result in a return value of a
            YTProjection, with method 'mip' for maximum intensity.  If the max has
            already been requested, it will use the cached extrema value.
    
            Parameters
            ----------
            field : string or tuple field name
                The field to maximize.
            axis : string, optional
                If supplied, the axis to project the maximum along.
    
            Returns
            -------
            Either a scalar or a YTProjection.
    
            Examples
            --------
    
            >>> max_temp = reg.max("temperature")
            >>> max_temp_proj = reg.max("temperature", axis="x")
            """
            if axis is None:
                rv = ()
                fields = ensure_list(field)
                for f in fields:
                    rv += (self._compute_extrema(f)[[0;36m1],)
                if len(fields) == [0;36m1:
                    return rv[[0;36m0]
                else:
                    return rv
            elif axis in self.ds.coordinates.axis_name:
                r = self.ds.proj(field, axis, data_source=self, method="mip")
                return r
            else:
                raise NotImplementedError("Unknown axis %s" % axis)
    File:      ~/yt/yt/yt/data_objects/data_containers.py
    Type:      method



(One fun bit here is that if you supply an axis and it's a spatial axis, this will project along the axis.)

Looks like it calls `._compute_extrema` so let's take a look there:


```python
dd._compute_extrema??
```


    Signature: dd._compute_extrema(field)
    Docstring: <no docstring>
    Source:   
        def _compute_extrema(self, field):
            if self._extrema_cache is None:
                self._extrema_cache = {}
            if field not in self._extrema_cache:
                # Note we still need to call extrema for each field, as of right
                # now
                mi, ma = self.quantities.extrema(field)
                self._extrema_cache[field] = (mi, ma)
            return self._extrema_cache[field]
    File:      ~/yt/yt/yt/data_objects/data_containers.py
    Type:      method



(Fun fact: until I saw the source code right now, I was prepared to say that it computed all the extrema in a single go.  Glad there's a backspace key.  I should probably file an issue.)

This calls `self.quantities.extrema` for each field, since it's nearly just as cheap to do both min and max in a single pass, and sometimes folks'll want both.

So we're starting to see the underpinnings here -- `.quantities` is where lots of the fun things happen.  What is it?


```python
dd.quantities.extrema??
```


    Signature:      dd.quantities.extrema(fields, non_zero=False)
    Type:           Extrema
    String form:    <yt.data_objects.derived_quantities.Extrema object at 0x7db454d7a2e8>
    File:           ~/yt/yt/yt/data_objects/derived_quantities.py
    Source:        
    class Extrema(DerivedQuantity):
        r"""
        Calculates the min and max value of a field or list of fields.
        Returns a YTArray for each field requested.  If one, a single YTArray
        is returned, if many, a list of YTArrays in order of field list is
        returned.  The first element of each YTArray is the minimum of the
        field and the second is the maximum of the field.
    
        Parameters
        ----------
        fields
            The field or list of fields over which the extrema are to be
            calculated.
        non_zero : bool
            If True, only positive values are considered in the calculation.
            Default: False
    
        Examples
        --------
    
        >>> ds = load("IsolatedGalaxy/galaxy0030/galaxy0030")
        >>> ad = ds.all_data()
        >>> print ad.quantities.extrema([("gas", "density"),
        ...                              ("gas", "temperature")])
    
        """
        def count_values(self, fields, non_zero):
            self.num_vals = len(fields) * [0;36m2
    
        def __call__(self, fields, non_zero = False):
            fields = ensure_list(fields)
            rv = super(Extrema, self).__call__(fields, non_zero)
            if len(rv) == [0;36m1: rv = rv[[0;36m0]
            return rv
    
        def process_chunk(self, data, fields, non_zero):
            vals = []
            for field in fields:
                field = data._determine_fields(field)[[0;36m0]
                fd = data[field]
                if non_zero: fd = fd[fd > [0;36m0.0]
                if fd.size > [0;36m0:
                    vals += [fd.min(), fd.max()]
                else:
                    vals += [array_like_field(data, HUGE, field),
                             array_like_field(data, -HUGE, field)]
            return vals
    
        def reduce_intermediate(self, values):
            # The values get turned into arrays here.
            return [self.data_source.ds.arr([mis.min(), mas.max()])
                    for mis, mas in zip(values[::[0;36m2], values[[0;36m1::[0;36m2])]
    Call docstring: Calculate results for the derived quantity



Ah, this is starting to make sense!

All the `DerivedQuantity` objects 

What all do we have?


```python
dd.quantities.keys()
```




    dict_keys(['WeightedAverageQuantity', 'TotalQuantity', 'TotalMass', 'CenterOfMass', 'BulkVelocity', 'WeightedVariance', 'AngularMomentumVector', 'Extrema', 'SampleAtMaxFieldValues', 'MaxLocation', 'SampleAtMinFieldValues', 'MinLocation', 'SpinParameter'])



Looking at these, there's likely a common theme that is pretty obvious -- they're all pretty easily parallelizable things!  Sure, there might need to be some reductions at the end, but these are all pretty straightforward combinations of fields and parameters.

The way the base class works is interesting, and we can use that to break down what is going on here in a way that demonstrates how this relies on chunking:


```python
yt.data_objects.derived_quantities.DerivedQuantity??
```


    Init signature: yt.data_objects.derived_quantities.DerivedQuantity(data_source)
    Docstring:      <no docstring>
    Source:        
    class DerivedQuantity(ParallelAnalysisInterface):
        num_vals = -[0;36m1
    
        def __init__(self, data_source):
            self.data_source = data_source
    
        def count_values(self, *args, **kwargs):
            return
    
        def __call__(self, *args, **kwargs):
            """Calculate results for the derived quantity"""
            # create the index if it doesn't exist yet
            self.data_source.ds.index
            self.count_values(*args, **kwargs)
            chunks = self.data_source.chunks([], chunking_style="io")
            storage = {}
            for sto, ds in parallel_objects(chunks, -[0;36m1, storage = storage):
                sto.result = self.process_chunk(ds, *args, **kwargs)
            # Now storage will have everything, and will be done via pickling, so
            # the units will be preserved.  (Credit to Nathan for this
            # idea/implementation.)
            values = [ [] for i in range(self.num_vals) ]
            for key in sorted(storage):
                for i in range(self.num_vals):
                    values[i].append(storage[key][i])
            # These will be YTArrays
            values = [self.data_source.ds.arr(values[i]) for i in range(self.num_vals)]
            values = self.reduce_intermediate(values)
            return values
    
        def process_chunk(self, data, *args, **kwargs):
            raise NotImplementedError
    
        def reduce_intermediate(self, values):
            raise NotImplementedError
    File:           ~/yt/yt/yt/data_objects/derived_quantities.py
    Type:           RegisteredDerivedQuantity
    Subclasses:     WeightedAverageQuantity, TotalQuantity, CenterOfMass, BulkVelocity, WeightedVariance, AngularMomentumVector, Extrema, SampleAtMaxFieldValues, SpinParameter



The key thing I want to highlight here is that this is rather simple in concept; the chunks are iterated over in parallel (via the `parallel_objects` routine, which parcels them out to different processors), processed, and then the reduction happens through `reduce_intermediate`.

There are a few things to note here -- this is actually units-aware, which means that even if you've got (for some reason) `cm` for a quantity on one processor and `km` on another, it will correctly convert them.  The other is that the set up is such that only the `process_chunk` and `reduce_intermediate` operations need to be implemented, along with setting some properties.

But, we're getting a bit far away from the topic at hand, which is why how chunking is set up can cause some issues with exposing data to dask.  And so I want to return to the notion of the `"io"` chunking and how this works for differently indexed datasets.

## Fine- and Coarse-grained Indexing

What yt does during the selection of data is key to how it thinks about the processings of that data.  The way that data can be provided to yt takes several forms:

 * Regularly structured grids and grid based data, where there may be overlapping regions (typically with one "authoritative source of truth" as in [adaptive mesh refinement](https://en.wikipedia.org/wiki/Adaptive_mesh_refinement))
 * Irregular grids, where the distance between points may vary along each spatial axis
 * Unstructured mesh, where the data arrives in hexahedra, tetrahedra, etc, and there is typically a well-defined form for evaluating field values internal to each polyhedra
 * Discrete, or particle-based datasets, where each point is sampled at some location that we don't know *a priori* -- for instance, N-body simulations
 * Octree or block-structured data, which can in some cases be thought of as a special-case of grid based data but that follows a more regular form

Several of these have a common trait that comes in quite handy for yt -- namely, that the *index* of the data occupies considerably less memory than the data itself.

### Grid Indexing

For instance, when dealing with a grid of data, typically that grid can be defined by a set of properties such as:

 * "origin" corner of the grid ("left edge")
 * "terminal" corner of the grid ("right edge")
 * dimensions along each axis
 * if irregular, the cell-spacing along each axis
 
There are of course a handful of other attributes that might be useful (and which we can sometimes deduce) but these are the basics.  If we imagine that each of these requires 64-bits per axis per value, a standard (regular) grid requires 576 bits, or 72 bytes.  If we were storing the actual value locations, each would require 3 64-bit numbers -- which means that as soon as we were storing 3 of them, we would 

(Of course, one probably doesn't need to store dimensions as 64 bits, and there are also probably some other ways to reduce the info necessary, but as [straw-person arguments](https://en.wikipedia.org/wiki/Straw_man) go, this isn't so bad.)

What we can get to with this is that for grid and other regular datasets, it's reasonably *cheap* to index the data.  So when we create a data object, for instance:


```python
sp = ds.sphere("center", (100.0, "kpc"))
```

yt can determine *without touching the disk* how many grid cells intersect it, and thus it can pre-allocate arrays of the correct size and fill them in progressively, in whatever fashion it deems best for IO purposes.

This isn't without cost -- computing the intersections can be quite costly, and so we do some things to cache those.  (The cost/benefit of caching often bites us when we are dealing with large unigrid datasets, though.)  This was all designed to prevent having to call a big `np.concatenate` at some point in the operation when chunking based on `"all"`, but it's not always obvious to me that the balance was correctly struck here.

When an object is created, no selection is conducted until a field is requested.  At some point in the call stack once a field is asked for, the function `index._identify_base_chunk` is called.  This is where things are different for particles, but we'll get to that later.

### Particle Indexing

When dealing with particles, our indexing requirements are very different.  Here, the cost of storing the index values is very high -- but, we also don't want to have to perform too much IO.  So we're stuck minimizing how much IO we do, while also minimizing the amount of information we store in-memory once we "index" a dataset.

In yt-4.0, we accomplish this through the use of bitmap indices, which I described a little bit in the first post.  The basic idea of this is that each "file" (which can be subsets of a single file, and is better thought of as an IO block of some type) is assigned some unique ID.  All the files are iterated over and for each discrete point included in that file, an index into a space-filling curve is generated.  We use a resonably coarse space filling curve for the first iteration -- say, a level 2 curve -- and that allows ambiguities.  This is essentially a binning operation.

(Incidentally, we often use [Morton Z-Ordering](https://en.wikipedia.org/wiki/Z-order) because it's just easier to explain.  We might get better compression if we used [Hilbert](https://en.wikipedia.org/wiki/Hilbert_curve) since consecutive values may be more likely to be identical.)

At the end of the first iteration, we have a key-value store of bitarrays, where the key is the file ID and the value is a set of 1's and 0's, where a 1 indicates that a particle is found in a given region identified by the space-filling curve index corresponding with that 1's index in the array.  So, for instance, if we had a level 3 index, we would have a set of bitarrays that looked like:

```
001 000 101
010 011 011
...
```

So, if we read from left-to-right, the first file has particles that live in (zero-indexed) indices 2, 6 and 8.  The second file has particles in indices 1, 4, 5, 7 and 8.

If we know that our selector only intersects areas touched by index 2, then we only have to read from the first file.

This would work great if we had particles that were distributed pretty homogeneously on large scales, but in many cases, we don't.  Sometimes when particles are written to disk they are sorted on some high-order index and then written out in that order.  What yt does is perform a secondary, as-needed indexing based on where there are "collisions" -- i.e., ambiguities.  A set of logical operations is performed across all the bitarrays to identify where multiple files overlap; following this, a *second* round of indexing is conducted at a much higher spatial order.

In doing this, we are able to pinpoint with reasonably high precision the file or files that need to be read to get data from a given selector, and minimize very precisely the amount of over-reading that is done.

Unfortunately, this doesn't give us the ability to explicitly allocate arrays of the correct size.  (And, the memory overhead of regaining that ability would be quite high.)  But as we saw above, yt doesn't want to do big concatenation operations!  So it does the thing I really wish it didn't, which is ... it reads all the position data in IO chunks, figures out how big it is (which only requires a running tally, not a set of allocated arrays), then allocates and fills that single big array.

This isn't really that efficient, and it arises from the case where the indexing is comparatively cheap.

But all of this arises out of the design decision that we need to optimize for the case that we want a single big array, rather than a bunch of small arrays -- i.e., for the case of:

```python
ds.r[:]["density"].max()
```

as opposed to

```python
ds.r[:].max("density")
```

## ...didn't you say you'd be talking about Dask?

Well, this is where dask comes in!  And, it's also why interfacing to dask is a bit tricky -- because we do a lot of work ahead of time before allocating any arrays, and then we get rid of the information generated during that work.

In an ideal world, what we would be able to do is to export a *data object* (such as a sphere or cylinder or rectangular prism) and a *field-type* (so we knew if it was a vector, or particles, or nodal/zonal data) as a dask array.  For instance, if instead of returning an array (specifically, a `YTArray` or `unyt_array`) when we accessed `sp["density"]`, it returned a `DaskArray`, we would open up a number of new and interesting techniques.

But to do that, we need to be able to know in advance the chunk sizes, and more to the point, we need to be able to specify a function that returns each chunk *uniquely*.

## Next Entry: Iterables and IO

Turns out, I thought I'd be done with this entry a lot sooner than I was!

In the next blog post, which hopefully will take less than the 8 days this one did, I'll talk about why this is (currently) hard, how to fix that, and what we're doing to fix it.
