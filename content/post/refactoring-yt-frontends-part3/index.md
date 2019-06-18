+++
title = "Refactoring yt Frontends - Part 3"
subtitle = ""

# Add a summary to display on homepage (optional).
summary = ""

date = 2019-06-17T22:32:09-04:00
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

Welcome to part 3 of a series on how yt deals with data and the ways that helps and hinders things!  This time, I am going to describe what "chunks" of data (`YTDataChunk`) in yt are, and a few characteristics of them that wouldn't be obvious from the previous blog posts.

### Chunks have spatial attributes

Data chunks in yt have a set of special attributes that help yt to put them in the context of the coordinate domain.

(This is as good a time as any to note to myself that I should probably write up a blog post about how coordinate handling works, as opposed to index handling.)

When you receive a chunk of a data object, that chunk will have with it information about the coordinate space of the data contained within it -- specifically, if it is a dataset that is volumetrically discretized in some regular way, it will have information about the centers of the individual grid cells, the sizes of those cells, and some measure of their resolution.

For a grid dataset specifically, these attributes will always exist on a data chunk:

 * `fcoords` - the centers of the grid cells, of shape `(..., 3)`
 * `fwidth` - the "width" of the grid cells, of shape `(..., 3)`
 * `icoords` - the *integer* coordinates, with respect to the *current* resolution, of the grid cells, of shape `(..., 3)`
 * `ires` - the "level of resolution" of a given cell; this mostly makes sense for datasets where there is some universal refinement ratio and a fixed value for the domain dimensions, of shape `(...,)`
 
There are a couple more that are specific to the individual data selection operation -- for instance, `tcoords` only makes sense if there is a parameterized vector being pushed through the domain.  There's also `fcoords_vertex` but it is seldom used except in unstructured mesh datasets.

Let's take a look at these values, and how they correspond to grid attributes:


```python
import yt
ds = yt.load("data/IsolatedGalaxy/galaxy0030/galaxy0030")
```

    /home/matthewturk/conda-py3/lib/python3.7/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 88 from C header, got 96 from PyObject
      return f(*args, **kwds)
    yt : [INFO     ] 2019-06-17 22:23:02,723 Parameters: current_time              = 0.0060000200028298
    yt : [INFO     ] 2019-06-17 22:23:02,724 Parameters: domain_dimensions         = [32 32 32]
    yt : [INFO     ] 2019-06-17 22:23:02,726 Parameters: domain_left_edge          = [0. 0. 0.]
    yt : [INFO     ] 2019-06-17 22:23:02,728 Parameters: domain_right_edge         = [1. 1. 1.]
    yt : [INFO     ] 2019-06-17 22:23:02,734 Parameters: cosmological_simulation   = 0.0


What yt is doing here is taking an individual, in-memory *sparse* index and then *expanding* that index into a fully-fledged array.  Internally, when we ask for any of these attributes, yt iterates over all the objects that belong in a given chunk and then it expands the values associated with those objects.

We can see this if we poke a little closer at the `Grid` objects in this case.


```python
for g in ds.index.grids:
    pass
```

    Parsing Hierarchy : 100%|██████████| 173/173 [00:00<00:00, 3064.43it/s]
    yt : [INFO     ] 2019-06-17 22:23:02,831 Gathering a field list (this may take a moment.)


