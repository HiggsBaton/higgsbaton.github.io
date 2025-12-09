---
layout: default
category: jekyll
---
<div class="home_table">
<div class="to_left_hidden">
  <h2>Category: {{ page.category }}</h2>
  <ul>
    {% for category in site.categories %}
      {% for post in category.last %}
        {% for category in post.categories %}
	  {% if category == page.category %}
            <li><a href="{{ post.url }}">{{ post.title }}</a></li>
          {% endif %}
        {% endfor %}
      {% endfor %}
    {% endfor %}
  </ul>
</div>
<div class="to_right_hidden">
  <h3> Recent posts:</h3>
  <ul>
  {% for post in site.posts offset: 0 limit: 10  %}
     <li>
       <a href="{{ site.baseurl }}{{ post.url }}">[{{ post.date | date: "%F" }}]: {{ post.title }}</a>
     </li>
  {% endfor %}
  </ul>
  <br>
  <h3>Categories:</h3>
  <ul>
  {% for category in site.categories %}
    <li>
      <a href="{{ site.baseurl }}/{{ category | first }}/">{{ category | first }}</a>
    </li>
  {% endfor %}
  </ul>
</div>
</div>
