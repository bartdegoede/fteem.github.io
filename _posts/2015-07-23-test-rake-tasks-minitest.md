---
layout: post
title: "Test Rake tasks with Minitest"
tags: [testing, minitest, rake, tasks]
---

Notes:

- https://robots.thoughtbot.com/test-rake-tasks-like-a-boss
- http://edelpero.svbtle.com/homemade-decorators
- http://edelpero.svbtle.com/everything-you-always-wanted-to-know-about-writing-good-rake-tasks-but-were-afraid-to-ask

Rake is a Make-like program implemented in Ruby. Tasks and dependencies are specified 
in standard Ruby syntax. It's a really neat tool that you should have under your belt.
Let's see why and how we can test our Rake tasks.


## But, why?

Yeah, it's a legit question. You can always say "I already tested my classes!".
But, there are couple of reasons why you should always test your Rake tasks:

1. Rake tasks is code. And all code should be tested.
2. Rake tasks can use any part of your app. Models, service classes and what not. If one of the classes that the Rake task relies on changes, you have to know if it will break it. 
3. Rake tasks can do heavy lifting. Perhaps you have a cron on Heroku that runs an 
email campaign that calls a Rake task. Or you generate reports with a Rake task. This
is important and you need to know if it actually works.
4. Forgetting about your Rake tasks is easy. Or, if you inherit a codebase it's easy
to not even notice them when beginning the project. Rake tasks are code as well 
and breaking it is easy.

So, yeah, test your Rake tasks!

## Our Rake task

For example, we have an application where users pay subscription for whatever reason. 
The application has a Rake task that notifies users that their subscription will expire
in seven days. 

{% highlight ruby %}
# lib/tasks/users/notify.rb
namespace :users do
  desc "Notify users 7 days before subscription expires"
  task :notify do
    Subscription.expires_in_seven_days.each |subscription|
      UserMailer.notify_subscription_expiry(subscription.user)
    end
  end
end
{% endhighlight %}

This task is run with as a cron job. Let's see how we can test it.

## Testing with Minitest

Okay, if you got to this point, I guess you agree with me. Lately, I prefer testing
with Minitest. I like it because it's really tiny, quite verbose, magic-less and 
it's pure Ruby. 

We will take for granted that the Subscription model and the UserMailer are already tested.
Our next objective is to test this Rake task.

First, we need to setup the tests. Whenever you are testing Rake tasks, you need to 
create a new Rake Applicaiton object. 

{% highlight ruby %}
# test/lib/user_notify_test.rb
class UserNotifyTest < Minitest::Test
  def setup
    rake      = Rake::Application.new
    task_name = 'notify'
    task_path = File.expand_path "../../lib/tasks/users/#{task_name}", __FILE__
    @task = rake[task_name]

    Rake.application = rake
    Rake.application.rake_require(task_path)
  end

  def test_notification_is_sent
    subscription = Minitest::Mock.new
    user = Minitest::Mock.new
    subscription.expect(:user, user)

    Subscription.stub :expires_in_seven_deys, [subscription] do
      UserMailer.stub :notify_subscription_exipry, true
    end

    @task.invoke
    subscription.verify
    user.verify
  end
end
{% endhighlight %}

