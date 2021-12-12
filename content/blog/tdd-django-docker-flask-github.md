---
title: "Test-Driven-Development with Django: Unit Testing & Integration testing with Docker, Flask & Github Actions"
date: 2021-12-12T9:28:02+01:00
draft: false
author: "Mangabo Kolawole"
featured_image: "stencil.png"
---

Software testing is an essential step in the software development process. It's a method to check whether the software is aligned with the requirements of the business or the project. 

However, many developers ignore this step and only rely on manual testing. 

Manual testing becomes difficult as the application size grows. By writing automated tests - unit tests, integrations tests, and E2E tests, we can make sure that every component added to the codebase or application works without breaking the whole application.

There are two main types of tests mostly used in Sofware development: Unit tests and Integration tests. We'll focus on these concepts for this article.

### Unit Testing

A Unit Test is an isolated test that tests one specific function.
For example, if you have an API to help you manage discount creation, and this application relied on a function to compute the discounted amount named `apply_discount`, a unit test here will be to test the function `apply_discount` to make sure it behaves accordingly. 
Or even more simple, you want to make sure that your `sum(a,b)` function returns exactly 10 when you do `sum(4,6)`.

Unit tests need a lot of focus. It's a great habit to write a lot of these. 

## Setup project

We'll be working on a simple project to create an e-commerce `cart`. 
Here are the requirements of this project: 
- Be able to create and list discount
- Be able to create a cart and apply a discount
- Be able to pay the amount on the cart 

First of all, let's configure the environment. I'll be using `virtualenv` to create a virtual python environment.

```shell
virtualenv --python=/usr/bin/python.9 venv
source venv/bin/activate
```

Now, let's install Django and create a project.

```shell
pip install django
django-admin startproject CoreRoot .

python manage.py migrate && python manage.py runserver
```

Well, let's create our application and start coding.

## Discount application

At the root of the project, enter this command

```shell
python manage.py startapp discount
```

The application is created, but we need to register it in the `settings.py` file.

```python
...
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    'discount'
]
...
```

### Discount Model

The discount model comes with the following fields.

- code
- value
- description
- created
- ended

Following the TDD methodology, we'll write the tests first and then write the features. 
Let's write a test for the model. But first, we have to install the modules we'll be using.

```shell
pip install pytest pytest-django
```

Once the installation is finished, create a directory named `tests` in the discount application.

It'll contain the tests we'll write for the models and the viewsets. 

```
└── tests
    ├── __init__.py
    └── test_models.py
    └── test_viewsets.py
    └── test_serializers.py
```
At the root of the project, add `pytest.ini` file. This will contain configurations such as the `DJANGO_SETTINGS_MODULE` and the python files to watch for the tests.

Let's write the test for the `Discount` model.

```python

import pytest

from django.utils.timezone import now, timedelta
from discount.models import Discount


@pytest.mark.django_db
def test_discount_model():
    discount = Discount(code="DIS20", value=5, description="Some discount",
                        created=now(), ended=now() + timedelta(days=2))
    discount.save()

    assert discount.code == "DIS20"
    assert discount.created < discount.ended
    assert discount.value == 5
```

Now use the `python manage.py test` command to run the tests.
Naturally, the tests will fail. As the Discount model is not created.

```shell
Found 1 test(s).
System check identified no issues (0 silenced).
E
======================================================================
ERROR: discount.tests.test_model (unittest.loader._FailedTest)
----------------------------------------------------------------------
ImportError: Failed to import test module: discount.tests.test_model
Traceback (most recent call last):
  File "/usr/lib/python3.9/unittest/loader.py", line 436, in _find_test_path
    module = self._get_module_from_name(name)
  File "/usr/lib/python3.9/unittest/loader.py", line 377, in _get_module_from_name
    __import__(name)
  File "/home/koladev/PycharmProjects/Django-Testing/discount/tests/test_model.py", line 4, in <module>
    from discount.models import Discount
ImportError: cannot import name 'Discount' from 'discount.models' (/home/koladev/PycharmProjects/Django-Testing/discount/models.py)


----------------------------------------------------------------------
Ran 1 test in 0.000s

FAILED (errors=1)
```

Let's create the Discount model then.

```python
from django.db import models


class Discount(models.Model):
    code = models.CharField(max_length=35, unique=True, db_index=True)
    value = models.FloatField()
    description = models.TextField(max_length=1000)
    created = models.DateTimeField(auto_now_add=True)
    ended = models.DateTimeField()
```

Now, let's create the migrations for this model.

```
python manage.py makemigrations
```

And let's run the migration.

```
python manage.py migrate
```

And now, run the test again. 

