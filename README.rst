*********************************
pyramid-jsonapi Documentation
*********************************

Create a JSON-API (`<http://jsonapi.org/>`_) standard API from a database using
the sqlAlchemy ORM and pyramid framework.

The core idea behind pyramid-jsonapi is to create a working JSON-API
automatically, starting from the sort of ``models.py`` file shipped with a
typical pyramid + sqlalchemy application.

If you are reading this document directly from the repository on github
(`<https://github.com/colinhiggs/pyramid-jsonapi>`_) you may notice some
oddities. In particular, internal links to things like ``:py:func:`blah``` which
don't work. That's because those directives are designed to be consumed by
sphinx (`<http://www.sphinx-doc.org/>`_). You can view the same document after
it has been run through sphinx, as well as the API documentation, at the
pyramid_jsonapi documentation page
(`<https://colinhiggs.github.io/pyramid-jsonapi/>`_).

Installation
============

* Stable releases are now uploaded to pypi:
  `<https://pypi.python.org/pypi?:action=display&name=pyramid_jsonapi>`_. You
  can install it in the usual way:

  .. code-block:: bash

    pip install -i pyramid_jsonapi

* Test releases are sometimes uploaded to testpypi:
  `<https://testpypi.python.org/pypi?:action=display&name=pyramid_jsonapi>`_.
  These may or may not be ahead of the current stable version. You
  can install it (perhaps into a virtualenv for play purposes) with

  .. code-block:: bash

    pip install -i https://testpypi.python.org/pypi pyramid_jsonapi

* Since there is only one file, you can download the development version from
  `<https://github.com/colinhiggs/pyramid-jsonapi>`_ and copy the
  pyramid_jsonapi directory into your PYTHONPATH or into your project.

Auto-Creating an API
====================

Declare your models somewhere using sqlalchemy's
:py:func:`sqlalchemy.ext.declarative.declarative_base`. In this documentation we
assume that you have done so in a file called ``models.py``:

.. code-block:: python

  class Person(Base):
      __tablename__ = 'people'
      id = Column(BigInteger, primary_key=True, autoincrement=True)
      name = Column(Text)
      blogs = relationship('Blog', backref='owner')
      posts = relationship('Post', backref='author')

  # and the rest...

If you are happy with the defaults, you can get away with the following
additions to the standard pyramid alchemy scaffold's top level ``__init__.py``:

.. code-block:: python

  import pyramid_jsonapi
  # Or 'from . import pyramid_jsonapi' if you copied pyramid_jsonapi directly
  # into your project.

  from . import models # Your models module.

  def main(global_config, **settings):

    # The usual stuff from the pyramid alchemy scaffold.
    engine = engine_from_config(settings, 'sqlalchemy.')
    models.DBSession.configure(bind=engine)
    models.Base.metadata.bind = engine
    config = Configurator(settings=settings)

    # pyramid_jsonapi uses the renderer labeled 'json'. As usual, if you have
    # any types to serialise that the default JSON renderer can't handle, you
    # must alter it. For example:
    renderer = JSON()
    renderer.add_adapter(datetime.date, datetime_adapter)
    config.add_renderer('json', renderer)

    # Create the routes and views automagically:
    pyramid_jsonapi.create_jsonapi_using_magic_and_pixie_dust(
      config, models, lambda view: models.DBSession
    )
    # The third argument above should be a callable which accepts a
    # CollectionView instance as an argument and returns a database session.
    # Notably the request is available as view.request, so if you're doing
    # something like this post
    # https://metaclassical.com/what-the-zope-transaction-manager-means-to-me-and-you
    # you can return the per-request session. In this case we just return the
    # usual DBSession from the models module.

    # Routes and views are added imperatively, so no need for a scan - unless
    # you have defined other routes and views declaratively.

Yes, there really is a method called
:py:func:`pyramid_jsonapi.create_jsonapi_using_magic_and_pixie_dust`. No, you
don't *have* to call it that. If you are feeling more sensible you can use the
synonym :py:func:`pyramid_jsonapi.create_jsonapi`.

