---
date: 2014-02-12T00:00:00
draft: false
title: "Gomu: Introduction"
tags: ["gomu", "golang", "web"]
categories: ["programming"]
authors: ["pksunkara"]
---

I have recently started tinkering with the [Go][] programming language and I am impressed by its potential.

[Go][] is better than some of the major programming languages at many tasks like concurrency, run time, memory, etc. In my opinion, its ease of deployment ranks high on that list. Everything gets compiled into a single binary and it's just a matter of uploading it.

I have used Go for my recent project [Alpaca](https://github.com/pksunkara/alpaca) which given a web api, generates client libraries in ruby, python, node and php. You can use a tool called [goxc](https://github.com/laher/goxc) to build binaries for all the important operating systems and architectures. All users have to do is download the appropriate binary and start running it.

I can definitely say that [Go][] is my new hammer.

I have been entertaining the thoughts of building an easy-to-use web framework for a long time. I contributed to the development of [flatiron](https://flatironjs.org) and learned a lot. This is a big nail I am going to use my new hammer on.

## Why use Go?

[Go][] is faster and takes less memory than ruby/node and it will only become better in the future.

## Why another framework?

Most of the programmers who use [Go][] would prefer to develop using a few utility libraries rather than using a web framework. **This is not for them**. This is for people who want to rapidly develop a website leveraging the benefits provided by [Go][].

There are a few good attempts at this like [Martini](https://github.com/codegangsta/martini), [Revel](http://robfig.github.io/revel) and [Beego](http://beego.me). But I am not satisfied with any of them. Martini is more similar to [Sinatra](https://sinatrarb.com) and [Express](https://expressjs.org) rather than [Rails](https://rubyonrails.org) therefore requiring a lot more effort from developer.

## Why are you saying all this?

This blog is going to be my journal for ideas and discussions about the web framework project design and execution. I also hope to list the necessary requirements and pain points in the cycle of web application development.

## What kind of name is Gomu?

I wanted to use the letters *go* and **"Gomu"** means rubber in Japanese. It fits since I plan to make the framework very flexible and opinionated.

[Go]: https://golang.org
