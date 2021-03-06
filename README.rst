
Graphene-Django-Subscriptions
=============================

This package adds support to Subscription's requests and its integration with websockets using Channels package. You can test websockets notifications with this mini web tool. It's intuitive and simple: `websocket_example_client <https://github.com/eamigo86/graphene-django-subscriptions/tree/master/example_websocket_client>`_


Installation:
-------------

For installing graphene-django-subscriptions, just run this command in your shell:

.. code:: bash

    pip install "graphene-django-subscriptions"

Documentation:
--------------

***************************************
Extra functionalities  (Subscriptions):
***************************************
    1.  Subscription  (Abstract class to define subscriptions to a DjangoSerializerMutation)
    2.  GraphqlAPIDemultiplexer  (Custom WebSocket consumer subclass that handles demultiplexing streams)


Subscriptions:
--------------

This first approach to add Graphql subscriptions support  with Channels in graphene-django, use channels-api package.

*****************************************
1- Defining custom Subscriptions classes:
*****************************************

You must to have defined a DjangoSerializerMutation class for each model that you want to define a Subscription class:

.. code:: python

    # app/graphql/subscriptions.py
    import graphene
    from graphene_django_extras.subscription import Subscription
    from .mutations import UserMutation, GroupMutation


    class UserSubscription(Subscription):
        class Meta:
            mutation_class = UserMutation
            stream = 'users'
            description = 'User Subscription'


    class GroupSubscription(Subscription):
        class Meta:
            mutation_class = GroupMutation
            stream = 'groups'
            description = 'Group Subscription'


Add the subscriptions definitions into your app's schema:

.. code:: python

    # app/graphql/schema.py
    import graphene
    from .subscriptions import UserSubscription, GroupSubscription


    class Subscriptions(graphene.ObjectType):
        user_subscription = UserSubscription.Field()
        GroupSubscription = PersonSubscription.Field()


Add the app's schema into your project root schema:

.. code:: python

    # schema.py
    import graphene
    import custom.app.route.graphql.schema


    class RootQuery(custom.app.route.graphql.schema.Query, graphene.ObjectType):
        class Meta:
            description = 'The project root query definition'


    class RootMutation(custom.app.route.graphql.schema.Mutation, graphene.ObjectType):
        class Meta:
            description = 'The project root mutation definition'


    class RootSubscription(custom.app.route.graphql.schema.Subscriptions, graphene.ObjectType):
        class Meta:
            description = 'The project root subscription definition'


    schema = graphene.Schema(
        query=RootQuery,
        mutation=RootMutation,
        subscription=RootSubscription
    )


********************************************************
2- Defining Channels settings and custom routing config:
********************************************************
**Note**: For more information about this step see Channels documentation.

You must to have defined a DjangoSerializerMutation class for each model that you want to define a Subscription class:

We define app routing, as if they were app urls:

.. code:: python

    # app/routing.py
    from graphene_django_subscriptions.subscriptions import GraphqlAPIDemultiplexer
    from channels.routing import route_class
    from .graphql.subscriptions import UserSubscription, GroupSubscription


    class CustomAppDemultiplexer(GraphqlAPIDemultiplexer):
        consumers = {
          'users': UserSubscription.get_binding().consumer,
          'groups': GroupSubscription.get_binding().consumer
        }


    app_routing = [
        route_class(CustomAppDemultiplexer)
    ]


Defining our project routing, like custom root project urls:

.. code:: python

    # project/routing.py
    from channels import include

    project_routing = [
        include("custom.app.folder.routing.app_routing", path=r"^/custom_websocket_path"),
    ]


You should put into your INSTALLED_APPS the channels and channels_api modules and you must to add your project's routing definition into the CHANNEL_LAYERS setting:

.. code:: python

    # settings.py
    ...
    INSTALLED_APPS = (
        'django.contrib.auth',
        'django.contrib.contenttypes',
        'django.contrib.sessions',
        'django.contrib.sites',
        ...
        'channels',
        'channels_api',

        'custom_app'
    )

    CHANNEL_LAYERS = {
        "default": {
            "BACKEND": "asgiref.inmemory.ChannelLayer",
            "ROUTING": "myproject.routing.project_routing",  # Our project routing
        },
    }
    ...


***************************
3- Subscription's examples:
***************************

In your WEB client you must define websocket connection to: 'ws://host:port/custom_websocket_path'.
When the connection is established, the server return a websocket's message like this:
{"channel_id": "GthKdsYVrK!WxRCdJQMPi", "connect": "success"}, where you must store the channel_id value to later use in your graphql subscriptions request for subscribe or unsubscribe operations.

The graphql's subscription request accept five possible parameters:
1.  **operation**: Operation to perform: subscribe or unsubscribe. (required)
2.  **action**: Action to which you wish to subscribe: create, update, delete or all_actions. (required)
3.  **channelId**: Identification of the connection by websocket. (required)
4.  **id**: Object's ID field value that you wish to subscribe to. (optional)
5.  **data**: Model's fields that you want to appear in the subscription notifications. (optional)

.. code:: python

    subscription{
        userSubscription(
            action: UPDATE,
            operation: SUBSCRIBE,
            channelId: "GthKdsYVrK!WxRCdJQMPi",
            id: 5,
            data: [ID, USERNAME, FIRST_NAME, LAST_NAME, EMAIL, IS_SUPERUSER]
        ){
            ok
            error
            stream
        }
    }


In this case, the subscription request sent return a websocket message to client like this: *{"action": "update", "operation": "subscribe", "ok": true, "stream": "users", "error": null}* and from that moment each time than the user with id=5 get modified, you will receive a message through websocket's connection with the following format:

.. code:: python

    {
        "stream": "users",
        "payload": {
            "action": "update",
            "model": "auth.user",
            "data": {
                "id": 5,
                "username": "meaghan90",
                "first_name": "Meaghan",
                "last_name": "Ackerman",
                "email": "meaghan@gmail.com",
                "is_superuser": false
            }
        }
    }


For unsubscribe you must send a graphql request like this:

.. code:: python

    subscription{
        userSubscription(
            action: UPDATE,
            operation: UNSUBSCRIBE,
            channelId: "GthKdsYVrK!WxRCdJQMPi",
            id: 5
        ){
            ok
            error
            stream
        }
    }


*NOTE*: Each time than the graphql's server restart, you must to reestablish the websocket connection and resend the graphql's subscription request with the new websocket connection id.


Change Log:
-----------

*******
v0.0.1:
*******
1. First commit