```
python manage.py test
```

And everything looks good.

```
Found 0 test(s).
System check identified no issues (0 silenced).

----------------------------------------------------------------------
Ran 0 tests in 0.000s

OK
```

And voila. We've just written our first unit test for this project. 
For the next steps, we'll be adding a serializer and a viewset. 

### Discount Serializer

For this, we'll install the  [Django Rest Framework](https://www.django-rest-framework.org/)  package. It contains tools needed to create RESTful APIs with Django.

```
pip install djangorestframework
```

Once it's done, add `rest_framework` to your `INSTALLED_APPS` settings.

```python
INSTALLED_APPS = [
    ...
    'rest_framework',
]
```

Great, we can now create the serializer. But first of all, let's write a test for that. 

```python
# ./discount/tests/test_serializers.py

import pytest

from django.utils.timezone import now, timedelta
from discount.serializers import DiscountSerializer


@pytest.mark.django_db
def test_valid_discount_serializer():
    valid_serializer_data = {
        "code": "DIS21",
        "value": 5,
        "description": "Some lines",
        "ended": (now() + timedelta(days=2))
    }

    serializer = DiscountSerializer(data=valid_serializer_data)
    assert serializer.is_valid(raise_exception=True)
    assert serializer.validated_data == valid_serializer_data
```

If you are not familiar with DRF, Serializer allows you to convert complex Django complex data structures such as `querysets` or model instances in Python native objects that can be easily converted `JSON/XML` format, but Serializer also serializes `JSON/XML` to naive Python. 

If you run the tests, it'll fail. Let's add the `DiscountSerializer.`

```python
# discount/serializers.py

from rest_framework import serializers

from discount.models import Discount


class DiscountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Discount
        fields = ('id', 'code', 'description', 'created', 'value', 'created', 'ended',)
        read_only_fields = ('id', 'created',)
```
Here, we created a serializer class named `DiscountSerializer` which outputs all the fields from the `Discount` model. Having `read_only_fields`, we ensure that they won't be updated or created via the serializer.

And now run the tests again. 

```shell
pytest
```

You'll get similar output.

```shell
============================ test session starts =============================
platform linux -- Python 3.9.5, pytest-6.2.5, py-1.11.0, pluggy-1.0.0
django: settings: CoreRoot.settings (from ini)
rootdir: /home/koladev/PycharmProjects/Django-Testing, configfile: pytest.ini
plugins: django-4.5.2
collected 2 items                                                            

discount/tests/test_model.py .                                         [ 50%]
discount/tests/test_serializers.py .                                   [100%]

============================= 2 passed in 0.23s ==============================
```

Now that we have the model and the serializer set up, we can use them together to create our viewsets and endpoint, which represent the logic of the business. 

### Discount Viewsets

We'll have three endpoints. 

| Routes                    | HTTP Method | Result            |
|---------------------------|-------------|-------------------|
| api/discount/             | POST        | Create a discount |
| api/discount/             | GET         | Get all Discounts |
| api/discount/discount_id/ | GET         | Get a discount    |

Let's start by writing tests. 

```python
#  ./discount/tests/test_viewsets.py
import json

import pytest

from django.utils.timezone import now, timedelta
from discount.models import Discount


@pytest.mark.django_db
def test_add_discount(client):
    discounts = Discount.objects.all()
    assert discounts.count() == 0

    response = client.post(
        "/api/discount/",
        {
            "code": "DIS21",
            "value": 5,
            "description": "Some lines",
            "ended": (now() + timedelta(days=2))
        },
        content_type="application/json"
    )

    assert response.status_code == 201
    assert response.data['code'] == "DIS21"

    discounts = Discount.objects.all()
    assert discounts.count() == 1


@pytest.mark.django_db
def test_get_all_discounts(client):
    response = client.get(
        "/api/discount/",
    )

    assert response.status_code == 200


@pytest.mark.django_db
def test_retrieve_discount(client):

    response = client.post(
        "/api/discount/",
        {
            "code": "DIS21",
            "value": 5,
            "description": "Some lines",
            "ended": (now() + timedelta(days=2))
        },
        content_type="application/json"
    )

    assert response.status_code == 201

    response_data = response.data

    response = client.get(
        f"/api/discount/{response_data['id']}/",
    )

    assert response.status_code == 200
```
If you run the tests, they will fail. 

To make sure the tests pass, we'll be adding the discount viewset and registering it in the project routes. Create a file called `viewsets.py` in the `discount application`. 

