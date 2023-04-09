---
title: "이론"
layout: archive
permalink: categories/algorithmQuestion
---

<div id="post-list">
  <h1>{{page.title}}</h1>
  <hr>
  <h2 id="{{category | first}}">{{ esc_ctgname }}</h2>
  <ul>
    <li v-for="p in posts" v-if="p.categories.indexOf(hash) != -1">
      <time>
        {{ esc_date }}
      </time>
      <a v-bind:href="p.url">{{ esc_title }}</a>
    </li>
  </ul>
</div>
