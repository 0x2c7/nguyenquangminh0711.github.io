---
title:  "Rack, the hidden hero"
date:   2017-03-28
layout: post
description: Everything you need to know about Rack
---

You are reading this blog, I guess you are a Ruby (Ruby on Rails) developer, or at least you used to touch the Rails framework, right? So, perhaps you used to hear some terms like "framework based on Rack" or "server for Rack application", etc. Yes, we are talking about this [Rack](https://rack.github.io) in the Ruby world. I beleive that there aren't many people working with Rack in usual. Neither do I. To be honest, the last thing I did with Rack is to config a web server and add some bunch of low-level routing. However, I realise that Rack is a elegant and beautiful underneath any Ruby web frameworks. Researching Rack provides me a lot of knowledges about Ruby and deep understanding about how web framework (like Rails) works from scratch. This blog post is a place for me to note about everything I leanrt. If you have any question for me, please don't hesitate to comment. I really appreciate <3

# How does your web application handle the request?

In the Ruby on Rails project website, there is a dedicated section to talk about Rack: [Rails on Rack](http://guides.rubyonrails.org/rails_on_rack.html). In other frameworks' websites, there is at least a small part in the guide page mentioned Rack and how the framework related to Rack, for example: [Hanami](http://hanamirb.org/guides/actions/rack-integration/), [Sinatra](http://www.sinatrarb.com/intro.html#Rack%20Middleware), [Cuba](https://github.com/cuba-platform/cuba), or [Roda](https://github.com/jeremyevans/roda). Yes, nearly all the Ruby web frameworks available on the market are related to Rack. Actually, I can say that they are *based on Rack*. If you open any projects using such frameworks, it is likely that there exists a file name `config.ru`. If there aren't any, for example, in a hello world Sinatra application, the `config.ru` file is automatically interpolated in other way. Besides normal server starting commands such like `rails server -p 3000` or `puma -p 3000`, you can start your applications with the command `rackup`. Let's give it a try and perhaps you will be suprised. This is one of the most significant evidences for the existence of Rack.

Before I continue with Rack, I want to recall the definition of web server. By definition from Wikipedia, web servers are the software that accepts and supervises the HTTP requests. Yup, a web server listens continuously to the host and the port that it is working on. If there is a request waiting, it accepts the request if possible, then calls something else to handle that request and get the result, then it sends the response back to the client. Its job is just trying to receive and respond as fast as possible. All the logic about how to generate the response for a specific request belongs to something else. The one who handle the real logic could be just a simple script writen in Perl, or some bunch of PHP files, or a large and full-component web framework like Ruby on Rails. In the Ruby world, there are some famous web servers: Puma, Thin, Webrick, Passenger, etc.

Now let's back to our hero. Let's open a random Rails application and look at the `config.ru` file. For example: this is a typical Rails application's `config.ru` file. The `.ru` extension of the file comes from `rackup`. As its name, this file stores the configurations that define how your web application is structured.

```ruby
require_relative 'config/environment'

run Rails.application
```

Since Rails does all the thing, the configuration file of a Rails application is super simple and obvious. This is a more complicated configuration (disclamer: I know, this is ugly. I won't use it in production)

```ruby
use ExceptionLoggingMiddleware

map '/' do
  use ApiCORSMiddleware

  map '/v1' do
    run ApiApplication
  end
end

run MainApplication
```

You must wonder what language is that. Actually, this file is parsed and eval by Rack. All the above code is just Ruby. Rack provides us some DSLs: `use`, `map` or `run`. I'll talk about those DSLs later. Now you just assume those are used to define the application structure. You can add some bunch of Ruby codes redirectly or require from other files / gems. This makes the `config.ru` file flexible and easy to config. However, it also makes it hard to mantain if you are not careful. The most important command here is the `run` command. It decides who takes responsibility to handle the logic for a branch of requests or all requests. That is where your favourite framework takes place. All the frameworks implement an interface to plug into the Rack application. This interface is called *middleware*. In fact, Rack structures everything in middlewares. I'll talk about this later. For now, you could think the middleware as an ordered list of components.

At an early step of the server starting process, a *Rack application* is created by parsing this configuration file. This object is delegated to the web server. Whenever there is an incoming request, the web server handles it, and prepares an object called *environment* that contains the request needed information like headers, body, host, port, etc and some meta data that the web server wants Rack to use. The specification of environment object is defined at [Rack spec](http://www.rubydoc.info/github/rack/rack/master/file/SPEC), you should take a look. Then, the web server passes the environment object to the Rack application to handle. In the internal of Rack application, the environment object is passed through the middlewares. A middleware could transform the environment object before it goes to next middle.

![Rack working flow 1]({{ site.url  }}/assets/figures/rack-the-hidden-hero/work-flow-1.jpg)

When the environment object reaches the end of the middleware chain, the last middleware (usually your web framework) does all the magic thing and generates the real response for the HTTP request. This response is an array of three elements: response status, headers and body. The response pops up in the inverse order from the last middleware back to the first middleware. Again, each middleware in the backtrace path could transform the response before it returns the response to the previous middleware. Finally, Rack application returns the response back to the web server. Web server converts the response into valid HTTP format and sends back to the client.

![Rack working flow 2]({{ site.url  }}/assets/figures/rack-the-hidden-hero/work-flow-2.jpg)

# Middleware, the heart of Rack

NotImplementedError

# How does Rack build your application?

NotImplementedError

# Web server layer

NotImplementedError

<!-- # Build app from config.ru -->
<!--   Top - down strategy -->
<!--  -->
<!--   If meets map => add to mapping -->
<!--   If meets use -->
<!--     If mapping exists -->
<!--       Convert the mapping to URLMap middleware and add to use list -->
<!--     Add the current middleware to use list -->
<!--   If meets run -->
<!--     Set main handler for that builder partial -->
<!--  -->
<!--   Final build step: -->
<!--     - If mapping exists. Create URLMap wrapp main handler -->
<!--     - Otherwise, use main handler directly -->
<!--  -->
<!--     Attach that result into the middlewares -->

<!-- # Inject default middlewares -->
<!--   Rack::ContentLength -->
<!--   Rack::Chunked -->
<!--   Rack::CommonLogger -->
<!--   Rack::ShowExceptions -->
<!--   Rack::Lint -->
<!--   Rack::TempfileReaper -->
<!--   Rack provides me a lot knowledges about the hidden world b -->
