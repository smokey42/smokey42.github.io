---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults
layout: default
---
<h1>Hello, this is my blog.</h1>

My name is <b>Christoph</b> and this is <b>my (admittedly sparse) blog</b>. My posts will mainly feature:

<ul>
    <li>Python</li>
    <li>programming in general</li>
    <li>agile methods and how they affect developers and companies</li>
    <li>space related topics</li>
    <li>everything else I feel like writing about</li>
</ul>

<hr>

<h2>Posts</h2>

Here is the list of past posts I've made. Perhaps there is something which will interest you.

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a><br>
      Released: {{ post.date | date: "%Y-%m-%d" }}<br>
<!---
      {% assign counter = 0 %}
      {% assign first = 0 %}
      {% assign last = post.categories | size %}
      {% for category in post.categories %}
          {% if counter == first %}Categories: {% endif %}
          {{ category }}
          {% assign counter = counter | plus: 1 %}
          {% if counter != last %} &mdash; {% endif %}
      {% endfor %}
--->
    </li>
  {% endfor %}
</ul>
