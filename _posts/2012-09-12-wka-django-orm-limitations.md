---
layout: post
title: "Workaround Django ORM limitation - create views"
description: "When Django ORM is not enough, there is other solutions."
category: Django
tags: [Django, ORM]
---
{% include JB/setup %}

I faced this problem yesteday.

------------------      ------------------      ------------------
| Recommendation |      | Product        |      | Promotion      |
------------------      ------------------      ------------------
| ID             |  |-> | ID             | <-|  | ID             |
| Product        | -|   |                |   |- | Product        |
------------------      ------------------      ------------------


and I needed to sort promotion by the presence of recommendations, ie
"Promotion were a recommendation exists should be first".

The Django way
--------------

I still haven't found a proper way to do it with plain Django.
But, if one solution exists, that would be:
- Build 2 querysets and merge them by any way (Merging a queryset is quite painful). Problem: code messy + lose the queryset for a list
- For any promotion, do a query to get the recommendation. Problem: performance
- Use extra. Problem: Never managed to do it, unreadable and hard to maintain
- Raw SQL. Problem: We have many other filters to apply + lose the queryset for a rawQuerySet

The better solution
-------------------

What if we already have promotions and recommendations info in the same table, that would be easier.
That's what a view is designed for. Providing up-to-date information coming from other tables.

Django works nicely with views since 1.1. What you want is to add the ID of `Recommendation` to the promotion table and if there is no match, add Null.
That's a LEFT JOIN.

That's done with:

    DROP VIEW my_view
    CREATE VIEW my_view AS
        SELECT 
            promotion.*, recommendation.ID
        FROM
            promotion LEFT JOIN recommendation
            ON promotion.product == recommendation.product

Now, to avoid duplication, our promotion and recommendation view should extend the same base class `Promotion`

    class AbstractPromotion(models.Model):
        product = models.ForeignKey(Product, related_name="%(class)s")    #(1)
     
        class Meta:
            abstract = True

    class Promotion(AbstractPromotion): pass

    class PromotionView(AbstractPromotion):
        recommendation_id = models.IntegerField()     # (2)

        class Meta:
            ordering = ["recommendation_id"]     # (3)

1. You need to have different related names to not have Django complaining. That's the way to do it or (read the doc)[https://docs.djangoproject.com/en/dev/topics/db/models/#s-be-careful-with-related-name]

2. I haven't tested it with a ForeignKey but I don't see why that would work.
3. That was our main problem and we fixed it.

Job done?
---------

We fixed the problem nicely but now we should be able to integrate this on our workflow to not break our automated deployment or testing.

How to create automatically this view ? The answer is **SOUTH**

    def forwards(self, orm):
        db.execute("DROP VIEW IF EXISTS my_view")    #(1)
        db.execute("""CREATE VIEW my_view AS
                          SELECT
                              promotion.*, recommendation.ID
                          FROM
                              promotion LEFT JOIN recommendation
                          ON promotion.product == recommendation.product
                   """)

1. Why not "CREATE VIEW OR REPLACE"? simply because that's not compatible with (sqlite)[http://www.sqlite.org/lang_createview.html]
