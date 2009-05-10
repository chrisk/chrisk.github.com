---
layout: post
title: How to prevent database contention in continuous integration
---

We've used a few different continuous integration stacks for Rails over the last year at work—first [CruiseControl.rb](http://cruisecontrolrb.thoughtworks.com), which we found a little too complex to administer, then a custom bash script (which worked well, but took a lot of tweaking to get just right). When we eventually switched to git last year, we took the opportunity to try [Integrity](http://integrityapp.com), a cute lil' Sinatra app.

Integrity mostly "just works," and it's been a happy switch. One thing we lost in the move, though, was code that protected against resource contention when two builds are running at once. This is definitely a problem with Rails, since a typical `database.yml` tells Active Record to use the same database for all test runs. So you've got multiple builds hitting the database at once, dropping tables, creating records, and so on. Yikes.

Our old bash script used a filesystem lock and a queue to only run one build at a time, in order. In theory, this is the most sound approach, but hey—our build server has 8 cores and 16 GB of RAM, plenty of room for parallelism. During a pair-programming session this week with [Jared Grippe](http://www.jaredgrippe.com), we decided that the best approach is to solve the contention issues and allow multiple simultaneous builds. We figured that'd keep the [rapid feedback](http://martinfowler.com/articles/continuousIntegration.html#KeepTheBuildFast) up-to-speed when the commits are flying in.

1. Stop putting code in that little "build script" box in Integrity's configuration page. Instead, drop it in a rake task, so it's versioned and kept safe.
2. In your build script, set a unique database name for the current build, and use ERB in the server's `database.yml` to interpolate it in.
3. Have the build script run `rake db:create` and `rake db:drop`, so that your databases are created and cleaned-up automatically.

Here's an example script:

{% highlight ruby %}
desc "Run continuous integration suite"
task :build do
  ENV["RAILS_ENV"] = RAILS_ENV = "test"

  # use ERB in config/database.yml to make this the database name:
  # database: <%= ENV["DB_NAME"] %>
  now = Time.now.utc
  identifier = "#{Process.pid}#{now.to_i}#{now.usec}"
  ENV["DB_NAME"] = "myapp_test_#{identifier}"

  begin
    Rake::Task["db:create"].invoke
    Rake::Task["db:test:load"].invoke
    Rake::Task["default"].invoke
  ensure
    Rake::Task["db:drop"].invoke
  end
end
{% endhighlight %}

Appending the process ID and time in microseconds to the database name is about as unique as you can get, without generating a UUID or something. Note how we wrap the build in a block, and perform `db:drop` in the `ensure` section: that way, the database is removed even if the build fails (which would normally abort your rake task).

Keep in mind that the database might not be the only shared resource used by your build-watch out for filesystem use, in particular. You can probably use a similar strategy to solve that problem.