Queries & ObjectTypes
=====================

Introduction
------------

Graphene-Django offers a host of features for performing GraphQL queries.

Graphene-Django ships with a special ``DjangoObjectType`` that automatically transforms a Django Model
into a ``ObjectType`` for you.


Full example
~~~~~~~~~~~~

.. code:: python

    # my_app/schema.py

    import graphene

    from graphene_django.types import DjangoObjectType
    from .models import Question


    class QuestionType(DjangoObjectType):
        class Meta:
            model = Question


    class Query:
        questions = graphene.List(QuestionType)
        question = graphene.Field(QuestionType, question_id=graphene.String())

        def resolve_questions(self, info, **kwargs):
            # Querying a list
            return Question.objects.all()

        def resolve_question(self, info, question_id):
            # Querying a single question
            return Question.objects.get(pk=question_id)


Fields
------

By default, ``DjangoObjectType`` will present all fields on a Model through GraphQL.
If you don't want to do this you can change this by setting either ``only_fields`` and ``exclude_fields``.

only_fields
~~~~~~~~~~~

Show **only** these fields on the model:

.. code:: python

    class QuestionType(DjangoObjectType):
        class Meta:
            model = Question
            only_fields = ('question_text')


exclude_fields
~~~~~~~~~~~~~~

Show all fields **except** those in ``exclude_fields``:

.. code:: python

    class QuestionType(DjangoObjectType):
        class Meta:
            model = Question
            exclude_fields = ('question_text')


Customised fields
~~~~~~~~~~~~~~~~~

You can completely overwrite a field, or add new fields, to a ``DjangoObjectType`` using a Resolver:

.. code:: python

    class QuestionType(DjangoObjectType):

        class Meta:
            model = Question
            exclude_fields = ('question_text')

        extra_field = graphene.String()

        def resolve_extra_field(self, info):
            return 'hello!'


Choices to Enum conversion
~~~~~~~~~~~~~~~~~~~~~~~~~~

By default Graphene-Django will convert any Django fields that have `choices`_
defined into a GraphQL enum type.

.. _choices: https://docs.djangoproject.com/en/2.2/ref/models/fields/#choices

For example the following ``Model`` and ``DjangoObjectType``:

.. code:: python

   class PetModel(models.Model):
      kind = models.CharField(max_length=100, choices=(('cat', 'Cat'), ('dog', 'Dog')))

   class Pet(DjangoObjectType):
      class Meta:
         model = PetModel

Results in the following GraphQL schema definition:

.. code::

   type Pet {
     id: ID!
     kind: PetModelKind!
   }

   enum PetModelKind {
     CAT
     DOG
   }

You can disable this automatic conversion by setting
``convert_choices_to_enum`` attribute to ``False`` on the ``DjangoObjectType``
``Meta`` class.

.. code:: python

   class Pet(DjangoObjectType):
      class Meta:
         model = PetModel
         convert_choices_to_enum = False

.. code::

   type Pet {
     id: ID!
     kind: String!
   }

You can also set ``convert_choices_to_enum`` to a list of fields that should be
automatically converted into enums:

.. code:: python

   class Pet(DjangoObjectType):
      class Meta:
         model = PetModel
         convert_choices_to_enum = ['kind']

**Note:** Setting ``convert_choices_to_enum = []`` is the same as setting it to
``False``.


Related models
--------------

Say you have the following models:

.. code:: python

    class Category(models.Model):
        foo = models.CharField(max_length=256)

    class Question(models.Model):
        category = models.ForeignKey(Category, on_delete=models.CASCADE)


When ``Question`` is published as a ``DjangoObjectType`` and you want to add ``Category`` as a query-able field like so:

.. code:: python

    class QuestionType(DjangoObjectType):
        class Meta:
            model = Question
            only_fields = ('category',)

Then all query-able related models must be defined as DjangoObjectType subclass,
or they will fail to show if you are trying to query those relation fields. You only
need to create the most basic class for this to work:

.. code:: python

    class CategoryType(DjangoObjectType):
        class Meta:
            model = Category

Default QuerySet
-----------------

If you are using ``DjangoObjectType`` you can define a custom `get_queryset` method.
Use this to control filtering on the ObjectType level instead of the Query object level.

.. code:: python

    from graphene_django.types import DjangoObjectType
    from .models import Question


    class QuestionType(DjangoObjectType):
        class Meta:
            model = Question

        @classmethod
        def get_queryset(cls, queryset, info):
            if info.context.user.is_anonymous:
                return queryset.filter(published=True)
            return queryset

Resolvers
---------

