+++
title = "A Little Bit of Endianness"
subtitle = ""

# Add a summary to display on homepage (optional).
summary = ""

date = 2020-01-23T16:28:15-06:00
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
# projects = []

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder. 
[image]
  # Caption (optional)
  caption = ""

  # Focal point (optional)
  # Options: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight
  focal_point = ""
+++

(Confession time: I did not even realize I was mildly-punning in that title.)

Over the last few months, my hobby-time has mostly been focused on some reverse engineering of old DOS games I played as a kid.  The fruits of that will wait for another blog post, but this has led to some very fun investigations using the [DOSbox debugger](https://www.vogons.org/viewtopic.php?t=3944), [Ghidra](http://ghidra-sre.org/) and learning the Ghidra API to decompile MS-DOS system calls (especially [int 21h](http://bbc.nvg.org/doc/Master%20512%20Technical%20Guide/m512techb_int21.htm)) properly.

A particularly fun aspect of doing all this, though, has been realizing just how much this has in common with the early stages of developing yt, when we had to reverse engineer file formats like it was going out of style.

One of the things that constantly *constantly* showed up in reverse engineering these formats was making sure our endianness was right.  Since we were often dealing with 32-bit values that had uncertain units, and where the values themselves could be quite the extreme and not *too* far from the limits of representation, it wasn't always immediately obvious if we had it right or not.

To be completely honest, the thing that always threw me (and sometimes still does, when I'm thinking about ordering of numbers on the stack and in registers) is how numbers get represented as bits in memory, and more specifically, what the difference between "big" and "little" endian really is.

Also, for a much better description and walkthrough, please feel free to visit the [wikipedia article](https://en.wikipedia.org/wiki/Endianness).  Its load time is probably comparable to the reading time of this rather brief foray into views.


```python
import numpy as np
```

To get started, let's make an array that isn't symmetric at the "word" level.  I'll use my favorite value offset by one to do this.  (Remember for later that we're using a number as our base that is greater than can be represented in two bytes, so it truncates at `u2` size.)

We'll use numpy's ability to encode arrays using different internal representations -- the first array will be in "little" endian, 2-byte unsigned integers, and the second will be in "big" endian, 2-byte integers.  We toggle between the two with the `<` and `>` operators before the data type specification.


```python
arr_little = np.array([0x4d3d3 - 1]).astype("<u2")
arr_big = arr_little.astype(">u2")
arr_big, arr_little
```




    (array([54226], dtype=uint16), array([54226], dtype=uint16))



So, they're definitely the same values when we print them out!  But that's because even though they are encoded differently internally, when we print them, we use a representation that is encoding aware.  What happens if we force the issue, though?  Let's try, by asking to view them in "unsigned bytes" format by using the `view` method with the argument `B` to indicate the dtype.


```python
print("Little endian: {}\nBig endian:    {}".format(arr_little.view("B"), arr_big.view("B")))
```

    Little endian: [210 211]
    Big endian:    [211 210]


Interesting!  Note that the individual bytes are the same (and there are two in each one), but the order is different.

We can make this even more clear if we look at the binary and hex representations of them:


```python
print("Little endian: {}\nBig endian:    {}".format(*[[bin(_)[2:] for _ in arr.view("B")] for arr in (arr_little, arr_big)]))
```

    Little endian: ['11010010', '11010011']
    Big endian:    ['11010011', '11010010']



```python
print("Little endian: {}\nBig endian:    {}".format(*[[hex(_)[2:].upper() for _ in arr.view("B")] for arr in (arr_little, arr_big)]))
```

    Little endian: ['D2', 'D3']
    Big endian:    ['D3', 'D2']


What I always found the most difficult about endianness is that the **bits** aren't actually reversed -- what is reversed is the order of the bytes, but within a given byte the bits remain the same.  This gets even more fun with 32 and 64 bit integers.


```python
arr_little32 = np.array([0x4d3d3 - 1]).astype("<u4")
arr_big32 = arr_little32.astype(">u4")
arr_big32, arr_little32
```




    (array([316370], dtype=uint32), array([316370], dtype=uint32))




```python
print("Little endian: {}\nBig endian:    {}".format(*[[hex(_)[2:].upper() for _ in arr.view("B")] for arr in (arr_little32, arr_big32)]))
```

    Little endian: ['D2', 'D3', '4', '0']
    Big endian:    ['0', '4', 'D3', 'D2']


Of course, Wikipedia has an [awesome article](https://en.wikipedia.org/wiki/Endianness) with a great [illustration](https://en.wikipedia.org/wiki/Endianness#Illustration) that demonstrates all of this.
