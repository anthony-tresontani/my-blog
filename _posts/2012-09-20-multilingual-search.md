---
layout: post
title: "Multilingual search in Django with Haystack"
description: "How to perform independant multilingual search effortless"
category: Django
tags: [Django, Haystack, Solr, Search]
---
{% include JB/setup %}

Multilingual search in Django with Haystack
-------------------------------------------

When you build a website for a swiss company, like we do at tangent, that's likely you would need to perform search for 3 languages.

There is multiple way to perform multilingual search, and Steve Kearns explains that in this [slideshare](http://www.slideshare.net/lucenerevolution/steve-kearns-multilingualsearcheurocon2011)

We chose to implement the solution at indexing time, and that was, unpredictabily easy.


The solution
------------

The solution is really simple, let say you want to index a book title in 3 languages: german, french and italian.
Your book will look like:

    class Book(models.Model):
        title_de = models.CharField(max_length=100)
        title_fr = models.CharField(max_length=100)
        title_it = models.CharField(max_length=100)

Your Haystack index will look like:

    class BookIndex(indexes.RealTimeSearchIndex, indexes.Indexable):
        title =  indexes.CharField(document=True)

At indexing time, we will send an index to solr for each language to a specific solr core.

    ------------        "Mein Buch" ---|    ------------
    | SOLR FR  |<------ "Mon livre"    ---->| SOLR DE  |
    ------------        "Mio libro"         ------------
                             |
                             |
                             V
                        ------------
                        | SOLR IT  |
                        ------------


At query time, we will only query the core matching our language:

                                                                  ------------
   Please solr give me a french book called "Les miserables" ---->| SOLR FR  | ---> French results 
                                                                  ------------

Implementation
--------------
