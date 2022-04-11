---
title: "Custom Permissions in Django Rest"
date: 2022-04-11T02:28:02+01:00
draft: false
author: "Mangabo Kolawole"
featured_image: "optimization2.png"
description: "Write custom permissions in Django REST.🚀"
---

When building a web API using DRF, you are directly provided some permissions classes such as `AllowAny` or `IsAuthenticated` for example. 

However, it's possible to write your own permissions if these classes don't work for you.

## Problem

According to your needs, you might want to have specific permissions for an authenticated user for example. 
Suppose that your system is made of two types of users: simple users that we'll call `consumer` and shop owners that we'll call `owner`. 

An Owner is the admin of a shop. He can for example create products that the consumer can read. The consumer can't create, modify or delete the `product` resource and he must be authenticated before accessing the `product` resources. 

It's definitely intuitive to think about just using the `IsAuthenticated` class and move on.🤔

Hum! Let's give a quick look at the code of the `IsAuthenticated` class in Django. 

```python
class IsAuthenticated(BasePermission):
    """
    Allows access only to authenticated users.
    """

    def has_permission(self, request, view):
        return bool(request.user and request.user.is_authenticated)
```

Note that `permissions` in Django work at two levels. The resource level and the object level. 
The resource level is handled by `has_permission` and the object level is handled by `has_object_permission`. 
So here's how the `IsAuthenticated` class code will normally look.
```python
class IsAuthenticated(BasePermission):
    """
    Allows access only to authenticated users.
    """

    def has_permission(self, request, view):
        return bool(request.user and request.user.is_authenticated)

    def has_object_permission(self, request, view):
        return bool(request.user and request.user.is_authenticated)
```

But because it's already defined at the resource level that the user needs to be authenticated, no need to write the `has_object_permission` method. 

Well, let's write custom permission for our `consumers` and `owners`.

## Solution

We'll write two different permissions: 
- `ConsumerPermission`
- `OwnerPermission`

And here are the classes, subclasses of `BasePermission`. 

```python
class OwnerPermission(BasePermission):
    
    def has_permission(self, request, view):
        
        if view.basename == "product":
            return bool(request.user and request.user.is_authenticated and request.user.is_owner)
        return False
    
    def has_object_permission(self, request, view, obj):
        
        # Make sure that an owner can only modify and access its own products

        if obj.shop.owner == request.owner:
            return True
        
        return False
    
class ConsumerPermission(BasePermission):

    def has_permission(self, request, view):
        if view.basename == "product" and request.method in SAFE_METHODS:
            return bool(request.user and request.user.is_authenticated and request.user.is_consumer)

        return False      
```

The SAFE_METHODS is a list of just read methods `('GET', 'HEAD', 'OPTIONS')`.

And here's how you can write your own custom permissions in Django.🚀
You can learn more about permissions in Django in the following [documentation](https://www.django-rest-framework.org/api-guide/permissions/).
