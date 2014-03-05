---
layout: post
title: "Ruby, OOP, Events and the Tell Don't Ask Principle"
date: 2014-03-04 22:11
comments: true
categories: 
- ruby
- oop
---

## Background

Ignoring the existence of events in a program leads to harder to understand/debug code.  Events implicitly exist in all programs and if not explicitly utilized do not go away. 

It is obvious that events can help you write loosely coupled class and allow multiple objects to subscribe to a single event.  However, events can also help you conform to the Tell, Don't Ask Principle.

## Tell Don't Ask Principle

Instead of asking an object a qustion about it's state, making a descision, and proceeding forward we should strive to tell an object what to do.

> Procedural code gets information then makes decisions. Object-oriented code tells objects to do things. 
> 
> — Alec Sharp

Let's look at some simple examples of a Rails controller action:

 using the [wisper](https://github.com/krisleech/wisper) gem.
#### Normal Rails Controller

Here is an example you are sure to recognize as it is the normal way of writing a create action for a Rails controller.

{% highlight ruby %}
class TasksController < ApplicationController

  def create
    @task = Task.new(params[:task])

    if @task.save
      respond_to do |format|
        format.html { redirect_to task_path( @task ) }
      end
    else
      respond_to do |format|
        format.html { render :new }
      end
    end
  end

end
{% endhighlight %}

This implementation is not good because we are examining the state of the created task to determine what to do next.  Not to mention, lots of complex logic, most of which has nothing to do with the HTTP protocol, is happening in the controller.

#### Rails Controller Action With a Service Class Extracted

Now, we will extract that create logic to a service class the controller can use.

{% highlight ruby %}
class CreateTaskService

  attr_reader :task

  def initialize(attributes)
    @attributes = attributes
  end

  def call
    @task = Task.create(attributes)
  end

  def success?
    task.valid?
  end

protected

  attr_reader :attributes

end
{% endhighlight %}

{% highlight ruby %}
class TasksController < ApplicationController

  def create
    create_service.call

    if create_service.success?
      task = create_service.task
      respond_to do |format|
        format.html { redirect_to task_path(task) }
      end
    else
      respond_to do |format|
        format.html { render :new }
      end
    end
  end

protected

  def create_service
    @create_service ||= CreateTaskService.new(params[:task])
  end

end
{% endhighlight %}

While we know that extracting complex logic out of the controller into another class is a good idea, this particular implementation is just plain ugly.  The root of the problem with this implementation is the controller is asking the service class about it's state and then telling it to do additional tasks.

Why not let the service class tell the controller what happened?

{% highlight ruby %}
class CreateTaskService

  include Wisper::Publisher

  def initialize(attributes)
    @attributes = attributes
  end

  def call
    task = Task.create(attributes)

    if task.valid?
      publish :success, task
    else
      publish :validation_error, task
    end
  end

protected

  attr_reader :attributes

end
{% endhighlight %}

{% highlight ruby %}
class TasksController < ApplicationController

  def create
    create_service.on :success do |task|
      respond_to do |format|
        format.html { redirect_to task_path(task) }
      end
    end

    create_service.on :validation_error do |task|
      respond_to do |format|
        format.html do 
          @task = task
          render :new
        end
      end
    end

    create_service.call
  end

protected

  def create_service
    @create_service ||= CreateTaskService.new(params[:task])
  end

end
{% endhighlight %}

This is a much better implementation.  Without sacrificing the relatively linear flow of the code, we have obeyed the tell, don't ask principle.  In addition, we have explicitly acknowledged the presence of events, which makes the code easier to understand.

The ease of comprehension benefit may not seem like a big advantage in the prior example.  However, imagine the case where you are 5 levels deep in an object graph that is employing a strategy pattern.  Without the use of events to message directly to the outer most object  your only option is proxy methods.  Good luck tracing that quickly.

As a bonus, things get even better if we need some orthogonal piece of work to occur when the task is successfully created:

{% highlight ruby %}
class TasksController < ApplicationController

  def create
    create_service.subscribe( task_email_listener,
                              on: :success,
                              with: :task_created )

    create_service.on :success do |task|
      # ...
    end

    create_service.on :validation_error do |task|
      # ...
    end

    create_service.call
  end

protected

  def create_service
    @create_service ||= CreateTaskService.new(params[:task])
  end

  def task_email_listener
    TaskEmailListener.new
  end

end
{% endhighlight %}

{% highlight ruby %}
class TaskEmailListener

  def task_created( task )
    TaskMailer.assignment_email( task ).deliver
  end

end
{% endhighlight %}

Wow, we have now accomplished sending an email without tightly coupling it to the `CreateTaskService` or using ActiveRecord callbacks.  This means the `CreateTaskService` can be used other places in our code without sending an email.  This also means when we create a Task in the Rails console, we will not accidentally send an email.
