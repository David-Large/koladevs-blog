---
title: "Django ORM explained: selected_related and prefetch_related"
date: 2022-04-01T15:28:02+01:00
draft: false
author: "Mangabo Kolawole"
featured_image: "optimization.png"
description: "selected_related() vs prefetch_related()"
---

Working with the Django ORM provides ways to query data from the database without writing SQL statements. However, using an ORM instead of SQL statements definitely results in performance drops but for most CRUD applications, it's insignificant until you are looking for ways to reduce useless database queries and make your Django application much faster.

## An example of performance issues

Let's take an example of this model.

```python
from django.db import models


class User(models.Model):
    name = models.CharField(max_length=50)
    email = models.EmailField(max_length=254)

    
class Profile(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    first_name = models.CharField(max_length=50)
```

Great! We have two models here, and the `Profile` model has a `ForeignKey` -- One to Many -- relationship with the `User` model, meaning that a user can have many profiles. 

What is the default way to get the user if you have the profile object?

```python
# hit the database
profile = Profile.objects.first()

# hit the database again to get the related User object
user = profile.user
```
Well, we've just made two queries here. Not quite a big deal but we want to optimize these lines and avoid unnecessary queries. 

## Select Related

From the [Django docs](https://docs.djangoproject.com/en/4.0/ref/models/querysets/#select-related), `select_related()` Returns a QuerySet that will “follow” foreign-key relationships, selecting additional related-object data when it executes its query. This is a performance booster that results in a single more complex query but means later use of foreign-key relationships won’t require database queries.

Basically, we use `select_related` when the object that you're going to be selecting is a single object, so `OneToOneField` or a `ForeignKey`. 

Let's rewrite the line to fetch the `user` having the `profile` object.
```python
# Hits the database
profile = Profile.objects.select_related('user').first()

# Doesn't hit the database, because profile.user has been prepopulated
# in the previous query.
user = profile.user
```
And we've just transformed two queries into one. But what happens when you have `ManyToMany` relationships or just `ManyToOne`? 
Let's explore the case where we have to fetch all the profiles having the user object.

## Prefetch Related
Let's add a __str__ method to the User model. 
```python
class User(models.Model):
    name = models.CharField(max_length=50)
    email = models.EmailField(max_length=254)
    password = models.CharField(max_length=50)
    
    def __str__(self) -> str:
        return "%s (%s)" % (self.name,
            ", ".join(profile.first_name for profile in self.profile_set.all()))
```

>> **Note**: If you are using a `ForeignKey` relationship in Django between an M1 model and an M2 model -- which is symmetrical by default --, the  M1 model has automatically a reverse relationship field of type `ManyToOne`. 

If we run the following code, you'll have the following result. 

```
>>> User.objects.all()
[<User: User (John Doe)>, <User: User (Jane Doe)> ...]
```
The problem with this query is that every time the `__str__` method on the User model is called, it has to query the database for `self.profile_set.all()`. 
And the number of queries is proportional to the number of profiles by the user: well, a **nightmare**.🥶

But how can we reduce this to just two queries? Well, using `prefetch_related`.

```python
User.objects.prefetch_related('profile_set').all()
```
We use `prefetch_related` when you're going to get a "set" of things, so `ManyToMany` or reverse `ForeignKey` -- `ManyToOne`.

## Then what's the difference between the two? 
Both works on the same principle of prefetching data from the database, however, they take different approaches to do so. 

Already stated below but let's make the conclusion for this content: 
- We use `select_related` when the object that you're going to be selecting is a single object, so `OneToOneField` or a `ForeignKey`. 
- We use `prefetch_related` when you're going to get a "set" of things, so `ManyToMany` or reverse `ForeignKey` -- `ManyToOne`.

However, note that these methods should be used if you are looking to reduce database queries numbers and have no issues with getting a bigger size of data from the database. 
In my experience, these methods have really been helpful in running hungry cron jobs in Django.

What other methods do you use to make fewer database queries in your Django application? Let's discuss it in the comment section.
