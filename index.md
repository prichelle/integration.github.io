---
layout: default
title: Introduction
---

#Integration information

## CP4I
## APIC
## ACE
### Designer
### Software

This is a test

	<h1>{{ page.title }}</h1>
	<ul class="posts">

	  {% for post in site.posts %}
	    <li><span>{{ post.date | date_to_string }}</span> » <a href="{{ post.url }}" title="{{ post.title }}">{{ post.title }}</a></li>
	  {% endfor %}
	</ul>