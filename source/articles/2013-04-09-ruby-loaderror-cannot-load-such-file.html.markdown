---
layout: post
title: 'Ruby LoadError: cannot load such file'
date: 2013-04-09
---

So, you’ve just upgraded your Ruby app to Ruby 1.9.2, 1.9.3, 2.0.0, or greater, and you’re hit with:

``` ruby
 LoadError: cannot load such file -- somefile
```

Your code will most likely look something like this:

``` ruby
 require 'my-ruby-code'
 require 'some-gem'
 ...
```

Don’t panic! In Ruby 1.9.2, the current path was removed from the load path, so you simply need to change the ‘require’ statements to your own code to include the current load path:

``` ruby
 require './my-ruby-code'
 require 'some-gem'
 ...
```

You can also do this by using `File.expand_path(__FILE__)`

**Note:** You do not need to change any require statements for your Gems – the load path for your Gems is managed by RubyGems.
