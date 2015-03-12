# Deploying a Django Application

In this tutorial we're going to show you how to deploy a Django application on [CloudKilat]. You can find the [source code on Github][example-app] and check out the [Python buildpack][python buildpack] for supported features. The application follows the official [Django tutorial] and allows you to create, use and manage simple polls.

## The Django Application Explained

### Get the App

First, clone the Django application from our repository on Github:

~~~bash
$ git clone https://github.com/cloudControl/python-django-example-app.git
$ cd python-django-example-app
~~~

### Dependency Tracking

The Python buildpack tracks dependencies via [pip] and the `requirements.txt` file. It needs to be placed in the root directory of your repository. The example app specifies [Django][django], [MySQL driver][mysql-driver] and [gunicorn] as dependencies. The one you cloned as part of the example app looks like this:

~~~
Django==1.4.3
gunicorn==0.17.2
MySQL-python==1.2.4
~~~

### Production Server

In a production environment you normally don't want to use the development server. We have decided to use gunicorn for this purpose. To do so we had to include it in the list of installed applications (`INSTALLED_APPS` in `mysite/settings.py`):

~~~python
INSTALLED_APPS = (
    ...
    'gunicorn'
    ...
)
~~~

### Process Type Definition

CloudKilat uses a [Procfile] to know how to start your processes. The example code already includes a file called Procfile at the top level of your repository. It looks like this:

~~~
web: python manage.py run_gunicorn -b 0.0.0.0:$PORT
~~~

Left from the colon we specified the **required** process type called `web` followed by the command that starts the app and listens on the port specified by the environment variable `$PORT`.

### Production Database

The original tutorial application uses SQLite as the database in all environments, even the production one. It is not possible to use a SQLite database on CloudKilat because the filesystem is [not persistent][filesystem]. To use a database, you should choose a data storage Add-on from [the available addons][data-storage-addons].

In this tutorial we use the [Shared MySQL Add-on][mysqls]. Have a look at `mysite/settings.py` so you can find out how to [get the MySQL credentials][get-conf] provided by MySQLs Add-on:

~~~python
# Django Settings for mysite Project.

import os
import json

PROJECT_ROOT = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))

try:
    # production settings
    f = os.environ['CRED_FILE']
    db_data = json.load(open(f))['MYSQLS']

    db_config = {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': db_data['MYSQLS_DATABASE'],
        'USER': db_data['MYSQLS_USERNAME'],
        'PASSWORD': db_data['MYSQLS_PASSWORD'],
        'HOST': db_data['MYSQLS_HOSTNAME'],
        'PORT': db_data['MYSQLS_PORT'],
    }
except KeyError, IOError:
    # development/test settings:
    db_config = {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': '{0}/mysite.sqlite3'.format(PROJECT_ROOT),
    }
...
DATABASES = {
    'default': db_config,
}
...
~~~

## Pushing and Deploying your App

Choose a unique name to replace the `APP_NAME` placeholder for your application and create it on the CloudKilat platform: 

~~~bash
$ ironcliapp APP_NAME create python
~~~

Push your code to the application's repository, which triggers the deployment image build process:

~~~bash
$ ironcliapp APP_NAME/default push
Counting objects: 31, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (25/25), done.
Writing objects: 100% (31/31), 7.11 KiB, done.
Total 31 (delta 3), reused 24 (delta 0)

-----> Receiving push
-----> No runtime.txt provided; assuming python-2.7.3.
-----> Preparing Python runtime (python-2.7.3)
-----> Installing Distribute (0.6.36)
-----> Installing Pip (1.3.1)
-----> Installing dependencies using Pip (1.3.1)
       Downloading/unpacking Django==1.4.3 (from -r requirements.txt (line 1))
         Running setup.py egg_info for package Django
       ...
-----> Building image
-----> Uploading image (30M)

To ssh://APP_NAME@kilatiron.net/repository.git
 * [new branch]      master -> master
~~~

Add MySQLs Add-on with `free` plan to your deployment and deploy it:
~~~bash
$ ironcliapp APP_NAME/default addon.add mysqls.free
$ ironcliapp APP_NAME/default deploy
~~~

Finally, prepare the database using the [Run command][ssh-session] (when prompted create admin user):

~~~bash
$ ironcliapp APP_NAME/default run "python manage.py syncdb"
~~~

You can login to the admin console at `APP_NAME.kilatiron.net/admin`, create some polls and see them at `APP_NAME.kilatiron.net/polls`.

For additional information take a look at [Django Notes][django-notes] and other [python-specific documents][python-guides].

[django]: https://www.djangoproject.com/
[CloudKilat]: http://www.cloudkilat.com/
[CloudKilat]: http://www.cloudkilat.com/
[CloudKilat-doc-user]: /Platform%20Documentation.md/#user-accounts
[CloudKilat-doc-cmdline]: /Platform%20Documentation.md/#command-line-client-web-console-and-api
[Procfile]: /Platform%20Documentation.md/#buildpacks-and-the-procfile
[git]: https://help.github.com/articles/set-up-git
[filesystem]: /Platform%20Documentation.md/#non-persistent-filesystem
[data-storage-addons]: /Add-on%20Documentation/Data%20Storage
[mysqls]: /Add-on%20Documentation/Data%20Storage/MySQLs.md
[example-app]: https://github.com/cloudControl/python-django-example-app
[django-notes]: /Guides/Python/Django%20notes.md
[get-conf]: /Guides/Python/Add-on%20credentials.md
[Django tutorial]: https://docs.djangoproject.com/en/1.4/intro/tutorial01/
[python-guides]: /Guides/Python
[python buildpack]: https://github.com/cloudControl/buildpack-python
[pip]: http://www.pip-installer.org/
[gunicorn]: http://gunicorn.org/
[worker]: /Platform%20Documentation.md/#scheduled-jobs-and-background-workers
[db-commit]: https://github.com/cloudControl/python-django-example-app/commit/983f45e46ce0707476cec167ea062e19adcb53c9
[ssh-session]: /Platform%20Documentation.md/#secure-shell-ssh
[mysql-driver]: https://pypi.python.org/pypi/MySQL-python/1.2.4
