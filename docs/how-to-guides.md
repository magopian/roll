# How-to guides

A how-to guide:

* is goal-oriented
* shows how to solve a specific problem
* is a series of steps

*Analogy: a recipe in a cookery book*


## How to install Roll

Roll requires Python 3.5+ to be installed.

It is recommended to install Roll using
[pipenv](https://docs.pipenv.org/):

    pipenv install roll


## How to develop Roll

It is recommended to develop Roll using
[pipenv](https://docs.pipenv.org/):

    pipenv install '-e .' --dev --three

Then either activate the virtualenv:

    pipenv shell
    py.test

Or directly run a command via pipenv:

    pipenv run py.test

Create a dedicated branch, hack hack hack, pull-request as usual.

To (re)generate requirements files with hashes:

    pipenv lock --requirements > requirements.txt
    pipenv lock --requirements --dev > requirements-dev.txt


## How to create an extension

You can use extensions to achieve a lot of enhancements of the base
framework.

Basically, an extension is a function listening to
[events](reference.md#events), for instance:

```python
def cors(app, value='*'):

    @app.listen('response')
    async def add_cors_headers(response, request):
        response.headers['Access-Control-Allow-Origin'] = value
```

Here the `cors` extension can be applied to the Roll `app` object.
It listens to the `response` event and for each of those add a custom
header. The name of the inner function is not relevant but explicit is
always a bonus. The `response` object is modified in place.

*Note: more [extensions](reference.md#events) are available by default.
Make sure to check these out!*


## How to deal with content negociation

The [`content_negociation` extension](reference.md#content_negociation)
is made for this purpose, you can use it that way:

```python
extensions.content_negociation(app)

@app.route('/test', accepts=['text/html', 'application/json'])
async def get(req, resp):
    if req.headers['Accept'] == 'text/html':
        resp.headers['Content-Type'] = 'text/html'
        resp.body = '<h1>accepted</h1>'
    elif req.headers['Accept'] == 'application/json':
        resp.json = {'status': 'accepted'}
```

Requests with `Accept` header not matching `text/html` or
`application/json` will be honored with a `406 Not Acceptable` response.


## How to return an HTTP error

There are many reasons to return an HTTP error, with Roll you have to
raise an HttpError instance. Remember our
[base example from tutorial](tutorials.md#your-first-roll-application)?
What if we want to return an error to the user:

```python
from http import HTTPStatus

from roll import Roll, HttpError
from roll.extensions import simple_server

app = Roll()


@app.route('/hello/{parameter}')
async def hello(request, response, parameter):
    if parameter == 'foo':
        raise HttpError(HTTPStatus.BAD_REQUEST, 'Run, you foo(l)!')
    response.body = f'Hello {parameter}'


if __name__ == '__main__':
    simple_server(app)
```

Now when we try to reach the view with the `foo` parameter:

```
$ http :3579/hello/foo
HTTP/1.1 400 Bad Request
Content-Length: 16

Run, you foo(l)!
```

One advantage of using the exception mechanism is that you can raise an
HttpError from anywhere and let Roll handle it!


## How to return JSON content

There is a shortcut to return JSON content from a view. Remember our
[base example from tutorial](tutorials.md#your-first-roll-application)?

```python
from roll import Roll
from roll.extensions import simple_server

app = Roll()


@app.route('/hello/{parameter}')
async def hello(request, response, parameter):
    response.json = {'hello': parameter}


if __name__ == '__main__':
    simple_server(app)
```

Setting a `dict` to `response.json` will automagically dump it to
regular JSON and set the appropriated content type:

```
$ http :3579/hello/world
HTTP/1.1 200 OK
Content-Length: 17
Content-Type: application/json; charset=utf-8

{
    "hello": "world"
}
```

Especially useful for APIs.


## How to subclass Roll itself

Let’s say you want your own [Query](reference.md#core-objects) parser
to deal with GET parameters that should be converted as `datetime.date`
objects.

What you can do is subclass both the Roll class and the Protocol one
to set your custom Query class:

```python
from datetime import date

from roll import Roll, Query
from roll.extensions import simple_server


class MyQuery(Query):

    @property
    def date(self):
        return date(int(self.get('year')),
                    int(self.get('month')),
                    int(self.get('day')))


class MyProtocol(Roll.Protocol):
    Query = MyQuery


class MyRoll(Roll):
    Protocol = MyProtocol


app = MyRoll()


@app.route('/hello/')
async def hello(request, response):
    response.body = request.query.date.isoformat()


if __name__ == '__main__':
    simple_server(app)
```

And now when you pass appropriated parameters (for the sake of brievety,
no error handling is performed but hopefully you get the point!):

```
$ http :3579/hello/ year==2017 month==9 day==20
HTTP/1.1 200 OK
Content-Length: 10

2017-09-20
```


## How to deploy Roll into production

The recommended way to deploy Roll is using
[Gunicorn](http://docs.gunicorn.org/).

First install gunicorn in your virtualenv:

    pip install gunicorn

To run it, you need to pass it the pythonpath to your roll project
application. For example, if you have created a module `core.py`
in your package `mypackage`, where you create your application
with `app = Roll()`, then you need to issue this command line:

    gunicorn mypackage.core:app --worker roll.worker.Worker

See [gunicorn documentation](http://docs.gunicorn.org/en/stable/settings.html)
for more details about the available arguments.


## How to run Roll’s tests

Roll exposes a pytest fixture (`client`), and for this needs to be
properly installed so pytest sees it. Once in the roll root (and with
your virtualenv active), run:

    python setup.py develop

Then you can run the tests:

    py.test


## How to send custom events

Roll has a very small API for listening and sending events. It's possible to use
it in your project for your own events.

Events are useful when you want other users to extend your own code, whether
it's a Roll extension, or a full project built with Roll.
They differ from configuration in that they are more adapted for dynamic
modularity.

For example, say we develop a DB pooling extension for Roll. We
would use a simple configuration parameter to let users change the connection
credentials (host, username, password…). But if we want users to run some
code each time a new connection is created, we may use a custom event.

Our extension usage would look like this:

    app = Roll()
    db_pool_extension(app, dbname='mydb', username='foo', password='bar')

    @app.listen('new_connection')
    def listener(connection):
        # dosomething with the connection,
        # for example register some PostgreSQL custom types.

Then, in our extension, when creating a new connection, we'd do something like
that:
    app.hook('new_connection', connection=connection)


## How to use a livereload development server

First, install [hupper](https://pypi.python.org/pypi/hupper).

Then turn your Roll service into an importable module. Basically, a folder with
`__init__.py` and `__main__.py` files and put your `simple_server` call within
the `__main__.py` file (see the `example` folder for… an example!).

Once you did that, you can run the server using `hupper -m example`. Each and
every time you modify a python file, the server will reload and take into
account your modifications accordingly.

One of the pros of using hupper is that you can even set an `ipdb` call within
your code and it will work seamlessly (as opposed to using solutions like
[entr](http://www.entrproject.org/)).
