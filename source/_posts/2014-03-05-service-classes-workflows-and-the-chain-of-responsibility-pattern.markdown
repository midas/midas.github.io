---
layout: post
title: "Service Classes, Workflows and the Chain of Responsibility Pattern"
date: 2014-03-05 15:23:10 -0600
comments: true
published: false
categories: 
- ruby
- oop
- patterns
---

## Background

I have recently been utilizing service classes to extract business logic from Rails controllers into a more appropriate and testable object.  I define a service class as an object that performs complex and possibly multi-step tasks whose operations are across many models or domains within an application.  Be careful not to confuse a service with a form or query object;  see [7 Patterns to Refactor Fat ActiveRecord Models](http://blog.codeclimate.com/blog/2012/10/17/7-ways-to-decompose-fat-activerecord-models/) for further explanation.

Once this logic is extracted from a controller it becomes much easier to test and to reuse.  Here is an example of a service class:

## A Simple Service Class

{% highlight ruby %}
class AuthenticationService

  include Wisper::Publisher

  def initialize( username, password )
    @username = username
    @password = password
  end

  def call
    user = User.where( password: hashed_password ).first

    if user
      publish :success, user
    else
      publish :failure
    end
  end

protected

  attr_reader :password,
              :username

  def hashed_password
    # some algorithm to hash password
  end

end
{% endhighlight %}


I have standardized on the `#call` method as the main entrypoint into the functionality of the service class.  I also always use a pub/sub library ([wisper](https://github.com/krisleech/wisper)) to tell the calling objects what happened.  If you try other methods, you end up with lots of if statements and strangely named methods in the service class' public API.

This is much better than executing the authenticaiton logic in the controller action or a model method.  However, almost immediately upon starting to use service objects you will think, "what if I want to compose multiple service objects to achieve a larger task?"  Let's take a look at how we might achieve this.

## Chaining Multiple Service Classes to Achieve a Workflow

{% highlight ruby %}
module Chainable

  def self.included( base )
    base.extend ClassMethods
  end

  def next_in_chain=( next_obj )
    @_next_in_chain = next_obj
    next_obj || self
  end

  alias_method :set_next, :next_in_chain=

  def next_in_chain
    @_next_in_chain
  end

 private

  def abort_chain
    @_abort_chain = true
  end

  module ClassMethods

    def chain_method( method )
      original_method = "execute_#{method}".to_sym
      alias_method original_method, method

      self.class_eval <<-RUBY
        def #{method}( *args, &block )
          @_abort_chain = false
          #{original_method}( *args, &block )
          next_in_chain.#{method}( *args, &block ) if !@_abort_chain && next_in_chain
        end
      RUBY
    end

  end

end
{% endhighlight %}
