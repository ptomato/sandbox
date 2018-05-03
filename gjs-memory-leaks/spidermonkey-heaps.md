# GJS & SpiderMonkey Garbage Collector Heaps

Generally speaking, SpiderMonkey implements a mark-and-sweep garbage collector,
although it employs a number of strategies including generational collection
(with nursery and tenured regions), incremental collection, arena compacting and
others.

This a brief overview of the structure of SpiderMonkey garbage collector heaps
with some additional notes on GJS particulars. This isn't an exhaustive article
on garbage collection or SpiderMonkey internals, only a reasonably thorough
description of what a heap file looks like and how that relates to JavaScript
and GJS. There is a fairly thorough, and presumably up to date, description of
how the SpiderMonkey garbage collector works in [gc/GC.cpp][gc-cpp].

## The Roots Section

A garbage collector heap is divided into two parts, with the first being the
roots section. Along with a list of the top-level rooted objects it also has a
list of the [WeakMaps][weakmap]'s and [WeakSet][weakset]'s.

```
# Roots.
0x7f028442c060 B self-hosting global
0x7f0284480060 B on-stack compartment global
0x7f0284480060 B GJS global object
# Weak maps.
WeakMapEntry map=(nil) key=0x7f02844e1970 keyDelegate=(nil) value=0x7f02844e19a0
WeakMapEntry map=0x7f02844930c0 key=0x7f0284066c10 keyDelegate=(nil) value=0x7f02840c3cc0
```

### Roots

The roots portion always starts with the string `# Roots.` and each root entry
is made of three parts.

The first is the JavaScript engine's **address**. If you were to coerce a
GObject to its string description you could see that it carries two addresses;
one **jsobj address** and one **native address**. There won't be any GObjects in
the roots section, but this will be important later when we examine a **node**.

```js
gjs> const Gtk = imports.gi.Gtk;
gjs> Gtk.init(null);
gjs> new Gtk.Label();
[object instance proxy GIName:Gtk.Label jsobj@0x7f8bbf685300 native@0x55ca3c97e3b0]
```

The second part is the **color**, which could in principle be *black* (`B`),
*gray* (`G`) or *white* (`W`). This known as [Tri-color marking][tricolor], with
*black* being reachable and thus not collectable, *gray* yet to be scanned for
references to collectable roots and *white* able to be collected.

In practice all entries in the roots section will be *black*, and *gray* roots
could only appear if you were to dump a heap during a garbage collection cycle
between the *mark* and *sweep* phases. If you are dumping a heap using the
`dumpHeap()` function from GJS's System module, you will never see a *gray* root
since JavaScript execution is paused during a collection cycle. You may see them
if you are dumping a heap by sending a `SIGUSR1` to a GJS process with the env
variable `GJS_DEBUG_HEAP_OUTPUT` set, but they will still be very rare.

The third part is the **label**. Root labels are generally types of objects like
`script`, `persistent-Object`, `env chain`, `fp argv[0]` and so on. Each
**node** and **edge** will also have a label, but will usually be something more
descriptive, like a Function name.

### Weak Maps

The weak maps portion always starts with the string `# Weak maps.` and each line
contains the metadata of a WeakMap. Generally these aren't of much interest to
GJS users since SpiderMonkey can handle these just fine and they're pretty
uncommon anyways.

## Section Separator

The end of the roots section (including the roots portion and weak maps portion)
and the beginning of the graph section is always separated by a string of ten
equals characters (`==========`).

## The Graph Section

The second part of the heap contains the graph, which is the bulk of the data.
The graph is mostly comprised of a list of entries, each with a **node**
followed by a list of its **edge** references.

The graph is further annotated with lines describing how SpiderMonkey has
organized the allocations internally including what **zone**, **compartment**
and **arena** the **node** is allocated in.

### Zones, Compartments, Arenas and Cells

