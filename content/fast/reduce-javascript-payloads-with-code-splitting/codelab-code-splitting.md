---
page_type: glitch
title: Reduce JavaScript payloads with code-splitting
author: houssein
description: |
  In this codelab, learn how to improve the performance of a simple application
  through code-splitting.
web_updated_on: 2018-12-06
web_published_on: 2018-11-05
glitch: code-splitting-starter
---

Most web pages and applications are made up of many different parts. Instead of
sending all the JavaScript that makes up the application as soon as the first
page is loaded, **code-splitting** a bundle into multiple "pieces" (or chunks)
improves page performance.

In this codelab, the performance of a simple application that sorts three
numbers is optimized through code-splitting.

![image](./codelab-code-splitting-1.png)

## Measure

Like always, it's important to first measure how well a website performs before
attempting to add any optimizations.

-  Click on the **Show Live** button.

<web-screenshot type="show-live"></web-screenshot>

-  Open the DevTools by pressing `CMD + OPTION + i` / `CTRL + SHIFT + i`.
-  Click on the **Network** panel.
-  Make sure `Disable Cache` is checked and reload the app.

<img class="screenshot" src="./codelab-code-splitting-3.png" alt="Network panel showing 71.2 KB JavaScript bundle.">

71.2 KB worth of JavaScript just to sort a few numbers in a simple application.
What gives?

