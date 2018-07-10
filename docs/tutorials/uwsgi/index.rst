.. _uwsgi_tutorial:

Running a CookieCutter :app:`Pyramid` Application under ``uWSGI``
=================================================================

:term:`mod_wsgi` is an Apache module developed by Graham Dumpleton.
It allows :term:`WSGI` programs to be served using the Apache web
server.

This guide will outline broad steps that can be used to get a :app:`Pyramid`
application running under Nginx via ``uWSGI``.  This particular tutorial
was developed under Ubuntu 16.04, but the instructions should be largely
the same for all systems, delta specific path information for commands and
files.


#.  Install prerequisites

    .. code-block:: bash

       $ sudo apt install -y uwsgi-core uwsgi-plugin-python3 python3-cookiecutter \
         python3-pip nginx
       $ cookiecutter gh:Pylons/pyramid-cookiecutter-starter --checkout master


#.  Create a :app:`Pyramid` application. For this tutorial we'll use the
    ``starter`` :term:`cookiecutter`. See :ref:`project_narr` for more
    in-depth information about creating a new project.

    .. code-block:: bash

       $ cd
       $ cookiecutter gh:Pylons/pyramid-cookiecutter-starter --checkout master

    If prompted for the first item, accept the default ``yes`` by hitting return.

    .. code-block:: text

        You've cloned ~/.cookiecutters/pyramid-cookiecutter-starter before.
        Is it okay to delete and re-clone it? [yes]: yes
        project_name [Pyramid Scaffold]: myproject
        repo_name [myproject]: myproject
        Select template_language:
        1 - jinja2
        2 - chameleon
        3 - mako
        Choose from 1, 2, 3 [1]: 1

#.  Create a :term:`virtual environment` which we'll use to install our
    application.

    .. code-block:: bash

       $ cd myproject
       $ python3 -m venv env

#.  Install your :app:`Pyramid` application and its dependencies.

    .. code-block:: bash

       $ env/bin/pip install -e .

#.  Within the project directory (``~/myproject``), create a script
    named ``wsgi.py``.  Give it these contents:

    .. code-block:: python

        # Adapted from PServeCommand.run in site-packages/pyramid/scripts/pserve.py
        from pyramid.scripts.common import get_config_loader
        app_name    = 'main'
        config_vars = {}
        config_uri  = 'production.ini'

        loader = get_config_loader(config_uri)
        loader.setup_logging(config_vars)
        app = loader.get_wsgi_app(app_name, config_vars)


.
.
.
.
!!! Maybe app_name should point to [uwsgi] instead of [:main]...??
.
.
.
.

    :ref:`config_uri` is the project configuration file name.  It's best to use
    the ``production.ini`` file provided by your cookiecutter, as it contains
    settings appropriate for production.  :ref:`app_name` is the name of the section
    within the ``.ini`` file that should be loaded by ``uWSGI``.  The
    assignment to the name ``app`` is important: we will reference ``app`` and
    the name of the file, ``wsgi`` when we invoke uWSGI.

    The call to :func:`loader.setup_logging` initializes the standard
    library's `logging` module to allow logging within your application.
    See :ref:`logging_config`.


#   Create a new directory at ``~/myproject/tmp`` to house a pidfile and a unix
    socket.  However, you'll need to make sure that *two* users have access to
    change into the ``~/myproject/tmp`` directory: your current user (mine is
    ``ubuntu`` and the user that Nginx will run as often named ``www-data`` or
    ``nginx``).

#   Invoke uWSGI.


    .. code-block:: bash
      # 1. Invoke as sudo so you can masquerade as the users specfied in --uid and --gid
      # 2. Change permissions on socket to at least 020 so that in combination
      #    with "--gid www-data", Nginx will be able to write to it after
      #    uWSGI creates it
      # 3. Mount the path "/" on the symbol "app" found in the file wsgi.py

      cd ~/myproject
      sudo uwsgi \                          # See note 1 above
        --chmod-socket=020 \                # See note 2 above
        --enable-threads \                  # Execute threads that are in your app
        --plugin=python3 \                  # Use the python3 plugin
        -s ~/myproject/tmp/myproject.sock \ # Where to put the unix socket
        --manage-script-name \              #
        --mount /=wsgi:app \                # See note 3 above
        --uid ubuntu \                      # masquerade as the ubuntu user
        --gid www-data \                    # masquerade as the www-data group
        --virtualenv env                    # Use packages installed in your venv

#   Verify that the output of the previous step includes a line that looks approximately like this:
    .. code-block:: bash

       WSGI app 0 (mountpoint='/') ready in 1 seconds on interpreter 0x5615894a69a0 pid: 8827 (default app)

    If any errors occurred, you will need to correct them. If you get a
    ``callable not found or import error``, make sure you your ``--mount
    /=wsgi:app`` matches the ``app`` symbol in the ``wsgi.py`` file. An import
    error that looks like ``ImportError: No module named 'wsgi'`` probably
    indicates a mismatch in your --mount arguments. Any other import errors
    probably means that the package it's failing to import either is not
    installed or is not accessible by the user. That's why we chose to
    masquerade as the normal user that you log in as, so you would for sure
    have access to installed packages.

#.  Add a new file at ``/etc/nginx/sites-enabled/myproject.conf`` with
    the following contents. Also change any occurrences of the word `ubuntu`
    to your actual username.

    .. code-block:: nginx

       server{
        server_name _;

        root /home/ubuntu/myproject/;

        location /  {
          include uwsgi_params;
          # The socket location must match that used by uWSGI
          uwsgi_pass unix:/home/ubuntu/myproject/tmp/myproject.sock;
        }

      }


#.  Reload Nginx

    .. code-block:: bash

       $ sudo nginx -s reload

#.  Visit ``http://localhost`` in a browser.  You should see the
    sample application rendered in your browser.

:term:`uWSGI` has many knobs and a great variety of deployment modes. This
is just one representation of how you might use it to serve up a CookieCutter :app:`Pyramid`
application.  See the `uWSGI documentation
<https://uwsgi-docs.readthedocs.io/en/latest/>`
for more in-depth configuration information.
