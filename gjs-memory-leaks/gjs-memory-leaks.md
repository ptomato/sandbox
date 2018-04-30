# How To Cause Memory Leaks in GJS

This is document describes ways to reliably create memory in [GJS][wiki]. Some may not be specific to GJS and may also apply to JavaScript in general. At the time of writing they are tested in 1.52.

The tool we're going to use to examine the memory heap of the GJS process is from the [`heapgraph`][heapgraph-scripts] scripts. We'll just be using `heapgraph.py` which outputs a text tree, so you can just grab that one.

We'll also be using some of the functions provided by the built-in [`system`][system-module] module, to generate heap dumps and trigger garbage collection sweeps.

## Callbacks

Here's our test rig:

```js
'use strict';

const GLib = imports.gi.GLib;
const GObject = imports.gi.GObject;
const Gtk = imports.gi.Gtk;
const System = imports.system;


// Cleanup before each run. This is important because System.dumpHeap() doesn't
// overwrite (it appends) and that confuses `heapgraph.py`
GLib.unlink('gjs.1.heap');
GLib.unlink('gjs.2.heap');
GLib.unlink('gjs.3.heap');


Gtk.init(null);

let win = new Gtk.Window({
    default_height: 300,
    default_width: 300,
    title: 'Callbacks'
});

let button = new Gtk.Button({
    label: 'A Button',
    halign: Gtk.Align.CENTER,
    valign: Gtk.Align.CENTER
});

win.add(button);

//------------------------------------------------------------------------------
// This is where the snippets will go in our test rig
//------------------------------------------------------------------------------

win.show_all();
Gtk.main();
```

## References

* https://auth0.com/blog/four-types-of-leaks-in-your-javascript-code-and-how-to-get-rid-of-them/
* https://blog.sessionstack.com/how-javascript-works-memory-management-how-to-handle-4-common-memory-leaks-3f28b94cfbec


[wiki]: https://gitlab.gnome.org/GNOME/gjs/wikis/home
[heapgraph-scripts]: https://gitlab.gnome.org/GNOME/gjs/tree/master/tools
[system-module]: https://gitlab.gnome.org/GNOME/gjs/wikis/Modules#system