Calling :py:func:`pyramid_jsonapi.create_jsonapi`
-------------------------------------------------

Since :py:func:`pyramid_jsonapi.create_jsonapi` (or the one with pixie dust)
sits at the centre of the API creation, we'll spend a little time now explaining
the three mandatory arguments.

* ``config`` is the usual Configurator object used in pyramid.

* ``models`` can either be a module (as in the example above) defining classes
  which inherit from :py:func:`declarative_base` or an iterable of such classes.

* ``get_dbsession`` (to which we passed the lambda function above) should be a
  callable which accepts an instance of
  :py:class:`pyramid_jsonapi.CollectionViewBase` and returns a
  :py:class:`sqlalchemy.orm.session.Session` (or an equivalent, like a
  :py:func:`sqlalchemy.orm.scoped_session`)

Auto-Create Assumptions
-----------------------

#. Your model classes all inherit from a base class returned by sqlalchemy's
   ``declarative-base()``.

#. Each model has a single primary_key column. This will be auto-detected and
   copied to an attribute called ``_jsonapi_id``, so...

#. ...don't create any columns called ``_jsonapi_id``.

#. You are happy to give your collection end-points the same name as the
   corresponding database table (for now).

#. You have defined any relationships to exposed via the API using
   ``sqlalchemy.orm.relationship()`` (or ``backref()``).

#. You are happy to expose any so defined relationship via a relationship URL.

Some of those behaviours can be adjusted, see `Customising the Generated API`_.

Trying Your API Out
-------------------

You should now have a working JSON-API. A quick test. The following assumes that
you have already created and set up a pyramid project in development mode
(``python setup.py develop`` in pyramid 1.6, ``pip install -e`` in pyramid 1.7).

Make sure you have activated your virtualenv:

.. code-block:: bash

  $ source env/bin/activate

Start the server:

.. code-block:: bash

  $ pserv your_project/development.ini

Using the rather lovely httpie `<https://github.com/jkbrzt/httpie/>`_ to test:

.. code-block:: bash

  $ http http://localhost:6543/people

  HTTP/1.1 200 OK
  Content-Length: 1387
  Content-Type: application/vnd.api+json; charset=UTF-8
  Date: Fri, 28 Aug 2015 20:22:46 GMT
  Server: waitress

.. code-block:: json

  {
    "data": [
      {
        "attributes": {
          "name": "alice"
        },
        "id": "1",
        "links": {
          "self": "http://localhost:6543/people/1"
        },
        "relationships": {
          "<some_single_relationship>": {
            "data": {"type": "<rel_type>", "id": "<some_id>"}
          }
        }
      },
      {"<another_person>"}
    ]
  }


See ``test_project/test_project/__init__.py`` for a fully working
``__init__.py`` file.

You don't need a ``views.py`` unless you have some other routes and views.

Customising the Generated API
=============================

Selectively Passing Models for API Generation
---------------------------------------------

Your database may have some tables which you do not wish to expose as collections in the generated API. You can be selective by:

* writing a models module with only the model classes you wish to expose; or
* passing an iterable of only the model classes you wish to expose to
  :py:func:`pyramid_jsonapi.create_jsonapi`.

Callbacks
---------

At certain points during the processing of a request, ``pyramid_jsonapi`` will
invoke any callback functions which have been registered. Callback sequences are
currently implemented as ``collections.deque``: you add your callback functions
using ``.append()`` or ``.appendleft()``, remove them with ``.pop()`` or
``.popleft()`` and so on. The functions in each callback list will be called in
order at the appropriate point.

Getting the Callback Deque
--------------------------

Every view class (subclass of CollectionViewBase) has its own dictionary of
callback deques (``view_class.callbacks``). That dictionary is keyed by callback
deque name. For example, if you have a view_class and you would like to append
your ``my_after_get`` function to the ``after_get`` deque:

.. code-block:: python

  view_class.callbacks['after_get'].append(my_after_get)

