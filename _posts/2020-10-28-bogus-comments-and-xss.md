---
layout: post
title: "Bogus comments and XSS"
lead_words: 7
---

<style>

figure.highlight {
	margin-top: 25px;
	margin-bottom: 25px;
}

</style>

Let's say we have a sanitizer that:

1. Is not based on some HTML Parser
2. Is able to correctly strip away any 'dangerous' html (like script tags, on* attributes etc.) outside comments
3. Leaves `<!-- -->` as is
4. Does not replace `<`'s and `>`'s with html entities

This is of course rather unlikely to be seen in the wild, but bypassing such sanitizer can be a fun xss challenge. So how can we smuggle some javascript past this sanitizer?

# Bogus comments

According to the [whatwg's html spec](https://html.spec.whatwg.org/multipage/syntax.html#comments):

<blockquote>
Comments must have the following format: <br><br>
    1. The string "&lt;!--".<br><br>
    2. Optionally, text, with the additional restriction that the text must not start with the string ">", nor start with the string "->", nor contain the strings "<!--", "-->", or "--!>", nor end with the string "<!-". <br><br>
    3. The string "-->".<br>
</blockquote>

but there are other ways to get comments in our DOM.

For examle, when the parser comes across a `?` preceeded by a `<`, then the [unexpected-question-mark-instead-of-tag-name](https://html.spec.whatwg.org/multipage/parsing.html#parse-error-unexpected-question-mark-instead-of-tag-name) error occurs and everything till the nearest `>` will be treated as a comment.

{% highlight text %}
<? this is a comment > this is not a comment >
{% endhighlight %}

Cool, but how can we use this? Lets start simple:

{% highlight text %}
<script>alert(1)</script>
{% endhighlight %}

this will obviously be removed by the sanitizer and this:

{% highlight text %}
<!-- <script>alert(1)</script> -->
{% endhighlight %}

will be preserved, but will not lead to javascript execution. Hmm... but what if we could somehow comment out the `<!--`?  

Let's try:

{% highlight text %}
<? <!-- a> <script>alert(1)</script> -->
{% endhighlight %}

This produces the following DOM:

{% highlight text %}
├─ #text:
├─ #comment: ? <!-- a
├─ #text:
└─ SCRIPT
    └─ #text: alert(1)
{% endhighlight %}

Bingo! Since the `<!--` was embedded in `<? >`, it was treated by the parser as a comment's text and thus `<script>` was treated as an opening tag.

And that's one of many reasons to use sanitizers based on HTML parsers, as this would not be possible if `1.` wasn't the case.

Happy xss'ing! :)