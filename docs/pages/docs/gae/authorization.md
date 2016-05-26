---
title: Authorization
description: Details on how to restrict data access 1
---

# Authorization in Google App Engine

There are two main ways you may want to limit access to data when working
with Graphene and Google App Engine: limiting which fields are accessible via GraphQL
and limiting which objects a user can access.

Let's use a simple example model.

```python
from google.appengine.ext import ndb

class Article(ndb.Model):
    headline = ndb.StringProperty()
    summary = ndb.StringProperty()
    body = ndb.TextProperty()
    keywords = ndb.StringProperty(repeated=True)

    author_key = ndb.KeyProperty(kind='Author')

    created_at = ndb.DateTimeProperty(auto_now_add=True)
    updated_at = ndb.DateTimeProperty(auto_now=True)
```

## Limiting Field Access

This is easy, simply use the `only_fields` meta attribute:

```python
from graphene_gae import NdbNode
from .models import Article

class ArticleNode(DjangoNode):
    class Meta:
        model = Article
        only_fields = ('headline', 'summary', 'body')
```

Alternatively you can use the `exclude_fields` meta attribute:

```python
from graphene_gae import NdbNode
from .models import Article

class ArticleNode(DjangoNode):
    class Meta:
        model = Article
        exclude_fields = ('created_at', 'updated_at')
```

## User-based Queryset Filtering

You can use the GraphQL Context to pass values for your resolving code to check.
Adding the `with_context` decorator will send your resolver function an extra
argument - `context`:

```python
from graphene import ObjectType
from graphene_gae import NdbNode, NdbConnectionField
from .models import Article

class Query(ObjectType):
    my_articles = NdbConnectionField(PostNode)

    class Meta:
        abstract = True

    @with_context
    def resolve_my_articles(self, args, context, info):
        # context will reference to the Django request
        if not context.user:
            return []
        else:
            return Article.query().filter(Post.author_key == context.user.key)

```

When executing the GraphQL query you can set the context object to be passed
the resolvers:

```python
result = schema.execute(query, context_value=dict(user=self.current_user))
```

## Filtering ID-based node access

In order to add authorization to id-based node access, we need to add a method
to your `NdbNode`.

```python
from graphene_gae import NdbNode
from .models import Article

class ArticleNode(NdbNode):
    class Meta:
        model = Article
        only_fields = ('headline', 'summary', 'body')

    @classmethod
    @with_context
    def get_node(cls, id, context, info):
        try:
            article = ndb.Key(urlsafe=id).get()
            if not article:
                return None
        except:
            return None

        if context.user.key == article.author_key:
            return cls(instance)
        else:
            return None
```
