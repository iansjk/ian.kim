---
layout: "../../layouts/BlogPost.astro"
title: "“How did we get here?”–A primer on JavaScript build systems"
description: "An attempt to explain why we have so many JavaScript build tools now, and what problems they help solve."
pubDate: "Aug 28 2022"
tags: ["js"]
---

Modern frontend development has a great deal of build complexity. If you’ve worked with a framework like React, it’s likely that you’ve had to transform your JavaScript files in one way or another before you’re able to run them in the browser. Maybe you’ve already been through the pain of configuring Webpack or Babel. What happened to writing some JavaScript, including it with a `<script>` tag, and being done with it?

I think it’s helpful to think of what we’re trying to achieve in order to understand how frontend build chains have grown more complex.

## Our wishlist

So, what _do_ we want?

- **Cross-browser support**–We want JavaScript that works consistently across different browsers.
- **Future features**–We want to use the latest JavaScript features, even if not every browser that we want to support has implemented them.
- **Performance**–We want a website that is fast. One important aspect of this is delivering as small an amount of JavaScript over the network as possible.[1]
- **Modularity/reusability**–We want to be able to easily use JavaScript code that other people have made published to NPM, or reuse our own JavaScript code.
- **Consistency**–We want our code to be resilient no matter which packages we use or what order we import them in.
- **Static typing**–We want to use TypeScript so that we can add static typing to JavaScript, which is a dynamically typed language.
- **Hot module reloading**–While we’re developing our site, if we make a change to a file we’d our browser to update immediately with the change.

That’s quite a lot! And to be clear, not everyone will want all of these things–there are trade-offs, as with anything in software. But people have made various tools to address these desires over the years that have become part of the frontend build system.

### Cross-browser support and future features

I’m going to group these together, because the overall aim is similar: we’re trying to use a certain set of JavaScript features, and want them to run the same way on a variety of browsers.

We achieve this by _transpilation_–changing the original JavaScript code that not every browser understands into different JavaScript code that _is_ universally understood.

The first transpiler was Babel in 2014, which is still commonly used today. Transpilers take a JavaScript file as input and give you another JavaScript file as output–one file in, one file out.

### Performance

One important consideration is minimizing the number of network requests. Rather than requesting 500 separate tiny JavaScript files, it would be better to request just one larger file.[2] This is called _bundling._ Unlike transpilation, we put many files into the bundler, and get one file out.

Another consideration is the number of bytes required to send the file. Although programmers add a lot of whitespace and comments to aid development, these can be removed from the code without changing its results. Similarly, longer variable names can be renamed to shorter ones to save on size. This is called _minification._ We put one file into the minifier, and we get one smaller file out.

Finally, we might want to use a particular library, but just a small piece of it. If that library is very large and we include it with a `<script>` tag, the _entire library_ will be downloaded and evaluated–most of which will be wasted effort, since we’re actually only using a small part of it. What we really want is to include the part that we’re actually using. This is called _tree-shaking_.

### Modularity, reusability, consistency

In the earlier days of the Web, when you wanted to use a library, you needed to add a `<script>` tag referencing that library with a URL–there was no equivalent to `import some-package` like in Python, because JavaScript had no module system.

There were a few downsides of this approach:

- `<script>`s were downloaded and executed eagerly, so if you placed them higher up in the page, you’d block the rest of the page from downloading and displaying until the script finished.[3]
- Order mattered: if you had a plugin that only worked with a library, but accidentally put the plugin’s `<script>` tag before the library’s `<script>` tag, you’d likely run into a script error (since the library that the plugin is trying to attach to doesn’t exist yet).
- Scripts could affect other scripts: there was only one global scope (`window`), so importing one script could have an effect on another script without your knowledge.

Because there was no native module system in JavaScript, some folks banded together to create an unofficial one–CommonJS. This is still the most common module system today, thanks to Node and NPM popularizing it. The downside is that CommonJS’s syntax isn’t supported by a browser, so it must be _transpiled_ first before a browser can understand it.

With EcmaScript 2015 came EcmaScript Modules (ESM), which finally added a native module system to JavaScript. However, by this time there was already a large amount of JavaScript code that had been published to NPM in CommonJS, since no other module system was available.

The consequence is that we have two different module systems. CommonJS has a large amount of published code, but isn’t understood by browsers. ESM is natively understood by modern browsers, but can’t easily work with existing CommonJS code. As mentioned before, one must be _transpiled_ into the other.

### Static typing

JavaScript is a dynamically typed language, and its type coercion rules have been the subject of many memes for how surprising they can be.

