# David Hubbard's Blog

<p>
  In which random thoughts meet one another for a merry unheedy haste
  into the mind, blind but not blinded or that the bright things would
  come to confusion. Rather, emerges something more clear and bright than
  before. Hic sunt dracones. 🐲
  {% for post in site.posts %}
    {% assign cur_yr = post.date | date: "%Y" %}
    {% if cur_yr != date %}
      <div id="y{{cur_yr}}"><b>{{ cur_yr }}</b></div>
      {% assign date = cur_yr %}
    {% endif %}

    <h3><a href="{{ post.url }}">{{ post.title }}</a></h3><br/>
  {% endfor %}
</p>
