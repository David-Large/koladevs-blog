---
title: " Optimize query size in Django "
date: 2022-04-02T15:28:02+01:00
draft: false
author: "Mangabo Kolawole"
featured_image: "optimization2.png"
---

### Problem

When your project grows, it's possible that with time you'll start having models with more and more fields. 
What are the consequences of that? Well, if you have a model with more than 30 fields and you are trying to retrieve an object or a list of objects from the database, the return `queryset` will be really heavy. 
And it's possible you won't definitely need all the fields. 

## How to optimize the query?

Django provides the `only()` method that returns a `queryset` with objects that only contain the list of fields and the values specified in the `only(*args)`.
As stated in Django, 

Here's an example: 

```python
from django.db import models


class User(models.Model):
    name = models.CharField(max_length=50)
    email = models.EmailField(max_length=254)
    bio = models.CharField(max_length=50)
    
    def __str__(self) -> str:
        return "%s (%s)" % (self.name,
            ", ".join(profile.first_name for profile in self.profile_set.all()))
```
And here's the `queryset`:
```python
users = User.objects.only('name', 'email').all()
```

There is also the `defer()` method -- opposite to `only` -- that can be used to remove fields you don't want to use in a particular query. 
Following this, these two queries are identical if the `User` table has only `name`, `email`, and `bio` fields.

```python
users = User.objects.only('name', 'email').all()

users = User.objects.defer('bio').all()
```
