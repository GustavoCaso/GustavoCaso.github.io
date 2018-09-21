---
layout: post
title: "Integrate Disqus with Octopress"
date: 2014-04-14 13:20:23 +0200
comments: true
tags: [ruby,octopress]
---

I'm new to [Octopress](http://octopress.org/) platform for Blogging, but so far I think is great and simple.

Creating my first post I felt so excited, as when you discover a new treasure or a secret path in your favorite game. But everything at the beginning is not going to be smooth.
My first problem was that my post weren't displaying the [Disqus](https://disqus.com/) comments, that are a very common system for comments in the blog community.

READMORE

After hours of searching the web and reading lot of post about the topic I had a goodunderstand of how to integrated them.

So I thought of sharing my experience:

Fisrt you have to register you site in [Disqus](https://disqus.com/), you will receive a short code. After that go to your `_config.yml` file inside your project and add it.

```ruby
  disqus_short_name: *****************
  disqus_show_comment_count: false
```

After that make sure that you allow comments in your post. Inside `source/_posts/` there will be all your post, and on the top are some configuration :

```
---
layout: post
title: "File rename"
date: 2014-04-13 18:48:47 +0200
comments: true
tags: ruby
---
```

And that's it, or wasn't ?

But it didn't work, I went on a quest for solving this.
[Octopress](http://octopress.org/) uses the `liquid syntax` for loading the different layouts, reading the post layout inside `source/_layouts/_posts` you will see that load a file call `disqus_thread.html`,
I looked for it and found it, and saw this:
```html
<noscript>Please enable JavaScript to view the <a href="http://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
```
Felt a little weird, that's not what you expect, searching in the wide Internet some people talked about another file `disqus.html` inside the `_include` folder,
once there the code felt more appropiated. Copying that inside `disqus_thread.html`
the problem solve.

```html
<script type="text/javascript">
      var disqus_shortname = '{{ site.disqus_short_name }}';
      {% if page.comments == true %}
        {% comment %} `page.comments` can be only be set to true on pages/posts, so we embed the comments here. {% endcomment %}
        // var disqus_developer = 1;
        var disqus_identifier = '{{ site.url }}{{ page.url }}';
        var disqus_url = '{{ site.url }}{{ page.url }}';
        var disqus_script = 'embed.js';
      {% else %}
        {% comment %} As `page.comments` is empty, we must be on the index page. {% endcomment %}
        var disqus_script = 'count.js';
      {% endif %}
    (function () {
      var dsq = document.createElement('script'); dsq.type = 'text/javascript'; dsq.async = true;
      dsq.src = 'http://' + disqus_shortname + '.disqus.com/' + disqus_script;
      (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
    }());
</script>
<noscript>Please enable JavaScript to view the <a href="http://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>

```

Hope I could help.
