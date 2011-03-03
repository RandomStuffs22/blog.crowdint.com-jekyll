---
layout: post
title: How to make your Models look lean
author: Luis Galaviz
email: luis.galaviz@crowdint.com
avatar: 1e7f8fb8733b6193cf3bdbc85693f515
---

Rails is strongly based in the MVC concept, that means each part of
this framework is built to work smoothly under this concept.
Conceptually, we understand what we have to deliver in each case.
With dumb Views, lean RESTful Controllers and FAT Models(?) we can build
whatever we want, or more precisely, whatever we need.

Views and Controllers rely on the Model to deliver the business logic. Our Models,
empty at the beginning, slowly becomes into big monsters sometimes difficult to
maintain, scale, and usually, forget.
So with all that code, our *Models become FAT, and therefore we have to put
them on a diet sometimes*.

A common practice in order to get our models as Lean as possible is *extracting from
them all the duplicated code into modules*.
This way we can make modules that contains our common methods, scopes, method calls, etc.

For example, we have a Model named Product as follows:

{% highlight ruby %}
class Product < ActiveRecord::Base

  # Some shared behavior code.

  scope :active, where(:state => "active")

  state_machine :state, :initial=>:active do
    state :active
    state :disabled
    state :deleted

    event :activate do
      transition all => :active
    end

    event :disable do
      transition all => :disabled
    end

    event :soft_delete do
      transition all => :deleted
    end
  end

  def self.activate_all
    # This method Activates all the products.
  end

  def self.disable_all
    # This method Disables all the products.
  end

  def destroy
    self.soft_delete
  end
  
  # Below this point there is some code for the Product specific behavior...
end
{% endhighlight %}

Let's assume we have Type and Category models that shares the Product shared behavior.
So we have duplication along the three models.
**One of RoR rules is DRY and therefore we could send the duplicate code into modules**.

{% highlight ruby %}
class Product < ActiveRecord::Base
  extend CommonClassMethods
  include CommonInstanceMethods

  # Shared behavior code.

  scope :active, where(:state => "active")

  state_machine :state, :initial=>:active do
    state :active
    state :disabled
    state :deleted

    event :activate do
      transition all => :active
    end

    event :disable do
      transition all => :disabled
    end

    event :soft_delete do
      transition all => :deleted
    end
  end

  # Below this point there is some code for the Product specific behavior...
end

module CommonClassMethods

  def activate_all
    # This method Activates all the products.
  end

  def disable_all
    # This method Disables all the products.
  end
end

module CommonInstanceMethods
  def destroy
    self.soft_delete
  end
end
{% endhighlight %}

This way we can reuse all the class methods extending the CommonClassMethods
module and the instance methods including the CommonInstanceMethods module in our
models.
Anyway, we still have the state machine and the active scope duplicated.
What could we do to avoid duplication in that case? Well, Rails provides us a
great tool trough the *ActiveSupport::Concern*.

The ActiveSupport::Concern has a method named **included** where we can include all
the common behavior like scopes, validations or method calls.

{% highlight ruby %}
class Product < ActiveRecord::Base
  include CommonBehavior

  # Below this point there is some code for the Product specific behavior...
end

method CommonBehavior
  extend ActiveSupport::Concern

  included do
    scope :active, where(:state => "active")

    state_machine :state, :initial=>:active do
      state :active
      state :disabled
      state :deleted

      event :activate do
        transition all => :active
      end

      event :disable do
        transition all => :disabled
      end

      event :soft_delete do
        transition all => :deleted
      end
    end
  end
  
  module ClassMethods
    def activate_all
      # This method Activates all the elements.
    end

    def disable_all
      # This method Disables all the elements.
    end
  end

  module InstanceMethods
    def destroy
      self.soft_delete
    end
  end
end
{% endhighlight %}

By extending the ActiveSupport::Concern we can include our states and active scope
in the CommonBehavior module, and reuse them.
Also, we include the instance methods and extend the class methods just including the CommonBehavior
module into our Product model. Those methods also looks more readable by putting
them into modules named ClassMethods and InstanceMethods.

*Great! Now our models looks more Lean*.

*WARNING!*: We have to be careful when using modules. Remember to initialize them to
make them available for your classes in *development* mode, this is because the
development environment loads the classes in a lazy mode!.

You can use something like this in your **"~development.rb"** to pre-load this modules:

{% highlight ruby %}
Dir[Rails.root.join("lib", "*.rb")].each {|f| require f}
{% endhighlight %}
