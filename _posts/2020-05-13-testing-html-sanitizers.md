---
layout: post
title: "Two tips for html sanitizers testing"
---

So, you're looking for an xss, trying to bypass an html sanitizer with complex behavior? Here are two things you can start your testing with to grasp a better understanding of how does this sanitizer work.

## Tip 0. Just try to submit a list of all html tags.

First, if you can submit a lot of text, getting a list of all allowed tags can be easy. Just submit a list of all html tags and the sanitizer will politely tell you which ones are allowed simply by preserving them in the result :). If there are any text length limits, you'll have to split it into smaller chunks.

You can start with this list:

<div class="filename">all_html_tags.html</div>
<div class="scroll">
{% highlight html scroll %}
<!--...-->
<!doctype dfasf>
<a></a>
<abbr></abbr>
<acronym></acronym>
<address></address>
<applet></applet>
<area>
<article></article>
<aside></aside>
<audio></audio>
<b></b>
<base>
<basefont></basefont>
<bb></bb>
<bdo></bdo>
<big></big>
<blockquote></blockquote>
<body></body>
<br />
<button></button>
<canvas></canvas>
<caption></caption>
<center></center>
<cite></cite>
<code></code>
<col>
<colgroup></colgroup>
<command></command>
<datagrid></datagrid>
<datalist></datalist>
<dd></dd>
<del></del>
<details></details>
<dfn></dfn>
<dialog></dialog>
<dir></dir>
<div></div>
<dl></dl>
<dt></dt>
<em></em>
<embed>
<eventsource></eventsource>
<fieldset></fieldset>
<figcaption></figcaption>
<figure></figure>
<font></font>
<footer></footer>
<form></form>
<frame></frame>
<frameset></frameset>
<h1></h1>
<head></head>
<header></header>
<hgroup></hgroup>
<hr />
<html></html>
<i></i>
<iframe></iframe>
<img>
<input>
<ins></ins>
<isindex></isindex>
<kbd></kbd>
<keygen>
<label></label>
<legend></legend>
<li></li>
<link>
<map></map>
<mark></mark>
<menu></menu>
<meta>
<meter></meter>
<nav></nav>
<noframes></noframes>
<object></object>
<ol></ol>
<optgroup></optgroup>
<option></option>
<output></output>
<p></p>
<param>
<pre></pre>
<progress></progress>
<q></q>
<rp></rp>
<rt></rt>
<ruby></ruby>
<s></s>
<samp></samp>
<script></script>
<section></section>
<select></select>
<small></small>
<source>
<span></span>
<strike></strike>
<strong></strong>
<style></style>
<sub></sub>
<sup></sup>
<table></table>
<tbody></tbody>
<td></td>
<textarea></textarea>
<tfoot>	</tfoot>
<th></th>
<thead></thead>
<time></time>
<title></title>
<tr></tr>
<track>
<tt></tt>
<u></u>
<ul></ul>
<var></var>
<video></video>
<wbr>
<noscript></noscript>
{% endhighlight %}
</div>

## Tip 1. Submit a list of attributes for chosen tags.

Now that we know which tags are allowed without any tricks or complex filter bypasses, let's find out which attributes are allowed for those tags.

With a bit of python scripting we can generate list of chosen tags with chosen attributes, one attribute per line:

<div class="filename">attributes.py</div>
{% highlight python %}
TAGS = [
    'button',
    'p',
#   ... and more allowed tags
]   

EMPTY_ELEMENT_TAGS = [
    'img',
    'meta',
#   ... and more non empty element tags
]   

ATTRIBUTES = [
    'onerror',
    'style',
    'src',
#   ... and more attributes you want to check
]   

for tag in TAGS:
    for attribute in ATTRIBUTES:
        print("<{} {}=foo></{}>".format(tag, attribute, tag))

for tag in EMPTY_ELEMENT_TAGS:
    for attribute in ATTRIBUTES:
        print("<{} {}=foo>".format(tag, attribute))

{% endhighlight %}

{% highlight bash %}
$ python3 attributes.py
<button onerror=foo></button>
<button style=foo></button>
<button src=foo></button>
<p onerror=foo></p>
# ... etc.
{% endhighlight %}

To create a nice list of potentially useful attributes, google for articles like [W3schools HTML Event Attributes](https://www.w3schools.com/tags/ref_eventattributes.asp), [MDN's HTML attribute reference](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes) or <u><b>especially</b></u>: [MDN's Global attributes list](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes). 

Who knows? Maybe developers overlooked or didn't fully understand how some attribute/tag could be used while implementing a white/black list? It is possible that you'll find something juicy before trying to fool sanitizer with filter bypass techniques.

Happy hunting!
