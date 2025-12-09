---
layout: default
title: Блог 
---
<div class="home_table">
<div class="to_left_hidden">
    <h2>Welcome to my Blog!</h2>
    <small>A tiny little blog in a big big world :)</small>
    <br><br>
    <hr>
    <ul>
        {% for post in site.posts %}
        <li>
            <h2>
                <a class="post-link" href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a>
            </h2>
            <div class="label">
                <div class="label-card">
                    <i class="fa fa-calendar"></i>{{ post.date | date: "%F" }}
                </div>
                <div class="label-card">
                    {% if post.author %}<i class="fa fa-user"></i>{{ post.author }}
                    {% endif %}
                </div>
                <div class="label-card">
                    {% if page.meta %}<i class="fa fa-key"></i>{{ page.meta }}  {% endif %}
                </div>
            </div>
            <div class="excerpt">
                {{post.excerpt}}
            </div>
            <div class="read-all">
                <a  href="{{ post.url | prepend: site.baseurl }}"><i class="fa fa-newspaper-o"></i>Read All</a>
            </div>
            <hr>
        </li>
        {% endfor %}
    </ul>
</div>
<div class="to_right_hidden">
  <h3>Pages:</h3>
  <h4>
    <img src="assets/pictures/arrow-right-1.svg" class="icon_contacts">
    <a href="{{ site.baseurl }}/about/">Who_Я?</a>
  </h4>
  <h3>Recent posts:</h3>
  {% for post in site.posts offset: 0 limit: 10  %}
     <img src="assets/pictures/calendar.svg" class="icon_contacts">
     <a href="{{ site.baseurl }}{{ post.url }}">[{{ post.date | date: "%F" }}]: {{ post.title }}</a>
     <br>
  {% endfor %}
  <br>
  <h3>Categories:</h3>
  {% for category in site.categories %}
    <img src="assets/pictures/loading-1.svg" class="icon_contacts">
    <a href="{{ site.baseurl }}/{{ category | first }}/">{{ category | first }}</a>
    <br>
  {% endfor %}
</div>
</div>
