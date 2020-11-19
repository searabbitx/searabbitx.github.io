---
layout: post
title: "HTML attributes without spaces"
lead_words: 7
---

<style>

figure.highlight {
	margin-top: 25px;
	margin-bottom: 25px;
}

</style>

This is just another idea for an xss challenge, that can teach us something new about HTML parsing.

Let's say we need to sneak `<img src onerror="alert(1)">` past a sanitizer that rejects any input if it contains whitespace characters. What can we do in such situation?

<br>
### Attributes tokenizing

According to the [WHATWG HTML Spec](https://html.spec.whatwg.org/), after the tokenizer consumes `<img` characters it is in the [tag name state](https://html.spec.whatwg.org/multipage/parsing.html#tag-name-state) and our goal will be to get into the [attribute name state](https://html.spec.whatwg.org/multipage/parsing.html#attribute-name-state) from there. To do it in the intended way we should append our payload with one of these characters:

<div class="filename"><a href="https://html.spec.whatwg.org/multipage/parsing.html#tag-name-state" target="blank">Tag name state 
 | html.spec.whatwg.org</a></div>
{% highlight text %}
U+0009 CHARACTER TABULATION (tab)
U+000A LINE FEED (LF)
U+000C FORM FEED (FF)
U+0020 SPACE
    Switch to the before attribute name state.
{% endhighlight %}

but the sanitizer won't allow us to use any of these. What are the other options?

<div class="filename"><a href="https://html.spec.whatwg.org/multipage/parsing.html#tag-name-state" target="blank">Tag name state 
 | html.spec.whatwg.org</a></div>
{% highlight text %}
U+002F SOLIDUS (/)
    Switch to the self-closing start tag state.

U+003E GREATER-THAN SIGN (>)
    Switch to the data state. Emit the current tag token.

ASCII upper alpha
    Append the lowercase version of the current input character
    (...) to the current tag token's tag name.

U+0000 NULL
    This is an unexpected-null-character parse error. 
    Append a U+FFFD REPLACEMENT CHARACTER character
        to the current tag token's tag name.

EOF
    This is an eof-in-tag parse error. Emit an end-of-file token.

Anything else
    Append the current input character to the current tag token's tag name.
{% endhighlight %}

`U+002F SOLIDUS (/)` looks interesting, let's see where that leads us.


<br>
After the tokenizer consumes `<img/` it is in the [self-closing tag state](https://html.spec.whatwg.org/multipage/parsing.html#self-closing-start-tag-state) and then we can try: 

```
U+003E GREATER-THAN SIGN (>)
    Set the self-closing flag of the current tag token. 
    Switch to the data state. Emit the current tag token.
```

which won't help us

```
EOF
    This is an eof-in-tag parse error. Emit an end-of-file token.
```

which also won't help us even if we could control where the file with our payload ends. 

What about the last option?

```
Anything else
    This is an unexpected-solidus-in-tag parse error.
    Reconsume in the before attribute name state.
```

*Reconsume in the before attribute name state*, **bingo**!

<br>
So, after the tokenizer consumes `<img` and `/`, it ends up in **the same state** as it would after consuming `<img` and a space!

Therefore:

```
<img/src/onerror="alert(1)">
```

is treated as:

```
<img src onerror="alert(1)">
```

which is what we need.
<br>
<br>
<br>
Happy xss'ing! :)