If you don't currently have a view class, you can get one from a model class
(for example, ``models.Person``) with:

.. code-block:: python

  person_view_class = pyramid_jsonapi.view_classes[models.Person]

Available Callback Deques
-------------------------

The following is a list of available callbacks. Note that each item in the list
has a name like ``pyramid_jsonapi.callbacks_doc.<callback_name>``. That's so
that sphinx will link to auto-built documentation from the module
``pyramid_jsonapi.callbacks_doc``. In practice you should use only the name
after the last '.' to get callback deques.

* :py:func:`pyramid_jsonapi.callbacks_doc.after_serialise_object`

* :py:func:`pyramid_jsonapi.callbacks_doc.after_serialise_identifier`

* :py:func:`pyramid_jsonapi.callbacks_doc.after_get`

* :py:func:`pyramid_jsonapi.callbacks_doc.before_patch`

* :py:func:`pyramid_jsonapi.callbacks_doc.before_delete`

* :py:func:`pyramid_jsonapi.callbacks_doc.after_collection_get`

* :py:func:`pyramid_jsonapi.callbacks_doc.before_collection_post`

* :py:func:`pyramid_jsonapi.callbacks_doc.after_related_get`

* :py:func:`pyramid_jsonapi.callbacks_doc.after_relationships_get`

* :py:func:`pyramid_jsonapi.callbacks_doc.before_relationships_post`

* :py:func:`pyramid_jsonapi.callbacks_doc.before_relationships_patch`

* :py:func:`pyramid_jsonapi.callbacks_doc.before_relationships_delete`


Canned Callbacks
----------------

Using the callbacks above, you could, in theory, do things like implement a
permissions system, generalised call-outs to other data sources, or many other
things. However, some of those would entail quite a lot of work as well as being
potentially generally useful. In the interests of reuse, pyramid_jsonapi
maintains sets of self consistent callbacks which cooperate towards one goal.

So far there is only one such set: ``access_control_serialised_objects``. This
set of callbacks implements an access control system based on the inspection of
serialised (as dictionaries) objects before POST, PATCH and DELETE operations
and after serialisation and GET operations.

Registering Canned Callbacks
----------------------------

Given a callback set name, you can register callback sets on each view class:

.. code-block:: python

  view_class.append_callback_set('access_control_serialised_objects')

or on all view classes:

.. code-block:: python

  pyramid_jsonapi.append_callback_set_to_all_views(
    'access_control_serialised_objects'
  )

Callback Sets
-------------

``access_control_serialised_objects``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

These callbacks will allow, deny, or manipulate the results of actions
dependent upon the return values of two methods of the calling view class:
:py:func:`pyramid_jsonapi.CollectionViewBase.allowed_object` and
:py:func:`pyramid_jsonapi.CollectionViewBase.allowed_fields`.

The default implementations allow everything. To do anything else, you need to
replace those methods with your own implementations.

* :py:func:`pyramid_jsonapi.CollectionViewBase.allowed_object` will be given two
  arguments: an instance of a view class, and the serialised object (so far). It
  should return ``True`` if the operation (available from view.request) is
  allowed on the object, or ``False`` if not.

* :py:func:`pyramid_jsonapi.CollectionViewBase.allowed_fields` will be given one
  argument: an instance of a view class. It should return the set of fields
  (attributes and relationships) on which the current operation is allowed.

Consuming the API from the Client End
=====================================

GET-ing Resources
--------------------

A Collection
~~~~~~~~~~~~

.. code-block:: bash

  $ http GET http://localhost:6543/posts


