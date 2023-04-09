---
title: "파이썬"
layout: archive
permalink: categories/python
author_profile: true
sidebar_main: true
---

{% assign posts = site.categories.python %}
{% for post in posts %} {% include categories.html type=page.entries_layout %} {% endfor %}