```
# zone 0x556629c14b40
# compartment <unknown> [in zone 0x556629c14b40]
# arena allockind=23 size=40
0x7f0284424038 B string <length 18> allowContentSpread
0x7f0284424060 B string <length 18> ArrayBufferSpecies
0x7f0284424088 B string <length 17> ArrayIteratorNext
```

A **zone** is essentially a container that serves as a boundary for objects in
the heap and for the garbage collector to operate in. In other words, a garbage
collection cycle can't cross a **zone** and neither will a **compartment**, an
**arena** or a **cell**.

A **compartment** contains can function as security boundary, although there are
some facilities in the SpiderMonkey API to allow objects to cross compartent
boundaries.

An **arena** is an internal unit of memory allocation (4096 bytes) and will only
contain objects of the same size and kind as indicated by the `allockind` field.
The `size` field seems to be present to make processing easier, as the type and
size are linked and both defined in [AllocKind.h][allockind-h].

A **cell** is the unit of memory that is allocated and collected by the garbage
collector, and is the base class for all object classes (such as JSObject). Each
**node** entry represents an object that is a type of **cell** (an `allockind`).

### Nodes & Edges

While a **node** represents an object (such as an Array, Function or GObject),
each **edge** entry represents a reference the **node** holds to another object,
thus rooting it. If the object a **node** represent were collected and it was
the only reference holder to an object in it's **edge** list, that object would
then be marked *white* (collectable) during the next collection cycle.

```
0x7f028449eca0 B GObject_Object 0x556629ed55f0
> 0x7f02844d83d0 B group
> 0x7f02840daba0 B shape
> 0x7f02844d2b80 B constructor
> 0x7f02844d0eb0 B get_parent
> 0x7f02844d0ee0 B get_path
> 0x7f0284057fd0 B monitor
> 0x7f0284094be0 B get_child
> 0x7f0284094c10 B query_exists
> 0x7f0284012b80 B load_contents
> 0x7f02840a5610 B query_info
```

The first line in the snippet above is the **node**, which in this case is an
instance of a `Gio.File`. Like a **root** entry, it's made of three parts; a
**jsobj address**, a **color** and a **label**. Unfortunately, the GType name is
not part of the label but the **native address** is making it possible to
inspect via `gdb`:

```sh
(gdb) call g_type_name(((ObjectInstance*)0x556629ed55f0)->gtype)
$1 = (const gchar *) 0x7ff7850bed68 "GLocalFile"
```

Following the **node** entry is the **edge** list for this node. Again, each has
three parts; the **jsobj address** of the object the **node** holds a reference
to, the **color** and the **label**. If we were to trace the address of the
line `> 0x7f0284094be0 B get_child`, for example, we would find it points to a
Function **node** elsewhere in the same **compartment**.

## Further Reading

* [MDN: *Garbage collection*](https://developer.mozilla.org/docs/Mozilla/Projects/SpiderMonkey/Internals/Garbage_collection)
* [MDN: *GC and CC logs*](https://developer.mozilla.org/docs/Mozilla/Performance/GC_and_CC_logs)
* [MDN: *Memory Management*](https://developer.mozilla.org/docs/Web/JavaScript/Memory_Management)
* [Mozilla JavaScript Blog: *Incremental GC in Firefox 16!*](https://blog.mozilla.org/javascript/2012/08/28/incremental-gc-in-firefox-16/)
* [Mozilla Hacks: *Generational Garbage Collection in Firefox*](https://hacks.mozilla.org/2014/09/generational-garbage-collection-in-firefox/)
* [Mozilla Hack: *Compacting Garbage Collection in SpiderMonkey*](https://hacks.mozilla.org/2015/07/compacting-garbage-collection-in-spidermonkey/)

[allockind-h]: https://dxr.mozilla.org/mozilla-central/source/js/src/gc/AllocKind.h
[gc-cpp]: https://dxr.mozilla.org/mozilla-central/source/js/src/gc/GC.cpp
[tricolor]: https://en.wikipedia.org/wiki/Tracing_garbage_collection#Tri-color_marking
[weakmap]: https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/WeakMap
[weakset]: https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/WeakSet

