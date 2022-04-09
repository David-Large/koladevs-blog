---
title: "Django Tip: Use DecimalField for money"
date: 2022-04-06T00:28:02+01:00
draft: false
author: "Mangabo Kolawole"
featured_image: "moneyfloatdecimals.png"
---

When working with money values in Django, the first thought is to use `FloatField` to represent currency in the model. However, `FloatField` uses the `float` type internally which comes with some precision issues. 

## The problem

Take a look at this piece of code. It's just a simple operation with float numbers in Python.

```python
>>> .1 + .1 + .1 == .3
False
>>> .1 + .1 + .1
0.30000000000000004
```
Normally, you think that the addition will make sense but because of some issues with the float approximation, the equation is not equal to 3.
You can fix these issues using rounding but if you are dealing with money values and precision matters, it might be time to use decimals.

## The solution

Basically, use decimals instead of floats when precision matters. Let's rewrite the previous example but with `Decimal`.
```python
>>> from decimal import Decimal
>>> Decimal('.1') + Decimal('.1') + Decimal('.1') == Decimal('.3')
True
>>> Decimal('.1') + Decimal('.1') + Decimal('.1')
Decimal('0.3')
```
Notice that here we are initializing the decimals values with string values. You can use floats but as we said earlier, floats have their approximation issues. 

Then when working with Decimal in Django with the `DecimalField`, it's always a good habit to precise the decimal_places attribute when defining the field.
```python

class Product(models.Model):
    title = models.CharField(max_length=64)
    description = models.TextField()

    price = models.DecimalField(max_digits=6, decimal_places=2)
```

You can learn more about `DecimalField` [here](https://docs.djangoproject.com/fr/4.0/ref/models/fields/#decimalfield).