.. code-block:: json

  {
    "data": [
      {
        "type": "posts",
        "id": "1",
        "attributes": {
          "content": "something insightful",
          "published_at": "2015-01-01T00:00:00",
          "title": "post1: alice.main"
        },
        "links": {
          "self": "http://localhost:6543/posts/1"
        },
        "relationships": {
          "author": {
            "data": {
              "id": "1",
              "type": "people"
            },
            "links": {
              "related": "http://localhost:6543/posts/1/author",
              "self": "http://localhost:6543/posts/1/relationships/author"
            },
            "meta": {
              "direction": "MANYTOONE",
              "results": {}
            }
          },
          "blog": {
            "data": {
              "id": "1",
              "type": "blogs"
            },
            "links": {
              "related": "http://localhost:6543/posts/1/blog",
              "self": "http://localhost:6543/posts/1/relationships/blog"
            },
            "meta": {
              "direction": "MANYTOONE",
              "results": {}
            }
          },
          "comments": {
            "data": [],
            "links": {
              "related": "http://localhost:6543/posts/1/comments",
              "self": "http://localhost:6543/posts/1/relationships/comments"
            },
            "meta": {
              "direction": "ONETOMANY",
              "results": {
                "available": 0,
                "limit": 10,
                "returned": 0
              }
            }
          }
        }
      },
      "... 5 more results ..."
    ],
    "links": {
      "first": "http://localhost:6543/posts?sort=id&page%5Boffset%5D=0",
      "last": "http://localhost:6543/posts?sort=id&page%5Boffset%5D=0",
      "self": "http://localhost:6543/posts"
    },
    "meta": {
      "results": {
        "available": 6,
        "limit": 10,
        "offset": 0,
        "returned": 6
      }
    }
  }


Note that we have:

* ``data`` which is an array of comments objects, each with:

  * ``attributes``, as expected

    * a ``links`` object with:

    * a ``self`` link

  * relationship objects for each relationship with:

    * ``data`` with resource identifiers for related objects

    * ``self`` and ``related`` links

    * some other information about the relationship in ``meta``

* ``links`` with:

  * ``self`` and

  * ``pagination`` links

* ``meta`` with:

  * some extra information about the number of results returned.

A Single Resource
~~~~~~~~~~~~~~~~~

.. code-block:: bash

  $ http GET http://localhost:6543/posts/1

Returns a single resource object in ``data`` and no pagination links.

.. code-block:: json

  {
    "data": {
      "type": "posts",
      "id": "1",
      "attributes": {
        "content": "something insightful",
        "published_at": "2015-01-01T00:00:00",
        "title": "post1: alice.main"
      },
      "links": {
        "self": "http://localhost:6543/posts/1"
      },
      "relationships": {
        "author": {
          "data": {
            "id": "1",
            "type": "people"
          },
          "links": {
            "related": "http://localhost:6543/posts/1/author",
            "self": "http://localhost:6543/posts/1/relationships/author"
          },
          "meta": {
            "direction": "MANYTOONE",
            "results": {}
          }
        },
        "blog": {
          "data": {
            "id": "1",
            "type": "blogs"
          },
          "links": {
            "related": "http://localhost:6543/posts/1/blog",
            "self": "http://localhost:6543/posts/1/relationships/blog"
          },
          "meta": {
            "direction": "MANYTOONE",
            "results": {}
          }
        },
        "comments": {
          "data": [],
          "links": {
            "related": "http://localhost:6543/posts/1/comments",
            "self": "http://localhost:6543/posts/1/relationships/comments"
          },
          "meta": {
            "direction": "ONETOMANY",
            "results": {
              "available": 0,
              "limit": 10,
              "returned": 0
            }
          }
        }
      }
    },
    "links": {
      "self": "http://localhost:6543/posts/1"
    },
    "meta": {}
  }

Sparse Fieldsets
~~~~~~~~~~~~~~~~

We can ask only for certain fields (attributes and relationships are
collectively known as fields).

Use the ``fields`` parameter, parameterized by collection name
(fields[collection]), with the value set as a comma separated list of field
names.

So, to return only the title attribute and author relationship of each post:

.. code-block:: bash

  $ http GET http://localhost:6543/posts?fields[posts]=title,author

The resulting json has a ``data`` element with a list of objects something like
this:

