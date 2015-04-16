asana
=====

Python client library for Asana.

![Build Status](https://api.travis-ci.org/Asana/python-asana.svg)

Authentication
--------------

### Basic Auth

Create a client using your Asana API key:

    client = asana.Client.basic_auth('ASANA_API_KEY')

### OAuth 2

Asana supports OAuth 2. `asana` handles some of the details of the OAuth flow for you.

Create a client using your OAuth Client ID and secret:

    client = asana.Client.oauth(
      client_id='ASANA_CLIENT_ID',
      client_secret='ASANA_CLIENT_SECRET',
      redirect_uri='https://yourapp.com/auth/asana/callback'
    )

Redirect the user to the authorization URL obtained from the client's `session` object:

    (url, state) = client.session.authorization_url()

When the user is redirected back to your callback, check the `state` URL parameter matches, then pass the `code` parameter to obtain a bearer token:

    if request.params['state'] == state:
      token = client.session.fetch_token(code=request.params['code'])
      # ...
    else:
      # error! possible CSRF attack

Note: if you're writing a non-browser-based application (e.x. a command line tool) you can use the special redirect URI `urn:ietf:wg:oauth:2.0:oob` to prompt the user to copy and paste the code into the application.

Usage
-----

The client's methods are divided into several resources: `attachments`, `events`, `projects`, `stories`, `tags`, `tasks`, `teams`, `users`, and `workspaces`.

Methods that return a single object return that object directly:

    me = client.users.me()
    print "Hello " + me['name']

    workspace_id = me['workspaces'][0]['id']
    project = client.projects.create_in_workspace(workspace_id, { 'name': 'new project' })
    print "Created project with id: " + project['id']

Methods that return multiple items (e.x. `find_all`) return an item iterator by default. See the "Collections" section

Options
-------

Various options can be set globally on the `Client.DEFAULTS` object, per-client on `client.options`, or per-request as additional named arguments. For example:

    # global:
    asana.Client.DEFAULTS['page_size'] = 1000

    # per-client:
    client.options['page_size'] = 1000

    # per-request:
    client.tasks.find_all({ 'project': 1234 }, page_size=1000)

### Available options

* `base_url` (default: "https://app.asana.com/api/1.0"): API endpoint base URL to connect to
* `max_retries` (default: 5): number to times to retry if API rate limit is reached or a server error occures. Rate limit retries delay until the rate limit expires, server errors exponentially backoff starting with a 1 second delay.
* `full_payload` (default: False): return the entire JSON response instead of the 'data' propery (default for collection methods and `events.get`)
* `fields` and `expand`: array of field names to include in the response, or sub-objects to expand in the response. For example `fields=['followers', 'assignee']`. See [API documentation](https://asana.com/developers/documentation/getting-started/input-output-options)

Collections (methods returning an array as it's 'data' property):

* `iterator_type` (default: "items"): specifies which type of iterator (or not) to return. Valid values are "items" and `None`.
* `item_limit` (default: None): limits the total number of items of a collection to return (spanning multiple requests in the case of an iterator).
* `page_size` (default: 50): limits the number of items per page to fetch at a time.
* `offset`: offset token returned by previous calls to the same method (in `response['next_page']['offset']`)

Events:

* `poll_interval` (default: 5): polling interval for getting new events via `events.get_next` and `events.get_iterator`
* `sync`: sync token returned by previous calls to `events.get` (in `response['sync']`)

Collections
-----------

### Items Iterator

By default, methods that return a collection of objects return an item iterator:

    workspaces = client.workspaces.find_all(item_limit=1)
    print workspaces.next()
    print workspaces.next() # raises StopIteration if there are no more items

Or:

    for workspace in client.workspaces.find_all()
      print workspace

Internally the iterator may make multiple HTTP requests, with the number of requested results per page being controlled by the `page_size` option.

### Raw API

You can also use the raw API to fetch a page at a time:

    offset = None
    while True:
      page = client.workspaces.find_all(offset=offset, iterator_type=None)
      print page['data']
      if 'next_page' in page:
        offset = page['next_page']['offset']
      else:
        break