```python
from rest_framework import viewsets
from rest_framework.permissions import AllowAny
from discount.serializers import DiscountSerializer


class DiscountViewSet(viewsets.ModelViewSet):
    http_method_names = ['get', 'post']
    serializer_class = DiscountSerializer
    permission_classes = (AllowAny,)
    queryset = Discount.objects.all()
```

For the moment, we are only allowing `GET` and `POST` methods and for simplicity, no permissions are required to access this route.

Let's add now the routers. 

In the route of the project, create a file called `routers.py`. We'll register the viewsets into this file.

```python
from rest_framework import routers

from discount.viewsets import DiscountViewSet

router = routers.SimpleRouter()

router.register(r'discount', DiscountViewSet, basename='discount')

urlpatterns = router.urls
```

And the last step, let's add the new `urlpatterns` to the project `urls.py` file, here located in `CoreRoot` dir. 

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include(("routers", 'api'), namespace="core-api"))
]
``` 
Now run `pytest` again and the tests should pass. 

```shell
============================ test session starts =============================
platform linux -- Python 3.9.5, pytest-6.2.5, py-1.11.0, pluggy-1.0.0
django: settings: CoreRoot.settings (from ini)
rootdir: /home/koladev/PycharmProjects/Django-Testing, configfile: pytest.ini
plugins: django-4.5.2
collected 5 items                                                            

discount/tests/test_model.py .                                         [ 20%]
discount/tests/test_serializers.py .                                   [ 40%]
discount/tests/test_viewset.py ...                                     [100%]

============================= 5 passed in 0.35s ==============================
```

Great, now we have written unit tests for each part of the application. We can now move to integration testing. 


## Integration Testing

Integration testing is a phase of software testing where individual modules of the same software or external services are combined and tested as a group. 

Notice here the emphasis on "modules of the same software or external services". For example, integration testing will be combining a module handling authentication and permissions and a module handling payments. 
A test scenario will be to make sure that only authenticated and authorized users can initialize a payment. We will call this internal integration testing. 

Another case will be when you are using an external service for payment and you want to include this service in your test. We will call external integration testing.
Here in our project, we'll add another module and integrate it with our discount application. We'll also be adding a service for payment and integrating it into our environment.
For this, we will create a `cart` application. 
It'll behave as a cart as you see on an e-commerce website. But for sake of simplicity, we'll just tackle a few requirements and ignore some rules. Just keep it in mind. :) 
- the client can create a cart with a total, items_number, and currency
- the client can apply a discount to the cart

A cart here can be created with just the total, the items number, and the currency.

### Cart application

Let's create the application using the `django-admin` command. 

```shell
django-admin startapp cart
```
Now, go into the `settings.py` file of the project and register the newly created application.

```python
...
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    'discount',
    'cart'
]
...
```

As we did for the discount application, let's follow the TDD rules. 
Create a directory named `tests` containing files such as `test_models.py`, `test_serializers.py`, and `test_viewsets.py`.

### Adding the Cart model

Let's write the test to create a `Cart` instance first.

```python
import pytest

from cart.models import Cart


@pytest.mark.django_db
def test_cart_model():
    cart = Cart(total=10, currency="USD", items_number=5)
    cart.save()

    assert cart.total == 10
    assert cart.currency == "USD"
    assert cart.items_number == 5
    assert cart.payment_status == "pending"
```

Make sure the tests fail. 

```shell
============================ test session starts =============================
platform linux -- Python 3.9.5, pytest-6.2.5, py-1.11.0, pluggy-1.0.0
django: settings: CoreRoot.settings (from ini)
rootdir: /home/koladev/PycharmProjects/Django-Testing, configfile: pytest.ini
plugins: django-4.5.2
collected 5 items / 1 error / 4 selected                                     

=================================== ERRORS ===================================
_________________ ERROR collecting cart/tests/test_models.py _________________
ImportError while importing test module '/home/koladev/PycharmProjects/Django-Testing/cart/tests/test_models.py'.
Hint: make sure your test modules/packages have valid Python names.
Traceback:
/usr/lib/python3.9/importlib/__init__.py:127: in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
cart/tests/test_models.py:3: in <module>
    from cart.models import Cart
E   ImportError: cannot import name 'Cart' from 'cart.models' (/home/koladev/PycharmProjects/Django-Testing/cart/models.py)
========================== short test summary info ===========================
ERROR cart/tests/test_models.py
!!!!!!!!!!!!!!!!!!! Interrupted: 1 error during collection !!!!!!!!!!!!!!!!!!!
============================== 1 error in 0.12s ==============================
```

Let's create the Cart model. 

```python
from django.db import models


