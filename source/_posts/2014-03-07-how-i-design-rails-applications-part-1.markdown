---
layout: post
title: "How I Design Rails Applications: Part 1"
date: 2014-03-07 18:56:30 -0600
comments: true
published: true
categories: 
- oop
- patterns
- rails
- ruby
---

I have been thinking and experimenting with different techniques to make Rails applications manageable as they grow for about 2 years now.  This exercise is not because I desire to add layers/complexity to my Rails applications, but because I have experienced the pain that a Rails application can provide as it matures.  In this multi-part series I will break down some of the successful techniques I am using to achieve my goals.

One of my starting places was this [article by Bryan Helmkamp](http://blog.codeclimate.com/blog/2012/10/17/7-ways-to-decompose-fat-activerecord-models/).  I will often reference this article in this series.  Thanks Bryan for sharing your techniques.

I agree with Bryan the suggestion to make "fat models" is not a best practice.  The model should not contain logic that does not directly correspond to reading/writing to the database.

## Part 1

The first technique Bryan discusses in his article is to "Extract Value Objects."  According to Bryan, value objects are, "simple objects whose equality is dependent on their value rather than an identity."  This is an important conecpt taken from domain driven design.  In his book on DDD, Eric Evans states: "Many objects have no conceptual identity.  These objects describe characteristics of a thing."

A great example of a value object is color:

{% highlight ruby %}
class Color
  include Comparable

  def initialize(key)
    @key = key
  end

  def <=>(other)
    other.to_s <=> to_s
  end

  def eql?(other)
    to_s == other.to_s
  end

  def to_s
    @key.to_s
  end
end
{% endhighlight %}

## Value Objects in the Model

It is common practice for developers to define value objects in a model, usually as a constant in the model.

{% highlight ruby %}
class Car < ActiveRecord::Base
  COLOR = %w(
    black
    blue
    green
  )
end
{% endhighlight %}

This practice is not good because the value object may be used in attribute(s) of a model, but has nothing directly to do with the domain the model is representing.  Defining value objects inline in the model class carries the at least following disadvantages:

* Impossible to add functionality to value object without further polluting the model
* Limits reuse of the value object
* Harder to test

## Value Objects in the Database

Quite often, developers will include value objects in the database as a model.  This is also bad form.  The value object is not dynamic so it does not necessitate use of the database.  In addition, the value object has no identity, so we do not need a keying system.  The only value the database really provides in this case is the limiting of the valid values for a value object to some predefined set that can be enforced.

## Enumerative Gem

Enter the [enumerative gem](https://github.com/ninja-loss/enumerative).  This gem was authored by [Nils Jonsson](https://github.com/njonsson) and myself.  The enumerative gem provides the tools necessary to create value objects that are limited to a finite set of valid values.

An enumeration for color might be implemented like:

{% highlight ruby %}
class Color

  def self.valid_keys
    %w(
      black
      blue
      green
    )
  end

  include Enumerative::Enumeration

end
{% endhighlight %}

In addition, you must add the translations for the enumeration's values to the `config/en.yml` file:

{% highlight ruby %}
en:
  enumerations:
    color:
      black: Black
      blue: Blue
      green: Green
{% endhighlight %}

Now you can use the enumeration.

{% highlight ruby %}
Color::BLACK # #<Color:0x000001015e6aa8 @key="black">
Color::BLACK.key # "black"
Color::BLACK.value # "Black"
Color.to_select # [["Black", "black"], ["Blue", "blue"], ["Green", "green"]]
Color.new('black').valid? # true
Color.new('some invalid value').valid? # false
{% endhighlight %}

The initiatlizer is very forgiving of the values it will accept so that it can be used to easily standardize input, etc.

{% highlight ruby %}
Color.new(Color::BLACK) # #<Color:0x000001015e7aa0 @key="black">
Color.new('black') # #<Color:0x000001015e7aa0 @key="black">
Color.new(key: 'black') # #<Color:0x000001015e7aa0 @key="black">
Color.new(key: :black) # #<Color:0x000001015e7aa0 @key="black">
{% endhighlight %}

####  HasEnumeration Module

It gets even more exciting when you want to use it in a model.  If you included the `Enumerative::HasEnumeration` module you get automatic casting from the key that is stored in the database to the type of the enumeration your attribute is defined as on a read and back to the key for storage on a write.

{% highlight ruby %}
class Car < ActiveRecord::Base
  include Enumerative::HasEnumeration

  has_enumeration color, from: Color
end
{% endhighlight %}

Convenience when you read from the database:

{% highlight ruby %}
car = Car.first
car.color_before_type_cast # "black"
car.color # #<Color:0x000001015e7aa0 @key="black">
{% endhighlight %}

Convenience when you write to the database

{% highlight ruby %}
car = Car.new
car.color = Color::BLACK
color.save!
car.reload
car.color_before_type_cast # "black"
car.color # #<Color:0x000001015e7aa0 @key="black">
{% endhighlight %}

#### Testing Enumerations

Enumerative provides some a shared spec for easier specing of enumerations.  Following is an example spec for the color enumeration.

{% highlight ruby %}
require 'spec_helper'
require 'enumerative/enumeration_sharedspec'

describe Color do

  it_should_behave_like 'an Enumeration'

  def self.keys
    %w(
      black
      blue
      green
    )
  end

  self.keys.each do |key|
    const = key.upcase

    it "should have #{const}" do
      described_class.const_get( const ).should be_valid
    end
  end

  it "should have the correct select-box values" do
    described_class.to_select.should == [
      ["Black", "black"],
      ["Blue", "blue"],
      ["Green", "green"]
    ]
  end
end
{% endhighlight %}

## Project Organization

I recommend placing enumerations in the `app/enumerations` directory.

## Conclusion

With very little effort we have extracted our value object out of the database and retained the ability to limit the valid values to a finite set.  We can also use our value object in a model and persist the model's value for the enumeration in the database.  Most importantly, we have successfully removed a common contributer to making Rails models bloated and unmanageable.

Stay tuned for the next installment in the series.
