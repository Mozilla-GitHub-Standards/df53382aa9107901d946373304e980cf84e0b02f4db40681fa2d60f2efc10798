# Hawk authentication for ExpressJS

This module provides an [Hawk](https://github.com/hueniverse/hawk)
authentication middleware for express applications.  More specifically, for
applications which uses the connect middleware facility.

Hawk itself does not provide any mechanism for obtaining or transmitting the
set of shared credentials required, but this project proposes a scheme we use
accross mozilla-services projects.

## Installation

    npm install express-hawkauth

## How do I plug that in my application?

In order to plug express-hawk in your application, you'll need to use it as
a middleware.

    var express = require("express");
    var hawk = require("express-hawkauth");
    app = express();

    var hawkMiddleware = hawk.getMiddleware({
      hawkOptions: {},
      getSession: function(tokenId, cb) {
        // A function which pass to the cb the key and algorithm for the
        // given token id. First argument of the callback is a potential
        // error.
        cb(null, {
          key: "key",
          algorithm: "sha256"
        });
      },
      createSession: function(id, key, cb) {
        // A function which stores a session for the given id and key.
        // Argument returned is a potential error.
        cb(null);
      },
      setUser: function(req, res, credentials, cb) {

        // A function that uses req and res and the credentials so
        // that it can tweak it. For instance, you can store the tokenId
        // as the user.

        req.user = credentials.id;
      }
    });

    app.get('/hawk-enabled-endpoint', hawkMiddleware);

You can also pass a `sendError` parameter which is a function that's being
passed the errors generated by the library. Takes (res, status, payload) as
parameters.

If you want to only check a valid hawk session exists (without creating a new
one), just create a middleware which doesn't have any `createSession` parameter
defined.

## What's returned to the clients

In case an hawk session is created (e.g. when createSession had been defined
and no credentials were provided in the request, a `Hawk-Session-Token` header
will be set to the response, containing the session token to be derived.

## How are the shared credentials shared?

Okay, on to the actual details.

The server gives you a session token, that you'll need to derive to get the
hawk credentials:

Do an HKDF derivation on the given session token. You’ll need to use the
following parameters::

    key_material = HKDF(hawk_session, “”, ‘identity.mozilla.com/picl/v1/sessionToken’, 32*2);

The key material you’ll get out of the HKDF need to be separated into two
parts, the first 32 hex characters are the hawk id, and the next 32 ones are the
hawk key:

    credentials = {
        'id': keyMaterial[0:32]
        'key': keyMaterial[32:64]
        'algorithm': 'sha256'
    }