class Cart(models.Model):
    total = models.FloatField(default=0)
    currency = models.CharField(max_length=5)
    items_number = models.IntegerField(default=0)
    total_discounted = models.FloatField(default=0)
    amount_discounted = models.FloatField(default=0)
    payment_status = models.CharField(default="pending", max_length=35)
    created = models.DateTimeField(auto_now_add=True)
```

Now, let's generate the migrations for this model and migrate it. 

```shell
python manage.py makemigrations
python manage.py migrate
```

Once it's done, run the tests again and everything should be green.

```shell
============================ test session starts =============================
platform linux -- Python 3.9.5, pytest-6.2.5, py-1.11.0, pluggy-1.0.0
django: settings: CoreRoot.settings (from ini)
rootdir: /home/koladev/PycharmProjects/Django-Testing, configfile: pytest.ini
plugins: django-4.5.2
collected 6 items                                                            

cart/tests/test_models.py .                                            [ 16%]
discount/tests/test_model.py .                                         [ 33%]
discount/tests/test_serializers.py .                                   [ 50%]
discount/tests/test_viewset.py ...                                     [100%]

============================= 6 passed in 0.28s ==============================
```

Great, let's add the tests for the serializer and write the serializer.

```python
import pytest

from cart.serializers import CartSerializer


@pytest.mark.django_db
def test_valid_cart_serializer():
    valid_serializer_data = {
        "total": 10,
        "currency": "USD",
        "items_number": 5,
    }

    serializer = CartSerializer(data=valid_serializer_data)
    assert serializer.is_valid(raise_exception=True)
    assert serializer.validated_data == valid_serializer_data
```

Make sure the tests fail and now let's add the serializer.

```python
from rest_framework import serializers

from cart.models import Cart


class CartSerializer(serializers.ModelSerializer):
    class Meta:
        model = Cart
        fields = ('id', 'total', 'total_discounted', 'amount_discounted', 'items_number', 'created', 'currency', 'payment_status',)
        read_only_fields = ('id', 'total_discounted', 'amount_discounted', 'created', 'payment_status',)
```

Run the tests and it should pass now. And finally, let's add the routes and viewsets.
Here we'll need an endpoint to apply a discount. It should be a `POST` request containing a discount code in the body. 
We'll add it as an action to `CartViewset`.

```python
import pytest

from cart.models import Cart


@pytest.mark.django_db
def test_add_cart(client):
    carts = Cart.objects.all()
    assert carts.count() == 0

    response = client.post(
        "/api/cart/",
        {
            "total": 10,
            "currency": "USD",
            "items_number": 5
        },
        content_type="application/json"
    )

    assert response.status_code == 201

    carts = Cart.objects.all()
    assert carts.count() == 1


@pytest.mark.django_db
def test_get_all_carts(client):
    response = client.get(
        "/api/cart/",
    )

    assert response.status_code == 200


@pytest.mark.django_db
def test_retrieve_cart(client):
    response = client.post(
        "/api/cart/",
        {
            "total": 10,
            "currency": "USD",
            "items_number": 5
        },
        content_type="application/json"
    )
    assert response.status_code == 201

    response_data = response.data

    response = client.get(
        f"/api/cart/{response_data['id']}/",
    )

    assert response.status_code == 200
```
Make sure the tests fail. 

Let's now add the Viewset.

```python
from rest_framework import viewsets
from rest_framework.permissions import AllowAny
from cart.serializers import CartSerializer
from cart.models import Cart


class CartViewSet(viewsets.ModelViewSet):
    http_method_names = ['get', 'post']
    serializer_class = CartSerializer
    permission_classes = (AllowAny,)
    queryset = Cart.objects.all()
```
And register the viewset into the `routers.py` file.

```python
from rest_framework import routers

from discount.viewsets import DiscountViewSet
from cart.viewsets import CartViewSet

router = routers.SimpleRouter()

router.register(r'discount', DiscountViewSet, basename='discount')
router.register(r'cart', CartViewSet, basename='cart')

urlpatterns = router.urls
```

### Applying the discount

To apply the discount, we'll need to verify that a discount with the provided code exists and that this discount has not expired. 
And once it's done, we can now apply this discount to the cart.
Then here's how we can proceed: 
- add a method to the Cart model, which applies the discount and the value to the total
- Write a serializer specific to the `apply_discount` endpoint

Following the TDD principles, let's write a test for the `apply_discount` method on the `Cart` model.

```python
# cart/tests/test_model.py
...
@pytest.mark.django_db
def test_apply_discount_to_cart():
    cart = Cart(total=20, currency="USD", items_number=5)
    cart.save()

    # Creating the discount
    discount = Discount(code="DIS20", value=5, description="Some discount",
                        ended=now() + timedelta(days=2))
    discount.save()

    cart.apply_discount(discount)

    assert cart.amount_discounted == 5
    assert cart.total == 15
    assert cart.total_discounted == 15
