---
title: Setup Jekyll  and Jekyll Admin in Ubuntu
layout: post
---

Setting up Jekyll is quite easy, but to have a full pleasant experience it is required to find a good tool to edit your page. 

The first thing to do is to create a Hello World Jekyll site. There are many different ways to install Jekyll, but I found this quite easy. It took me 5 minutes. The only thing you need to do is to fork [jekyll-now](https://github.com/barryclark/jekyll-now) with the name ** ${your_github_username}.github.io**

Modify the _config.yml file including your name and your job role.  

In around 10 minutes github will process your repository and the site would be available at ** ${your_github_username}.github.io**

Then I started looking into Jekyll editors to make the blog posting more user friendly. I googled it, and in this webpage I found a good way to start:

[https://github.com/planetjekyll/awesome-jekyll-editors](http://https://github.com/planetjekyll/awesome-jekyll-editors)

I tried a few options before setting up the Jekyll Admin:
* Sublime + Jekyll plugin: installing the sublime plugin in linux was a bit tedious and after installing it, my initial experience was not so user friendly. 
* WebStorm: after installing it I realized it was one month trial. And I couldn't find an activation code... 

I realized that the Jekyll Admin was the most popular option. 

These are the steps I had to do to use the Jekyll Admin plugin in my new blog:

1. Install the latest Ruby version:
	`sudo apt-get install ruby2.4
	sudo ln -sf /usr/bin/ruby2.4  /usr/bin/ruby
	sudo apt-get install ruby2.4-dev
	`
2. Install [latest version of jekyll](https://jekyllrb.com/docs/upgrading/3-to-4/):
	`https://jekyllrb.com/docs/upgrading/3-to-4/`
3. Add a GemFile in the root folder of your blog repository:
```
gem 'jekyll-admin', group: :jekyll_plugins
gem 'jekyll-sitemap'
gem 'jekyll-feed'
```
5. Install the ruby bundle:
```
 bundle install
```
6. Start the Jekyll localhost server:
```
bundle exec jekyll serve
```
7. Administrate your page from your browser as a CMS:
```
http://localhost:4000/admin/
```