# GJS & SpiderMonkey Garbage Collector Heaps

Generally speaking, SpiderMonkey implements a mark-and-sweep garbage collector,
although it employs a number of strategies including generational collection
(with nursery and tenured regions), incremental collection, arena compacting and
others.

This a brief overview of the structure of SpiderMonkey garbage collector heaps
with some additional notes on GJS particulars. Heaps are first organized into
two sections; the roots and the graph.

## Roots Section

The roots section contains a list of the top-level rooted objects and
rooted objects and `WeakMap`'s (possibly also `WeakSet`'s).

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

The second part is the root **color**, which could in principle be black (`B`),
gray (`G`) or white (`W`). In practice all entries in the roots section will be
black, and gray roots could only appear if you were to dump a heap during a
garbage collection cycle between the *mark* and *sweep* phases. If you are
dumping a heap using the `dumpHeap()` function from GJS's System module, you
will never see gray roots since JavaScript execution is paused during a
collection cycle. If you are dumping a heap by sending a `SIGUSR1` to a process
with the env variable `GJS_DEBUG_HEAP_OUTPUT` set, you may see gray roots, but
they will still be very rare.

The third part is the root label. Root labels are generally types of objects
like `script`, `persistent-Object`, `env chain`, `fp argv[0]` and so on. Each
**node** and **edge** will also have a label and are usually more descriptive,
such as a Function name.

### Weak Maps

The weak maps portion always starts with the string `# Weak maps.` and each line
contains the metadata of a WeakMap. Generally these aren't of much interest to
GJS users since SpiderMonkey can handle these just fine and they're pretty
uncommon anyways.

### End of the Roots Section

The end of the roots section (including the roots portion and weak maps portion)
is always marked by the string `==========` (10 equals characters).

## The Graph

The graph section of a heap contains the bulk of the data in a heap and is
mostly comprised of a list of entries, each with a **node** followed by a list
of its **edge** references. The graph is organized by lines describing how
SpiderMonkey has organized the allocations internally including what **zone**,
**compartment** and **arena** the **node** is allocated in.

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

A **compartment** contains arenas and can function as security boundary,
although there are some facilities in the SpiderMonkey API to allow objects to
cross compartent boundaries. They may also allow compartment specific garbage
collection, but it's unclear from the scattered documention the *current* place
of compartments in how the garbage collector operates.

An **arena** is an internal unit of memory allocation (4096 bytes) and only
contains objects of the same size and kind as defined by the `allockind` field.
The `size` field seems to be present to make processing easier, as the type and
size are linked and both defined in [AllocKind.h](allockind).

A **cell** is the unit of memory that is allocated and collected by the GC, and
is the base class for all classes such as JSObject.

### Nodes & Edges

A **node** represents an object (such as Arrays, Functions and string) and an
**edge** represents a reference held between those objects. Each **node** is a
type of **cell** (an `allockind`) that can only be allocated in an **arena**
specified to hold that type of `allockind`, however a **node** can be in more
than one **arena**.

Some **node** types contain actual data, like strings or numbers

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

The first line in the snippet above is the node, which in this case is a
GObject.

Like a root, a node has three parts; an address, a color and a label.
Unfortunately, the GType is not part of the label but the native address is
which makes it possible to inspect via `gdb`.

> The following gdb snippet was not actually performed on this object

> Printing out the `ObjectInstance` for an address

```sh
(gdb) print *((ObjectInstance*)0x5590ae10f560)
$1 = {info = 0x5590ae416140, gobj = 0x0, keep_alive = {m_rooted = false, m_has_weakref = false, m_cx = 0x0, m_heap = {<js::HeapBase<JSObject*>> = {<No data fields>}, ptr = 0x0},
    m_root = 0x0, m_notify = 0x0, m_data = 0x0}, gtype = 94079882365632, signals = std::set with 0 elements, klass = 0x5590addad800, vfuncs = std::deque with 0 elements,
  js_object_finalized = 0}
```

> Calling `g_type_name()` on the gtype member of the ObjectInstance

```sh
(gdb) call g_type_name(((ObjectInstance*)0x5590ae10f560)->gtype)
$2 = (const gchar *) 0x7ff7850bed68 "GtkLabel"
```

Following the **node** entry are the edges for this node. Again, each has three
parts; the address of the object the **edge** forms a reference with, the color
and the label. If you're familiar with the Gnome API you have probably discerned
that this is a `Gio.File` object. If we were to trace the address of, for
example, `> 0x7f0284094be0 B get_child` we would find it points to a `Function`
**node** (not the child itself) elsewhere in the same **compartment**.

## References

* [MDN: *GC and CC logs*](https://developer.mozilla.org/docs/Mozilla/Performance/GC_and_CC_logs)
* [MDN: *Garbage collection*](https://developer.mozilla.org/docs/Mozilla/Projects/SpiderMonkey/Internals/Garbage_collection)
* [Mozilla JavaScript Blog: *Incremental GC in Firefox 16!*](https://blog.mozilla.org/javascript/2012/08/28/incremental-gc-in-firefox-16/)
* [Mozilla Hacks: *Generational Garbage Collection in Firefox*](https://hacks.mozilla.org/2014/09/generational-garbage-collection-in-firefox/)
* [Mozilla Hack: *Compacting Garbage Collection in SpiderMonkey*](https://hacks.mozilla.org/2015/07/compacting-garbage-collection-in-spidermonkey/)

[allockind]: https://dxr.mozilla.org/mozilla-central/source/js/src/gc/AllocKind.h

