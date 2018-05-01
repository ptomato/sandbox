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

### Cyclic References

Cyclic references are well-known way to cause memory leaks, or uncollectable objects, in JavaScript. Here's some quick examples of how that might play out in GJS with GObject signals:


```js
//------------------------------------------------------------------------------
function connectTest(button) {
    // CYCLIC REFERENCE
    // myVariable will hold reference to `button` from outside its scope
    button.connect('clicked', (widget) => {
        let myVariable = button; // Cyclic Reference
    });
 
    // NOT CYCLIC REFERENCE
    // Even though we're referencing the same object (address), we're not
    // holding a reference to `button`
    button.connect('clicked', (widget) => {
        let myVariable = widget;
    });
   
    // NOT CYCLIC REFERENCE
    // Because `button` is re-defined in our arrow function's arguments, we're
    // not holding a reference to `button`, we're using `@button`.
    button.connect('clicked', (button) => {
        let myVariable = button;
    });
 
    // CYCLIC REFERENCE
    // Using a regular anonymous function won't make a difference here.
    button.connect('clicked', function(widget) {
        let myVariable = button;
    });
}
 
let button = new Gtk.Button({ label: 'Button' });
connectTest(button);
```


### Snippets to sort out

Considering the example from https://gitlab.gnome.org/GNOME/gjs/blob/master/installed-tests/js/testEverythingEncapsulated.js#L277

```js
describe('Garbage collection of introspected objects', function () {
    // This tests a regression that would very rarely crash, but
    // when run under valgrind this code would show use-after-free.
    it('collects objects properly with signals connected', function (done) {
        function orphanObject() {
            let obj = new Regress.TestObj();
            obj.connect('notify', () => {});
        }
        orphanObject();
        System.gc();
        GLib.idle_add(GLib.PRIORITY_LOW, () => done());
    });
});
```

> **Philip Chimento**

> in this case the `this` is a red herring
> it would be a reference cycle if it was `function () {}` as well

> so, here, `() => {}` traces `orphanObject`'s lexical environment, which traces `obj`

> and `obj` traces `() => {}` because it's a signal connection

> but nothing else traces `obj` because the interpreter stops tracing, or else unroots, the lexical environment when `orphanObject` is done executing (I'm a bit shaky on this part)

> so the whole cycle can be collected

> if `let obj` were placed outside of `orphanObject`, then it wouldn't be able to be collected


## References

* https://auth0.com/blog/four-types-of-leaks-in-your-javascript-code-and-how-to-get-rid-of-them/
* https://blog.sessionstack.com/how-javascript-works-memory-management-how-to-handle-4-common-memory-leaks-3f28b94cfbec
* [Mozilla Hacks - *ES6 In Depth: Arrow functions*](https://hacks.mozilla.org/2015/06/es6-in-depth-arrow-functions/)


[wiki]: https://gitlab.gnome.org/GNOME/gjs/wikis/home
[heapgraph-scripts]: https://gitlab.gnome.org/GNOME/gjs/tree/master/tools
[system-module]: https://gitlab.gnome.org/GNOME/gjs/wikis/Modules#system
