..
  Technote content.

  See https://developer.lsst.io/docs/rst_styleguide.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label
     :target: http://target.link/url

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1
.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. Add content below. Do not include the document title.

Overview
========

The site `api.lsst.codes` is intended to be a unified front-end for API
services designed for programmatic consumption to support LSST/DM.  This
technote will document how to use the tools that SQuaRE has provided to
more easily write and deploy these API microservices.

In general, a microservice will be a way to extract data from an
underlying back-end source, possibly to extract from multiple sources
and correlate and aggregate, and to reformat or reduce that data in a
way that is sensible to consume programatically.

You may ask, "why not just hit the backend?  What value does the
microservice provide?"  There are two, somewhat complementary, answers.
First is that if you put the service behind a unified framework and
enforce certain standards for discoverability and output format, you can
make writing tools to query your services a great deal easier.  Second
is that the first use case for this is in a chatops service, and we
wanted to make the chatbot, in so far as possible, a presentation-layer
front end and keep data manipulation and reduction logic out of it, all
the more so since (if we stick with Hubot) the chat logic is written in
CoffeeScript, and is much more painful to work with than good old
Python.

The DM/SQuaRE tooling assumes that you will write your microservice in
Python 3 and Flask (if you're working in Python, it's nice if it works
under Python 2 as well), and that you will use the `sqre_apikit`_ module
(source at `github_apikit`_) to provide the required metadata routes.

Let's start with those.

Required Metadata
=================

The metadata must be accessible at the routes `/metadata` and
`/{{api_version}}/metadata` from the root of the microservice.  As of
the time of writing, the current (and only) API version is `1.0`.

The metadata is also documented at `github_apikit`_.

The metadata must be presented as a JSON object.  All fields except are
type `str`. 

.. code-block:: json

    {
        name
	version
	repository
	description
	api_version
	auth
    }

The fields `name`, `description`, and `version` are arbitrary.
`semantic_versioning`_ is strongly encouraged for `version`.
`api_version` should reflect the version of the API in use (at time of
writing, `1.0`).  `repository` is the URL for the source of your
project.  If you want your microservice to be published on
`api.lsst.codes` its source must be publicly available.  We extremely
strongly recommend hosting it on Github.

The `auth` field is constrained.  It must be one of `none`, `basic`, or
`bitly-proxy`.  These represent the three choices available to the
microservice for authentication to Github (we have standardized on
Github as a canonical source of authentication data, since LSST is
fairly fundamentally coupled to Github and it functions as a widely
available OAuth2 provider):

 - `none`: no authentication required.
 - `basic`: HTTP Basic Auth.  Typically used with a Github username and
   token, although if you didn't have two-factor authentication enabled
   at Github you could use a password here as well.
 - `bitly-proxy`: Authenticate through the Bitly OAuth2 proxy.
   Typically used with a Github username and password, and basically
   converts two-factor authentication back into username-and-password
   authentication.

The good news is, if you're writing in Python and your application is a
Flask app, you don't need to implement the metadata route.  Just use
`apikit`. 

Using apikit
============

The `apikit` module is documented at `_github_apikit`_, but basically:
it provides one module, `apikit`, which has two classes,
:class:`apikit.APIFlask` and :class:`apikit.BackendError` and two
functions: `set_flask_metadata` and `add_metadata_route`.

The :class:`apikit.APIFlask` class is what you should generally use: it
is a sublclass of a Flask application (:class:`flask.Flask`) which
already has metadata added and the route baked into it.

If you have an existing Flask application, you might want to use
`apikit.set_flask_metadata` on that application rather than the
:class:`apikit.APIFlask` class.  You will find `add_route_prefix` useful
to add additional routes to the metadata, which is useful, for instance,
for Kubernetes Ingress resources, which provide routing but not path
rewriting, so it's your responsibility to make sure the metadata is
available at `/{{app_name}}/metadata` as well as `/metadata`.

The :class:`apikit.BackendError` class is useful with Flask decorators
to return diagnostic information when something goes wrong with your
application.  You'll see it in the example below.

Example APIKit usage
--------------------

The :class:`apikit.APIFlask` class takes the same arguments as the
object returned by metadata, with the following exception: `auth`
becomes an object with two fields, `type`, and `data`, unless it is
`None`, the empty string, or the string `none`.  The `type` field must
be one of the strings `none`, `basic`, or `bitly-proxy`.

