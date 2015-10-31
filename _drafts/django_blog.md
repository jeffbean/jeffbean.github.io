# notes for django blog

#### django_demo/urls.py

Now to add the new blog app URL's

    :::python hl_lines="3 7 8"
    from django.conf.urls import patterns, include, url
    from django.contrib import admin
    from django.views.generic import RedirectView

    urlpatterns = patterns(
        '',
        url(r'^$', RedirectView.as_view(url='/blog/'), name='home'),
        url(r'^blog/', include('blog.urls')),

        url(r'^admin/', include(admin.site.urls)),
    )


#### blog/urls.py

    from django.conf.urls import patterns, url
    from blog.views import BlogIndexView, ArticleDetailView

    urlpatterns = patterns(
        '',
        url('^(?P<slug>[\w\-]+)/$', ArticleDetailView.as_view(), name="blog-article-single"),
        url('^$', BlogIndexView.as_view(), name="blog-article-index"),
    )


#### blog/views.py

    from django.views.generic import ListView
    from django.views.generic.detail import DetailView
    from blog.models import Article


    class BlogIndexView(ListView):
        model = Article
        queryset = Article.objects.order_by('-publish_date')
        context_object_name = 'articles'
        paginate_by = 5


    class ArticleDetailView(DetailView):
        model = Article
        slug_field = 'slug'
        slug_url_kwarg = 'slug'
        paginate_by = 5

#### blog/models.py

    from django.db import models
    from django.utils.translation import ugettext as _


    class Article(models.Model):
        title = models.CharField(
            verbose_name=_(u'Title'),
            help_text=_(u' '),
            max_length=255
        )
        slug = models.SlugField(
            verbose_name=_(u'Slug'),
            help_text=_(u'Uri identifier.'),
            max_length=255,
            unique=True
        )
        article = models.TextField(verbose_name=_(u'Article'))
        publish_date = models.DateField(
            verbose_name=_(u'Publish Date'),
            help_text=_(u'The date the Blog Article was published')
        )

        class Meta:
            ordering = ['-publish_date']

        def __str__(self):
            return u"{0:s}".format(self.title, )

        def __unicode__(self):
            return u"{0:s}".format(self.title, )


#### blog/admin.py

    from django.contrib import admin

    from blog.models import Article


    class ArticleAdmin(admin.ModelAdmin):
        prepopulated_fields = {'slug': ('title',)}
        list_display = ('title', 'publish_date')
        fieldsets = (
            (
                None,
                {
                    'fields': ('title', 'slug', 'article', 'publish_date',)
                }
            ),
        )


    admin.site.register(Article, ArticleAdmin)

#### blog/templates/blog/blog_base.html

    <!DOCTYPE html>
    <html>
    <head lang="en">
        <meta charset="UTF-8">
        <title></title>
    </head>
    <body>
        {% block content%}

        {% endblock %}
    </body>
    </html>



#### blog/templates/blog/article_list.html

    {% extends 'blog/blog_base.html' %}

    {% block content %}
        <h1>Blog</h1>
        {% for item in object_list %}
            <article class="news-item">
                <h3 class="title">
                    <a href="{% url 'blog:blog-article-single' slug=item.slug %}">{{ item.title }}</a>
                </h3>

                <div class="meta">{{ item.publish_date|date:"F j, Y" }}</div>
            </article>
        {% empty %}
            <p>No Blog Articles</p>
        {% endfor %}
    {% endblock %}
