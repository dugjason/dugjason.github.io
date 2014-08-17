---
layout: post
title: 'Socket Timeouts in Ruby'
---

I recently had a problem, which required me to close an open HTTP connection if it had not received any new data for a certain amount of time.

After a bit of Googling, I found that the [Timeout module](http://ruby-doc.org/stdlib-1.9.3/libdoc/timeout/rdoc/Timeout.html) baked into Ruby was probably not my best option. I looked deeper at my options, and arrived at using [IO.select](http://ruby-doc.org/core-2.0/IO.html#method-c-select):

{% highlight ruby linenos %}
begin
  ready = IO.select([@socket], nil, [@socket], @stream_timeout)
  unless ready
    reconnect()
    next
  end
end
{% endhighlight %}

This code lives in a loop that listens for data on the socket. IO.select takes four arguments:

{% highlight ruby %}
select( read_array [, write_array [, error_array [, timeout] ] ] )
{% endhighlight %}

In this case, the data we want to read (including errors) comes from the socket. The timeout value is how long you want to wait until the timeout should occur.

So, in the example above, we listen for data or errors on the socket for up to the number of seconds specified by ```@stream_timeout```. If we do not receive any data within the specified timeout, we call the ```reconnect()``` method. In this application, if we have not received any data within the specified timeout, something is wrong with the socket connection, and we need to reconnect, so we call the ```reconnect()``` method, and skip to the next iteration of the loop listening on the socket. You can see this reconnect in action in the [DataSift API Client Library for Ruby (v.2.*)](https://github.com/datasift/datasift-ruby/releases/tag/2.1.0).

I’m still learning my way around Ruby, and Ruby’s best practises, so if anyone has any comments or criticisms, please leave a comment, or raise an issue on the Github repo.
