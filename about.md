---
layout: page
permalink: /about/index.html
title: Nguyễn Đình Phi - Oni
tags: [Phi, Linux, Oni, Kernel]
imagefeature: fourseasons.jpg
chart: true
---
<figure>
  <img src="{{ site.url }}/images/hmfaysal.jpg" alt="Oni Ranger">
  <figcaption>Nguyễn Đình Phi</figcaption>
</figure>

{% assign total_words = 0 %}
{% assign total_readtime = 0 %}
{% assign featuredcount = 0 %}
{% assign statuscount = 0 %}

{% for post in site.posts %}
    {% assign post_words = post.content | strip_html | number_of_words %}
    {% assign readtime = post_words | append: '.0' | divided_by:200 %}
    {% assign total_words = total_words | plus: post_words %}
    {% assign total_readtime = total_readtime | plus: readtime %}
    {% if post.featured %}
    {% assign featuredcount = featuredcount | plus: 1 %}
    {% endif %}
{% endfor %}


My name is **Nguyễn Đình Phi**, and this is my personal blog. It currently has {{ site.posts | size }} posts in {{ site.categories | size }} categories which combinedly have {{ total_words }} words, which will take an average reader ({{ site.wpm }} WPM) approximately <span class="time">{{ total_readtime }}</span> minutes to read. {% if featuredcount != 0 %}There are <a href="{{ site.url }}/featured">{{ featuredcount }} featured posts</a>, you should definitely check those out.{% endif %} The most recent post is {% for post in site.posts limit:1 %}{% if post.description %}<a href="{{ site.url }}{{ post.url }}" title="{{ post.description }}">"{{ post.title }}"</a>{% else %}<a href="{{ site.url }}{{ post.url }}" title="{{ post.description }}" title="Read more about {{ post.title }}">"{{ post.title }}"</a>{% endif %}{% endfor %} which was published on {% for post in site.posts limit:1 %}{% assign modifiedtime = post.modified | date: "%Y%m%d" %}{% assign posttime = post.date | date: "%Y%m%d" %}<time datetime="{{ post.date | date_to_xmlschema }}" class="post-time">{{ post.date | date: "%d %b %Y" }}</time>{% if post.modified %}{% if modifiedtime != posttime %} and last modified on <time datetime="{{ post.modified | date: "%Y-%m-%d" }}" itemprop="dateModified">{{ post.modified | date: "%d %b %Y" }}</time>{% endif %}{% endif %}{% endfor %}. The last commit was on {{ site.time | date: "%A, %d %b %Y" }} at {{ site.time | date: "%I:%M %p" }} [UTC](http://en.wikipedia.org/wiki/Coordinated_Universal_Time "Temps Universel Coordonné").

I was born in 1993 spring. I am a developer, with more than 2 years of professional developer experience. Currently, I am working as a Embedded Engineer at Compex System, a local company in Singapore. Back to the past, I graduated from Vietnam National University, Hanoi with major in Information Technology in 2015. Afterward, I went to work at Toshiba Software Development Vietnam in 2 years, at there, I develop a Thin client solution called TZCS and a broad casting system for Japanese companies. After 2 years, I want to find a chance to work oversea, then I left toshiba and got a new job in singapore. I consider myself as a developer who loves to write clean and maintainable code. I can work indepently or within a team, however, I prefer the latter because sharing is better.

<figure class="third">
	<a href="{{ site.url }}/images/about/1.JPG"><img src="{{ site.url }}/images/about/1.JPG"></a>
	<a href="{{ site.url }}/images/about/2.JPG"><img src="{{ site.url }}/images/about/2.JPG"></a>
	<a href="{{ site.url }}/images/about/3.JPG"><img src="{{ site.url }}/images/about/3.JPG"></a>
</figure>
<figure class="half">
	<a href="{{ site.url }}/images/about/4.JPG"><img src="{{ site.url }}/images/about/4.JPG"></a>
	<a href="{{ site.url }}/images/about/5.JPG"><img src="{{ site.url }}/images/about/5.JPG"></a>
</figure>
<figure class="third">
	<a href="{{ site.url }}/images/about/6.JPG"><img src="{{ site.url }}/images/about/6.JPG"></a>
	<a href="{{ site.url }}/images/about/7.JPG"><img src="{{ site.url }}/images/about/7.JPG"></a>
	<a href="{{ site.url }}/images/about/8.JPG"><img src="{{ site.url }}/images/about/8.JPG"></a>
	<figcaption>I love trees.</figcaption>
</figure>

TZCS project.<br/>
Toshiba appreciates that combining effective mobile working with tight security measures for company data is a major priority for moderm business, so we develop a mobile zero client, designated for operate on Toshiba's lastest standard business laptops. These laptops don't have storage device like SSD or HDD and it doesn't allow any data to be hosted on the device instead, both functionality and data are made available through a VDI, removing threat of malware being stored on the device and data theft in the event of device is lost or stolen. Because everything is hosted on a central server, so these laptops don't need to be build with strong and expensive hardware, but they still get good performance. Technically, these laptops contain a small Linux based OS, that be stored permanently in ROM. When User power on the laptop, I will connect to known SSID and Company's VDI server, then login, only device that is authorised will be allow to access company VDI, all necessary authentication is stored in BIOS level so no one can change it manually. This feature secures the environment from the moment it is power on. 
TZCS has a small Linux OS, call firmware (only 4MB), It is too small to contain enough software for run a VDI endpoint. So, infact, when user power on the laptop, it will need to download neccessary packages to to make it can work as an Endpoint.
Every data on laptop will be erase when user turn it off. <br/>

Tokyo broadcasting system.<br/>
Develop a auto program controller for some Japanese broacasting services

https://smtebooks.com/