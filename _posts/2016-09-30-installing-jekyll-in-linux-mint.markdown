---
layout: post
title:  "Installing jekyll in Linux Mint"
date:   2016-09-30 17:15:35 -0600
categories: mint
---
For whatever reason, Mint 17.3 includes a really old version of ruby, where git jekyll requires at least version 2.0. Following [this post](https://theoryl1.wordpress.com/2015/12/30/install-ruby-2-x-on-linux-mint-17-3/) you can load up a PPA with a more modern ruby.

{% highlight shell %}
sudo apt-get install software-properties-common
sudo apt-add-repository ppa:brightbox/ruby-ng
sudo apt-get update
sudo apt-get install ruby2.2 ruby2.2-dev
 
ruby -v
# Output: ruby 2.2.4p230 (2015-12-16 revision 53155) [x86_64-linux-gnu]
{% endhighlight %}

I also had to install the following packages.

```
sudo apt-get install zlib1g-dev nodejs
sudo gem install bundler
```

Now, following the [github guide](https://help.github.com/articles/setting-up-your-github-pages-site-locally-with-jekyll/) you can get things going.

{% highlight shell %}
echo "source 'https://rubygems.org'
gem 'github-pages', group: :jekyll_plugins" > Gemfile
bundle install
bundle exec jekyll new . --force
{% endhighlight %}

Be sure to actually follow the guide, as this is just my notes after completing install.
