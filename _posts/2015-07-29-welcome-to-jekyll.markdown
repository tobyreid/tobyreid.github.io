---
layout: post
title:  "Welcome to Jekyll!?!"
date:   2015-07-29 17:00:50
categories: blog github jekyll
---
So after a lot of faffing about trying to decide what blog engine to use, I decide to use one that is hosted and supported by [github pages][github-pages] natively rather than trying to host markdown or bbcode in Azure/Amazon/private server/whatever and have the overhead of generating it. [Jekyll][jekyll] it is then.

The process of installing [Jekyll][jekyll] was fairly painless even on windows, assuming you have [chocolatey][chocolatey] installed:

{% highlight bat %}
cinst ruby
cinst ruby2.devkit
gem install bundler
{% endhighlight %}

Then navigate to your repo where you'd like to create, create a `GemFile.` (no extension) with the following content:

{% highlight bat%}
source 'https://rubygems.org'
gem 'github-pages'
{% endhighlight%}

Now either you can go ahead and start building everything from scratch, or you can run:

{% highlight bat%}
jekyll new .
{% endhighlight%}

Which should populate your directory with a site with a really basic layout and a couple of sample pages to get you started.

Don't forget to commit the `.gitignore` before you commit all the rest of your files, or you'll end up with `_site` subdirectory committed, which you don't need.

As this default page used to say:

> Check out the [Jekyll docs][jekyll] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll’s dedicated Help repository][jekyll-help].

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
[github-pages]:https://pages.github.com
[chocolatey]:  https://chocolatey.org/