```
Make sure the tests fail. We can now add the `apply_discount` method.

```python
from django.db import models


class Cart(models.Model):
    total = models.FloatField(default=0)
    currency = models.CharField(max_length=5)
    items_number = models.IntegerField(default=0)
    total_discounted = models.FloatField(default=0)
    amount_discounted = models.FloatField(default=0)
    created = models.DateTimeField(auto_now_add=True)

    def apply_discount(self, discount):

        self.amount_discounted += discount.value
        self.total_discounted = self.total - self.amount_discounted

        self.total = self.total_discounted

        self.save(update_fields=['total', 'amount_discounted', 'total_discounted'])
```

Run the tests again and it should pass. This is the beginning of the integration testing between `cart` and `discount` applications. But it's only done at a model level. 

Let's add the `apply_discount` serializer.

### ApplyDiscount Serializer

Let's add the tests first.

```python 
#  discount/tests/test_serializers.py

...
@pytest.mark.django_db
def test_apply_discount_serializer():
    # CREATING THE DISCOUNT
    discount = Discount(code="DIS20", value=5, description="Some discount",
                        created=now(), ended=now() + timedelta(days=2))
    discount.save()

    discount_data = {
        "code": "DIS20"
    }

    serializer = ApplyDiscountSerializer(data=discount_data)
    assert serializer.is_valid()


@pytest.mark.django_db
def test_apply_expired_discount_serializer():
    # CREATING THE DISCOUNT
    discount = Discount(code="DIS19", value=5, description="Some discount",
                        created=now(), ended=now() - timedelta(days=2))
    discount.save()

    discount_data = {
        "code": "DIS19"
    }

    serializer = ApplyDiscountSerializer(data=discount_data)
    assert not serializer.is_valid()


@pytest.mark.django_db
def test_apply_non_exist_discount_serializer():
    discount_data = {
        "code": "DIS32"
    }

    serializer = ApplyDiscountSerializer(data=discount_data)
    assert not serializer.is_valid()
```

Make sure the test fails and let's add the serializer.

```python
# discount/serializer.
```
Let's add this serializer in the `serializer.py` file of the discount application.

```python
class ApplyDiscountSerializer(serializers.Serializer):
    code = serializers.CharField(max_length=35)

    def validate(self, attrs):
        code = attrs.get('code')
        try:
            discount = Discount.objects.get(code=code)
        except Discount.DoesNotExist:
            raise validators.ValidationError("This discount doesn't exist.")

        if discount.ended < now():
            raise validators.ValidationError("This discount has expired.")

       attrs['discount'] = discount

        return attrs
```

Run the tests again and it should pass. We are now sure that a valid discount will pass and be applied. 

Next step, we'll be adding an extra route for this on the `CartViewset`, thanks to viewsets `@action`.

### `apply_discount` action on CartViewSet

`apply_discount` will be an action on the `CartViewSet`. 
As described in the  [official docs of DRF](https://www.django-rest-framework.org/api-guide/viewsets/#marking-extra-actions-for-routing) , if you have ad-hoc methods that should be routable, you can mark them as such with the @action decorator. Like regular actions, extra actions may be intended for either a single object, or an entire collection.

Let's add some tests for this in the `test_viewsets.py` file of the `cart` application.

```python

...
@pytest.mark.django_db
def test_apply_discount_to_cart(client):
    # CREATING THE DISCOUNT
    discount = Discount(code="DIS20", value=5, description="Some discount",
                        ended=now() + timedelta(days=2))
    discount.save()

    response = client.post(
        "/api/cart/",
        {
            "total": 20,
            "currency": "USD",
            "items_number": 5
        },
        content_type="application/json"
    )
    assert response.status_code == 201

    response_data = response.data

    response = client.post(
        f"/api/cart/{response_data['id']}/apply_discount/",
        {
            "code": "DIS20"
        },
        content_type="application/json"
    )

    assert response.status_code == 200
    assert response.data['total'] == 15
    assert response.data['total_discounted'] == 15
    assert response.data['amount_discounted'] == 5
```

Make sure the tests fail. Once it's done, we can now add the `apply_discount` action.

```python

from rest_framework import viewsets, status
from rest_framework.permissions import AllowAny
from rest_framework.decorators import action
from rest_framework.response import Response

from cart.serializers import CartSerializer
from cart.models import Cart
from discount.serializers import ApplyDiscountSerializer

