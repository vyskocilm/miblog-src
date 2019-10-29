---
title: "'Modern' Javascript Development for Oldies"
date: 2018-04-04T22:45:29+02:00
draft: true
---

## My Javascript experience

When I started my career, I was working as Junior Developer on web based application. We have had parts written in [Java Server Pages](https://en.wikipedia.org/wiki/JavaServer_Pages), which was great technology combining elegance of PHP with easy to use of Java. We had something in [Java servlets](https://en.wikipedia.org/wiki/Java_servlet), which allows us easier testing and compilation up front. Other parts were in [PL/SQL](
https://en.wikipedia.org/wiki/PL/SQL). And of course, the Javascript. And most of our clients has been using [Internet Explorer 6](https://davidwalsh.name/6-reasons-why-ie6-must-die) at the time! XmlHttpRequest was a neat new thing, jQuery was a hot new technology. The common way to have JS integrated was through session objects, so the logic of application was scattered through several pages (JSPs or Servlets) and it relied on server side rendering heavily. At that time I decided to **never** touch Javascript at all.

## Fast forward to 2018

The world has changed. Browsers are more and more ubiquitous, we got more platforms (TVs, consoles and phones). So if you want to spread your small application to your friends, then web application is the easiest way to do. Fortunately Javascript become more powerful and compatible across browsers. The main shift of paradigm is client side rendering. There is no need to do horrible things like passing session objects and initialize all you JS again and again. You can manage all your state in one place. And that's not all. With advent of technologies like [Webassembly](http://webassembly.org/) and [Emscripten](https://github.com/kripken/emscripten) you can have crazy things like [native sqlite](https://github.com/kripken/sql.js) database running in your browser. Plus you can offload heavy CPU intensive tasks (like SQL queries) to [Web worker](https://en.wikipedia.org/wiki/Web_worker).

## "Modern" JS development for dummies

Of course things are not _that_ easy for dummies. You have [nodejs](https://nodejs.org/en/), [npm](https://www.npmjs.com/) package manager (or shall I use [yarn](https://yarnpkg.com/en/)?). Should I learn AngularJS, ReactJs, Redux, Webpack, Gulp, Vue.js, Grunt? You can easily fall into the trap of choosing the right tools, frameworks and workflows, spending hours on watching youtube videos and reading documentation, while not implementing a single line. The thing is - it **does not matter**. Keep it simple, implement first version using technologies you know or you can learn easily and finish it as soon as possible. Then you will encounter problems like deployment, which you can solve using existing technologies. So start with stupid solutions and enhance it over times.

## jQuery

First thing - use and learn jQuery. The DOM API (especially old parts like `getElementById`) is horrible and jQuery have nicer and easier to use API. Plus all you need to use jQuery is to add `scr="https://code.jquery.com/jquery-3.3.1.slim.js"` to your HTML and it works. You can get nicer looking code with a minimal effort


{{< highlight javascript >}}
// DOM API
var element = document.getElementById("id");
// jQuery
var element = $("#id");
{{< / highlight >}}
</pre>

Sadly there are things jQuery is not compatible with modern API like needed for `sql.js` like `arraybuffer` type for `XmlHttpRequest`. Fortunately you can combine it with native browser APIs (unlike [ReactJS](https://reactjs.org/docs/integrating-with-other-libraries.html) which builds own Virtual DOM). Which is important in case you're junior and exploring the world.

## Avoid nested code

For a simple application you'll be building, try to avoid having nested code. If you define function inside function inside callback definition, you're doing something wrong. This is problematic given the fact Javascript is event based and you register callback all around. The problem I have had was

1.  You need to use `XmlHttpRequest` to download your database file
2.  THEN you can instantiate database and send first request to worker
3.  THEN is your application ready

And while this is trivial to write in traditional environment, `XmlHttpRequest` is asynchronous, so all you know if that you're ready when *callback* is called.

{{< highlight javascript >}}
var worker = new Worker ("worker.sql.js");
...
var xhr = new XmlHttpRequest ();

...
xhr.onload = function (event) {
    ...
    worker.postMessage({
        id:id,
        action:'open',
        buffer: uInt8Array,
    });
    // enable execute button, so you can post SQL queries to worker
    $("#execute").removeAttr ("disabled");
}
{{< / highlight >}}

## console.log and browser's REPL

Can't emphasize how great tool is `console.log`. It does not work like Python or similar REPL printing strings.  Browser will deal with logged things like with objects, so all the structure become visible to you immediately. This is very powerful tool, especially for novices in the field.

This becomes extremely important when you're dealing with a relatively simple tasks, like how to iterate through `Object` or `Map` in the language.

## Kinkip: The result

[Kinkip](/kinkip/index.html)

[Piknik slov - Word Snack](https://play.google.com/store/apps/details?id=com.apnax.wordsnack.csk&hl=en)

[sql.js](https://github.com/kripken/sql.js)

[svobodneslovniky.cz](https://www.svobodneslovniky.cz/)

Logo by pxhere: [https://pxhere.com/en/photo/1267185]