In the source code (`src/index.js`), the `lodash` library is imported and used
in this application. [Lodash](https://lodash.com/) provides many useful utility
functions, but only a single method from the package is being used here.
Installing and importing entire third-party dependencies where only a small
portion of it is being utilized is a common mistake.

## Optimize

There are a few ways the bundle size can be trimmed:

1.  Write a custom sorting method instead of importing a third-party library
2.  Use the built in `Array.prototype.sort()` method to sort numerically
3.  Only import the `sortBy` method from `lodash` and not the entire library
4.  Download the code for sorting only when the user clicks the button

Options 1 and 2 are perfectly appropriate methods to reduce the bundle size (and
would probably make the most sense for a real application). However, those are
not used in this tutorial for the sake of teaching 😈.

<div class="aside note">
The concept of removing unused code is explored in further detail in a
<a href="/fast/remove-unused-code">separate guide</a>.
</div>

Both options 3 and 4 help improve the performance of this application. The
next few sections of this codelab cover these steps. Like any coding
tutorial, always try to write the code yourself instead of copy and pasting.

### Only import what you need

A few files need to be modified to only import the single method from `lodash`.
To begin with, replace this dependency in `package.json`:

```
"lodash": "^4.7.0",
```

with this:

```
"lodash.sortby": "^4.7.0",
```

Now in `src/index.js`, import this specific module:

<pre class="prettyprint">
import "./style.css";
<s>import _ from "lodash";</s>
<strong>import sortBy from "lodash.sortby";</strong>
</pre>

And update how the values are sorted::

<pre class="prettyprint">
form.addEventListener("submit", e => {
  e.preventDefault();
  const values = [input1.valueAsNumber, input2.valueAsNumber, input3.valueAsNumber];
  <s>const sortedValues = _.sortBy(values);</s>
  <strong>const sortedValues = sortBy(values);</strong>

  results.innerHTML = `
    &lt;h2&gt;
      ${sortedValues}
    &lt;/h2&gt;
  `
});
</pre>

Reload the application, open DevTools, and take a look at the **Network** panel
once again.

<img class="screenshot" src="./codelab-code-splitting-4.png" alt="Network panel showing 15.2 KB JavaScript bundle.">

For this application, the bundle size was reduced by over 4X with very little
work, but there's still more room for improvement.

### Code splitting

[**webpack**](https://webpack.js.org/) is one of the most popular open-source
module bundlers used today. In short, it _bundles_ all JavaScript modules (as
well as other assets) that make up a web application into static files that can
be read by the browser.

The single bundle used in this application can be split into two separate
chunks:

- One responsible for the code that makes up our initial route
- A secondary chunk that contains our sorting code

With the use of **dynamic imports**, a secondary chunk can be _lazy loaded,_ or
loaded on demand. In this application, the code that makes up the chunk can be
loaded only when the user presses the button.

Begin by removing the top-level import for the sort method in `src/index.js`:

<pre class="prettyprint">
<s>import sortBy from "lodash.sortby";</s>
</pre>

And import it within the event listener that fires when the button is pressed:

<pre class="prettyprint">
form.addEventListener("submit", e => {
  e.preventDefault();
  <strong>import('lodash.sortby')</strong>
    <strong>.then(module => module.default)</strong>
    <strong>.then(sortInput())</strong>
    <strong>.catch(err => { alert(err) });</strong>
});
</pre>

The `import()` feature is part of a
[proposal](https://github.com/tc39/proposal-dynamic-import) (currently at stage
3 of the TC39 process) to include the capability to dynamically import a module.
webpack has already included support for this and follows the same syntax laid
out by the proposal.

<div class="aside note">
Read more about how
dynamic imports work in this
<a href="https://developers.google.com/web/updates/2017/11/dynamic-import">Web Updates article</a>.
</div>


`import()` returns a promise and when it resolves, the selected
module is provided which is split out into a separate chunk. After the module is
returned, `module.default` is used to reference the default
export provided by lodash. The promise is chained with another `.then` that
calls a `sortInput` method to sort the three input values. At the end of the
promise chain, .`catch()` is used to handle cases where the promise is rejected
due to an error.

<div class="aside caution">
In a production application, always handle dynamic import
errors appropriately. A simple alert message similar to what is used here may
not provide the best user experience to let the user know something has
failed.
</div>

<div class="aside note">
You may see a linting error that says <code>Parsing error: 'import' and
'export' may only appear at the top level.</code> This is due to the fact that
the dynamic import syntax is still in the proposal stage and has not been
finalized. Although webpack already supports it, the settings for
<a href="https://eslint.org/">ESLint</a> (a JavaScript linting utility) used by
Glitch has not been updated to include this syntax yet, but it still works!
</div>

The last thing that needs to be done is to write the `sortInput` method at the
end of the file. This needs to be a function that _returns_ a function that
takes in the imported method from `lodash.sortBy`. The nested function can then
sort the three input values and update the DOM.

```
const sortInput = () => {
  return (sortBy) => {
    const values = [
      input1.valueAsNumber,
      input2.valueAsNumber,
      input3.valueAsNumber
    ];
    const sortedValues = sortBy(values);

    results.innerHTML = `
      <h2>
        ${sortedValues}
      </h2>
    `
  };
}
```

## Monitor

Reload the application one last time and keep a close eye on the **Network**
panel again. Only a small initial bundle is downloaded as soon as the app
loads.

<img class="screenshot" src="./codelab-code-splitting-5.png" alt="Network panel showing 2.7 KB JavaScript bundle.">

After the button is pressed to sort the input numbers, the chunk that contains
the sorting code gets fetched and executed.

<img class="screenshot" src="./codelab-code-splitting-6.png" alt="Network panel showing 2.7 KB JavaScript bundle followed by a 13.9 KB JavaScript bundle.">

Notice how the numbers still get sorted!

## Conclusion

Code splitting and lazy loading can be extremely useful techniques to trim down
the initial bundle size of your application, and this can directly result in
much faster page load times. However, there are some important things that need
to be considered before including this optimization in your application.

### Lazy loading UI

When lazy loading specific modules of code, it's important to consider how the
experience would be for users with weaker network connections. Splitting and
loading a very large chunk of code when a user submits an action can make it
seem like the application may have stopped working, so consider showing a
loading indicator of some sort.

### Lazy loading third-party node modules

It is not always the best approach to lazy load third-party dependencies in your
application and it depends on where you use them. Usually, third party
dependencies are split into a separate `vendor` bundle that can be cached since
they don't update as often. Read more about how the
[SplitChunksPlugin](https://webpack.js.org/plugins/split-chunks-plugin/) can
help you do this.

### Lazy loading with a JavaScript framework

Many popular frameworks and libraries that use webpack provide abstractions to
make lazy loading easier than using dynamic imports in the middle of your
application.

+  [Lazy loading modules with Angular](https://angular.io/guide/lazy-loading-ngmodules)
+  [Code splitting with React Router](https://reacttraining.com/react-router/web/guides/code-splitting)
+  [Lazy loading with Vue Router](https://router.vuejs.org/guide/advanced/lazy-loading.html)

Although it is useful to understand how dynamic imports work, always use the
method recommended by your framework/library to lazy load specific modules.

### Preloading and prefetching

Where possible, take advantage of browser hints such as `<link rel="preload">`
or `<link rel="prefetch">` in order to try and load critical modules even
sooner. webpack supports both hints through the use of magic comments in import
statements. This is explained in more detail in the
[Preload critical chunks](/fast/preload-critical-assets) guide.

### Lazy loading more than code

Images can make up a significant part of an application. Lazy loading those that
are below the fold, or outside the device viewport, can speed up a website. Read
more about this in the
[Lazysizes](/fast/use-lazysizes-to-lazyload-images) guide.
