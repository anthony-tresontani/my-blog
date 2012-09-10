---
layout: post
title: "Packaging a python app"
description: "Package an app for the other goods"
category: "Python"
tags: ["Python"]
---

or what you should know to package a python application.

This blog is about the easiest place to put your code and do not fit for big project.
But if you have a really nice open source personnal project and want to share it with the community, it's a good starting point.


**Put in on github**


The first place where you project should be visible is on GitHub.
Just signup on github if not already done and create a new repo for your project.

**Document it !**

Even obvious usage of your package should be documented for new python user.
The easiest way is readthedoc.org. 

It rely mainly on sphinx documentation. You can find everything you need [here](http://sphinx.pocoo.org/) 

Once done, sign in and set your project to your github repo.

**Pypi**

Be finding a default setup.py, hacking it a bit, sign in pypi and run:

    python setup.py register
    python setup.py sdist upload

That's it.

You will be amazed of how many people will start downloading your code (and maybe even using it).

