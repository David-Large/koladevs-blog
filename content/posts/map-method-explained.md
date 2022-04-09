---
title: "Python map() Method Explained"
date: 2022-04-08T00:28:02+01:00
draft: false
author: "Mangabo Kolawole"
featured_image: "map.png"
---

Do you know that it's possible to process **iterables** without a Loop? The `map` method is really useful when you need to perform the same operation on all items of an iterable (list, sets, etc).

The `map()` method takes two arguments: 
- a function object
- an iterable or multiple iterable

The function passed to the `map()` method will perform some action on each element of the iterable passed as an argument.

### Example

```python
>>> fruits = ["lemon", "orange", "banana"]
>>> def add_is_a_fruit(fruit):
...     return fruit + " is a fruit."
...
>>> new_fruits = map(add_is_a_fruit, fruits)
<map object at 0x7f7fe85e6460>
>>> new_fruits = list(new_fruits)
>>> new_fruits
['lemon is a fruit.', 'orange is a fruit.', 'banana is a fruit.']
```

Notice that you have to convert the returned map object into an iterable so you can work with it easily.

It's also possible to use `map()` with lambda functions. 

```python
>>> new_fruits = map(lambda fruit: fruit + " is a fruit.", fruits)
>>> new_fruits = list(new_fruits)
>>> new_fruits
['lemon is a fruit.', 'orange is a fruit.', 'banana is a fruit.']
```

You can learn more about the method [here](https://docs.python.org/3/library/functions.html#map).
