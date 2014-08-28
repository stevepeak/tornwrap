# tornwrap [![Build Status](https://secure.travis-ci.org/stevepeak/tornwrap.png)](http://travis-ci.org/stevepeak/tornwrap) [![codecov.io](https://codecov.io/github/stevepeak/tornwrap/coverage.png?branch=master)](https://codecov.io/github/stevepeak/tornwrap)

> Collection of commonly used metheods. Feedback and PRs welcome!

```sh
pip install tornwrap
```

## Projet Contents

- [`@authenticated`](#authenticated)
  - via `Basic realm=Restricted`
- [`@ratelimited`](#ratelimited)
  - limit usage for guests and authenticated users
- [`@validated`](#validated)
  - using [valideer](https://github.com/podio/valideer) to validate and adapt body and/or url args
- *future* [`@cached`](#cached)
  - cache requests, ex page builds
- *future* [`@gist`](#gist)
  - retrieve contents of a [gist](https://gist.github.com/) and return it to the handler
- *future* [`@markdown`](#markdown)
  - generate html pages from a Github repo containing markdown



# `@authenticated`

```py
from tornwrap import authenticated

class Handler(RequestHandler):
    def get_authenticated_user(self, user, password):
        return self.db.get("select * from users where username=%s and password=%s limit 1", user, password)
        if result:
            return result # will call `self.current_user = result` for you
        else:
            return None # will raise HTTPError(401)

    @authenticated
    def get(self):
        self.write("Hello, world!")
```

# `@ratelimited`
> Requires `redis`

```py
from tornwrap import ratelimited

class Handler(RequestHandler):
    def initialize(self, redis):
        self.redis = redis # required

    @ratelimited(guest=(1000, 3600), user=(10000, 1800))
    def get(self):
        # users get 10k requests every 30 min
        # guests get 1k every 1h
        self.write("Hello, world!")

    def was_rate_limited(self, tokens, remaining, ttl):
        # notice we do not raise HTTPError here
        # because it will reset headers, which kinda sucks
        self.set_status(403)
        self.finish({"message": "You have been rate limited."})
```

# `@validated`
> Uses [valideer](https://github.com/podio/valideer)

```py
from tornwrap import validated

class Handler(RequestHandler):
    @validated({"+name":"string"})
    def get(self):
        # all your data is in `self.validated`
        self.finish("Hello, %s!" % self.validated['name'])
```

# `@cached` (future feature)
> Cache the results of the http request.

```python
from tornwrap import cached

class Handler(RequestHandler):
    @cached(key="%(arg)s")
    def get(self, arg):
        # maybe a long api request
        return results # will call `self.finish(results)` for you

    def set_cache(self, key, data):
        self.memecached.set(key, data)

    def get_cache(self, key):
        return self.memecached.get(key)

```
> Will create methods for `async` methods to `get` and `set` too. We all <3 async!

# `@gist` (future feature)
> Fetch a Github gist content and return it to the handler method. 
> Useful for chaning home page on the fly.

```python
from tornwrap import gist

class Handler(RequestHandler):
    @gist("stevepeak/5592167", refresh=False)
    def get(self, gist):
        self.render("template.html", gist=gist)
```
> `refresh` can be: `True`: fetch on every request, `False`: fetch right away, cache forever, `:seconds` till expires then refetch
> For those using Heroku, keep in mind dynos reset every 24 hours-ish.

# `@markdown` (future feature)
> Merge [torndown](https://github.com/stevepeak/torndown) project