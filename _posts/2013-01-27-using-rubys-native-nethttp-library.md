---
layout: post
title: "Using Ruby's Native Net::HTTP Library"
description: "Eliminate unnecessary dependencies by using the ruby
standard library for basic HTTP requests."
category: ruby
tags: [ruby general http]
---
{% include JB/setup %}

In this post, I am going to step through constructing a basic wrapper
class for simplifying interaction with the `Net::HTTP` library. I came
across a few answers on stack overflow recently that seemed to give the
impression that working with `Net::HTTP` is more difficult than it needs
to be.

While there are many great ruby HTTP client gems out there, the native
library can be useful when you need to make a few simple HTTP requests.
Try giving it a shot first before reaching for a heavier weight solution.
If nothing else, you will have a better understanding of what is
happening behind the scenes of your favorite HTTP client gem.

### The Basics

If you are already familiar with basic usage of the `Net::HTTP` library,
you may want to [skip ahead to the good stuff](#skip-basics).

After browsing the [documentation][1] it is clear that there are many
different ways to interact with the native library. For our purposes
we are going to go with what is, in my opinion, one of the most
straightforward but least well documented.

First, we will create a new instance of the `Net::HTTP` class
with a reference to our main API endpoint. The endpoint is the full
address of the API server without any of the paths for the specific
requests. In our case, we will use the new API based temporary email
service that I am building. It is currently located at
`http://dev.threedotloft.com`.

{% highlight ruby %}
require "net/http"
require "uri"

uri = URI.parse("http://api.random.com")
http = Net::HTTP.new(uri.host, uri.port)
{% endhighlight %}

We now have our http instance that we can use to perform requests. A
basic request is constructed using the appropriate class for the HTTP
request method that you wish to perform. For an HTTP **GET**, you would
use `Net::HTTP::Get`.

{% highlight ruby %}
# Continuing our example from above

request = Net::HTTP::Get.new("/search?question=somequestion")
response = http.request(request)

response.code
=> "200"
response.body
=> # Raw response body would go here needing to be parsed
{% endhighlight %}

If you don't care for checking the response code, the response object's
class happens to reflect the status of the response as well.

{% highlight ruby %}
response.code
=> "200"
response.class
=> Net::HTTPSuccess

case response
when HTTPSuccess
  response.body
when HTTPRedirect
  follow_redirect(response) # you would need to implement this method
else
  raise StandardError, "Something went wrong :("
end
{% endhighlight %}

I personally find it more succinct to operate on the response
code.

{% highlight ruby %}
case response.code.to_i
when 200 || 201
  p [:success]
when (400..499)
  p [:bad_request]
when (500..599)
  p [:server_problems]
end
{% endhighlight %}

Now that we are familiar with the basics of interacting with the
library, we can begin packaging all of this up into a nice class. In the
interest of brevity, I am going to leave out a few details such as error
handling. I am also not including the tests
behind this class, although I may follow up with another post that
covers building out this class in a test driven manner and adding more
advanced functionality.

### <a id="skip-basics"></a> Creating Our Connection Class

The first thing we need to do is store our http instance upon
initialization so we can use it later to perform requests.

{% highlight ruby %}
require "net/http"
require "uri"

class Connection

  ENDPOINT = "http://dev.threedotloft.com"

  def initialize(endpoint = ENDPOINT)
    uri = URI.parse(endpoint)
    @http = Net::HTTP.new(uri.host, uri.port)
  end

end
{% endhighlight %}

Since we know the address of the API endpoint that we are targeting, we
use that as the default option. However, we allow for overriding that at
runtime to provide a little more flexibility.

The next thing we need is a way to perform the actual requests. Our
first shot may look something like this.

{% highlight ruby %}
# Inside the Connection class
def get(path, params)
  full_path = encode_path_params(path, params)
  request = Net::HTTP::Get.new(full_path)
  @http.request(request)
end

def post(path, params)
  request = Net::HTTP::Post.new(path)
  request.set_form_data(params)
  @http.request(request)
end

def put(path, params)
  request = Net::HTTP::Put.new(path)
  request.set_form_data(params)
  @http.request(request)
end

private

def encode_path_params(path, params)
  encoded = URI.encode_www_form(params)
  [path, encoded].join("?")
end
{% endhighlight %}

So far we have only implemented three of our HTTP verbs and we are
already starting to see a lot of duplication. Lets factor the bulk of
that duplication into a common `#request` method. This refactoring will
give us a central place to make changes in the event that
we want to implement things like response body deserialization. 

Since each request type needs a different class instantiated, we will
include a hash to map the HTTP verbs to their respective request classes.

{% highlight ruby %}
class Connection

  VERB_MAP = {
    :get    => Net::HTTP::Get,
    :post   => Net::HTTP::Post,
    :put    => Net::HTTP::Put,
    :delete => Net::HTTP::Delete
  }

  # ENDPOINT declaration and initialization goes here.
  
  private
  
  def request(method, path, params)
    case method
    when :get
      full_path = encode_path_params(path, params)
      request = VERB_MAP[method].new(full_path)
    else
      request = VERB_MAP[method].new(path)
      request.set_form_data(params)
    end

    @http.request(request)
  end
{% endhighlight %}

Now we can go back and refactor our original implementation to use this
common request method and implement the remaining HTTP verbs while we
are at it.

{% highlight ruby %}
def get(path, params)
  request :get, path, params
end

def post(path, params)
  request :post, path, params
end

def put(path, params)
  request :put, path, params
end

def delete(path, params)
  request :delete, path, params
end
{% endhighlight %}

Since the TmpMail API only makes use of these four common verbs, we
are finished with our request logic. If we need to add more later it is
now a trivial matter of adding the correct class to the `VERB_MAP` and
implementing any special parameter handling within the request method.

We are almost finished. Our class can now perform the most common http
actions, exposing a simple and intuitive public API. However, our
API endpoint returns JSON response bodies and it would be a pain
to have to deserialize the response each time we made a request so lets go
ahead and add deserialization to the Connection class.

As with anything in ruby, there are many different ways we could go
about this. I am going to add a `#request_json` method that calls our
original `#request` method, parses the response body and returns the
deserialized request.

{% highlight ruby %}
require "ostruct"
require "json"

# Still in the Connection class

private

def request_json(method, path, params)
  response = request(method, path, params)
  body = JSON.parse(response.body)

  OpenStruct.new(:code => response.code, :body => body)
rescue JSON::ParserError
  response
end
{% endhighlight %}

> In a real application it would typically not be a good idea to
silently swallow the JSON parser error. It would be far better to define
your own custom exception class and raise that. Just make sure to
clearly document the exceptions your method may raise and why.

Since I prefer to also return the status code of the response, I
chose to package the code and deserialized body together using ruby's
OpenStruct library. For those of you not familiar with the OpenStruct
library, I recommend checking out the [documentation][2]. It provides a
quick way to bundle together attributes within an object. If the
requirements grow and you need things like dynamic content type
deserialization, I would likely factor this out into a custom Response class.

Now we will go ahead and update our public verb methods to use this new
`request_json` method. Let's take a look at the final product.

{% highlight ruby %}
require "net/http"
require "uri"
require "ostruct"
require "json"

class Connection

  ENDPOINT = "http://api.random.com"

  VERB_MAP = {
    :get    => Net::HTTP::Get,
    :post   => Net::HTTP::Post,
    :put    => Net::HTTP::Put,
    :delete => Net::HTTP::Delete
  }

  def initialize(endpoint = ENDPOINT)
    uri = URI.parse(endpoint)
    @http = Net::HTTP.new(uri.host, uri.port)
  end

  def get(path, params)
    request_json :get, path, params
  end

  def post(path, params)
    request_json :post, path, params
  end

  def put(path, params)
    request_json :put, path, params
  end

  def delete(path, params)
    request_json :delete, path, params
  end

  private

  def request_json(method, path, params)
    response = request(method, path, params)
    body = JSON.parse(response.body)

    OpenStruct.new(:code => response.code, :body => body)
  rescue JSON::ParserError
    response
  end

  def request(method, path, params = {})
    case method
    when :get
      full_path = encode_path_params(path, params)
      request = VERB_MAP[method.to_sym].new(full_path)
    else
      request = VERB_MAP[method.to_sym].new(path)
      request.set_form_data(params)
    end

    @http.request(request)
  end

  def encode_path_params(path, params)
    encoded = URI.encode_www_form(params)
    [path, encoded].join("?")
  end

end
{% endhighlight %}

There we have it. Our finished HTTP client. Lets take it out for a quick
spin and check a few emails.

{% highlight ruby %}
> connection = Connection.new
 => #<Connection:0x007ffac20f5270 @http=#<Net::HTTP dev.threedotloft.com:80 open=false>>

> res = connection.post("/account", {email: 'dknox@threedotloft.com',
>                                    password: 'some temp pass'})
> res.code
 => 200
> res.body
 => {
  "email"=>"dknox@threedotloft.com",
  "auth_token"=>"SnpuWVlPYmx2R052amVrQnd6RVV3TFtYcg=="
}
{% endhighlight %}

I'm going to store the authentication token in a variable for use
in the next few API calls.

{% highlight ruby %}
> auth_token = res.body["auth_token"]
{% endhighlight %}

Next we are going to request a list of the available domains on the
server and then join one so we can start receiving emails.

{% highlight ruby %}
> res = connection.get("/domains",{token: auth_token})
> res.body
 => 
[
  {
    "name"=>"dev.threedotloft.com"
  }, 
  {
    "name"=>"tmpmail.threedotloft.com"
  }
] 

# Join the domain first
> res = connection.post("/domains/join",{token: auth_token,
                                         name: 'dev.threedotloft.com'})

 => #<OpenStruct code="200", body=[{"name"=>"dev.threedotloft.com"}]>

# Claim your email second
> res = connection.post("/inboxes/claim",{token: auth_token,
                                          name: 'dknox',
                                          domain: 'dev.threedotloft.com'})
> res.body
 => [
 {
   "name"=>"dank",
   "message_count"=>0,
   "domain"=>"dev.threedotloft.com",
   "full_address"=>"dank@dev.threedotloft.com"
 },
 {
   "name"=>"dknox",
   "message_count"=>0,
   "domain"=>"dev.threedotloft.com",
   "full_address"=>"dknox@dev.threedotloft.com"
 }
]
{% endhighlight %}

Based on the output from the last call, you can see that I currently own
two email addresses. The first is `dank@dev.threedotloft.com` and the
second is `dknox@dev.threedotloft.com`. They both currently have zero
messages in their Inbox. I am going to send a test email out and see if
it shows up in my email service.

You will have to imagine the sending of the email as it is currently
taking place off-camera...

Alright, the test email has been sent. Lets call the API again to see if
it has arrived.

{% highlight ruby %}
> res = connection.get("/messages", {token: auth_token,
                                     inbox: 'dknox',
                                     domain: 'dev.threedotloft.com'})
> res.body.count
 => 1
> res.body
 =>
  [{
    "id"=>"512ea4661d41c8adf7000001", 
    "date"=>"2013-02-27T16:27:51-08:00", 
    "subject"=>"Testing Email API", 
    "to"=>["dknox@dev.threedotloft.com"], 
    "from"=>["dknox@threedotloft.com"], 
    "reply_to"=>nil, 
    "cc_addrs"=>[], "bcc_addrs"=>[], 
    "headers"=>{}, 
    "raw"=> **Omitted Due To Length**
  }]
{% endhighlight %}

Awesome, it looks like my test email arrived successfully, thus concluding this
demonstration.

Feel free to use this code yourself. As mentioned above, this basic example is mostly devoid of error
handling. If you choose to use it you should probably modify it a bit
first.
 
[1]: http://ruby-doc.org/stdlib-1.9.3/libdoc/net/http/rdoc/Net/HTTP.html
[2]: http://ruby-doc.org/stdlib-1.9.3/libdoc/ostruct/rdoc/OpenStruct.html