When a GraphQL query is received by the ``Schema`` object, it will map it to a "Resolver" related to it.

This resolve method should follow this format:

.. code:: python

    def resolve_foo(self, info, **kwargs):

Where "foo" is the name of the field declared in the ``Query`` object.

.. code:: python

    class Query:
        foo = graphene.List(QuestionType)

        def resolve_foo(self, info, **kwargs):
            id = kwargs.get('id')
            return QuestionModel.objects.get(id)

Arguments
~~~~~~~~~

Additionally, Resolvers will receive **any arguments declared in the field definition**. This allows you to provide input arguments in your GraphQL server and can be useful for custom queries.

.. code:: python

    class Query:
        question = graphene.Field(Question, foo=graphene.String(), bar=graphene.Int())

        def resolve_question(self, info, foo, bar):
            # If `foo` or `bar` are declared in the GraphQL query they will be here, else None.
            return Question.objects.filter(foo=foo, bar=bar).first()


Info
~~~~

The ``info`` argument passed to all resolve methods holds some useful information.
For Graphene-Django, the ``info.context`` attribute is the ``HTTPRequest`` object
that would be familiar to any Django developer. This gives you the full functionality
of Django's ``HTTPRequest`` in your resolve methods, such as checking for authenticated users:

.. code:: python

    def resolve_questions(self, info, **kwargs):
        # See if a user is authenticated
        if info.context.user.is_authenticated():
            return Question.objects.all()
        else:
            return Question.objects.none()


Plain ObjectTypes
-----------------

With Graphene-Django you are not limited to just Django Models - you can use the standard
``ObjectType`` to create custom fields or to provide an abstraction between your internal
Django models and your external API.

.. code:: python

    import graphene
    from .models import Question


    class MyQuestion(graphene.ObjectType):
        text = graphene.String()


    class Query:
        question = graphene.Field(MyQuestion, question_id=graphene.String())

        def resolve_question(self, info, question_id):
            question = Question.objects.get(pk=question_id)
            return MyQuestion(
                text=question.question_text
            )

For more information and more examples, please see the `core object type documentation <https://docs.graphene-python.org/en/latest/types/objecttypes/>`__.


Relay
-----

`Relay <http://docs.graphene-python.org/en/latest/relay/>`__ with Graphene-Django gives us some additional features:

- Pagination and slicing.
- An abstract ``id`` value which contains enough info for the server to know its type and its id.

There is one additional import and a single line of code needed to adopt this:

Full example
~~~~~~~~~~~~
See the `Relay documentation <https://docs.graphene-python.org/en/latest/relay/nodes/>`__ on
the core graphene pages for more information on customizing the Relay experience.

.. code:: python

    from graphene import relay
    from graphene_django import DjangoObjectType
    from .models import Question


    class QuestionType(DjangoObjectType):
        class Meta:
            model = Question
            interfaces = (relay.Node,)


    class QuestionConnection(relay.Connection):
        class Meta:
            node = QuestionType


    class Query:
        questions = relay.ConnectionField(QuestionConnection)

        def resolve_questions(root, info, **kwargs):
            return Question.objects.all()


You can now execute queries like:


.. code:: python

    {
        questions (first: 2, after: "YXJyYXljb25uZWN0aW9uOjEwNQ==") {
            pageInfo {
            startCursor
            endCursor
            hasNextPage
            hasPreviousPage
            }
            edges {
            cursor
            node {
                id
                question_text
            }
            }
        }
    }

Which returns:

.. code:: python

    {
        "data": {
            "questions": {
            "pageInfo": {
                "startCursor": "YXJyYXljb25uZWN0aW9uOjEwNg==",
                "endCursor": "YXJyYXljb25uZWN0aW9uOjEwNw==",
                "hasNextPage": true,
                "hasPreviousPage": false
            },
            "edges": [
                {
                "cursor": "YXJyYXljb25uZWN0aW9uOjEwNg==",
                "node": {
                    "id": "UGxhY2VUeXBlOjEwNw==",
                    "question_text": "How did we get here?"
                }
                },
                {
                "cursor": "YXJyYXljb25uZWN0aW9uOjEwNw==",
                "node": {
                    "id": "UGxhY2VUeXBlOjEwOA==",
                    "name": "Where are we?"
                }
                }
            ]
            }
        }
    }

Note that relay implements :code:`pagination` capabilities automatically, adding a :code:`pageInfo` element, and including :code:`cursor` on nodes. These elements are included in the above example for illustration.

To learn more about Pagination in general, take a look at `Pagination <https://graphql.org/learn/pagination/>`__  on the GraphQL community site.
