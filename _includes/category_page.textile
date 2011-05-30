<ul class="posts">
{% for post in category_posts %}
		<li><p class="date" cate="{{ post.categories }}">{{ post.date | date:"%Y-%m-%d" }}</p> <a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %} 
</ul>
