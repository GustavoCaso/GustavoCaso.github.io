+++
title = 'Integrate Disqus with Octopress'
date = 2014-04-14
tags = ["ruby", "octopress"]
+++

I'm new to [Octopress](http://octopress.org/) platform for Blogging, but I think it is excellent and straightforward.

Creating my first post, I felt so excited, similar to when you discover a new treasure or secret path in my favourite game. But everything at the beginning is going to be challenging.

My first problem was that my post wasn't displaying the comments. I use [Disqus](https://disqus.com/), a widespread commenting system.

After hours of searching the web and reading a lot of posts about the topic, I had a good understanding of how to integrate them.

So I thought of sharing my experience:

First, you must register your site in [Disqus](https://disqus.com/), and you will receive a short code. After that, go to your `_config.yml` file inside your project and add it.

```ruby
  disqus_short_name: *****************
  disqus_show_comment_count: false
```

After that, make sure that you allow comments in your post. Inside `source/_posts/`, there will be all your posts, and on the top are some configurations:

```
---
layout: post
title: "File rename"
date: 2014-04-13 18:48:47 +0200

tags: ruby
---
```

But it didn't work; I went on a quest to solve this.
[Octopress](http://octopress.org/) uses the `liquid syntax` for loading the different layouts; reading the post layout inside `source/_layouts/_posts`, you will see that it loads a file called `disqus_thread.html,`
I looked for it and found it, and saw this:

```html
<noscript>Please enable JavaScript to view the <a href="http://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
```

Felt a little weird; that's not what you expected; searching on the Internet, some people talked about another file, `disqus.html`, inside the `_include` folder,
and once there, the code felt more appropriate.

```html
<script type="text/javascript">
      var disqus_shortname = '{{ site.disqus_short_name }}';
      {% of page.comments == true %}
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
      var dsq = document.createElement('script'); dsq.type = 'text/javascript; dsq.async = true;
      dsq.src = 'http://' + disqus_shortname + '.disqus.com/' + disqus_script;
      (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
    }());
</script>
<noscript>Please enable JavaScript to view the <a href="http://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
```

Copying that inside `disqus_thread.html` solved the problem.
