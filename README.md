Software developer, occasional rambler, highly opinionated.

I develop software for a living!

Have an idea and want me to build it? Get in touch via [email](mailto:bl.panuto@gmail.com), or reach me through [twitter](https://twitter.com/panuto_).

# Posts
{% for post in site.posts %}

- [{{post.title}}]({{ post.url }})

{% endfor %}

# Open Source projects

{% for project in site.projects %}

### {{project.title}} [link]({{ project.project_url }})

{{project.content}}

{% endfor %}
