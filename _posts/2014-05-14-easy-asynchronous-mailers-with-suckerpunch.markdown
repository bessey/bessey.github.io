---
layout: post
title: "Easy SuckerPunch Asynchronous Mailers "
date: 2014-05-14 23:27
comments: true
categories: ruby rails suckerpunch
---

For a Rails side project recently I finally had the need for a background task handler. Now traditionally I'd jump at [Sidekiq](http://sidekiq.org/), and for good reason. It's sexy, relatively simple, and comes with a ton of niceties like automatic exponential back-off retries. For this project however, at least at this stage, Sidekiq is overkill. We just need to run some ActionMailers out of the render loop, so we'd like to avoid booting up another server if possible.

So, in comes [SuckerPunch](https://github.com/brandonhilkert/sucker_punch), a gem built to address exactly this issue. SuckerPunch uses [Celluloid](https://github.com/celluloid/celluloid/) to operate a barebones background task handler *inside the server process*, avoiding the significant memory overhead of running two Rails instances. Now while its worth noting thanks to the Global Interpreter Lock present in MRI, CPU intensive jobs are still liable to slow down your requests, however the GIL does not apply to external I/O, and so the bulk of what makes a mailer slow (negotiating SMTP) **doesn't block the server**.

<!-- more -->

This is great! So now we just need to write a load of jobs for each mailer method we want to run in the background, like so:

{% highlight ruby %}
# jobs/user_mailer_job.rb
class UserRegistrationMailerJob
  include SuckerPunch::Job

  def perform(user)
    UserMailer.registration(user).deliver
  end
end
{% endhighlight %}

### Refactor, refactor, refactor!

Clean as this DSL is, its clearly going to get unwieldy. It would be much nicer to be able to have a job that could handle *any* email for *any* mailer. Fortunately this isn't too hard to achieve with just a pinch of meta-programming:

{% highlight ruby %}
# jobs/async_mailer_job.rb
class AsyncMailerJob
  include SuckerPunch::Job

  # Enables us to turn any mailer into an asyncronous one
  def perform(mailer, method, *args)
    Rails.logger.info("Asyncronously running #{mailer.to_s}.#{method.to_s}")
    mailer.send(method, *args).deliver
  end
end
{% endhighlight %}

Now we can slightly modify our existing mailer calls slightly to achieve the same effect as before, without a specific job for each of them:

`AsyncMailerJob.new.async.perform(UserMailer, :registration, user)`

### But wait, there's more!

Hmmm, it seems we can do a little better than that though. The syntax is a little ugly, wouldn't it be nice if all we had to do to run a mailer asynchronously was (like with SuckerPunch jobs) add an `.async` call before calling the message method? If we knock the meta-programming dial up a notch this isn't actually too difficult. First up we need to build a module that defines the `.async` method on a mailer, lets not worry about whats going in there for now:

{% highlight ruby %}
# async_mailer.rb

module AsyncMailer
  module ClassMethods
    def async
      AsyncMailerJobRunner.new(self)
    end
  end

  def self.included(base)
    base.extend ClassMethods
  end

end
{% endhighlight %}

Nothing too crazy yet, just using the `included` callback to allow a module to define methods on the class (as opposed to instances of the class). So what's going in `async`? Lets look at the `AsyncMailerJobRunner` to find out:

{% highlight ruby %}
# async_mailer_job_runner.rb
class AsyncMailerJobRunner
  def initialize(mailer)
    @mailer = mailer
  end

  def method_missing(meth, *args, &block)
    AsyncMailerJob.new.async.perform(@mailer, meth, *args)
  end
end
{% endhighlight %}

Remember `async` has to return an object that **acts like a mailer** while actually running the mailer commands asynchronously. To do that, it effectively mocks a mailer, delegating all messages not to the mailer, but instead to the `AsyncMailerJob` we wrote earlier. Thanks to `method_missing` we don't have to worry about which mailer method is called, and thanks to splat `*args` we don't have to worry about that method's footprint. Meta-programming at its most finest!

I hope all that was helpful to you. If you see any improvements or errata please throw me a comment! For the lazier here's the source all together in its entirety:

{% highlight ruby %}
# mailers/async_mailer.rb

# Gives an ActionMailer the ability to run any method asynchronously by simply
# prepending an .async call.
#
# Example:
#   MyMailer.async.new_user_email(user)

module AsyncMailer
  # Takes care of transforming an ActionMailer method call into a SuckerPunch
  # perform call.
  #
  # Example:
  #   MyMailer.new_user_email(user)
  # becomes
  #   AsyncMailerJob.new.async.perform(MyMailer, :new_user_email, user)

  class AsyncMailerJobRunner
    def initialize(mailer)
      @mailer = mailer
    end

    def method_missing(meth, *args, &block)
      AsyncMailerJob.new.async.perform(@mailer, meth, *args)
    end
  end

  class AsyncMailerJob
    include SuckerPunch::Job

    # Enables us to turn any mailer into an asyncronous one
    def perform(mailer, method, *args)
      Rails.logger.info("Asyncronously running #{mailer.to_s}.#{method.to_s}")
      mailer.send(method, *args).deliver
    end
  end

  module ClassMethods
    def async
      AsyncMailerJobRunner.new(self)
    end
  end

  def self.included(base)
    base.extend ClassMethods
  end

end
{% endhighlight %}

Enjoy!
