+++
title = "Kaitai Struct and Scientific Data"
subtitle = ""

# Add a summary to display on homepage (optional).
summary = ""

date = 2019-06-20T13:15:41-05:00
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

**tl;dr**: kaitai struct is awesome.

File formats can be pretty annoying -- *especially* when you figure them out through weird combinations of reverse-engineering, hand-me-down code and trial-and-error.

What we've ended up with in yt is a bunch of data formats where the process of conducting the IO is all mixed up with the description of that IO.  This means that any attempt to update things (which I've alluded to in [these](https://matthewturk.github.io/post/refactoring-yt-frontends-part1/) [blog](https://matthewturk.github.io/post/refactoring-yt-frontends-part2/) [posts](https://matthewturk.github.io/post/refactoring-yt-frontends-part3/)) requires a fair bit of care to make sure that the process is not disruptive in any way.

In preparation for this, I set out to start writing down some of the file formats in a flat text format by going through my old notes and the code in the various yt `io.py` files.  Before too long, I decided to instead start utilizing [kaitai struct](kaitai.io) instead.

I've found Kaitai struct to be useful in other projects -- for instance, I have been using it as a method to reverse engineer the data files from an old DOS game I used to play with my brother back in the early 90s.  The combination of the simple (but reasonably flexible) syntax, the ruby-based [visualizer](https://github.com/kaitai-io/kaitai_struct_visualizer) and the [WebIDE](https://ide.kaitai.io) make it really good for exploratory reverse engineering.

And, it generates code!  You can run:

```bash
$ ksc -t python somefile.ksy
```

and it'll generate a python reader.  It can also generate javascript, java, go, etc, and I think rust is in the works.

So the question became: could we use kaitai struct for documenting a data format like, say, the binary format from [GADGET-2](https://wwwmpa.mpa-garching.mpg.de/gadget/)?  After that we can really stretch our legs and try it on more complicated ones, but let's start with this one.

## Gadget-2 Data Format

The (vanilla) Gadget-2 data format is reasonably straightforward, and it's even described in some detail in the Gadget manual.  You have a header, then sets of records.  Each of these sets of records has a separating value (sometimes!).  According to table 3 in the user guide, the file looks something like: 

 * Header
 * Position
 * Velocity
 * ID
 * Mass (only for variable mass particles)
 * Energy (only for gas particles)
 * Density (Only for gas particles)
 * Smoothing length (Only for gas particles)
 * Potential (if enabled in makefile)
 * Acceleration (if enabled in makefile)
 * Endtime (if enabled in makefile)
 * Timestep (if enabled in makefile)

Gadget-2 is probably the most widely-used astrophysics simulation code, but one of its defining characteristics is that it is quite readily hackable to include additional parameters, additional particle attributes, and lots more physics.  This usually means that when you receive binary gadget data, you need to know what to expect in the file -- the individual file formats are typically not self-describing.

The header is often left unchanged, at least in the data I've seen.  In yt we have a simple specification system to allow for modifications to both the fields and the header, but the "vanilla" header looks something like this (in python [struct](https://docs.python.org/3/library/struct.html) notation):

```
default      = (('Npart', 6, 'i'),
                ('Massarr', 6, 'd'),
                ('Time', 1, 'd'),
                ('Redshift', 1, 'd'),
                ('FlagSfr', 1, 'i'),
                ('FlagFeedback', 1, 'i'),
                ('Nall', 6, 'i'),
                ('FlagCooling', 1, 'i'),
                ('NumFiles', 1, 'i'),
                ('BoxSize', 1, 'd'),
                ('Omega0', 1, 'd'),
                ('OmegaLambda', 1, 'd'),
                ('HubbleParam', 1, 'd'),
                ('FlagAge', 1, 'i'),
                ('FlagMetals', 1, 'i'),
                ('NallHW', 6, 'i'),
                ('unused', 16, 'i'))
```

Before we get any further, let's check if we can parse *just* this with kaitai struct.

## Our First ksy File

In its simplest form, Kaitai specifications allow specifying the format of data and the order that different types of data will arrive in.  The base data types it has are standard numerical data types, bytes, strings, arrays and so on.

Kaitai has a little bit of metadata we will add at the beginning, so we start with this preamble:

```yaml
meta:
  id: gadget_format1
  endian: le
```

This gives it a name and notes it as little endian.

If we wanted to build a parser for the header, the most obvious thing we could do would be to parse the different items in order.  The very first item is an array of 6 integers that describes the number of particles in the simulation, separated by each particle type.  This comes in a sequence starting at the beginning of our "stream," so we can use the `seq` top-level type to start parsing:

```yaml
seq:
  - id: recsize_0
    type: u4
  - id: npart1
    type: u4
  - id: npart2
    type: u4
  - id: npart3
    type: u4
  - id: npart4
    type: u4
  - id: npart5
    type: u4
  - id: npart6
    type: u4
```

`recsize_0` here is because we know that our header will be preceded by a value indicating how many bytes are in it -- this is a common practice, but not universal.  And I've said that each of the variables (`npart1` ... `npart6`) is of type `u4`, which means "unsigned four byte integers."

This isn't that efficient, though -- each particle type has its own variable.  KS provides us with the option to `repeat` which can take an expression.  Now we can enable grabbing arrays of values, so we can use that, instead.  Let's change this to:

```yaml
seq:
  - id: recsize_0
    type: u4
  - id: npart
    type: u4
    repeat: expr
    repeat-expr: 6
```

(You can also do `repeat: eos` for end of stream and `repeat-until`.)  The result now is that `npart` is an array -- so later on when we loop, we can look it up by using the special variable `_index` inside another `repeat-expr`.  We'll look at that a bit later.

## Data Types in ksy

Parsing the entire header in this way is certainly possible, but it also can be a bit clunky -- and can be harder for maintainability.  Instead of writing all of our header values out, let's use a user-defined type.

The top-level key `types` is where user-defined types are described; you can do some pretty fancy things like supply parameters to them, but we'll just use it to hold the data we know.

```yaml
types:
  gadget_header:
    seq:
      ...
```

Now, in our top-level `seq` section, we can just reference the header type:

```yaml
seq:
  -id: header
   type: gadget_header
```

## Big Arrays and Lazy-reading

The tricky bit comes in when we get to our arrays of values.  Kaitai has several different languages it can generate for -- Python, C#, C++, ruby, and others.  When reading lots of little items, python can get bogged down.  For instance, this is the most obvious way of describing one of the position arrays:

```yaml
  -id: coordinates_part1
   type: f4
   repeat: expr
   repeat-expr: header.npart[0] * 3
```

The generated python code will read a series of 4-byte objects, one at a time, and append them to a list.  The combination of these things results in a lot of overhead from the python language runtime and the type system, so it ends up being rather slow.

One way to get around this is to use what's called an `instance`.  This allows parsing to happen only when requested, and it can also exist outside of the rest of the parsing structure.

**Important note**: There are some fiddly issues with making `instance` objects work and how they relate to substreams of IO which I'm going to sidestep here.  Let's just assume that it works, instead!

By default, we can read in bytes for these individual objects; this lets us read a big block of data (which we can deal with later -- for instance, inside python!) and only parse into individual floats when we request.  This might look something like defining a custom type with both a `seq` and an `instances` section:

```yaml
types:
  f4_array_type:
    params:
      - id: count
        type: u4
    seq:
      - id: buffer
        size: 4 * count
    instances:
      values:
        pos: 0
        size-eos: true
        id: entries:
        type: f4
        repeat: eos
```

Here we are saying, just read the bytes.  But, if anybody asks, this is what the attribute `values` should look like -- you should parse it starting at 0, go to the end of the stream, and assume it's all `f4`-typed items.  (I am not certain that both `size-eos` and `repeat-eos` are necessary.)

With these components, we should be able to parse our entire gadget binary file, a

## Working Implementation

I've gotten this to the point that it mostly works, and I've been committing to [data-exp-lab/astro-data-formats](https://github.com/data-exp-lab/astro-data-formats).  I ended up splitting the array stuff into its own parameterized type that I store in a separate file, so that we can potentially reuse it in other file formats.

Here's some of the high-level stuff:

```yaml
meta:
  id: gadget_format1
  endian: le
  ks-opaque-types: true
  imports:
    - array_buffer
seq:
  - id: gadget_header
    type: header
  - id: coordinates
    type: particle_fields('f4', 3)
  - id: velocities
    type: particle_fields('f4', 3)
  - id: particle_ids
    type: particle_fields('u4', 1)
types:
  header:
    seq:
      - id: recsize_0
        type: u4
      - id: npart
        type: u4
        repeat: expr
        repeat-expr: 6
      - id: massarr
        type: f8
        repeat: expr
        repeat-expr: 6

[ ... ]

    particle_fields:
    params:
      - id: field_type
        type: str
      - id: components
        type: u1
    seq:
      - id: magic1
        type: u4
      - id: fields
        type: field(_index, components, field_type)
        repeat: expr
        repeat-expr: 6
      - id: magic2
        type: u4
  field:
    params:
      - id: index
        type: u1
      - id: components
        type: u1
      - id: field_type
        type: str
    seq:
      - id: field
        size: _root.gadget_header.npart[index] * components * 4
        type: array_buffer(field_type)
```

And then the `array_buffer` is an instance-based way of getting the raw bytes.  Right now this is reasonably fast, and there's a possibility I could simplify the structure and flatten it out a bit, but it works exactly as I want it to.

## Future

The next thing I'm going to do is work on porting the RAMSES frontend to this format.  That will enable some more exploration and validation of the different outputs, and hopefully use that as guidance for any future modifications to the IO systems.  One of the stickier wickets I've seen is that it's possible to use memory-mapping, but I think there are some subtle places where data is read into memory and copied -- this kind of gets rid of the purpose of the memory mapping.  I hope to explore and dig more deeply into this later.

Ultimately, I believe a combination of ksy files for binary formats and a ksy-*like* dialect for describing the connection between chunks of data and physical space locations will simplify a considerable amount of data analysis.

I'd also like to thank @GreyCat on gitter, who was super helpful in helping me work out some of the bits I wasn't clear on.
