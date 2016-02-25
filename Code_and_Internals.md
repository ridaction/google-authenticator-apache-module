# Introduction #

About the source code and internals of the project


# Call Graph #

Here is a basic call graph of the project

![http://s3.bradgoodman.com/ga_callgraph.png](http://s3.bradgoodman.com/ga_callgraph.png)

## Top Level ##

`authn_google_module` is akin to the "primary entry point". It defines the initialization and hook registration functions.

`register_hooks` Hooks three functions into Apache:

  * `ga_child_init` - a general initialization function
  * `auth_provider` - Registers some authorization provider functions (discussed below)
  * `check_user_id` - This hook is used to "supersede" normal authentication. We use it to detect the presence of our cookie which we use to tell us that we have already authenticated, and therefore do not need to use our whole two-factor system again.

## Authentication Provider ##

`register_hook` (as described above) registers a `authn_google_provider` structure which defines the two primary authorization functions. When we register these hooks, Apache uses these two functions in it's authorization chain, (if it has not previously cleared the `check_user_id` above. This is where the main body of our two-factor authorization is done. Per HTTP, this may be done in one of two different ways, and thus two separate functions are required:

### check\_password ###

`ga_check_password` is called when a client has attempted to connect via [Basic Access Authentication](http://en.wikipedia.org/wiki/Basic_access_authentication). With basic access authentication, a user's username and password are sent in cleartext from the client to the server. Using this on a non-encrypted session is considered to be **very insecure**, as this information can easily be snooped.

When this authentication is used, the `ga_check_password` function is given the username and password that the user presented, and it is up to the module to validate it.

### get\_realm\_hash ###

The `ga_get_realm_hash` function is used when a client has used [Digest Access Authentication](http://en.wikipedia.org/wiki/Digest_access_authentication).

With this method, the HTTP client generates a hash ("HA1") consisting of:
  * The server-provided "realm" (Apache-configured cleartext string)
  * Username
  * Password

The service sends a one-time value ("nonce") to the client. The client hashes this value with the previous hash, generating a second hash ("HA2") sending it to the server.

The `ga_realm_hash` function is passed the username and realm. After running our 2-Factor authorization algorithm, it determines what the password _should be_, then runs the same hash algorithm of it, along with the specified username and realm.

It them returns the (HA1)_hash result_ back to Apache, which hashes it with the previously generated nonce value, and decides if it matches that (HA2) which the user had specified.

This is much more secure because:

  * The password is never transmitted over the wire, in cleartext or otherwise.
  * The transmitted hash is only "good" for a given request. Any subsequent request would have been hashed against a different nonce, and thus not work.
  * Even if an unencrypted connection is used, the password or hash cannot be snooped.
  * Even if a client were to cache the HA1 hash, it does not contain the cleartext password.


### "Third Level" Code ###

The "Third level" code are the common functions that the two provider functions above both use to perform the authentication. They are:

  * `computeTimeCode` - Given the secret key, and the current time, returns the correct security code for the given time.
  * `get_timestamp` - Returns current time. Number represents number of 30-second intervals since epoch, per authentication protocol standard
  * `addCookie` - Add cookie to authentication request - allowing subsequent access withouth re-authentication (within alloted re-authorization time)
  * `getUserSecret` - From the username - fetch the secret key. This works now by reading static files. Returns Base32 encoded key
  * `get_shared_secret` - Converts key from getUserSecret from Base32 into usable binary key.
  * `getSharedKey` - Reads specified filename - extracts plaintext key.
  * `hash_cookie` - Performs cryptographic hash of authentication cookie - so we can set and, and/or determine if it is valid

## Other References and Acknowledgements ##

  * [Apache HTTP server and Portable Runtime API Docs](http://ci.apache.org/projects/httpd/trunk/doxygen/)
  * [Nick Kew's](http://www.apachetutor.org/) incredible [Apache Modules Book](http://www.amazon.com/Apache-Modules-Book-Application-Development/dp/0132409674)