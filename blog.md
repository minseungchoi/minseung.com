---
layout: page
title: Blog
permalink: /blog/
sitemap: false
---

I might blog occasionally. My blog entries---{{ site.posts | size }} in total since I started this website in 2021---may span neuroscience, computer science, music, political science, and public policy. 

[//]: # To date, <a href="/categories">categories on my blog</a> (and number of blog posts in that category) include {% assign sorted_cats = site.categories | sort  %}{% for category in sorted_cats %}{% if forloop.last == true %}and {% endif %}<a href="/categories/#{{category[0]}}" style="font-weight:normal;"> {{category[0] | camelcase }}</a> ({{ category[1].size  }}){% if forloop.last == false %}, {% endif %}{% endfor %}. 

A full listing of my blog posts follows.


<!-- {% assign sorted_cats = site.categories | sort  %}{% for category in sorted_cats %}{% if forloop.last == true %}and {% endif %}<a href="/categories/#{{category[0]}}" style="font-weight:normal;">{{category[0] | camelcase }}</a> ({{ category[1].size  }}){% if forloop.last == false %},{% endif %} {% endfor %} -->



<ul id="archive">
{% for post in site.posts %}
  {% capture y %}{{post.date | date:"%Y"}}{% endcapture %}
  {% if year != y %}
    {% assign year = y %}
    <h2 class="blogyear">{{ y}}</h2>
  {% endif %}
<li class="archiveposturl"><span><a href="{{ post.url }}" title="{{ post.title }}">{{ post.title }}</a></span><br/>
<span class = "postlower">

<!--<strong>Author:</strong> {{post.author}} -->
<strong>Category:</strong>  {% if post.categories %}
 
  {% for cat in post.categories %}
  <a href="/categories/#{{ cat }}" title="{{ cat }}">{{ cat }}</a>&nbsp;
  {% endfor %}

{% endif %} <!-- {{ post.categories | first }} -->
<strong style="font-size:100%; font-family: 'Titillium Web', sans-serif; float:right; padding-right: .5em">{{ post.date | date: '%d %b %Y' }}</strong> 
</span> 

</li>
{% endfor %}
</ul>

<!-- {{ post.date | date: '%m %d, %Y' }} -->
