# SystemJS HMR
Hot Module Replacement for SystemJS

SystemJS HMR extends SystemJS with a ```System.reload``` function and overrides ```System.delete``` to introduce plugin unload behaviours.

##Goal
The goal of this project is to implement HMR primitives for SystemJS that can be battle tested and later be added to the core project.
***SystemJS HMR*** is meant to be used as an HMR enabler for library creators rather then providing a full HMR experience
for application developers, if you're looking to implement HMR in your own project take a look at
[capaj/systemjs-hot-reloader](https://github.com/capaj/systemjs-hot-reloader).

We want to introduce a minimal API change to SystemJS and build in such a fashion as to enable smooth assimilation into core further down the line.
This project will only implement the logic required to enable HMR,
and as such notifying ***SystemJS HMR*** of file changes is left to the library/application implementer.

## Motivation
Integrating HMR into core will add to the tooling capacity of the project and allow for interesting integrations.
While [capaj/systemjs-hot-reloader](https://github.com/capaj/systemjs-hot-reloader) is great at an application level,
it forces you to use its own events API which is problematic for anyone trying to create tooling around SystemJS & HMR.
And while it is possible to implement your own HMR logic void of an events system, HMR is a straight forward enough problem
(at least where SystemJS is concerned) that this logic could be appropriately included into the core project.
The lack of supporting tooling built around SystemJS is one of the core reasons SystemJS feels 'hard'
for new users and proper HMR support would go a long way to increase developer interest in the project.

## Loader Plugin Unload Hook

In a traditional application one does not usually have to deal with modules being deleted/unloaded, naturally HMR requires
us to start thinking about what happens when we unload a module in-order to replace it. Now unloading ```js``` is naturally
different then say to ```css```. With ```javascript``` we need to let the module itself handle some of the unloading
(unsubscribing from event listeners, unmounting dom nodes, etc) and then delete it from the SystemJS registry.
With ```css``` however, we simply need to delete the *link* node from the DOM.

Evidently this is a plugin level decision. As such we augment the ```System.delete``` function with a call to the plugins unload
hook, providing it with the module name, before calling the original ```System.delete``` function. [This needs thought]

This ```unload``` extension ends up cleanly catering for the general case where a module is forcefully deleted as well.

### JS Plugin Unload Proposal

Any JS module can optionally export an ```__unload``` function, which will be triggered by the js loader.

module.js
```js
...

export function __unload() {
    unsubscribe()
    ...
}
```

### CSS Plugin Unload Proposal

The css ```unload``` hook would simply remove the *link* node from the DOM.

This should solve the following issues:
- https://github.com/systemjs/plugin-css/issues/81
- https://github.com/capaj/systemjs-hot-reloader/issues/37.

Providing us with true CSS/SCSS reloading.

## Loader Plugin Reload Hook

When a new version of a module is imported it will probably want to reinitialize its own state based on the state of the
previous version. For example a component containing the client application state might want to perform the following operation.

```js
if (isReload) {
    this.state = oldModule.state
}
```

to account for this, and the fact that future plugins might want to be able to handle their own reload behaviour. We call an
optionally exported plugin reload hook.

### JS Plugin Reload Proposal #1
I can see three ways of implementing the reload hook for JS.
The first would be the way [capaj/systemjs-hot-reloader](https://github.com/capaj/systemjs-hot-reloader) currently does it,
which is to export a ```__reload(module)``` function from each module.

In this case, once the module is reloaded, the exported ```__reload``` function is run and additional init logic can be placed here.
Since ES6 modules are live, you could override exports, and since ```__reload``` would be run before any of the modules dependants, this would
be transparent as far as the dependants are concerned.

**Pros**
- Clear separation between reload behaviour and standard module initialisation
- Potentially simplest reload hook implementation out of #1, #2 and #3

**Cons**
- Harder to implement more complex reload behaviour (need to abstract init code into functions that can be rerun from ```__reload```)
- Not clear when ```__reload``` is actually run (I've seen people duplicate module init behaviour in ```__reload``` function, so init code gets run twice).
- Different to webpack

### JS Plugin Reload Proposal #2
Identical to proposal #1, however standard module init code is not run, only ```__reload``` function, essentially acting as an
alternate module constructor.

**Pros**
- Clear separation between reload behaviour and standard module initialisation
- ```__reload``` is now clearly init code
- How webpack does this (I think?) and so will be familiar for people moving over.

**Cons**
- Harder to implement more complex reload behaviour (need to abstract init code into functions that can be rerun from ```__reload```)
- Actually don't think this is even possible in SystemJS currently.

### JS Plugin Reload Proposal #3
\#1 and #2 work well until you want to incorporate more complex reload logic into your standard module initialisation code.

This approach would be to pass in the old module version instance as a 'local/global' variable directly available to the module.
This would allow for more complex handling of reload logic when required.

Example
```js
...

let foo = () => {};

if (module.previous) {
    foo = () => true
}

export {foo}

...
```

Cleaning up reload behaviour for production (if required) could be easily handled via dead code removal
(via JSPM's ``` bundle --production ``` flag?).

This would be my personal preference.

**Pros**
- Super clear what is init code and what isn't
- Easy to implement more complex reload behaviour
- Kinda like how webpack does this and so aspects will be familiar for people moving over (closer then #1)

**Cons**
- Blurs the lines between reload behaviour and standard module initialisation... kinda
- Actually don't think this is even possible in SystemJS currently
- Not actually how webpack does this...

## Reload API

```js
SystemJS.reload(moduleName, moduleSource)
```
Where
- moduleName is a String of the same kind you would provide to ```System.import``` if importing a module.
- moduleSource is an optional String parameter containing the source code of the module that SystemJS would fetch if
```System.import(moduleName)``` was called.

The ```reload``` function recursively calls the plugin loader unload hook and deletes the module, and all modules that depend on it, directly or indirectly.
It then reimports the root of the deleted dependency tree (which will first load it's missing dependencies),
and runs the plugin loader reload hooks of each imported module.