.. _configuration_points:

Configuring :mod:`repoze.who`
=============================

Configuration Points
--------------------

Classifiers
+++++++++++

:mod:`repoze.who` "classifies" the request on middleware ingress.
Request classification happens before identification and
authentication.  A request from a browser might be classified a
different way than a request from an XML-RPC client.
:mod:`repoze.who` uses request classifiers to decide which other
components to consult during subsequent identification,
authentication, and challenge steps.  Plugins are free to advertise
themselves as willing to participate in identification and
authorization for a request based on this classification.  The request
classification system is pluggable.  :mod:`repoze.who` provides a
default classifier that you may use.

You may extend the classification system by making :mod:`repoze.who` aware
of a different request classifier implementation.

Challenge Deciders
++++++++++++++++++

:mod:`repoze.who` uses a "challenge decider" to decide whether the
response returned from a downstream application requires a challenge
plugin to fire.  When using the default challenge decider, only the
status is used (if it starts with ``401``, a challenge is required).

:mod:`repoze.who` also provides an alternate challenge decider,
``repoze.who.classifiers.passthrough_challenge_decider``, which avoids
challenging ``401`` responses which have been "pre-challenged" by the
application.

You may supply a different challenge decider as necessary.

Plugins
+++++++

:mod:`repoze.who` has core functionality designed around the concept
of plugins.  Plugins are instances that are willing to perform one or
more identification- and/or authentication-related duties.  Each
plugin can be configured arbitrarily.

:mod:`repoze.who` consults the set of configured plugins when it
intercepts a WSGI request, and gives some subset of them a chance to
influence what :mod:`repoze.who` does for the current request.

.. note:: As of :mod:`repoze.who` 1.0.7, the ``repoze.who.plugins``
   package is a namespace package, intended to make it possible for
   people to ship eggs which are who plugins as,
   e.g. ``repoze.who.plugins.mycoolplugin``.


.. _imperative_configuration:

Configuring :mod:`repoze.who` via Python Code
---------------------------------------------

.. module:: repoze.who.middleware

.. class:: PluggableAuthenticationMiddleware(app, identifiers, challengers, mdproviders, classifier, challenge_decider [, log_stream=None [, log_level=logging.INFO[, remote_user_key='REMOTE_USER']]])

  The primary method of configuring the :mod:`repoze.who` middleware is
  to use straight Python code, meant to be consumed by frameworks
  which construct and compose middleware pipelines without using a
  configuration file.

  In the middleware constructor: *app* is the "next" application in
  the WSGI pipeline. *identifiers* is a sequence of ``IIdentifier``
  plugins, *challengers* is a sequence of ``IChallenger`` plugins,
  *mdproviders* is a sequence of ``IMetadataProvider`` plugins.  Any
  of these can be specified as the empty sequence.  *classifier* is a
  request classifier callable, *challenge_decider* is a challenge
  decision callable.  *log_stream* is a stream object (an object with
  a ``write`` method) *or* a ``logging.Logger`` object, *log_level* is
  a numeric value that maps to the ``logging`` module's notion of log
  levels, *remote_user_key* is the key in which the ``REMOTE_USER``
  (userid) value should be placed in the WSGI environment for
  consumption by downstream applications.