...
    @action(methods=['post'], detail=True)
    def apply_discount(self, request, pk=None):
        obj = self.get_object()

        serializer = ApplyDiscountSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        obj.apply_discount(serializer.validated_data['discount'])

        return Response(CartSerializer(obj).data, status=status.HTTP_200_OK)
```
We are passing to the `@action` decorator two parameters: 
- The list of HTTP methods accepted, here `POST`. 
- And the detail parameter. This will tell Django that the client has to provide an `id` and also, the URL will have this structure: `cart/<cart_id>/apply_discount/`.

Now, let's run the tests again. And everything should be green. 

Congratulations! We've just written integrations tests. But there is the last feature we have to develop. Yes, the  `payment` feature. 

## External Integration testing

If your application depends on external services, it's a good industrial habit to include those services in your testing. But this comes with some questions. 

Are you going to integrate a Real API into your tests? 

Even if the service provides a testing environment, how do you come with solutions if your tests are firing many requests and they get blocked? Or what if the service is unavailable? 

We can avoid hitting the real services API by running our own fake servers while running the integrations tests.
Then most of the time you have two choices: 
- Mock the API using `unittest.mock`. But what is **Mocking**? 
**Mocking** means substituting or imitating a real object or service within a testing environment. This  [article](https://realpython.com/python-mock-library/#what-is-mocking)  from RealPython illustrates it well.
- Building your own fake server for integration tests: This solution is a good idea in some scenarios. 
As stated in this  [article](https://www.cosmicpython.com/blog/2020-01-25-testing_external_api_calls.html), you should go with this: 
- if the integration is not core to your application, i.e it’s an incidental feature
- if the bulk of the code you write, and the feedback you want, is not about integration issues, but about other things in your app
- if you really can’t figure out how to fix the problems with your integration tests another way (retries? perhaps they’d be a good idea anyway?)

For this tutorial, we'll go with the second choice. We'll build a simple server to imitate a card payment API and spin it up in a docker container. 
Then, we'll need to dockerize our project. Bur, first of all, let's quickly create the Flask server.

### Flask application

 [Flask](https://flask.palletsprojects.com/en/2.0.x/)  is a very lightweight framework coming with the necessary tools to create an API or start a simple server. 
We'll use this to imitate a payment provider API for our tests. 

First of all, make sure to have `flask` installed on your project. We'll also install `python-dotenv` to load environment variables.

```shell
pip install flask python-dotenv
```

Once it's done, create a file called `fake_payment_server.py`. And add the following content. 

```python
from flask import Flask, jsonify

app = Flask(__name__)


@app.route('/inspect')
def inspect():
    return jsonify({
        "available": 1
    })


if __name__ == '__main__':
    app.run()
```

Create a `.env` file at the root of the project. This will contain some configs for the `Flask` server but also the `Django` server.

```
FAKE_PAYMENT_API=fake_payment_api:5005
FLASK_APP=fake_payment_server.py
```

Now run the Flask server with `flask run`. The server will normally be running at `localhost:5000`. This is to make sure there is no issue before we proceed to the next step.

Now, we can dockerize the project.

### Dockerizing the project

But why Docker? 
It helps you separate your applications from your infrastructure and helps in delivering code faster. In this case, `Docker` allows us to run your tests in containers as well as isolate your tests in development and deployment.
If it's your first time working with Docker, I highly recommend you go through a quick tutorial and read some documentation about it.

Here are some great resources that helped me:

-  [Docker Tutorial](https://www.youtube.com/watch?v=eN_O4zd4D9o&list=PLPoSdR46FgI5wOJuzcPQCNqS37t39zKkg) 
-  [Docker curriculum](https://docker-curriculum.com/) 

For this step, make sure you have `docker` and `docker-compose` installed on your machine.

At the root of the project, create a file named `Dockerfile`.

```docker
# pull the official base image
FROM python:3.10-alpine

# set work directory
WORKDIR /app

# set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# install psycopg2 dependencies
RUN apk update \
    && apk add gcc python3-dev musl-dev

# install python dependencies
COPY requirements.txt /app/requirements.txt
RUN pip install --upgrade pip
RUN pip install --no-cache-dir -r requirements.txt