.. code-block:: json

  {
    "attributes": {
      "title": "post1: bob.second"
    },
    "id": "6",
    "links": {
      "self": "http://localhost:6543/posts/6"
    },
    "relationships": {
      "author": {
        "data": {
          "id": "2",
          "type": "people"
        },
        "links": {
          "related": "http://localhost:6543/posts/6/author",
          "self": "http://localhost:6543/posts/6/relationships/author"
        },
        "meta": {
          "direction": "MANYTOONE",
          "results": {}
        }
      }
    },
    "type": "posts"
  }

Sorting
~~~~~~~

You can specify a sorting attribute and order with the sort query parameter.

Sort posts by title:

.. code-block:: bash

  $ http GET http://localhost:6543/posts?sort=title

and in reverse:

.. code-block:: bash

  $ http GET http://localhost:6543/posts?sort=-title

Sorting by multiple attributes (e.g. ``sort=title,content``) and sorting by attributes of related objects (`sort=author.name`) are supported.

A sort on id is assumed unless the sort parameter is specified.

Pagination
~~~~~~~~~~

You can specify the pagination limit and offset:

.. code-block:: bash

  $ http GET http://localhost:6543/posts?fields[posts]=title\&page[limit]=2\&page[offset]=2

We asked for only the ``title`` field above so that the results would be more
compact...

.. code-block:: json

  {
    "data": [
      {
        "attributes": {
          "title": "post1: alice.second"
        },
        "id": "3",
        "links": {
          "self": "http://localhost:6543/posts/3"
        },
        "relationships": {},
        "type": "posts"
      },
      {
        "attributes": {
          "title": "post1: bob.main"
        },
        "id": "4",
        "links": {
          "self": "http://localhost:6543/posts/4"
        },
        "relationships": {},
        "type": "posts"
      }
    ],
    "links": {
      "first": "http://localhost:6543/posts?page%5Blimit%5D=2&sort=id&page%5Boffset%5D=0",
      "last": "http://localhost:6543/posts?page%5Blimit%5D=2&sort=id&page%5Boffset%5D=4",
      "next": "http://localhost:6543/posts?page%5Blimit%5D=2&sort=id&page%5Boffset%5D=4",
      "prev": "http://localhost:6543/posts?page%5Blimit%5D=2&sort=id&page%5Boffset%5D=0",
      "self": "http://localhost:6543/posts?fields[posts]=title&page[limit]=2&page[offset]=2"
    },
    "meta": {
      "results": {
        "available": 6,
        "limit": 2,
        "offset": 2,
        "returned": 2
      }
    }
  }

There's a default page limit which is used if the limit is not specified and a
maximum limit that the server will allow. Both of these can be set in the ini
file.

Filtering
~~~~~~~~~

The JSON API spec doesn't say much about filtering syntax, other than that it
should use the parameter key ``filter``. In this implementation, we use syntax
like the following:

.. code::

  filter[<attribute_spec>:<operator>]=<value>

where:

* ``attribute_spec`` is either a direct attribute name or a dotted path to an
  attribute via relationhips.

* ``operator`` is one of the list of supported operators (`Filter Operators`_).

* ``value`` is the value to match on.

This is simple and reasonably effective. It's a little awkward on readability though. If you feel that you have a syntax that is more readable, more powerful, easier to parse or has some other advantage, let me know - I'd be interested in any thoughts.

Filter Operators
^^^^^^^^^^^^^^^^

* ``eq``
* ``ne``
* ``startswith``
* ``endswith``
* ``contains``
* ``lt``
* ``gt``
* ``le``
* ``ge``
* ``like`` or ``ilike``. Note that both of these use '*' in place of '%' to
  avoid much URL escaping.

Filter Examples
^^^^^^^^^^^^^^^

Find all the people with name 'alice':

.. code-block:: bash

  http GET http://localhost:6543/people?filter[name:eq]=alice

Find all the posts published after 2015-01-03:

.. code-block:: bash

  http GET http://localhost:6543/posts?filter[published_at:gt]=2015-01-03

Find all the posts with 'bob' somewhere in the title:

.. code-block:: bash

  http GET http://localhost:6543/posts?filter[title:like]=*bob*