This is used internally in yt whenever we want to deal with a geometric selection, or when we apply geometric processing.  For instance, the "projection" operator can use the integer coordinate system to very rapidly insert values into a [quadtree](https://en.wikipedia.org/wiki/Quadtree).  yt can iterate over all of the chunks that belong to a data object and insert them into the quadtree based on their `icoords`, and it can refine nodes by bit shifting the appropriate amount.

For grid data, `icoords` also gives us some handy things for evaluating relationships between objects.  For instance, we might have a root grid with one child grid.  We can figure out their relationship by looking at their `icoords`, `ires` and the result of `get_global_startindex()`.  In our sample dataset, let's look at some random grid -- I happen to know the first level 8 grid is index 27, so let's use that.


```python
child = ds.index.grids[27]
parent = child.Parent
```


```python
parent.icoords
```




    array([[2048, 2048, 2048],
           [2048, 2048, 2049],
           [2048, 2048, 2050],
           ...,
           [2087, 2089, 2069],
           [2087, 2089, 2070],
           [2087, 2089, 2071]])




```python
child.icoords
```




    array([[4096, 4096, 4096],
           [4096, 4096, 4097],
           [4096, 4096, 4098],
           ...,
           [4133, 4129, 4119],
           [4133, 4129, 4120],
           [4133, 4129, 4121]])




```python
parent.LeftEdge, child.LeftEdge
```




    (YTArray([0.5, 0.5, 0.5]) code_length, YTArray([0.5, 0.5, 0.5]) code_length)



They both start at the same place -- so that would suggest that their minimum icoords should refer to the same location.


```python
difference = child.ires.min() - parent.ires.max()
left_edge_child = child.icoords.min(axis=0)
left_edge_parent = parent.icoords.min(axis=0)
(left_edge_child - left_edge_parent)
```




    array([2048, 2048, 2048])



This seems odd until we recognize that `ires` differs between the two, and in this case refers to the bit shifting we need to do to match them up.


```python
(left_edge_child >> difference) - left_edge_parent
```




    array([0, 0, 0])



We can do this with the next grid up, as well.


```python
left_edge_gparent = parent.Parent.icoords.min(axis=0)
difference_g = child.ires.min() - parent.Parent.ires.max()
(left_edge_child >> difference_g) - left_edge_gparent
```




    array([0, 0, 0])



There's more to this story -- for instance, non-power-of-two differences, but the idea is consistent; both integer and float positioning can be used between chunks, sub-chunks, and so on.

We can also pretty easily get cell positions, and if we access data objects, we receive all the values across all sub-objects.


```python
sp = ds.sphere("c", 0.1)
print(sp.fwidth)
```

    [[0.00024414 0.00024414 0.00024414]
     [0.00024414 0.00024414 0.00024414]
     [0.00024414 0.00024414 0.00024414]
     ...
     [0.00012207 0.00012207 0.00012207]
     [0.00012207 0.00012207 0.00012207]
     [0.00012207 0.00012207 0.00012207]] code_length


### Chunks can be sub-chunked

The other main advantage of chunks is that they can be sub-chunked.  Usually this only means up to one level, but it means that we could in principle do something like this:


```python
dd = ds.all_data()
for i, chunk1 in enumerate(dd.chunks([], "io")):
    print("Chunk ", i, len(chunk1._current_chunk.objs))
    print("      ", end = "")
    for j, chunk2 in enumerate(dd.chunks([], "spatial")):
        print(chunk2._current_chunk.objs, end = " ")
    print()
    print()
```

    Chunk  0 5
          [EnzoGrid_0001] [EnzoGrid_0075] [EnzoGrid_0076] [EnzoGrid_0082] [EnzoGrid_0110] 
    
    Chunk  1 1
          [EnzoGrid_0073] 
    
    Chunk  2 20
          [EnzoGrid_0009] [EnzoGrid_0010] [EnzoGrid_0011] [EnzoGrid_0012] [EnzoGrid_0013] [EnzoGrid_0014] [EnzoGrid_0015] [EnzoGrid_0016] [EnzoGrid_0017] [EnzoGrid_0018] [EnzoGrid_0019] [EnzoGrid_0020] [EnzoGrid_0021] [EnzoGrid_0022] [EnzoGrid_0023] [EnzoGrid_0024] [EnzoGrid_0025] [EnzoGrid_0026] [EnzoGrid_0027] [EnzoGrid_0028] 
    
    Chunk  3 28
          [EnzoGrid_0008] [EnzoGrid_0029] [EnzoGrid_0030] [EnzoGrid_0031] [EnzoGrid_0032] [EnzoGrid_0033] [EnzoGrid_0034] [EnzoGrid_0035] [EnzoGrid_0036] [EnzoGrid_0037] [EnzoGrid_0038] [EnzoGrid_0039] [EnzoGrid_0040] [EnzoGrid_0041] [EnzoGrid_0042] [EnzoGrid_0043] [EnzoGrid_0044] [EnzoGrid_0045] [EnzoGrid_0046] [EnzoGrid_0047] [EnzoGrid_0048] [EnzoGrid_0049] [EnzoGrid_0050] [EnzoGrid_0051] [EnzoGrid_0052] [EnzoGrid_0053] [EnzoGrid_0054] [EnzoGrid_0055] 
    
    Chunk  4 20
          [EnzoGrid_0007] [EnzoGrid_0056] [EnzoGrid_0057] [EnzoGrid_0058] [EnzoGrid_0059] [EnzoGrid_0060] [EnzoGrid_0061] [EnzoGrid_0062] [EnzoGrid_0063] [EnzoGrid_0064] [EnzoGrid_0065] [EnzoGrid_0066] [EnzoGrid_0067] [EnzoGrid_0068] [EnzoGrid_0069] [EnzoGrid_0070] [EnzoGrid_0071] [EnzoGrid_0072] [EnzoGrid_0074] [EnzoGrid_0077] 
    
    Chunk  5 16
          [EnzoGrid_0006] [EnzoGrid_0078] [EnzoGrid_0079] [EnzoGrid_0080] [EnzoGrid_0081] [EnzoGrid_0083] [EnzoGrid_0084] [EnzoGrid_0085] [EnzoGrid_0086] [EnzoGrid_0087] [EnzoGrid_0088] [EnzoGrid_0089] [EnzoGrid_0090] [EnzoGrid_0091] [EnzoGrid_0092] [EnzoGrid_0093] 
    
    Chunk  6 13
          [EnzoGrid_0005] [EnzoGrid_0094] [EnzoGrid_0095] [EnzoGrid_0096] [EnzoGrid_0097] [EnzoGrid_0098] [EnzoGrid_0099] [EnzoGrid_0100] [EnzoGrid_0101] [EnzoGrid_0102] [EnzoGrid_0103] [EnzoGrid_0104] [EnzoGrid_0105] 
    
    Chunk  7 14
          [EnzoGrid_0004] [EnzoGrid_0106] [EnzoGrid_0107] [EnzoGrid_0108] [EnzoGrid_0109] [EnzoGrid_0111] [EnzoGrid_0112] [EnzoGrid_0113] [EnzoGrid_0114] [EnzoGrid_0115] [EnzoGrid_0116] [EnzoGrid_0117] [EnzoGrid_0118] [EnzoGrid_0119] 
    
    Chunk  8 26
          [EnzoGrid_0003] [EnzoGrid_0120] [EnzoGrid_0121] [EnzoGrid_0122] [EnzoGrid_0123] [EnzoGrid_0124] [EnzoGrid_0125] [EnzoGrid_0126] [EnzoGrid_0127] [EnzoGrid_0128] [EnzoGrid_0129] [EnzoGrid_0130] [EnzoGrid_0131] [EnzoGrid_0132] [EnzoGrid_0133] [EnzoGrid_0134] [EnzoGrid_0135] [EnzoGrid_0136] [EnzoGrid_0137] [EnzoGrid_0138] [EnzoGrid_0139] [EnzoGrid_0140] [EnzoGrid_0141] [EnzoGrid_0142] [EnzoGrid_0143] [EnzoGrid_0144] 
    
    Chunk  9 30
          [EnzoGrid_0002] [EnzoGrid_0145] [EnzoGrid_0146] [EnzoGrid_0147] [EnzoGrid_0148] [EnzoGrid_0149] [EnzoGrid_0150] [EnzoGrid_0151] [EnzoGrid_0152] [EnzoGrid_0153] [EnzoGrid_0154] [EnzoGrid_0155] [EnzoGrid_0156] [EnzoGrid_0157] [EnzoGrid_0158] [EnzoGrid_0159] [EnzoGrid_0160] [EnzoGrid_0161] [EnzoGrid_0162] [EnzoGrid_0163] [EnzoGrid_0164] [EnzoGrid_0165] [EnzoGrid_0166] [EnzoGrid_0167] [EnzoGrid_0168] [EnzoGrid_0169] [EnzoGrid_0170] [EnzoGrid_0171] [EnzoGrid_0172] [EnzoGrid_0173] 
    


(In general, this is about as deep as it goes, although in principle we could do lots more sub-chunking.)  This lets you chunk over IO, then chunk over individual objects, and even request things like the number of ghost zones you need in that sub-chunking.

### Chunks can retain state during a long-lived IO task

We can also store field parameters and reset them during an individual chunking operation.  So for instance, we could have a custom field that requires one field parameter that is then swapped out during the next level of chunking.

This feature is probably not used much.

## Iteration and Chunks

But here's the issue we run into, which actually shows up whenever an error is raised during a chunking operation.

**All of this is done via generator expressions.**  This seemed like the right thing to do!  Just `yield` everywhere! It was so hip.

But, there's an even more problematic part: **often, the generator expressions get unrolled into lists anyway.**  And, it turns out, I can't blame anybody else for this: this particular core element of yt was something I not only put in, but something I felt rather self-satisfied about.

Let's take a look at this in the routine that does chunking for the `io` type in grid index datasets:


```python
ds.index._chunk_io??
```


    Signature:
    ds.index._chunk_io(
        dobj,
        cache=True,
        local_only=False,
        preload_fields=None,
        chunk_sizing='auto',
    )
    Docstring: <no docstring>
    Source:   
        def _chunk_io(self, dobj, cache=True, local_only=False,
                      preload_fields=None, chunk_sizing="auto"):
            # local_only is only useful for inline datasets and requires
            # implementation by subclasses.
            if preload_fields is None:
                preload_fields = []
            preload_fields, _ = self._split_fields(preload_fields)
            gfiles = defaultdict(list)
            gobjs = getattr(dobj._current_chunk, "objs", dobj._chunk_info)
            fast_index = dobj._current_chunk._fast_index
            for g in gobjs:
                # Force to be a string because sometimes g.filename is None.
                gfiles[str(g.filename)].append(g)
            # We can apply a heuristic here to make sure we aren't loading too
            # many grids all at once.
            if chunk_sizing == "auto":
                chunk_ngrids = len(gobjs)
                if chunk_ngrids > 0:
                    nproc = np.float(ytcfg.getint("yt", "__global_parallel_size"))
                    chunking_factor = np.ceil(self._grid_chunksize*nproc/chunk_ngrids).astype("int")
                    size = max(self._grid_chunksize//chunking_factor, 1)
                else:
                    size = self._grid_chunksize
            elif chunk_sizing == "config_file":
                size = ytcfg.getint("yt", "chunk_size")
            elif chunk_sizing == "just_one":
                size = 1
            elif chunk_sizing == "old":
                size = self._grid_chunksize
            else:
                raise RuntimeError("%s is an invalid value for the 'chunk_sizing' argument." % chunk_sizing)
            for fn in sorted(gfiles):
                gs = gfiles[fn]
                for grids in (gs[pos:pos + size] for pos
                              in range(0, len(gs), size)):
                    dc = YTDataChunk(dobj, "io", grids,
                            self._count_selection(dobj, grids),
                            cache = cache, fast_index = fast_index)
                    # We allow four full chunks to be included.
                    with self.io.preload(dc, preload_fields, 
                                4.0 * size):
                        yield dc
    File:      ~/yt/yt/yt/geometry/grid_geometry_handler.py
    Type:      method



There's lots going on here, so I'll just grab a few of the most important lines.  Specifically, I want to highlight this:

```python
        for fn in sorted(gfiles):
            gs = gfiles[fn]
            for grids in (gs[pos:pos + size] for pos
                          in range(0, len(gs), size)):
                dc = YTDataChunk(dobj, "io", grids,
                        self._count_selection(dobj, grids),
                        cache = cache, fast_index = fast_index)
                # We allow four full chunks to be included.
                with self.io.preload(dc, preload_fields, 
                            4.0 * size):
                    yield dc
```

The upshot of this is that we first sorted our grids by which file they're in (on the assumption that we probably want to minimize open/close operations), but then in that, we split them up based on the grid counts we want in each chunk, and then we spit out the chunks (with optional preloading of data) to whatever consumes them.

As long as we don't do any preloading it's alright for parallel IO, but we're not going to see any of the benefits of this if we ever turn this into a list.

Now for the Enzo frontend specifically, let's see how the IO routine works:


```python
ds.index.io._read_fluid_selection??
```


    Signature: ds.index.io._read_fluid_selection(chunks, selector, fields, size)
    Docstring: <no docstring>
    Source:   
        def _read_fluid_selection(self, chunks, selector, fields, size):
            # This function has an interesting history.  It previously was mandate
            # to be defined by all of the subclasses.  But, to avoid having to
            # rewrite a whole bunch of IO handlers all at once, and to allow a
            # better abstraction for grid-based frontends, we're now defining it in
            # the base class.
            rv = {}
            nodal_fields = []
            for field in fields:
                finfo = self.ds.field_info[field]
                nodal_flag = finfo.nodal_flag
                if np.any(nodal_flag):
                    num_nodes = 2**sum(nodal_flag)
                    rv[field] = np.empty((size, num_nodes), dtype="=f8")
                    nodal_fields.append(field)
                else:
                    rv[field] = np.empty(size, dtype="=f8")
            ind = {field: 0 for field in fields}
            for field, obj, data in self.io_iter(chunks, fields):
                if data is None:
                    continue
                if isinstance(selector, GridSelector) and field not in nodal_fields:
                    ind[field] += data.size
                    rv[field] = data.copy()
                else:
                    ind[field] += obj.select(selector, data, rv[field], ind[field])
            return rv
    File:      ~/yt/yt/yt/utilities/io_handler.py
    Type:      method



This is in the base class for the IO handler; some of the grid-based frontends implement it.  In this *particular* case we aren't unrolling the generator, but you can see some of the issues here anyway: we need to know a fair bit about the IO method (thus the `io_iter` method, which I will show below) and we need to do a lot of `obj.select` and whatnot.

This isn't terribly efficient, and it *also* means that since we are yielding a generator expression from within a generator expression, we end up having a nested set of loops that don't know their sizes or allow seeking in their stream of yields.

This makes interoperating with something like dask -- which works best when it knows the sizes and shapes and can do its own distribution -- much more challenging.  And it also means that we have a few layers of relatively opaque routines that conspire to keep us a ways from the file-based abstraction.

Let's look at the `io_iter` function to see how it works for Enzo.  You can see that it does do a few fun things; most importantly, it keeps the file handle open if it can.  This can save a surprising amount of time on parallel file systems, as it reduces the number of metadata lookups necessary.


```python
ds.index.io.io_iter??
```


    Signature: ds.index.io.io_iter(chunks, fields)
    Docstring: <no docstring>
    Source:   
        def io_iter(self, chunks, fields):
            h5_dtype = self._field_dtype
            for chunk in chunks:
                fid = None
                filename = -1
                for obj in chunk.objs:
                    if obj.filename is None: continue
                    if obj.filename != filename:
                        # Note one really important thing here: even if we do
                        # implement LRU caching in the _read_obj_field function,
                        # we'll still be doing file opening and whatnot.  This is a
                        # problem, but one we can return to.
                        if fid is not None:
                            fid.close()
                        fid = h5py.h5f.open(b(obj.filename), h5py.h5f.ACC_RDONLY)
                        filename = obj.filename
                    for field in fields:
                        nodal_flag = self.ds.field_info[field].nodal_flag
                        dims = obj.ActiveDimensions[::-1] + nodal_flag[::-1]
                        data = np.empty(dims, dtype=h5_dtype)
                        yield field, obj, self._read_obj_field(
                            obj, field, (fid, data))
            if fid is not None:
                fid.close()
    File:      ~/yt/yt/yt/frontends/enzo/io.py
    Type:      method



So to recap, right now: making chunks work nicely with non-yt operations is tricky because of some early design decisions

## Next Up

In the next blog post, I'm going to present a bit about:

 * How particle IO is handled -- and the differences between grid IO (which has lots of differently-*shaped* chunks) and particle IO
 * Some efforts to refactor particle IO (before it gets released!)
 * A future for how to make all this stuff work better with dask (yes, really, I promise)