# copy project
COPY . .
```

Here, we started with an **Alpine-based Docker Image for Python**. It's a lightweight Linux distribution designed for security and resource efficiency. 
After that, we set a working directory followed by two environment variables: 
 
1 - `PYTHONDONTWRITEBYTECODE` to prevent Python from writing `.pyc` files to disc
2 - `PYTHONUNBUFFERED` to prevent Python from buffering `stdout` and `stderr`

After that, we perform operations like: 
- Setting up environment variables
- Copying there `requirements.txt` file to our app path, upgrading pip, and installing the python package to run our application
- And last copying the entire project

Also, let's add a `.dockerignore` file. 

```
env
venv
.dockerignore
Dockerfile
```

Once it's done, create a file called `docker-compose.yml`. 
[Docker Compose](https://docs.docker.com/compose/)  is a great tool (<3). You can use it to define and run multi-container Docker applications.
 
What do we need? Well, just a YAML file containing all the configuration of our application's services. 
Then, with the `docker-compose` command, we can create and start all those services.

```yml
version: '3.9'
services:
  api:
    container_name: api
    build: .
    restart: always
    env_file: .env
    ports:
      - "8000:8000"
    command: >
      sh -c " python manage.py migrate &&
          gunicorn CoreRoot.wsgi:application --bind 0.0.0.0:8000"
    volumes:
     - .:/app

  flask_api:
    container_name: fake_payment_api
    build: .
    restart: on-failure
    ports:
      - "5005:5005"
    command: >
      sh -c "flask run --host=0.0.0.0 --port=5005"
    volumes:
      - .:/app/
```
Now, we can build the containers and start running the services.

```
docker-compose up -d --build
```

To make sure everything works well, we can run the tests on the API and see how it goes.

```
docker-compose exec api pytest
```

Great, we can start integrating the Flask server to our API now.

### Testing Flask server availability

We'll simply make a `GET` request on the `/inspect` endpoint of the flask server. And as we are running these tests into the docker environment, we will be using the name of the flask container as a network host. 

That's why into the .env file we've included the `FAKE_PAYMENT_API=fake_payment_api:5005` line.

In the `settings.py file`, import `dotenv` and force env vars loading at the beginning of the file.

```python
from dotenv import load_dotenv

load_dotenv()

ENV = os.environ.get('ENV', 'DEV')

...
FAKE_PAYMENT_API = os.environ.get('FAKE_PAYMENT_API')
```

Inside, the `test_viewsets.py` file of the `cart` application, let's add a test to check the fake server is reachable.

Before this, make sure to have the `requests` package installed and make sure to have it in your `requirements.txt` file.

```python
...
import requests
from django.conf import settings
...

def test_inspect_payment_api(client):
    response = requests.get(f'http://{settings.FAKE_PAYMENT_API}/inspect')

    assert response.status_code == 200
    assert response.json()['available'] == 1
```
Rebuild the containers again and run the tests. 

```shell
docker-compose exec api pytest
```

Everything should be green. We can now add an endpoint to the flask API which will handle payments.

### Payment feature 

In a real-world scenario, you'll be integrating services like Stripe or Paypal APIs for such things.

For the sake of simplicity here, we'll suppose that there is actually 20 USD on the wallet/the card used for the payment. 

To make a payment then, a request will be made on `request_payment` endpoint of the `Flask` server. This request should contain in the body the `cart_id` and the `amount` to be debited. 

We'll then compare the `amount` to the `balance` constant of 20 to make sure the payment is doable.

If the amount is superior or equal to 20, we authorize the payment and return the `cart_id` and a `payment_status` set to `success` of successful payment and `failed` for failed payment.

Here's the endpoint. 

```python
...
app = Flask(__name__)

CARD_BALANCE = 20
...

@app.route('/request_payment', methods=['POST'])
def request_payment():
    data = request.get_json(force=True)

    cart_id = data.get('cart_id')
    amount = data.get('amount')

    if cart_id is None:
        abort(400, {'cart_id': "This field is required"})

    if amount is None:
        abort(400, {'amount': "This field is required"})

    if not isinstance(cart_id, int) or not isinstance(amount, int):
        abort(400, {'type': "The fields should be integers."})

    if amount >= CARD_BALANCE:
        return jsonify({
            'cart_id': cart_id,
            'payment_status': "failed"
        })

    return jsonify({
        'cart_id': cart_id,
        'payment_status': "success"
    })
``` 

And very simple, we have a payment endpoint. Let's integrate it to the `cart` application.

### Integrating the payment API

A good practice when integrating API is to write a wrapper. 
This is actually useful if the API doesn't provide a module in the language your are working with. 
An **API wrapper** is a language-specific package or kit that encapsulates multiple API calls to make complicated functions easy to use. It creates an abstraction from the API endpoints providing readable methods or functions that can be reused anywhere in the code.

We'll be adding `pay` method on the `Cart` model. 
But first of all, let's write a test for that and make it crash.

```python
...
@pytest.mark.django_db
def test_cart_pay():
    cart = Cart(total=10, currency="USD", items_number=5)
    cart.save()

    assert cart.total == 10
    assert cart.currency == "USD"
    assert cart.items_number == 5
    assert cart.payment_status == "pending"

    cart.pay()

    assert cart.payment_status == "success"