An example configuration which uses the default plugins follows::

    from repoze.who.middleware import PluggableAuthenticationMiddleware
    from repoze.who.interfaces import IIdentifier
    from repoze.who.interfaces import IChallenger
    from repoze.who.plugins.basicauth import BasicAuthPlugin
    from repoze.who.plugins.auth_tkt import AuthTktCookiePlugin
    from repoze.who.plugins.cookie import InsecureCookiePlugin
    from repoze.who.plugins.form import FormPlugin
    from repoze.who.plugins.htpasswd import HTPasswdPlugin

    io = StringIO()
    salt = 'aa'
    for name, password in [ ('admin', 'admin'), ('chris', 'chris') ]:
        io.write('%s:%s\n' % (name, password))
    io.seek(0)
    def cleartext_check(password, hashed):
        return password == hashed
    htpasswd = HTPasswdPlugin(io, cleartext_check)
    basicauth = BasicAuthPlugin('repoze.who')
    auth_tkt = AuthTktCookiePlugin('secret', 'auth_tkt')
    form = FormPlugin('__do_login', rememberer_name='auth_tkt')
    form.classifications = { IIdentifier:['browser'],
                             IChallenger:['browser'] } # only for browser
    identifiers = [('form', form),('auth_tkt',auth_tkt),('basicauth',basicauth)]
    authenticators = [('htpasswd', htpasswd)]
    challengers = [('form',form), ('basicauth',basicauth)]
    mdproviders = []

    from repoze.who.classifiers import default_request_classifier
    from repoze.who.classifiers import default_challenge_decider
    log_stream = None
    import os
    if os.environ.get('WHO_LOG'):
        log_stream = sys.stdout

    middleware = PluggableAuthenticationMiddleware(
        app,
        identifiers,
        authenticators,
        challengers,
        mdproviders,
        default_request_classifier,
        default_challenge_decider,
        log_stream = log_stream,
        log_level = logging.DEBUG
        )

The above example configures the repoze.who middleware with:

- Three ``IIdentifier`` plugins (form auth, auth_tkt cookie, and a
  basic auth plugin).  The form auth plugin is set up to fire only
  when the request is a ``browser`` request (as per the combination of
  the request classifier returning ``browser`` and the framework
  checking against the *classifications* attribute of the plugin,
  which limits ``IIdentifier`` and ``IChallenger`` to the ``browser``
  classification only).  In this setup, when "identification" needs to
  be performed, the form auth plugin will be checked first (if the
  request is a browser request), then the auth_tkt cookie plugin, then
  the basic auth plugin.

- One ``IAuthenticator`` plugin: an htpasswd one.  This htpasswd
  plugin is configured with two valid username/password combinations:
  chris/chris, and admin/admin.  When an username and password is
  found via any identifier, it will be checked against this
  authenticator.

- Two ``IChallenger`` plugins: the form plugin, then the basic auth
  plugin.  The form auth will fire if the request is a ``browser``
  request, otherwise the basic auth plugin will fire.

The rest of the middleware configuration is for values like logging
and the classifier and decider implementations.  These use the "stock"
implementations.

.. note:: The ``app`` referred to in the example is the "downstream"
   WSGI application that who is wrapping.


.. _declarative_configuration:

Configuring :mod:`repoze.who` via Config File
---------------------------------------------

:mod:`repoze.who` may be configured using a ConfigParser-style .INI
file.  The configuration file has five main types of sections: plugin
sections, a general section, an identifiers section, an authenticators
section, and a challengers section.  Each "plugin" section defines a
configuration for a particular plugin.  The identifiers,
authenticators, and challengers sections refer to these plugins to
form a site configuration.  The general section is general middleware
configuration.

To configure :mod:`repoze.who` in Python, using an .INI file, call
the `make_middleware_with_config` entry point, passing the right-hand
application and the path to the confi file ::

    from repoze.who.config import make_middleware_with_config
    who = make_middleware_with_config(app, '/path/to/who.ini')

:mod:`repoze.who`'s configuration file can be pointed to within a PasteDeploy
configuration file ::

    [filter:who]
    use = egg:repoze.who#config
    config_file = %(here)s/who.ini
    log_file = stdout
    log_level = debug

Below is an example of a configuration file (what ``config_file``
might point at above ) that might be used to configure the
:mod:`repoze.who` middleware.  A set of plugins are defined, and they
are referred to by following non-plugin sections.

