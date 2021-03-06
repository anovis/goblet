========
Topics
========


Routing
^^^^^^^^^^^^^

The Goblet.route() method is used to construct which routes you want to create for your API. 
The concept is the same mechanism used by Flask. You decorate a function with @app.route(...), 
and whenever a user requests that URL, the function you’ve decorated is called. For example, 
suppose you deployed this app:

.. code:: python 

    from goblet import Goblet

    app = Goblet(function_name='helloworld')

    @app.route('/')
    def index():
        return {'view': 'index'}

    @app.route('/a')
    def a():
        return {'view': 'a'}

    @app.route('/b')
    def b():
        return {'view': 'b'}

If you go to https://endpoint/, the index() function would be called. If you went to https://endpoint/a and https://endpoint/b, then the a() and b() function would be called, respectively.

You can also create a route that captures part of the URL. This captured value will then be passed in as arguments to your view function:

.. code:: python 

    @app.route('/users/{name}')
    def users(name):
        return {'name': name}

If you then go to https://endpoint/users/james, then the view function will be called as: users('james'). 
The parameters are passed as keyword parameters based on the name as they appear in the URL. 
The argument names for the view function must match the name of the captured argument:

.. code:: python 

    @app.route('/a/{first}/b/{second}')
    def users(first, second):
        return {'first': first, 'second': second}


Scheduled Jobs
^^^^^^^^^^^^^

To deploy scheduled jobs using a cron schedule use the Goblet.schedule decorator. The cron schedule follows the unix-cron format. 
More information on the cron format can be found `here`_. Make sure `Cloud Scheduler`_ is enabled in your account if you want to deploy
scheduled jobs.

Example usage:

.. code:: python 

    @app.schedule('5 * * * *')
    def scheduled_job():
        return app.jsonify("success")


.. _HERE: https://cloud.google.com/scheduler/docs/configuring/cron-job-schedules
.. _CLOUD SCHEDULER: https://cloud.google.com/scheduler


Config
^^^^^^^^^^^^^

You can provide custom configurations for your cloudfunctions and goblet deployment by using the config.json file which should be 
located in the .goblet folder. If one doesn't exist then you should add one. 

To provide custom values for the cloudfunction configuration pass in your desired overrides in the `cloudfunction` key. See below for example.

Example fields include 

- environmentVariables
- labels
- availableMemoryMb
- timeout

Example config.json: 

.. code:: json

    {
        "cloudfunction":{
            "environmentVariables": {"env1":"var1"},
            "labels": {"label1":"val1"},
            "availableMemoryMb": 256,
            "timeout": "30s"
        }
    }

see the `cloudfunction`_ docs for more details on the fields.

.. _CLOUDFUNCTION: https://cloud.google.com/functions/docs/reference/rest/v1/projects.locations.functions#CloudFunction

By default goblet includes all python files located in the directory. To include other files use the `customFiles` key
which takes in a list of python `glob`_ formatted strings.

Example config.json: 

.. code:: json

    {
        "customFiles": ["*.yaml"]
    }   

.. _GLOB: https://docs.python.org/3/library/glob.html


Run Locally
^^^^^^^^^^^

Running your functions locally for testing and debugging is easy to do with goblet. First set a local param in the goblet class

.. code:: python

    from goblet import Goblet

    app = Goblet(function_name="goblet_example",region='us-central-1', local='test')


Then run `goblet local test` and replace test with whatever variable you decide to use.
Now you can hit your functions endpoint at `localhost:8080`.


Authentication
^^^^^^^^^^^^^
API gateway supports several authentication options including, `jwt`_, `firebase`_, `auth0`_, `Okta`_, `google_id`_, 

.. _JWT: https://cloud.google.com/api-gateway/docs/authenticating-users-jwt
.. _firebase: https://cloud.google.com/api-gateway/docs/authenticating-users-firebase
.. _auth0: https://cloud.google.com/api-gateway/docs/authenticating-users-auth0
.. _Okta: https://cloud.google.com/api-gateway/docs/authenticating-users-okta
.. _google_id: https://cloud.google.com/api-gateway/docs/authenticating-users-googleid

To configure authentication with goblet simply add the desired configuration in the `securityDefinitions` option in config.json. See the 
API gateway docs linked above for more details on how to set up the configuration. 

An api using JWT authentication would require the following in `config.json`

.. code:: json

    {
        "securityDefinitions":{
            "your_custom_auth_id":{
                "authorizationUrl": "",
                "flow": "implicit",
                "type": "oauth2",
                "x-google-issuer": "issuer of the token",
                "x-google-jwks_uri": "url to the public key"
            }
        }
    }

Request
^^^^^^^^^^^^^ 
 
The route path can only contain [a-zA-Z0-9._-] chars and curly braces for parts of the URL you want to capture. 
To access other parts of the request including headers, query strings, and post data you can use `app.current_request` to get
the request object. To see all fields see `Request`_. Note, that this also means you cannot control the routing based on query strings or headers. 
Here’s an example for accessing query string data in a view function:

.. _Request: https://werkzeug.palletsprojects.com/en/1.0.x/wrappers/#werkzeug.wrappers.Request

.. code:: python 

    @app.route('/users/{name}')
    def users(name):
        result = {'name': name}
        if app.current_request.query_params.get('include-greeting') == 'true':
            result['greeting'] = 'Hello, %s' % name
        return result

Here’s an example for accessing post data in a view function:

.. code:: python 

    @app.route('/users}', methods=["POST"])
    def users():
        json_data = app.current_request.json
        return json_data

Response
^^^^^^^^^^^^^ 

Goblet http function response should be of the form a flask `Response`_. See more at the `cloudfunctions`_ documentation

.. _RESPONSE: https://flask.palletsprojects.com/en/1.1.x/api/#flask.Response
.. _CLOUDFUNCTIONS: https://cloud.google.com/functions/docs/writing/http

jsonify is a helper to create response objects.

```Goblet.jsonify(*args, **kwargs)```

This function wraps dumps() to add a few enhancements that make life easier. It turns the JSON output into a Response 
object with the application/json mimetype. For convenience, it also converts multiple arguments into an array or 
multiple keyword arguments into a dict. This means that both jsonify(1,2,3) and jsonify([1,2,3]) serialize to [1,2,3].

For clarity, the JSON serialization behavior has the following differences from dumps():

Single argument: Passed straight through to dumps().

Multiple arguments: Converted to an array before being passed to dumps().

Multiple keyword arguments: Converted to a dict before being passed to dumps().

Both args and kwargs: Behavior undefined and will throw an exception.

Example usage:

.. code:: python 

    @app.route('/get_current_user')
    def get_current_user():
        return app.jsonify(username=g.user.username,
                    email=g.user.email,
                    id=g.user.id)

This will send a JSON response like this to the browser:

.. code:: json 

    {
        "username": "admin",
        "email": "admin@localhost",
        "id": 42
    }