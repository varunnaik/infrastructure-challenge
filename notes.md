#### The Mandate
An authentication + authorisation system that allows:

1. Logging in with a username/password
2. Logging in with a third party (e.g google) via OAuth and/or SAML
3. SSO across domains and subdomains
4. Efficient checking of request authorisation for both the microservcies and the monolith
5. Efficient interservice communication with the ability to easily impersonate a user from either the monolith or the microservice
6. The ability to communicate between services outside the context of a request. e.g. from an asynchronous worker or cron job

---

This is a considerably complicated challenge and is hard to solve lacking context and scope for the problem.

One simple way to solve this would be to use multisite cookies, eg.
`SESSION_COOKIE_DOMAIN=".lildatum.com"` in `settings.py` which will allow the login cookie to be used on subdomains. This would not work on entirely different domains, where something like [Django-simple-sso](https://github.com/divio/django-simple-sso) might be used instead.

While these are good options, ultimately they do not scale to a robust and extensible architecture.

I would suggest a more extensive overhaul as described below.

-------

![Architecture](propeller.png)

- An incoming request from the client - web or mobile - is sent by the API gateway (the evolution of the reverse-proxy) to an auth microservice. This can be done with the [auth proxy](https://docs.nginx.com/nginx/admin-guide/security-controls/configuring-subrequest-authentication/) nginx module.

- The auth microservice verifies the incoming Auth0 JWT and then does a database lookup for ACLs, or other authorisation parmeters and returns them to the gateway.

- The API gateway strips the original JWT out of the request, puts these authorisation parameters into an internal auth header - maybe a JWT or a simple JSON string, and forwards the request to the correct service.

- If the monolith or a microservice gets the request, it checks the internal auth header and allow access if the embedded scopes permit it. If more fine-grained access control is needed it hits the auth microservice again to verify ACLs.

- If, for example, this reaches the monolith and it needs to access a microservice to fulfill its business logic, it simply crafts a new request, appends the internal auth header, and sends it off to the microservice.

- If an out-of-band service - maybe a cron job - needs to impersonate a user it simply makes a request to the auth microservice and gets an internal auth token for that user. The servers must have an internal token based auth for talking to each other and the auth service must also be secured with AWS security groups or network policies.

A more detailed look at ths process:

#### 1. To login with a username, third-party Oauth, and across domains and subdomains.

Use an auth domain (eg. `auth.lildatum.com`) - any user visiting any domain or subdomain of the company gets redirected to auth.lildatum.com and this controls the login cookie and redirects user back to originating (sub)domain with a JWT token if logged in, else redirects to a login page and prompts for login before sending them back. This JWT can authenticate subsequent API calls.

This can either be build in-house or delegated to a third-party service (Auth0/Okta/Cognito). Auth0 can import existing users although in this case the differences in hashing (pbkdf for lildatum, while Auth0 [needs bcrypt](https://auth0.com/docs/users/guides/bulk-user-imports)) means we might need to use a [custom db connection](https://auth0.com/docs/connections/database/custom-db/overview-custom-db-connections) failing which users will have to change their passwords on first visit.

Auth0 will also provide Oauth logins, e.g. with Google.


#### 2. Efficient checking of request authorisation for both the microservcies and the monolith

This can be done with an API gateway and centralised auth microservice. There are two ways to do this:
- Every microservice or the monolith checks each request against this service, or,
- The API gateway appends a list of permissions and scopes (as returned by the auth microservice) to each request before passing it upstream. This can be appended as an internal JWT token with a list of claims or more generally as asimple json object saving the need for decryption on each request.

Now if there are requirements for fine-grained resource level permissions then the second options is not sufficient, as the metadata for all resources and permissions could be much lager than would fit onto a HTTP header. In general the second option is likely not sufficient for anything but very simple use-cases.

The first approach is much more standard, and is likely to benefit greatly from a Redis or similar cache.

But the second could still be implemented as an optimisation depending on context - if a majority of requests are simple ones that don't access access-controlled resources, maybe the two could be used in conjunction.

#### 3. Efficient interservice communication...

In this case, simply making a new request to the target service copying the required headers should suffice. If the destination microservice needs to perform additional ACL checks it would touch the auth microservice instead of the monolith.

#### 4. ...with the ability to easily impersonate a user from either the monolith or the microservice and the ability to communicate between services outside the context of a request. e.g. from an asynchronous worker or cron job

This makes many assumptions - are all tasks performed on user's behalf or are they general tasks? I am not sure if impersonation is a good idea - ideally requests originating from the user would be better treated separately from system-originating requests, eg. for auditing, debugging - tracing a request from browser to time of processing, logging. If internal services can impersonate users this might confuse things. Of course, this depends on context. This certainly warrants discussion.

However, if impersonation is required then a separate endpoint on the auth API where a microservice can request to impersonate a user would be the way to go.

#### Notes:

- [Vouch-proxy](https://github.com/vouch/vouch-proxy) can perform the role of the auth microservice.

- This involves a separate network request for each incoming request. It also involves a database lookup (i.e., for the ACLs). This latter one can be cached with Redis, this is an ideal place for Redis to improve performance.

- This can extend auth to cover future developments - for example, if a public API is an offering then this same auth proxy can handle authentication in that system as well.

- Since auth is handled by a separate microservice, this is language agnostic - as long as the language/framework used in the microservice can verify JWT tokens and parse JSON it should work just fine.

----
1
All in all, this has been a very interesting and well thought out challenge. Ultimately, besides fixing a couple of bugs, I have not written any code to implement this as it is beyond the scope of a 3-4 hour programming test.