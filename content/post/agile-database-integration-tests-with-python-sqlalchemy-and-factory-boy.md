---
title: 'Agile database integration tests with Python, SQLAlchemy and Factory Boy'
date: '2019-04-22T18:52:14+02:00'
draft: false
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


Do it, and you’ll most likely face walls of SQL code, hardly reusable, hard to
read and even harder to maintain. A very tedious and error prone method. <br>
Moreover you can encounter performance issues, especially if more tables are
involved or if there are cascading deletes.

<figure class="image">
  <img src="/images/addams.gif" alt="shocked and shattered expression">
  <i><figcaption>You, after reading 300 lines of SQL statements</figcaption></i>
</figure>

The best way to isolate your test case is an already present feature of SQL:
**transactions**. Yes, they can help you.

Let’s suppose you are using the excellent
[pytest](https://docs.pytest.org/en/latest/) framework for your tests, a couple
of well designed fixtures will do the work:

```python3
import pytest
from sqlalchemy.orm import sessionmaker
from sqlalchemy import create_engine

engine = create_engine('DB_CONNECTION_URL')
Session = sessionmaker()

@pytest.fixture(scope='module')
def connection():
    connection = engine.connect()
    yield connection
    connection.close()

@pytest.fixture(scope='function')
def session(connection):
    transaction = connection.begin()
    session = Session(bind=connection)
    yield session
    session.close()
    transaction.rollback()
```

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


```python3
import factory

class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email

class UserFactory(factory.Factory):
    name = factory.Faker('name')
    email = factory.Faker('email')

    class Meta:
        model = User
```

Now, just calling `UserFactory.build()` will generate a *filled* `User`
instance.<br> There are lots of features, such as properties overriding,
sequences and so on. Take a look at the
[docs](http://factoryboy.readthedocs.io/en/latest/)!

> *… Umm, nice! But what has this got to do with database? …*

Well, what if I say you can actually bind factories with SQLAlchemy models?

```python3
class UserModel(Base):
    __tablename__ = 'account'

    id = Column(BigInteger, primary_key=True)
    name = Column(String, nullable=False)
    email = Column(String, nullable=False)

class UserFactory(factory.alchemy.SQLAlchemyModelFactory):
    id = factory.Sequence(lambda n: '%s' % n)
    name = factory.Faker('name')
    email = factory.Faker('email')

    class Meta:
        model = UserModel
```

That’s all! Now creating and **having a user on your database** is simple as
writing` UserFactory.create()`.

*Mind. Blown.*

Want more objects? `UserFactory.create_batch(6)`

*Mind. Blown. ***Again***.*

If you think you didn’t get it, here is a full integration test in its splendor:

```python3
@pytest.fixture(scope='function')
def session(connection):
    transaction = connection.begin()
    session = Session(bind=connection)
    UserFactory._meta.sqlalchemy_session = session # NB: This line added
    yield session
    session.close()
    transaction.rollback()

def my_func_to_delete_user(session, user_id):
    session.query(UserModel).filter(UserModel.id == user_id).delete()

def test_case(session):
    user = UserFactory.create()
    assert session.query(UserModel).one()

    my_func_to_delete_user(session, user.id)

    result = session.query(UserModel).one_or_none()
    assert result is None
```

At line 5, you can see how SQLAlchemy session is bound to the factory class.
Every action performed by the factory will use our session, whose lifecycle is
handled by fixtures themselves.

Easy, few lines long, clean, understandable and … obviously it works! 😃<br>
Such a smooth integration testing!

![clean sensation](/images/clean_sensation.png)

Of course this example is intentionally trivial, you can find slightly more real
and organized examples in this
[repository](https://github.com/ProvoK/article_agile_database_integration) I
created on purpose.

#### Recap

![testing flow](/images/python_integration_testing_flow.png)

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
