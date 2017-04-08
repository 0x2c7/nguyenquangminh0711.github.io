---
title:  "Rack, the hidden hero"
date:   2017-03-28
layout: post
description: Everything you need to know about Rack
---

You are reading this blog, I guess you are a Ruby (Ruby on Rails) developer, or at least you used to touch the Rails framework, right? So, perhaps you used to hear some terms like "framework based on Rack" or "server for Rack application", etc. Yes, we are talking about this [Rack](https://rack.github.io) in the Ruby world. I believe that there aren't many people working with Rack in usual. Neither do I. To be honest, the last thing I did with Rack is to config a web server and add some bunch of low-level routing. However, I realize that Rack is an elegant and beautiful underneath any Ruby web frameworks. Researching Rack provides me a lot of knowledge about Ruby and deep understanding about how web framework (like Rails) works from scratch. This blog post is a place for me to note about everything I learned. If you have any question for me, please don't hesitate to comment. I really appreciate <3

# What is Rack?

In the Ruby on Rails project website, there is a dedicated section to talk about Rack: [Rails on Rack](http://guides.rubyonrails.org/rails_on_rack.html). In other frameworks' websites, there is at least a small part in the guide page mentioned Rack and how the framework related to Rack, for example, [Hanami](http://hanamirb.org/guides/actions/rack-integration/), [Sinatra](http://www.sinatrarb.com/intro.html#Rack%20Middleware), [Cuba](https://github.com/cuba-platform/cuba), or [Roda](https://github.com/jeremyevans/roda). Yes, nearly all the Ruby web frameworks available on the market are related to Rack. Actually, I can say that they are **based on Rack**. If you open any projects using such frameworks, it is likely that there exists a file name `config.ru`. If there aren't any, for example, in a hello world Sinatra application, the `config.ru` file is automatically interpolated in other way. Besides normal server starting commands such like `rails server -p 3000` or `puma -p 3000`, you can start your applications with the command `rackup`. Let's give it a try and perhaps you will be surprised. This is one of the most significant evidence for the existence of Rack.

By the self-definition of the Rack team: "Rack provides a minimal interface between webservers that support Ruby and Ruby frameworks.". Before talking anything about this definition, I want to recall the definition of the web server. By definition from Wikipedia, web servers are the software that accepts and supervises the HTTP requests. Yup, a web server listens continuously to the host and the port that it is working on. If there is a request waiting, it accepts the request if possible, then calls something else to handle that request and get the result, then it sends the response back to the client. Its job is just trying to receive and respond as fast as possible. All the logic about how to generate the response for a specific request belongs to something else. The one who handles the real logic could be just a simple script written in Perl, or some bunch of PHP files, or a large and full-component web framework like Ruby on Rails.

Back to the definition of Rack, it is obvious that Rack lays between the Ruby-supported web servers and the Ruby web framework. Web servers take of handling raw requests, web frameworks rake care of the logics to generate the response for the HTTP requests. What does Rack do between them? Why don't we just let the web servers talk directly to the web frameworks? Actually, yes, they can talk directly to each other. However, everything soon becomes chaotic. In the Ruby world, there is a bunch of famous web servers: Puma, Thin, Webrick, Passenger, etc. and even more web frameworks: Rails, Cuba, Roda, Hanami, Sinatra, etc. A web server, for example, Puma, wants to run Rails, they must have the same standard so that Puma can pass something that Rails understands and Rails returns a format that Puma understands. Soon, if Puma wants to support more frameworks, it must implement the adapters for each framework. You can imagine the efforts putting in the development process and maintaining the compatibilities among them are extremely high. That's why Rack exists. It provides a common **standard interface**. The web servers and web frameworks just need to follow the standard and they can talk to each other.

Usually, at an early step of the server starting process, a **Rack application** is created based on the configs of the user. These configurations are usually stored in a file name `config.ru` which I mentioned above. For each incoming request, the web server prepares an object called **environment** that contains the request needed information like headers, body, host, port, and some data that the web server wants Rack to use. The specification of environment object is defined at [http://www.rubydoc.info/github/rack/rack/master/file/SPEC](http://www.rubydoc.info/github/rack/rack/master/file/SPEC), you should take a look. This environment object is passed into the Rack application. Inside that Rack object, the environment object goes through a **middleware chain**. Middleware is a separated stage to process the request. The input of each middleware is the environment object, the output is an array of three elements: response status, a hash of response headers and response body. Each middleware takes a single responsibility. It could process the request for real by using the information from environment object and return the response by itself or call the next middleware in the middleware chain to handle. Before passing to next middleware, it could enhance the environment object with useful information such as fetching session from HTTP cookie header, uploading the file and provides the file path, etc. It can also enhance the response by adding more response headers, transform the body or even replace the whole response. Some middlewares do not transform anything, it just does the monitoring or logging works. Perhaps you will be surprised to know that your favorite framework is actually a middleware in the middleware chain. Yes, a big one. Usually, it stays at the end of the middleware chain. Of course, if the request is caught and handled by a middleware, it won't reach the next middlewares in the chain. When the environment object stops at an execution middleware (usually at the end of the chain, which is the web framework), the response is generated and it pops up back to the beginning of the chain. At the end, the web server receives the full response and converts the response into valid HTTP format and sends back to the client.

In the figure, let's assume that there are 3 middlewares (in fact, there are usually 10 - 20 middlewares in a Rails application). The request from the user is parsed and passed to the Rack application under environment object (we can consider it as a hash). After going through Warden ([Warden authentication framework](https://github.com/hassox/warden)), a new field is added to the environment object and the next middleware receives it. When reaching Rails, some magic happens and the response is generated. The response pops back to the beginning of the middleware chain. When it reaches Rack::ContentLength, a response header `Content-Length` is added. Finally, the response array reaches the web server and it is transformed into valid HTTP response and sent back to the client.

![Rack working flow 1]({{ site.url  }}/assets/figures/rack-the-hidden-hero/work-flow-1.jpg)

This next example is just like above except the Warden middleware stops the environment and returns the responses right away. The framework doesn't receive the environment object.

![Rack working flow 2]({{ site.url  }}/assets/figures/rack-the-hidden-hero/work-flow-2.jpg)

# Middleware, the heart of Rack

# How does Rack build your application?

NotImplementedError


Let's open a random Rails application and look at the `config.ru` file. For example: this is a typical Rails application's `config.ru` file. The `.ru` extension of the file comes from `rackup`. As its name, this file stores the configurations that define how your web application is structured.

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

To define the structure of your web application, Rack provides us some DSLs: `use`, `map` or `run`. The details of those DSL will be described in next few sections. In fact, you can add some bunch of Ruby codes redirectly or require from other files / gems. This makes the `config.ru` file flexible and easy to config. However, it also makes it hard to mantain if you are not careful. The most important command here is the `run` command. It decides who takes responsibility to handle the logic for a branch of requests or all requests. That is where your favourite framework takes place. All the frameworks implement an interface to plug into the Rack application. This interface is called **middleware**.

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