```

Let's add the `pay` method on the `Cart` model to handle the payment.

```python
    ...
    def pay(self):
        initialized_payment = PaymentAPI()
        payment = initialized_payment.request_payment(cart_id=self.pk, amount=self.total)

        assert payment['cart_id'] == self.pk

        self.payment_status = payment['payment_status']
        
        self.save(update_fields=['payment_status'])
```

The last thing remaining is to add the `PaymentAPI` wrapper, here a `class`.

Create a file called `utils.py` at the root of the project and enter the following code.

```python
from typing import Optional

from django.conf import settings
import requests


class PaymentAPI:
    API_URL = None

    def __init__(self):
        if settings.ENV in ['PROD']:
            self.API_URL = settings.REAL_PAYMENT_API
        else:
            self.API_URL = settings.FAKE_PAYMENT_API

    def request_payment(self, cart_id: int, amount: int ) -> Optional[dict]:

        data = {
            "cart_id": cart_id,
            "amount": amount
        }

        response = requests.post(f"http://{self.API_URL}/request_payment", json=data)

        return response.json()
```


And here, we have the `wrapper`. As you can see, we are rewriting the `__init__()` method to assign a value to `API_URL`. And then we have the `request_payment()` method to make a request on the API and returns the response.

Run the tests again and everything should be green.

Let's finally add a test to check payment failure. In this case, the amount is superior to 20.

```python
...
@pytest.mark.django_db
def test_cart_pay_failed():
    cart = Cart(total=25, currency="USD", items_number=5)
    cart.save()

    assert cart.total == 25
    assert cart.payment_status == "pending"

    cart.pay()

    assert cart.payment_status == "failed"
```

And here it is. We have write some external integration tests.

## Github actions

As a bonus part, let's create CI/CD pipeline using Github actions. This is useful when you want to make sure you are deploying non-failing software in a production environment.

 [GitHub actions](https://github.com/features/actions)  are one of the greatest features of Github. it helps you build, test or deploy your application and more.

Here, we'll create a `YAML` file named django.yml to run some Django tests.

In the root project, create a directory named .github. Inside that directory, create another directory named `workflows` and create the `django.yml` file.

Here's the code.

```yml
name: Django CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Creating env file
      run: |
        touch .env
        echo FAKE_PAYMENT_API=fake_payment_api:5005 >> .env
        echo FLASK_APP=fake_payment_server.py >> .env
    - name: Building containers
      run: |
        docker-compose up -d --build
    - name: Running Tests
      run: |
        docker-compose exec -T api pytest
```

Basically, what we are doing here is setting rules for the  [GitHub action workflow](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions) , installing dependencies, and running the tests.

- Make sure that this workflow is triggered only when there is a push or pull_request on the main branch
- Choose ubuntu-latest as the OS.
- Next, we create a `.env` file which Docker will use to run the containers.
- After that as we build the containers and run the tests.

![Django Github CI/CD](https://cdn.hashnode.com/res/hashnode/image/upload/v1639299710688/5Kc0mPmay.png)

And voilà. Here's how you can start with Unit testing and Integration testing in your Django projects using Docker, Flask, and Github actions. 

But let's quickly talk about good testing practices

### Best practices

This part of the article, I believe is the most important. The world of Software Testing is not only limited to unit test and integration testing. There is a lot more. You can have E2E (End-to-end) testing, contract testing, Exploratory Testing, Acceptance testing and a lot more. 
The good news about this is that it's up to you what testing strategy you are adopting and what type of tests to include as well. 
Just make sure that the quality of the software at the end is always high. You can learn more about testing in the  [article](https://martinfowler.com/articles/practical-test-pyramid.html)  on the Martin Fowler blog, which I recommend.

Now that there is some clarification about the testing terminology, here are some good practices when writing tests: 
- Avoid Test duplication: Avoid having the same tests at a different parts of the project. 
- Test one thing at  time: If you find yourself writing testing code unrelated to the role of what the function tests, just write another testing function.
- Tests should be fast: To make sure you can have very fast tests, focus on writing a lot of unit tests.

## Conclusion

In this article,  we've learned how to write unit and integrations tests for a Django application, using Docker, Flask, and Github actions too.

And as every article can be made better so your suggestion or questions are welcome in the comment section. 😉

Check the code of this tutorial  [here](https://github.com/koladev32/django-unit-integration-testing). 

*Article posted using [bloggu.io](https://bloggu.io). Try it for free.*