In the below configuration, five plugins are defined.  The form, and
basicauth plugins are nominated to act as challenger plugins.  The
form, cookie, and basicauth plugins are nominated to act as
identification plugins.  The htpasswd and sqlusers plugins are
nominated to act as authenticator plugins. ::

    [plugin:form]
    # identificaion and challenge
    use = repoze.who.plugins.form:make_plugin
    login_form_qs = __do_login
    rememberer_name = auth_tkt
    form = %(here)s/login_form.html

    [plugin:auth_tkt]
    # identification
    use = repoze.who.plugins.auth_tkt:make_plugin
    secret = s33kr1t
    cookie_name = oatmeal
    secure = False
    include_ip = False

    [plugin:basicauth]
    # identification and challenge
    use = repoze.who.plugins.basicauth:make_plugin
    realm = 'sample'

    [plugin:htpasswd]
    # authentication
    use = repoze.who.plugins.htpasswd:make_plugin
    filename = %(here)s/passwd
    check_fn = repoze.who.plugins.htpasswd:crypt_check

    [plugin:sqlusers]
    # authentication
    use = repoze.who.plugins.sql:make_authenticator_plugin
    query = "SELECT userid, password FROM users where login = %(login)s;"
    conn_factory = repoze.who.plugins.sql:make_psycopg_conn_factory
    compare_fn = repoze.who.plugins.sql:default_password_compare

    [plugin:sqlproperties]
    name = properties
    use = repoze.who.plugins.sql:make_metadata_plugin
    query = "SELECT firstname, lastname FROM users where userid = %(__userid)s;"
    filter = my.package:filter_propmd
    conn_factory = repoze.who.plugins.sql:make_psycopg_conn_factory

    [general]
    request_classifier = repoze.who.classifiers:default_request_classifier
    challenge_decider = repoze.who.classifiers:default_challenge_decider
    remote_user_key = REMOTE_USER

    [identifiers]
    # plugin_name;classifier_name:.. or just plugin_name (good for any)
    plugins =
          form;browser
          auth_tkt
          basicauth

    [authenticators]
    # plugin_name;classifier_name.. or just plugin_name (good for any)
    plugins =
          htpasswd
          sqlusers

    [challengers]
    # plugin_name;classifier_name:.. or just plugin_name (good for any)
    plugins =
          form;browser
          basicauth

    [mdproviders]
    plugins =
          sqlproperties

The basicauth section configures a plugin that does identification and
challenge for basic auth credentials.  The form section configures a
plugin that does identification and challenge (its implementation
defers to the cookie plugin for identification "forget" and "remember"
duties, thus the "identifier_impl_name" key; this is looked up at
runtime).  The auth_tkt section configures a plugin that does
identification for cookie auth credentials.  The htpasswd plugin
obtains its user info from a file.  The sqlusers plugin obtains its
user info from a Postgres database.

The identifiers section provides an ordered list of plugins that are
willing to provide identification capability.  These will be consulted
in the defined order.  The tokens on each line of the ``plugins=`` key
are in the form "plugin_name:requestclassifier_name:..."  (or just
"plugin_name" if the plugin can be consulted regardless of the
classification of the request).  The configuration above indicates
that the system will look for credentials using the form plugin (if
the request is classified as a browser request), then the cookie
identifier (unconditionally), then the basic auth plugin
(unconditionally).

The authenticators section provides an ordered list of plugins that
provide authenticator capability.  These will be consulted in the
defined order, so the system will look for users in the file, then in
the sql database when attempting to validate credentials.  No
classification prefixes are given to restrict which of the two plugins
are used, so both plugins are consulted regardless of the
classification of the request.  Each authenticator is called with each
set of identities found by the identifier plugins.  The first identity
that can be authenticated is used to set ``REMOTE_USER``.

The mdproviders section provides an ordered list of plugins that
provide metadata provider capability.  These will be consulted in the
defined order.  Each will have a chance (on ingress) to provide add
metadata to the authenticated identity.  Our example mdproviders
section shows one plugin configured: "sqlproperties".  The
sqlproperties plugin will add information related to user properties
(e.g. first name and last name) to the identity dictionary.

The challengers section provides an ordered list of plugins that
provide challenger capability.  These will be consulted in the defined
order, so the system will consult the cookie auth plugin first, then
the basic auth plugin.  Each will have a chance to initiate a
challenge.  The above configuration indicates that the form challenger
will fire if it's a browser request, and the basic auth challenger
will fire if it's not (fallback).