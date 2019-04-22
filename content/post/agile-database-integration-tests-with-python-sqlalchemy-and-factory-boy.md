---
title: 'Agile database integration tests with Python, SQLAlchemy and Factory Boy'
date: '2019-04-22T18:52:14+02:00'
draft: true
---
![](https://cdn-images-1.medium.com/max/800/1*P4nj9fJjSeJ9-c0rwSZqlg.png)

<span class="figcaption_hack">[xkcd](https://xkcd.com/327/)</span>

So you are interested in testing, aren’t you?<br> Not doing it yet? That’s the
right time to start then!

In this little example, I’m going to show a **possible** procedure to easily
test your piece of code that interacts with a database.

Doing database integration tests, you will generally need few things for each
test case:

* Setup test data in one or more tables (setup)
* Run the under-test functionality
* Check actual state on database is as expected (assertions)
* Clear all for the next test (teardown)

If not done carefully, most of these steps could bring *more-than-wanted* lines
of code. Moreover, the resulting test runs are often quite slow.

The most difficult part is, in my experience,** setting up** and **tearing
down** the database state to test the piece of code in question.

> *It’s simple, I’ll just insert data with a couple of raw “insert” queries and
> cleanup tables with “deletes”.*

> *Done!*


```python
def ciao():
    pass
```


Do it, and you’ll most likely face walls of SQL code, hardly reusable, hard to
read and even harder to maintain. A very tedious and error prone method. <br>
Moreover you can encounter performance issues, especially if more tables are
involved or if there are cascading deletes.

<span class="figcaption_hack">You, after reading 300 lines of SQL statements</span>

The best way to isolate your test case is an already present feature of SQL:
**transactions**. Yes, they can help you.

Let’s suppose you are using the excellent
[pytest](https://docs.pytest.org/en/latest/) framework for your tests, a couple
of well designed fixtures will do the work:

Woah! So easy?! Yes, it is.<br> In a few words:

* `connection` fixture makes sure there is only one database connection and that
it’s closed after all test are run.
* `session` fixture makes sure each test runs in a separate transaction, and
cleanup is guaranteed by the rollback. Rollback will be always faster that any
custom delete query and will not forget to restore anything 😉

> *… Ok, but we haven’t entered any data yet! …*

You’re right, let me introduce you to …

### **Factory Boy**

Factory Boy is a tool for creating on demand pre-configured objects. It’s very
handy and I personally suggest you to use it extensively.

Let’s say you have a `User` class with *name* and *email*. With Factory Boy you
can create a class bound with your `User` model.

Now, just calling `UserFactory.build()` will generate a *filled* `User`
instance.<br> There are lots of features, such as properties overriding,
sequences and so on. Take a look at the
[docs](http://factoryboy.readthedocs.io/en/latest/)!

> *… Umm, nice! But what has this got to do with database? …*

Well, what if I say you can actually bind factories with SQLAlchemy models?

That’s all! Now creating and **having a user on your database** is simple as
writing` UserFactory.create()`.

*Mind. Blown.*

Want more objects? `UserFactory.create_batch(6)`

*Mind. Blown. ***Again***.*

If you think you didn’t get it, here is a full integration test in its splendor:

At line 5, you can see how SQLAlchemy session is bound to the factory class.
Every action performed by the factory will use our session, whose lifecycle is
handled by fixtures themselves.

Easy, few lines long, clean, understandable and … obviously it works! 😃<br>
Such a smooth integration testing!

Of course this example is intentionally trivial, you can find slightly more real
and organized examples in this
[repository](https://github.com/ProvoK/article_agile_database_integration) I
created on purpose.

#### Recap

In order to have a clean database integration testing, you should aim for:

1.  Running your test cases in isolated transactions, and rollback them.
1.  Describe your entities as models and delegate their creation to a factory
system.

> *If there is not a factory library or system for your language, do it yourself
> and open source it!*

*****

> Thanks for your time!

> Let me know if you liked the article and what you think about it! 👋

#### Useful links

* [pytest](https://docs.pytest.org/en/latest/)
* [factory_boy](http://factoryboy.readthedocs.io/en/latest/index.html)
* [sqlalchemy](http://docs.sqlalchemy.org/en/latest/)
* [faker](http://faker.readthedocs.io/en/master/)

* [Testing](https://medium.com/tag/testing?source=post)
* [Python](https://medium.com/tag/python?source=post)
* [Database](https://medium.com/tag/database?source=post)
* [Learning](https://medium.com/tag/learning?source=post)
* [Sql](https://medium.com/tag/sql?source=post)

### [Vittorio Camisa](https://medium.com/@vittorio.camisa)

Software engineer, backend developer and Python enthusiast. Interested in
microservices and DevOps practices.

The problem here is if you have lots of factories or the factories use
SubFactories (i.e. dependent models). It gets quite unwieldy then to attach the
session to every single factory in the session fixture.

A better approach might be to define each factory inside a function:

    def…

Thanks for your comment!

As always when coding, any level of abstraction may be useful in some cases.

I’m not sure your approach will make more compact or easier the tests code
though: doing so you need to replicate the `make_...` function in every test,
and the issue with subfactories is exactly the…

Regardless whether you use Subfactories or not (I agree with the integrity
issues, I have run into them too), the problem with attaching the session to
every Factory instance in your system is that you need to remember to do it
every time you add a new factory. In a very large application where you might
have possibly dozens of models, and hence…

Yes, I agree with you.

I usually works with small services with an handful of tables, so I’ve not run
in your issues, but I definitely understand.

You should consider to make a PR to factoryboy itself, it would be awesome if
sqlalchemy subfactories inherits sessions from parent factories!

Great article — sadly, I am addicted to ModelMommy … but I assume its possible
to adapt your approach somewhat?

Thanks!

I never heard about ModelMommy but if it’s this
(http://model-mommy.readthedocs.io/) it’s pretty obvious you are using Django.

It seems a factory object builder tool like FactoryBoy, you probably can stick
with it!<br> I’m not very experienced in Django testing, but I suppose you can
play around…
