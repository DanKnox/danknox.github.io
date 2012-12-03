---
layout: post
title: "First Post"
description: ""
category : other 
tags : [general]
---
{% include JB/setup %}

Welcome to the very first post on my blog.

Here is some cool code:

{% highlight ruby linenos %}
class BlogPost
  
  def initialize(content)
    @content = content
  end

  def awesome?
    @content == awesome
  end

end

BlogPost.new.awesome?
 => true
{% endhighlight %}
