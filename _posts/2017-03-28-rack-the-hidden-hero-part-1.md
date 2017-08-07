---
title:  "Rack, the hidden hero"
date:   2017-03-28
layout: post
description: Everything you need to know about Rack
---
{:.full-image}
![Post cover]({{ site.url }}/assets/figures/rack-the-hidden-hero/post-cover.jpg)

You are reading this blog, I guess you are a Ruby (Ruby on Rails) developer, or at least you used to touch the Rails framework, right? So, perhaps you used to hear some terms like "framework based on Rack" or "server for Rack application", etc. Yes, we are talking about this [Rack](https://rack.github.io) in the Ruby world. I believe that there aren't many people working with Rack in usual. Neither do I. To be honest, the last thing I did with Rack is to config a web server and add some bunch of low-level routing. However, I realize that Rack is an elegant and beautiful underneath any Ruby web frameworks. Researching Rack provides me a lot of knowledge about Ruby and deep understanding about how web framework (like Rails) works from scratch. This blog post is a place for me to note about everything I learned. If you have any question for me, please don't hesitate to comment. I really appreciate <3

# What is Rack?

In the Ruby on Rails project website, there is a dedicated section to talk about Rack: [Rails on Rack](http://guides.rubyonrails.org/rails_on_rack.html). In other frameworks' websites, there is at least a small part in the guide page mentioned Rack and how the framework related to Rack, for example, [Hanami](http://hanamirb.org/guides/actions/rack-integration/), [Sinatra](http://www.sinatrarb.com/intro.html#Rack%20Middleware), [Cuba](https://github.com/cuba-platform/cuba), or [Roda](https://github.com/jeremyevans/roda). Yes, nearly all the Ruby web frameworks available on the market are related to Rack. Actually, I can say that they are **based on Rack**. If you open any projects using such frameworks, it is likely that there exists a file name `config.ru`. If there aren't any, for example, in a hello world Sinatra application, the `config.ru` file is automatically interpolated in other way. Besides normal server starting commands such like `rails server -p 3000` or `puma -p 3000`, you can start your applications with the command `rackup`. Let's give it a try and perhaps you will be surprised. This is one of the most significant evidence for the existence of Rack.

By the self-definition of the Rack team: "Rack provides a minimal interface between webservers that support Ruby and Ruby frameworks.". Before talking anything about this definition, I want to recall the definition of the web server. By definition from Wikipedia, web servers are the software that accepts and supervises the HTTP requests. Yup, a web server listens continuously to the host and the port that it is working on. If there is a request waiting, it accepts the request if possible, then calls something else to handle that request and get the result, then it sends the response back to the client. Its job is just trying to receive and respond as fast as possible. All the logic about how to generate the response for a specific request belongs to something else. The one who handles the real logic could be just a simple script written in Perl, or some bunch of PHP files, or a large and full-component web framework like Ruby on Rails.

Back to the definition of Rack, it is obvious that Rack lays between the Ruby-supported web servers and the Ruby web framework. Web servers take of handling raw requests, web frameworks rake care of the logics to generate the response for the HTTP requests. What does Rack do between them? Why don't we just let the web servers talk directly to the web frameworks? Actually, yes, they can talk directly to each other. However, everything soon becomes chaotic. In the Ruby world, there is a bunch of famous web servers: Puma, Thin, Webrick, Passenger, etc. and even more web frameworks: Rails, Cuba, Roda, Hanami, Sinatra, etc. A web server, for example, Puma, wants to run Rails, they must have the same standard so that Puma can pass something that Rails understands and Rails returns a format that Puma understands. Soon, if Puma wants to support more frameworks, it must implement the adapters for each framework. You can imagine the efforts putting in the development process and maintaining the compatibilities among them are extremely high. That's why Rack exists. It provides a common **standard interface**. The web servers and web frameworks just need to follow the standard and they can talk to each other.

Usually, at an early step of the server starting process, a **Rack application** is created based on the configs of the user. These configurations are usually stored in a file name `config.ru` which I mentioned above. For each incoming request, the web server prepares an object called **environment** that contains the request needed information like headers, body, host, port, and some data that the web server wants Rack to use. The specification of environment object is defined at [Rack specification](http://www.rubydoc.info/github/rack/rack/master/file/SPEC), you should take a look. This environment object is passed into the Rack application. Inside that Rack object, the environment object goes through a **middleware chain**. Middleware is a separated stage to process the request. The input of each middleware is the environment object, the output is an array of three elements: response status, a hash of response headers and response body. Each middleware takes a single responsibility. It could process the request for real by using the information from environment object and return the response by itself or call the next middleware in the middleware chain to handle. Before passing to next middleware, it could enhance the environment object with useful information such as fetching session from HTTP cookie header, uploading the file and provides the file path, etc. It can also enhance the response by adding more response headers, transform the body or even replace the whole response. Some middlewares do not transform anything, it just does the monitoring or logging works. Perhaps you will be surprised to know that your favorite framework is actually a middleware in the middleware chain. Yes, a big one. Usually, it stays at the end of the middleware chain. Of course, if the request is caught and handled by a middleware, it won't reach the next middlewares in the chain. When the environment object stops at an execution middleware (usually at the end of the chain, which is the web framework), the response is generated and it pops up back to the beginning of the chain. At the end, the web server receives the full response and converts the response into valid HTTP format and sends back to the client.

In the figure, let's assume that there are 3 middlewares (in fact, there are usually 10 - 20 middlewares in a Rails application). The request from the user is parsed and passed to the Rack application under environment object (we can consider it as a hash). After going through Warden ([Warden authentication framework](https://github.com/hassox/warden)), a new field is added to the environment object and the next middleware receives it. When reaching Rails, some magic happens and the response is generated. The response pops back to the beginning of the middleware chain. When it reaches Rack::ContentLength, a response header `Content-Length` is added. Finally, the response array reaches the web server and it is transformed into valid HTTP response and sent back to the client.

![Rack working flow 1]({{ site.url  }}/assets/figures/rack-the-hidden-hero/work-flow-1.jpg)

This next example is just like above except the Warden middleware stops the environment and returns the responses right away. The framework doesn't receive the environment object.

![Rack working flow 2]({{ site.url  }}/assets/figures/rack-the-hidden-hero/work-flow-2.jpg)

# Let's play with Rack

If you are familar with full-stack framework like Rails, perhaps you will be surprised about how simple it is to write a web application. You don't need fancy gems, `Rack` is just enough. I'll assume you use bundler :). Let's start with an empty project folder and a simple Gemfile and run `bundle install`.

```ruby
source 'https://rubygems.org'

gem 'rack'
```

Then, create a file named `config.ru` with the following content.

```ruby
application = proc do |env|
  [200, { 'Content-Type' => 'text/html' }, ['Anime is awesome']]
end

run application
```

And finally, run the command `bundle exec rackup` and tada, congratulation, you have just create your own web application with Rack. This application has just one simple responsibility: response a dummy JSON with header `Content-Type: text/html`. All the logic is handled by the proc object. Whenever a request comes to the webserver, that proc is executed with a environment variable attached as the first parameters. Actually, thanks to Ruby's duck-typing system, you can freely use any object which has `call` public interface. Let's structure the code differently:

```ruby
class MyAwesomeApplication
  def call(request)
    [200, { 'Content-Type' => 'text/html' }, ['Anime is awesome']]
  end
end

run MyAwesomeApplication.new
```

If you are curious enough, let's print out that object, you will see the environment variable I mentioned above.

```ruby
{
    "GATEWAY_INTERFACE" => "CGI/1.1",
            "PATH_INFO" => "/",
         "QUERY_STRING" => "",
          "REMOTE_ADDR" => "::1",
          "REMOTE_HOST" => "::1",
       "REQUEST_METHOD" => "GET",
          "REQUEST_URI" => "http://localhost:9292/",
          "SCRIPT_NAME" => "",
          "SERVER_NAME" => "localhost",
          "SERVER_PORT" => "9292",
      "SERVER_PROTOCOL" => "HTTP/1.1",
      "SERVER_SOFTWARE" => "WEBrick/1.3.1 (Ruby/2.3.3/2016-11-21)",
            "HTTP_HOST" => "localhost:9292",
      "HTTP_USER_AGENT" => "curl/7.51.0",
          "HTTP_ACCEPT" => "*/*",
         "rack.version" => [
        [0] 1,
        [1] 3
    ],
           "rack.input" => #<Rack::Lint::InputWrapper:0x007fd40d28f418 @input=#<StringIO:0x007fd40d295a20>>,
          "rack.errors" => #<Rack::Lint::ErrorWrapper:0x007fd40d28f3f0 @error=#<IO:<STDERR>>>,
     "rack.multithread" => true,
    "rack.multiprocess" => false,
        "rack.run_once" => false,
      "rack.url_scheme" => "http",
         "rack.hijack?" => true,
          "rack.hijack" => #<Proc:0x007fd40d28f710@/Users/dark_wing0711/.rvm/gems/ruby-2.3.3@eh/gems/rack-2.0.3/lib/rack/lint.rb:525>,
       "rack.hijack_io" => nil,
         "HTTP_VERSION" => "HTTP/1.1",
         "REQUEST_PATH" => "/",
       "rack.tempfiles" => []
}
```

This environment variable is generated by WEBrick web server, which is the default web server attached with Rack. This web server is extremely slow and simple, it is capitable to handle one request at once, other requests are blocked until the first request finishes its processing cycle. There is a trick usually used by famous web servers such as Puma that they will money patch and replace WEBrick by their own implementation so that you can continue to use `rackup` command beside web server's custom starting command.

Let's back to our web application. It is normal that you soon want to add more endpoint to the application to enlarge the application logic. Rack supports a special keyword to handle such case.

```ruby
class MyAwesomeAnime
  def call(env)
    [200, { 'Content-Type' => 'text/html' }, ['Anime is awesome']]
  end
end

class MyAwesomeManga
  def call(env)
    [200, { 'Content-Type' => 'text/html' }, ['Manga is the best']]
  end
end

class NotFoundHandler
  def call(env)
    [200, { 'Content-Type' => 'text/html' }, ['Not found']]
  end
end

map '/anime' do
  run MyAwesomeAnime.new
end

map '/manga' do
  run MyAwesomeManga.new
end

run NotFoundHandler.new
```

It's kinda straight forward about the logic of application now. `map` keyword initializes a special middleware, called `Rack::Routes`. That middleware takes the responsibility to check for the request path based on `PATH_INFO` of the rack object. If the request matches, it handles the request with the logic inside the block. Otherwise, it continues with the next middleware in the main brach. From now, we could see a rack application as a tree rather than a sequence of middlewares. That's the implementation of Rack's standard middleware. What if I want to implement my own middleware? Let's start with a simple use case. Your application gets the attention and expands to multiple countries. However, it is sad that some country has a super strict cencor policy that prohibits the words `anime` and `manga`. One of the obvious solution is to audit all of our handlers, but it is costly because we have a lot of handlers and it would be time-consuming to update of it. A global solution like middleware could easily solve the problem.

```ruby
class MyAwesomeAnime
  def call(env)
    [200, { 'Content-Type' => 'text/html' }, ['Anime is awesome']]
  end
end

class MyAwesomeManga
  def call(env)
    [200, { 'Content-Type' => 'text/html' }, ['Manga is the best']]
  end
end

class NotFoundHandler
  def call(env)
    [200, { 'Content-Type' => 'text/html' }, ['Not found']]
  end
end

class CensorMiddleware
  attr_reader :app

  def initialize(app)
    @app = app
  end

  def call(env)
    status, headers, contents = @app.call(env)
    censored_content = contents.map do |content|
      content.gsub(/(anime|manga)/i, '***')
    end
    [status, headers, censored_content]
  end
end

use CensorMiddleware

map '/anime' do
  run MyAwesomeAnime.new
end

map '/manga' do
  run MyAwesomeManga.new
end

run NotFoundHandler.new
```

Our `CensorMiddleware` middleware before two `Rack::Routes` middlewares, so whenever the response is generated, it steals and modifies the contents before sending back to the client. Internally, you must be surprised about how Rack structure the middlewares. The following diagram demonstrates how everything works with middleware

{:.full-image}
![Middleware diagram]({{ site.url  }}/assets/figures/rack-the-hidden-hero/middleware-diagram.jpg)

