---
title: Using Portier with Python's asyncio
issue_id: 3
---

[Mozilla Persona][mzp] was an authentication solution that sought to preserve
user privacy while making life easy for developers (as explained [here][det]).
You may have noticed the past tense there &mdash; Persona has gone away. A new
project, [Portier][prt], picks up where Persona left off, and I'm eager to see
it succeed. I'll do anything to avoid storing password hashes or dealing with
OAuth. With that in mind, here is my small contribution: a guide to using
Portier with [Python][pth]'s [asyncio][aio].

### The Big Idea

So far, the [official guide to using Portier][gde] is... short. It tells you to
inspect and reuse code from the [demo implementation][dmo]. So that's what
we'll do! But first, let's take a look at how this is supposed to work.

The guide says

>The Portier Broker's API is identical to [OpenID Connect][oidc]'s "[Implicit
>Flow][flo]."

From the developer's perspective, that means

1. A user submits their e-mail address via a `POST` to an endpoint on your
server.
2. Your server
    1. Generates and stores a [nonce][nce] for this request.
    2. Replies with a redirect to an authorization endpoint (as of today, that
    means hardcoding Portier's Broker URL: `https://broker.portier.io`).
3. Your server will eventually receive a `POST` to one of its endpoints
(`/verify` in the demo implementation). At this point
    1. If the data supplied are all valid (including checking against the nonce
    created in step 2.1.), then the user has been authenticated and your server
    can set a session cookie.
    2. If something is invalid, show the user an error.

This is all well and good... provided that you're using Python and your server
code is synchronous. I generally use [Tornado][tdo] with asyncio for my Python
web framework needs, so some tweaks need to be made to get everything working
together nicely.

If you want to use something other than Python, I can't really help
you. I did say Portier is new, didn't I?

### Enter asyncio

For some background, Python 3.4 [added][3.4] a way to write non-blocking
single-threaded code, and Python 3.5 [added][3.5] some language keywords to
make this feature easier to use. For the sake of brevity I'll include code that
works in Python 3.5 or later. [Here][snk] is a blog post describing the
changes in case you need to use an earlier version of Python.

For those of you using the same setup that I do (Tornado and asyncio), refer to
[this page][tas] for getting things up and running.

#### [The login endpoint][lgn]

This code does not need to be modified to work with asyncio. I'll include what
it should look like when using Tornado, though. Assuming that

- `REDIS` is a Redis connection object as from [redis-py][rds]
- `SETTINGS` is a dictionary containing your application's settings
- `SETTINGS['WebsiteURL']` is the URL of your application (such as
`https://www.example.com`)
- `SETTINGS['BrokerURL']` is the URL of the Portier Broker,
`https://broker.portier.io`

{% highlight python %}
from datetime import timedelta
from urllib.parse import urlencode
from uuid import uuid4
import tornado.web

class LoginHandler(tornado.web.RequestHandler):
    def post(self):
        nonce = uuid4().hex
        REDIS.setex(nonce, timedelta(minutes=15), '')
        query_args = urlencode({
            'login_hint': self.get_argument('email'),
            'scope': 'openid email',
            'nonce': nonce,
            'response_type': 'id_token',
            'response_mode': 'form_post',
            'client_id': SETTINGS['WebsiteURL'],
            'redirect_uri': SETTINGS['WebsiteURL'] + '/verify',
        })
        self.redirect(SETTINGS['BrokerURL'] + '/auth?' + query_args)
{% endhighlight %}

#### [The verify endpoint][ver]
 
This does need some modification to work. Assuming that you have defined an
exception class for your application called `ApplicationError`

{% highlight python %}
import tornado.web

class VerifyHandler(tornado.web.RequestHandler):
    def check_xsrf_cookie(self):
        """Disable XSRF check.
        OIDC Implicit Flow doesn't reply with _xsrf header.
        https://github.com/portier/demo-rp/issues/10
        """
        pass

    async def post(self):  # Make this method a coroutine with async def
        if 'error' in self.request.arguments:
            error = self.get_argument('error')
            description = self.get_argument('error_description')
            raise ApplicationError('Broker Error: {}: {}'.format(error, description))
        token = self.get_argument('id_token')
        email = await get_verified_email(token)  # Use await to make this asynchronous
        # The demo implementation handles RuntimeError here but you may want to
        # deal with errors in your own application-specific way

        # At this point, the user has authenticated, so set the user cookie in
        # whatever way makes sense for your application.
        self.set_secure_cookie(...)
        self.redirect(self.get_argument('next', '/'))
{% endhighlight %}

#### [get_verified_email][gve]

This function only needs two straightforward changes from the demo
implementation:

{% highlight python %}
async def get_verified_email(token):
{% endhighlight %}

and

{% highlight python %}
keys = await discover_keys(SETTINGS['BrokerURL'])
{% endhighlight %}

#### [discover_keys][dis]

This function needs three changes from the demo implementation. The first is
simple again:

{% highlight python %}
async def discover_keys(broker):
{% endhighlight %}

The second change is in the line with `res = urlopen(''.join((broker,
'/.well-known/openid-configuration')))`. The problem is that [`urlopen`][uop] is
blocking, so you can't just `await` it. If you're not using Tornado, I
recommend using the [aiohttp][aih] library (refer to the
[client example][cex]). If you *are* using Tornado, you can use the
[AsyncHTTPClient][ahc] class.

{% highlight python %}
http_client = tornado.httpclient.AsyncHTTPClient()
url = broker + '/.well-known/openid-configuration'
res = await http_client.fetch(url)
{% endhighlight %}

The third change is similar to the second: `raw_jwks =
urlopen(discovery['jwks_uri']).read()` uses `urlopen` again. Solve it the same
way:

{% highlight python %}
raw_jwks = (await http_client.fetch(discovery['jwks_uri'])).body
{% endhighlight %}

### Wrapping up

Down the line, there will be client-side libraries (or at least, other demo
implementations) for various languages. Until then, you'll need to do some of
the heavy lifting yourself.

[mzp]: https://en.wikipedia.org/wiki/Mozilla_Persona
[det]: https://developer.mozilla.org/en-US/Persona/Why_Persona
[prt]: https://portier.github.io/
[pth]: https://www.python.org/
[aio]: https://docs.python.org/3/library/asyncio.html
[gde]: https://github.com/portier/portier.github.io/blob/3a46c2f21ac3b748ec78ca9ac4099b073446cdc7/Using.md
[dmo]: https://github.com/portier/demo-rp/blob/6aee99fe126eceda527cae1f6da3f02a68401b6e/server.py
[oidc]: http://openid.net/specs/openid-connect-core-1_0.html
[flo]: http://openid.net/specs/openid-connect-core-1_0.html#ImplicitFlowAuth
[nce]: https://en.wikipedia.org/wiki/Cryptographic_nonce
[tdo]: http://www.tornadoweb.org/en/stable/
[3.4]: https://docs.python.org/3/whatsnew/3.4.html#whatsnew-asyncio
[3.5]: https://docs.python.org/3/whatsnew/3.5.html#pep-492-coroutines-with-async-and-await-syntax
[snk]: https://snarky.ca/how-the-heck-does-async-await-work-in-python-3-5/#goingfromyieldfromtoawaitinpython35
[lgn]: https://github.com/portier/demo-rp/blob/6aee99fe126eceda527cae1f6da3f02a68401b6e/server.py#L69-L103
[rds]: https://github.com/andymccurdy/redis-py
[tas]: http://www.tornadoweb.org/en/stable/asyncio.html
[ver]: https://github.com/portier/demo-rp/blob/6aee99fe126eceda527cae1f6da3f02a68401b6e/server.py#L112-L151
[gve]: https://github.com/portier/demo-rp/blob/6aee99fe126eceda527cae1f6da3f02a68401b6e/server.py#L240-L296
[dis]: https://github.com/portier/demo-rp/blob/6aee99fe126eceda527cae1f6da3f02a68401b6e/server.py#L187-L230
[uop]: https://docs.python.org/3/library/urllib.request.html#urllib.request.urlopen
[aih]: https://aiohttp.readthedocs.io/
[cex]: https://aiohttp.readthedocs.io/en/stable/#getting-started
[ahc]: http://www.tornadoweb.org/en/stable/httpclient.html#tornado.httpclient.AsyncHTTPClient
