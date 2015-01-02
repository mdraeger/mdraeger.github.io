---
layout: page
title: Welcome!

tagline: Programming, Maybe Math, Maybe Humor
---
{% include JB/setup %}

Programming, Maybe Math, Maybe Humor

I'm Marco Draeger, German, computer scientist, hobby programmer, 
__Firefly__ and __Doctor Who__ fan, 
martial arts interested (holding black belts in 
[Aikido](http://en.wikipedia.org/wiki/Aikido), 
[Iaido](http://en.wikipedia.org/wiki/Iaido), and 
[Battojutsu](http://en.wikipedia.org/wiki/Battojutsu)),
runner, fitness enthusiast, and general geek.

Interests
---------

I have studied Computer Science and Operations Research. Although these skills are rarely required
in my daily work, I have strong interests in (not exclusively):

-   programming languages (Haskell rules, though.)
-   fancy data structures
-   machine learning and applications
-   big data (just because it is fancy)
-   natural language processing
-   everything else that's geeky

Blogs
-----

Knowing myself, this will mainly remain a construction site with few updates in case I feel like it. 
However, should I really blog something, it will show up in the following list:

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

