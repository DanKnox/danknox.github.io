---
layout: post
title: "WebsocketRails on the Pogoapp Blog"
description: "The kind folks over at Pogoapp featured WebsocketRails on
their blog."
category: 
tags: []
---
{% include JB/setup %}

The awesome folks over at [Pogoapp][2] just added a post to their
Blog about using my [WebsocketRails][3] Gem to demonstrate the Pogoapp
WebSocket support. They were also nice enough to update the code from my
very outdated example application to work with the latest version of the
Gem and clean up some of the CoffeeScript while they were at it. You can
check out the post [here][6].

I appreciate them taking the time to update the example app code. 
It is something that I have been meaning to do for quite awhile. There 
are a ton of useful new features in the Gem that the example app does not
yet cover. Expect a post coming soon that walks through all of the new
functionality that the WebsocketRails gem has to offer

### You Should Check Out Pogoapp!

I met [Paul][1] from Pogoapp the other day on IRC and was quite
impressed with what he and the Pogoapp team have put together. We
originally began chatting about the current implementation of
[Gitlab's][4] new [gitlab-shell][5] project. For those of you that are
unfamiliar with the gitlab-shell project, it is a Ruby based git
repository management system that is heavily tied into the core Gitlab
code base. In version 5.0 of Gitlab, due out on the 22nd of this month,
gitlab-shell will fully replace Gitolite as their repo management system. 

Paul shared with me some of the secret sauce behind how Pogoapp developed
their own Redis backed git repository management system. He then went
above and beyond by walking me through the details of what the Pogoapp
team have spent several years building out infrastructure wise. Needless
to say, I am very impressed with their core infrastructure and the
powerful management interface that they have created. For those of you
looking for a clean way out of the seemingly endless Heroku money pit, I highly recommend you check out Pogoapp for your next application.

[1]: https://github.com/themgt
[2]: http://www.pogoapp.com
[3]: http://github.com/DanKnox/websocket-rails
[4]: http://gitlabhq.com
[5]: https://github.com/gitlabhq/gitlab-shell
[6]: http://www.pogoapp.com/blog/posts/websockets-on-rails-4-and-ruby-2