If `auth` is an object whose type field is `none`. `data` is the empty
object.  Otherwise it is an object with two fields, `username` and
`password`.  If `auth.type` is `bitly-proxy` then `data` must have a
third field, `endpoint`, which is the `start` point of the OAuth2 proxy
data flow for the underlying service.  Usually this is
`https://service.host/oauth2/start`.

The `api_version` field has a sane default (currently `1.0`) and can
normally be omitted.

Let's pretend that you have a service living at
https://myservice.lsst.codes, which you want to put an API wrapper
around using apikit.  Your service uses the Bitly OAuth2 proxy to use
the Github as its authentication source, so you need to leverage that.

.. code-block:: python

    import apikit
    from BitlyOAuth2ProxySession import Session
    

    backenduri = "https://myservice.lsst.codes"
    app = apkit.APIFlask(name="uservice-mymicroservice",
                         version="0.0.1",
                         repository="https://github.com/sqre-lsst/" +
                             "uservice-mymicroservice",
                         description="My delightful microservice",
                         route=["/", "/mymicroservice"],
                         auth={"type": "bitly-proxy",
                               "data": { "username": "",
                                         "password": "",
                                         "endpoint": backenduri +
                                             "/oauth2/start" } })


This creates a Flask application which presents the service metadata on
`/metadata`, `/v1.0/metadata`, `/mymicroservice/metadata`, and
`/mymicroservice/v1.0/metadata/`, as well as all of those with `.json`
appended.

Next we'll add a basic error handler.  Somewhere in your main method,
after you've created the app:

.. code-block:: python

    @app.errorhandler(apikit.BackendError)
    def handle_invalid_usage(error):
       """Custom error handler; bubble up status code, jsonify rest."""
        response = jsonify(error.to_dict())
        response.status_code = error.status_code
        return response       

Now, when you want to return an error based on something you got from
the service, create a new :class:`apikit.BackendError`.

And, because this is eventually going to run under GCE using an Ingress
TLS terminator and router (well, this is our assumption, anyway), you
want the actual application root to return a `200` very quickly, because
the Ingress controller will be pinging it often to determine service
health (GCE's Ingress defines a successful healthcheck as getting `200`
from an `HTTP GET /`.

    @app.route("/")
    def healthcheck():
        """Default route to keep Ingress controller happy."""
        return "OK"


Now, in order to actually access your data, you're going to need to make
your requests within a session with the appropriate authentication.
Let's assume that your caller is going to send you HTTP Basic
Authentication headers, and you're going to use those as username and
password to the proxy.

You'll need a place to store the session.  Fortunately, Flask provides a
mechanism for this: the app.config hash.

So, after initialization, you probably want:

.. code-block:: python

    app.config["SESSION"] = None

And then a _reauth method, so if an HTTP operation fails, you can try to
regenerate a session with your authentication data:

.. code-block:: python

    def _reauth(app, username, password):
    """Get a session with authentication data"""
    oaep = app.config["AUTH"]["data"]["endpoint"]
    # Session here comes from BitlyOAuth2Proxy
    session = Session.Session(oauth2_username=username,
                              oauth2_password=password,
                              authentication_session_url=None,
                              authentication_base_url=oaep)
    session.authenticate()
    app.config["SESSION"] = session

When we create the actual fetch of backend data, we'll see how to pull
the headers off the request we got and create an authorization object
for the session.

So let's do that now.  Let's say you call this service with something
like `GET /mymicroservice/jobname/metric` to retrieve the named metric
about jobname (imagine you want to do `GET
/mymicroservice/buildmyapp/time` to get back data about how long a build
took).  We'll pretend that your backend service is extraordinarily
ill-behaved, and conceptualizes services across metrics first and
secondarily returns the metric as a plain text value, rather than in
JSON or XML or anything sane.  You call it with `GET
/api/metric/jobname` and what you get is what you get, which you hope is
ASCII text, but it's not like the other side is going to guarantee that
to you.

What you have decided to return to your caller is, of course, JSON, and
you are going to return a structure that looks like:

.. code-block:: json

    {
        jobname
        metric
        value
    }

Where each of those fields are strings.



.. note::


   **This technote is not yet published.**

   A guide to writing microservices that will live behind api.lsst.codes and are intended for automated consumption

.. _github_apikit: https://github.com/lsst-sqre/sqre-apikit
.. _sqre_apikit: https://pypi.python.org/pypi/sqre-apikit
.. _semantic_versioning: http://semver.org