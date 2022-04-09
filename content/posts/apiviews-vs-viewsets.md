---
title: " APIView vs ViewSets"
date: 2022-04-04T15:28:02+01:00
draft: false
author: "Mangabo Kolawole"
featured_image: "viewsvsviewsets.png"
---

When building your API in Django using Django Rest Framework, you have two choices for writing the controllers: APIView or ViewSets. 

## APIView

`APIView` provides a methods handler for HTTP verbs: `get`, `post`, `put`, `patch` and `delete`. 
But you'll need to write a method for each HTTP verb. For example, you are writing a view for a `Product` resources with a `ProductSerializer`.

Here's an example of the View with a `get()` method to return all the products in the database. 

```python
from rest_framework.views import APIView


class ProductView(APIView):
    
    serializer_class = ProductSerializer
    
    def get(self, request, *args, **kwargs):
        
        products = Product.objects.all()
        
        serializer = self.serializer_class(products, many=True)
        
        return Response(serializer.data)
```

`APIView` works best for simple endpoints that won't really require many possibles HTTP requests. What do I mean? 
Suppose that you want an endpoint to create a product. Pretty simple. We can quickly handle it.

```python
from rest_framework.views import APIView


class ProductView(APIView):
    
    serializer_class = ProductSerializer
    
    def get(self, request, *args, **kwargs):
        
        products = Product.objects.all()
        
        serializer = self.serializer_class(products, many=True)
        
        return Response(serializer.data)
    
    def post(self, request, *args, **kwargs):
        
        serializer = self.serializer_class(data=request.data)
        
        serializer.is_valid(raise_exception=True)
        
        return Response(serializer.data)
```

We can now create a product, but let's suppose we want to make a certain action on the product, like make it unavailable. To do it, a `POST` request must be sent to the endpoint and we'll handle the rest with some methods on the `Product` model.

We can't do that in the `ProductView` class because there is already a `post()` method. A solution? 
Add another `APIView` ... and this starts becoming quite verbose.🤔
Not to add that you'll have to add two different URL routes for the same resource, which can be a bad API design.

A solution? 💡 `ViewSets`.

## ViewSets

A `ViewSet` is a class-based view, able to handle all of the basic HTTP requests: GET, POST, PUT, DELETE without hard coding any of the logic.
The `viewsets` module also provides a `ModelViewSet` class that maps HTTP request actions to CRUD actions in the Database. 
For example, let's rewrite the `ProductView` in a `ModelViewSet`.

```python
class ProductViewSet(ModelViewSet):
    
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    http_method_names = ['get', 'post']
```
Well, this is doing the same job as the `ProductView`. Interesting, right?🤩 This is even more interesting when we want a certain action on the same resource, no need to add a new class. 

```python
class ProductViewSet(ModelViewSet):
...
    @action(detail=True, methods=['post'])
    def unavailable(self, request, *args, **kwargs):
        obj = self.get_object()
        
        obj.unavailable()
        
        return Response({'status': 'unavailable'})
...
```
And that's not all. If you are working with `viewsets`, instead of URL you'll be working with routers.
And here's how to register a `viewsets` in your router file.

```python
router = routers.DefaultRouter()

router.register(r'products', ProductViewSet)
```
And with this, you'll have the following structure for the `products` endpoint.

| **Endpoint**                      | **HTTP Method** | **Description**            |
|-----------------------------------|-----------------|----------------------------|
| /products/                        | POST            | Creating a new product     |
| /products/                        | GET             | Get the list of all users  |
| /products/product_id/             | GET             | Retrieve a product         |
| /products/product_id/unavailable/ | POST            | Make a product unavailable |

## Summary

`APIView` and `ViewSet` all have their right use cases. But most of the time, if you are only doing CRUD on resources, you can directly use `ViewSets` to respect the DRY principle.
But if you are looking for more complex features, you can go low-level because after all, `viewsets` are also a subclass of `APIView`.

More on [viewsets](https://www.django-rest-framework.org/api-guide/viewsets/) and [views](https://www.django-rest-framework.org/api-guide/views/).