TypeScript allows us to add optional type hints to existing JavaScript code. Especially on larger projects, this aids development in adding additional safeguards before you run your code–for example, if you try to access an object property that you haven’t declared on its type, you’ll know ahead of time in your code editor rather than having it throw an error at runtime.

TypeScript syntax is not understood by browsers, so the type hints must be removed before you can run your TypeScript code in a browser.[4] Removing the type hints is another case of _transpilation_.

In addition, if you’re working on a library, other people generally like it when you add types to your library code, since it makes your library easier to use. This requires that you publish type definitions along with your final JavaScript code; these _type definition files_ are generated by the TypeScript compiler, `tsc`.

### Hot module reloading

When developing a website, we want to iterate quickly–if we make a change to our code, we’d like to see the change in the browser immediately to see if it had the desired effect. But if we’re already transpiling, bundling, and minifying our code, rebuilding after a small change can take a while.

In addition, even after making a new bundle, we’d have to reload the page to see the results of the bundle changing. If we do this, we lose any page state we had.

What we’d like is for just the code that changed to be swapped into the application without having to rebuild everything or reload the page. This is what’s called **hot module reloading** (HMR).

## The available tools

So the steps we have to execute in order to fulfill our wishlist are:

- transpilation
- bundling
- minification
- tree-shaking
- hot module reloading
- for libraries: type definitions

I’ve given some examples of tools that fulfill each of these tasks below (not exhaustive!)

### Transpilation

As before mentioned, **Babel** is the oldest transpiler and still frequently used. Its upside is that it is very mature and has a large number of plugins; its downside is that recent alternatives are significantly faster. These alternatives include **SWC** and **esbuild**.

### Bundling

The first of these tools was **Webpack**, which is still frequently used. (Most projects continue to be built with **Webpack + Babel** today.) Just like Babel, it is mature, feature-rich, and has a large plugin ecosystem. An increasingly popular, recent alternative is **Rollup**. Another option is **esbuild**, which has the advantage of being much faster than either Rollup or Webpack but is lacking some features. (Yes, it can transpile _and_ bundle!)

### Minification

The most common minifier is **Terser**; a more recent alternative is **esbuild** (yes, it does all three!)

### Tree-shaking

This is handled as part of the bundling step; Webpack, Rollup, and esbuild all have support for it.

### **Hot module reloading**

Webpack provides a tool, **webpack-dev-server**, that can be used as a hot module reloader. It will watch your code for changes and update your site’s assets when your files change. It works in conjunction with Webpack.

An alternative that has recently gained popularity is **Vite**. It is significantly faster than webpack-dev-server because it is not tied to Webpack (it uses esbuild instead).

### Type definitions

The only option to generate type definitions at the moment is **tsc**, the TypeScript compiler.[5]

## But why do I need this?

You might not! For example, if you have a simple website with only a small amount of dynamic functionality and only need to support recent browsers, then you could write a single JavaScript file and skip all of these tools. Even if you wanted to include a library or two from NPM, you could import those with `<script>` tags as well and use them. It’s really when you want more of those items on the wishlist that these build tools become useful.

Of course, these build tools come with additional complexity that must be considered. There’s also other downsides and complications of using these build tools that I haven’t mentioned here, like transpilation and minification making your code more difficult to debug (unless you set up source maps, but that’s yet another layer of complexity to work through). So these tools come with trade-offs, much like anything else in engineering. I just think it’s helpful to consider what problems we’re trying to solve, and how these build tools can help us solve them.

---

[1] To be clear, JavaScript bundle size isn’t everything; the complexity of the code (and thus the time required to parse it) is important too. [This article by Nolan Lawson](https://nolanlawson.com/2021/02/23/javascript-performance-beyond-bundle-size/) explores some of those additional considerations.

[2] There are some caveats here; if 499 of those files never change but 1 changes frequently, and we have a high number of repeat users, then retrieving 499 files from the cache and fetching the 1 changed file may be better than having to repeatedly fetch 1 large file. But in general, [we should keep bundling](https://v8.dev/features/modules#bundle).

[3] We now have the `async` attribute to download and evaluate scripts lazily to avoid this, but it [wasn’t widely supported until about 2012](https://caniuse.com/script-async) (when IE10 added support for it).

[4] Note that there is a proposal to add native type hints to JavaScript; there is also the option of using JSDoc, which allows for type hints in comments

[5] The creator of SWC is working on a TSC replacement in Go, but no preview release is available yet.
