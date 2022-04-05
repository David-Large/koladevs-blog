---
title: "Writing Custom Migrations in Django"
date: 2022-04-05T00:28:02+01:00
draft: false
author: "Mangabo Kolawole"
featured_image: "arrayfilter.png"
---

Filtering `queryset` when working with Django is one of the most important and fun tasks to be done. 
If you are familiar with working with array/list values, you might want to know how to optimize your queries when working with array fields or values. 🚀

Well, let's learn some filters methods or array fields or values. 

## The `in()` method for array values

Suppose that you have a list of ids. You want to filter a `queryset` to only have the entries with ids that match ids values in the list.

```python
ids = [1,4,8,9]

Products.objects.filter(id__in=ids)
```

This is the equivalent in SQL:
```SQL
SELECT * FROM products WHERE id IN (1,4,8,9);
```

## The `ArrayField`

The `ArrayField` is a field used to store a list of data. We'll dive deeper into this field in another article. But here's a basic example of a model using this field.

```python
from django.contrib.postgres.fields import ArrayField
from django.db import models

class Product(models.Model):
    name = models.CharField(max_length=255)
    tags = ArrayField(models.CharField(max_length=255), size=10)
```
Great! Let's say you want to retrieve from the database all products with the `shoe` tag, or even more interesting with the `shoe` and `pants` tags.

For this, you use the `contains` filter. 

```python
# Retrieving all products with shoe tag

Product.objects.filter(tags__contains=['shoe'])

# Retrieving all products with shoe and pant tags

Product.objects.filter(tags__contains=['shoe', 'pant'])
```
And that's it basically.👀

You can also find interesting filters such as `contained_by` which is the inverse of [`contains`](https://docs.djangoproject.com/fr/4.0/ref/contrib/postgres/fields/#contained-by) but also the [`overlap`](https://docs.djangoproject.com/fr/4.0/ref/contrib/postgres/fields/#overlap) filter that checks that the retrieved entries contain at least a value with those passed. 


