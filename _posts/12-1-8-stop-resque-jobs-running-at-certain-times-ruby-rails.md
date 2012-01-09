---
layout: post
title: "Stop resque jobs running at certain times [Ruby/Rails]"
excerpt: I needed a way to stop running [resque](https://github.com/defunkt/resque) jobs at certain times, but only for things sent to a crappy legacy api written in .NET (by some other company).
published: true
---

I needed a way to stop running [resque](https://github.com/defunkt/resque) jobs at certain times, but only for things sent to a crappy legacy
api written in .NET (by some other company). Luckily I'm using [resque scheduler](https://github.com/bvandenbos/resque-scheduler),
which makes things a lot easier as they add some great methods:

{% highlight ruby linenos %}
# run a job in 5 days
Resque.enqueue_in(5.days, SendFollowupEmail)
# or run SomeJob at a specific time
Resque.enqueue_at(5.days.from_now, SomeJob)
{% endhighlight %}

Lets make a simple list of days and times our workers should stop working and not send them data, as their .NET app goes down for "maintenance" every night but yet still returns success like it's business as usual...
except the data disapears and never gets created.  So it's our job as responsable programmers to compensate for this and not send them anything in those maintenance windows:

{% highlight ruby linenos %}
time_off = {
  saturday:  '10pm-8am',
  sunday:    '12am-6am',
  monday:    '12am-6am',
  tuesday:   '12am-6am',
  wednesday: '12am-6am',
  thursday:  '12am-6am',
  friday:    '12am-6am'
}
{% endhighlight %}
<span class='small'>You can use any type of time i.e. '12:34:12am-6:45am'</span>

Now we have our list, we need to grab and set the current day and time:

{% highlight ruby linenos %}
current_day  = Date.today.strftime('%A').downcase
current_time = Time.now
{% endhighlight %}

Once we have that, all we need to do is check if the current day and time we're about to run our job at falls inside one
of their <s>.NET memory leak clean ups</s> maintenance windows.  If it does use the `Resque.enqueue_at` method:

{% highlight ruby linenos %}
if workers_time_off = time_off[current_day.to_sym]

  workers_time_off = workers_time_off.split('-').collect { |t|
    Chronic.parse((t['am'] ? 'tomorrow ' : '') + t)
  }

  from = workers_time_off.first
  to   = workers_time_off.last

  if current_time >= from and current_time < to
    # It falls within the given time off range for that day
    Resque.enqueue_at(to, SomeWorker)
  end

end
{% endhighlight %}

You might have noticed on `line 4` I didn't use [Time](http://ruby-doc.org/core-1.9.3/Time.html) but
[Chronic](https://github.com/mojombo/chronic) instead.  Well the great thing about ruby is,
there's almost always a gem that will do what you need.  In this instance if the time contains `am` we
need to make sure that the date is tomorow, otherwise it would set it for 6am the current day and not work
as expected.

That's it, now we have our [resque](https://github.com/defunkt/resque) workers stopping at certain days and times.
If you have any questions, comments or improvements, please let me know by adding a comment :)