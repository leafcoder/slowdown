=========
Scripting
=========

.. contents::
    :depth: 1
    :local:
    :backlinks: none


Package initialization
----------------------

.. code-block:: apacheconf
    :emphasize-lines: 2-4,13

    <modules>
        load MY.PACKAGE
        load MY.MODULE
        load ..
    </modules>

    <routers>
        <router ROUTER_NAME>
            pattern REGEX
            <host HOST_NAME>
                pattern REGEX
                <path PATH_NAME>
                    handler MY.PACKAGE_OR_MODULE
                </path>
            </host>
        </router>
    </routers>

In fact, modules registered with ``<path>handler MODULE</path>`` have
the same import machanism as modules registered with ``<modules>`` section
and are stored in the **modules** attribute of the
:py:class:`~slowdown.__main__.Application` object.

When the server starts, it executes the existing
**initialize(application)** function of the registered modules/packages.
The **initialize(application)** function accepts an
:py:class:`~slowdown.__main__.Application` object as an argument.

Example:

.. code-block:: python

    # file: entrypoint.py           -- sample module
    #  or
    # file: entrypoint/__init__.py  -- sample package

    import datetime

    def initialize(application):
        global start_time
        start_time = f'Start time: {datetime.datetime.now()}'

    def handler(rw):
        rw.send_html_and_close(
            content=f'<html>{start_time}</html>'
        )


Mapfs architecture
------------------

If you create a package that doesn't contain a **handler**, the default
handler that is a :py:class:`~slowdown.mapfs.Mapfs` object is automatically
generated by the server at startup.

The automatically generated **handler** uses the ``__www__`` dir under the
package dir as the folder of static files, and the ``__cgi__`` dir under
the package dir as the folder of script files.

.. code-block:: text

    myproj/
        pkgs/
            mysite/
                __init__.py
                __www__/
                __cgi__/

Static files in the ``__www__`` folder shall be sent to the browser. And
scripts in the ``__cgi__`` folder will be executed when requested.

The static file will be sent if the file and script are matched by the same
URL. If no files or scripts are matched, the existing ``index.html`` file
or ``index.html.py`` script will be the choice.

Swap files and hidden files whose names start with dot ( ``.`` ), end with
tilde ( ``~`` ), and have ``.swp`` , ``.swx`` suffixes are ignored.

script samples:

.. code-block:: python

    # file: pkgs/__cgi__/a/b/c/test1.py
    # test: http(s)://ROUTER/a/b/c/test1/d/e/f

    import slowdown.cgi

    def GET(rw):  # only GET requests are processed
        path1       = rw.environ['PATH_INFO']  # -> the original path
        path2       = rw.environ['locals.path_info']    # -> /d/e/f/
        script_name = rw.environ['locals.script_name']  # -> /a/b/c/test1
        return \
            rw.start_html_and_close(
                content='<html>It works!</html>'
            )

    def POST(rw):  # only POST requests are processed
        form = slowdown.cgi.Form(rw)
        return \
            rw.start_html_and_close(
                content=f'<html>{form}</html>'
            )

.. code-block:: python

    # file: pkgs/__cgi__/a/b/c/d/test2.py
    # test: http(s)://ROUTER/a/b/c/d/test2/e/f/g

    import slowdown.cgi

    # You can define a handler called HTTP to handle all
    # request methods just in one place.
    #
    # Be ware, don't define HTTP and GET/POST at the same time.
    def HTTP(rw):
        path_info   = rw.environ['locals.path_info'  ]  # -> /e/f/g
        script_name = rw.environ['locals.script_name']  # -> /a/b/c/d/test2
        if 'GET' == rw.environ['REQUEST_METHOD']:
            return \
                rw.start_html_and_close(
                    content='<html>It works!</html>'
                )
        elif 'POST' == rw.environ['REQUEST_METHOD']:
            form = slowdown.cgi.Form(rw)
            return \
                rw.start_html_and_close(
                    content=f'<html>{form}</html>'
                )
        else:
            return rw.method_not_allowed()


Script initialization
^^^^^^^^^^^^^^^^^^^^^

The script can be loaded by multiple :py:class:`slowdown.mapfs.Mapfs`
handlers, and **initialize(mapfs)** will be called several times
independently.

