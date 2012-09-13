---
layout: post
title: "Patching django polymorphism"
description: "Django allow some incomplete polymorphism but there is a simple way to enhance it"
category: "Python"
tags: ["Python", "Django"]
---
{% include JB/setup %}

## Remember OOP

For the one who don't remember what is the abstract notion of [polymorphism](http://en.wikipedia.org/wiki/Polymorphism_in_object-oriented_programming). Here is a reminder:

    class Duck(object):
        def quack(self):
            print "QUACK"

    class FrenzyDuck(Duck):
        def quack(self):
            print "QUACK" * 4

    ducks = [Duck(), FrenzyDuck()]
    for duck in ducks:
        duck.quack()

And you will get:

    "QUACK"
    "QUACKQUACKQUACKQUACK"

Now, if we translate this in a Django way:

    class Duck(models.Model):
        name = models.CharField(max_length=15)
        def quack(self):
            print "QUACK"

    class FrenzyDuck(Duck):
        def quack(self):
            print "QUACK" * 4

    fifi = Duck.objects.create(name="fifi")
    loulou = FrenzyDuck.objects.create(name="loulou")

    fifi.quack()
    loulou.quack()

    for duck in Duck.objects.all():
        duck.quack()

you get:

    "QUACK"
    "QUACKQUACKQUACKQUACK"
    "QUACK"
    "QUACK"    # Should have been "QUACKQUACKQUACKQUACK"

Django provides some mechanisms to emulate polymorphism, but that works only in some restricted cases.
Why doesn't Django call the right method? Cause Django don't remember and we didn't give Django a way to remember.

In the database, we would have something like that:

    > select * from duck_duck;
    ID | NAME
    ---------
    1  | fifi
    2  | loulou

There is obviously nothing which says "Loulou is more than a duck, he is a frenzy duck".

## Why you should care?

That doesn't seem to be such a big deal.

I am working in a company where we build e-commerce websites. Different kinds of products have different behavior.
We can have:
- Normal product
- Product grouped together, like multiple sizes T-shirt
- Customisable product, like a T-shirt where you can define your own logo


Our code would have looked like:

    def my_view(request, product):
        if product.is_customised():
           product.do_customised_things()
        elif product.is_grouped():
           product.do_grouped_things()
        else:
           product.do_normal_things()

Not really a sexy code. With polymorphism, we would have:

    def my_view(request, product):
        product.do_things()

Much better.

We also have custom display of theses products, eg a template can look like:

   {% raw %} {% for product in products %}
        {% if product.is_customised %}
            {% include "customised.html" %}
        {% elif product.is_grouped %}
            {% include "grouped.html" %}
        {% else %}
            {% include "normal.html" %}
    {% endfor %}{% endraw %}

why not:

    {% raw %}{% for product in products %}
        {% include product.template_name %}
    {% endfor %}{% endraw %}

That's shorter, more DRY and easier to maintain. I want that in my project.

## Solution

If we want fully support polymorphism, we should find a way to tell Django which object is of which class.
From here, you can find many solutions, most of them relying on storing the real class in a table.

We have an original approach. Most of the time, you know the type of object you have indirectly because of another attribute having a particular value.
For example, a grouped product will have is_group value set to True. IE, there are some rules in your system which allow you to know the real class of an object.

Now, we should find a way to let Django know these rules. The `get_proxy_class` pattern (purely invented for the sake of this article).

    from django.db import models
    from django.db.models.signals import post_init
    from django.dispatch.dispatcher import receiver

    class Duck(models.Model):
        name = models.CharField(max_length=15)
        def quack(self):
            print "QUACK"

        def _get_proxy_class(self):     #(1)
            if self.name == "loulou":
                return FrenzyDuck
            return Duck

    class FrenzyDuck(Duck):
        def quack(self):
            print "QUACK" * 4

        class Meta:                      #(2)
            proxy = True

    @receiver(post_init)                 #(3)
    def update_proxy_object(sender, **kwargs):
        instance = kwargs['instance']
        if hasattr(instance, "_get_proxy_class") and not instance._meta.proxy:
            instance.__class__ = instance._get_proxy_object()

    fifi = Duck.objects.create(name="fifi")
    loulou = FrenzyDuck.objects.create(name="loulou")

    fifi.quack()
    loulou.quack()

    for duck in Duck.objects.all():
        duck.quack()

now, you get:

    "QUACK"
    "QUACKQUACKQUACKQUACK"
    "QUACK"
    "QUACKQUACKQUACKQUACK"

Here are some explanations:

(1) You define a method which knows which class should be returned depending on some rules.

(2) Your subclass should be a proxy. That's also working with real subclass, but I would avoid using it as you may have object not completely initialized and it's a nightmare to debug.

(3) When the object is initialized by Django, if the get_proxy_class is defined, we cast the object. We do that only if the object is not already a proxy.

With that solution, you can now have fully featured polymorphism at no extra (DB) cost.

## Too good to be real?

As any solution, there are some trade-offs. We are using it in [tangentlabs](http://www.tangentlabs.co.uk/) and we get great benefits from it.
But there is still:
- You cannot easily define model with different data. We tried and we failed, spending many hours trying to understand why this object is badly initialized.
- Your main class is responsible to provide this mechanism, IE your parent class knows all its children. That would prevent to use this mechanism for an external application. This limitation can be easily worked around by providing a function external to the class, like:

    def get_proxy_class(instance):
        if isinstance(instance, Duck):
            ... previous code ...

That's it.