.. code-block:: python

    # The first time the script is loaded, the "initialize(mapfs)" function
    # of the script is executed.
    def initialize(mapfs_):
        global application
        global mapfs
        application = mapfs_.application
        mapfs       = mapfs_

    def GET(rw):
        # Call `__cgi__/a.html.py`
        return mapfs.load_script('a.html').GET(rw)


Calling another cgi script
^^^^^^^^^^^^^^^^^^^^^^^^^^

The :py:class:`~slowdown.__main__.Application` 's **modules** attribute
holds the module specified in the ``<modules>`` section of the configuration. And you can get the script module under the ``__cgi__``
folder through :py:class:`~slowdown.mapfs.Mapfs` 's **load_script(PATH)**
method.

.. code-block:: python

    def GET(rw):
        # Call `MY/PACKAGE/__cgi__/a.html.py:GET`
        mapfsobj = rw.application.modules['MY.PACKAGE'].handler
        mapfsobj.load_script('a.html').GET(rw)

.. code-block:: python

    def GET(rw):
        # Transfer to another package
        rw.application.modules['MY.PACKAGE'].handler(rw)

Access the matching configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If you want to know which configuration match this request, you can access
the :py:class:`~slowdown.__main__.HTTPRWPair` object's **match** attribute.

.. code-block:: python

    def GET(rw):
        match = rw.match
        host_section = match.host_section  # HostSection object
        host_section.section  # Original ZConfig section object
        path_section = match.path_section  # PathSection object
        path_section.section  # Original ZConfig section object


Error log
^^^^^^^^^

.. code-block:: python

    def GET(rw):
        rw.errorlog.debug(msg)
        rw.errorlog.info(msg)
        rw.errorlog.warning(msg)
        rw.errorlog.error(msg)
        rw.errorlog.critical(msg)


The HTTPRWPair object
---------------------

The script accepts :py:class:`~slowdown.__main__.HTTPRWPair` object
(inherited from :py:class:`slowdown.http.File` ) as the only argument.
In general, the :py:class:`~slowdown.__main__.HTTPRWPair` object is
sometimes called **rw** for short.


HTTP Headers
^^^^^^^^^^^^

The script can access http headers by reading the **rw.environ** dict.

.. envvar:: locals.path_info

    The **router** sets the path matched by the named group to the
    envirment variable ``locals.path_info`` .

    In most cases, when the **rw** object comes from a
    :py:class:`~slowdown.mapfs.Mapfs` dispatcher, the path after the script
    name is set to the envirment variable ``locals.path_info`` .

.. envvar:: locals.script_name

    The name of the script that is in use.

.. envvar:: REMOTE_ADDR

    The IP address of the client.

.. envvar:: REMOTE_PORT

    The port of the remote client.

.. envvar:: CONTENT_TYPE

    The `Content-Type` header.

.. envvar:: REQUEST_URI

    Full URI of the request.

.. envvar:: REQUEST_METHOD

    The method of the request, usually **GET** and **POST** , etc.

.. envvar:: PATH_INFO

    The original path.

.. envvar:: SCRIPT_NAME

    The originall script name. Always empty.

.. envvar:: QUERY_STRING

    The query string contained by the URL.

.. envvar:: HTTP_*

    Other HTTP headers. See `RFC 3875`__

.. note::

    Always use ``locals.path_info`` instead of ``PATH_INFO`` unless you
    need access to the original path.

__ https://tools.ietf.org/html/rfc3875


Reading from the POST content
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

- :py:meth:`slowdown.__main__.HTTPRWPair.read`
- :py:meth:`slowdown.__main__.HTTPRWPair.readline`

Example:

.. code-block:: python

    def POST(rw):
        size = rw.readline()
        data = rw.read(size)
        rw.send_response_and_close(
             status='200 OK',
            headers=[('Content-Type', 'text/html')],
            content='<html>OK</html>'
        )

.. note::

    The POST content must be read completely in order to respond further.


Streaming responses
^^^^^^^^^^^^^^^^^^^

- :py:meth:`slowdown.__main__.HTTPRWPair.start_response`
- :py:meth:`slowdown.__main__.HTTPRWPair.start_chunked`
- :py:meth:`slowdown.__main__.HTTPRWPair.write`
- :py:meth:`slowdown.__main__.HTTPRWPair.close`

Example:

.. code-block:: python

    def GET(rw):
        rw.start_response(
             status='200 OK',
            header=[('Content-Type', 'text/html')]
        )
        rw.write('<html>')
        rw.write('Hello, World!')
        rw.write('</html>')
        rw.close()


Quick response utils
^^^^^^^^^^^^^^^^^^^^

- :py:meth:`slowdown.__main__.HTTPRWPair.send_response_and_close`
- :py:meth:`slowdown.__main__.HTTPRWPair.send_html_and_close`
- :py:meth:`slowdown.__main__.HTTPRWPair.not_modified`
- :py:meth:`slowdown.__main__.HTTPRWPair.bad_request`
- :py:meth:`slowdown.__main__.HTTPRWPair.forbidden`
- :py:meth:`slowdown.__main__.HTTPRWPair.not_found`
- :py:meth:`slowdown.__main__.HTTPRWPair.method_not_allowed`
- :py:meth:`slowdown.__main__.HTTPRWPair.request_entity_too_large`
- :py:meth:`slowdown.__main__.HTTPRWPair.request_uri_too_large`
- :py:meth:`slowdown.__main__.HTTPRWPair.internal_server_error`
- :py:meth:`slowdown.__main__.HTTPRWPair.multiple_choices`
- :py:meth:`slowdown.__main__.HTTPRWPair.moved_permanently`
- :py:meth:`slowdown.__main__.HTTPRWPair.found`
- :py:meth:`slowdown.__main__.HTTPRWPair.see_other`
- :py:meth:`slowdown.__main__.HTTPRWPair.temporary_redirect`

Example:

.. code-block:: python

    def GET(rw):
        return rw.not_found()


Cookies
^^^^^^^

Cookies can be readed by accessing the **cookie** attribute of the
:py:attr:`slowdown.__main__.HTTPRWPair` object .

.. code-block:: python

    # using cookies

    import http.cookies

    def GET(rw):
        # get cookies
        # `None` will be returned if there are no cookies exists.
        cookie = rw.cookie  # `http.cookies.SimpleCookie` object

        # set cookies
        new_cookie = http.cookies.SimpleCookie()
        new_cookie['key'] = 'value'
            rw.send_html_and_close(
                content='<html>OK</html>',
                cookie=new_cookie
            )


CGI
---

The :py:mod:`slowdown.cgi` module provides CGI protocol support.


Form
^^^^

:py:class:`slowdown.cgi.Form` is a CGI form parser.

    >>> form = \
    ...     slowdown.cgi.Form(
    ...         rw,             # the incoming `HTTPRWPair` object.
    ...
    ...         max_size=10240  # the length of the http content containing
    ...                         # the CGI form data should be less than
    ...                         # `max_size` (bytes).
    ... )
    >>> form['checkboxA']
    'a'
    >>> # If more than one form variable comes with the same name,
    >>> # a list is returned.
    >>> form['checkboxB']
    ['a', 'b', 'c']


Upload files
^^^^^^^^^^^^

:py:func:`slowdown.cgi.multipart` can be used to handle file uploads.

.. code-block:: python

    import slowdown.cgi

    def POST(rw):
        # The CGI message must be read completely in order to
        # respond further, so use 'for .. in' to ensure that
        # no parts are unprocessed.
        for part in \
            slowdown.cgi.multipart(
                rw,  # the incoming `slowdown.__main__.HTTPRWPair` object

                # Uploaded files always store their binary filenames in
                # multi-parts heads. Those filenames require an encoding
                # to convert to strings.
                filename_encoding='utf-8'  # the default is 'iso8859-1'
            ):
            # The reading of the current part must be completed
            # before the next part.
            if part.filename is None:  # ordinary form variable
                print (f'key  : {part.name  }')
                print (f'value: {part.read()}')
            else:  # file upload
                with open(part.filename, 'w') as file_out:
                    while True:
                        data = part.read(8192)
                        file_out.write(data)
                        if not data:
                            break
