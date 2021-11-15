---
layout: post
title:  "Getting Started With Pants and Django (Part 1)"
---

After writing about [our experience with moving a mature Django app to
Pants](https://g-cassie.github.io/2021/10/02/django-pants.html), I began wondering how difficult it would be to start a
brand new Django project on Pants.  While it overall wasn't too great an undertaking to move our project over, it sure
would have saved time if we had been using Pants from the beginning.

Of course, if you want to get up and running with Django as fast as possible, adding in the complexity of a build
tool like Pants is going to slow you down.  My hypothesis is that the early benefits of Pants outweigh the initial
complexity bump.  In particular, I believe the following features of Pants can pay large dividends early in a project's
lifecycle.

 - out-of-the-box linting and syntax formatting
 - eliminating the need for a virtualenvs
 - fast test runs with out-of-the-box caching and parallelization
 - painless deployments with `.pex` files

To test this hypothesis, I decided to follow the [official django beginner
tutorial](https://docs.djangoproject.com/en/3.2/intro/tutorial01/) and see what was involved to get it running on
Pants.

If you are trying to follow this tutorial, checkout [the repo for this
tutorial](https://github.com/g-cassie/django-pants-tutorial). I have created each step as its own commit so you can
refer to the diffs to see exactly how it works.  If you encounter any trouble, the [Pants Slack
group](https://join.slack.com/t/pantsbuild/shared_invite/zt-d0uh0mok-RLvVosDiX6JDpvStH~bFBA) is an amazing community
with some of the most helpful people I have met in OSS.

## Step 1: Initial Repo Setup

To begin, we want to create a new directory, initialize a git repo, and add a gitignore for python.  We are going to be
following the django tutorial so we will name our project “django-tutorial”.

```
cd /path/to/my/python/projects
mkdir django-tutorial
cd django-tutorial

# initialize git repo and add a standard python gitignore
git init
curl -L -O https://raw.githubusercontent.com/github/gitignore/master/Python.gitignore
mv Python.gitignore .gitignore
```

Now add the following Pants specific lines to end of your gitignore

```
# .gitignore
...
# Pants workspace files
/.pants.d/
/dist/
/.pids
/.pants.workdir.file_lock*
```

## Step 2: Setup Pants

Now we are going to setup Pants. To do this, I am going to loosely follow the [Getting Started section of their
docs](https://www.pantsbuild.org/docs/installation).

First, create a file named `pants.toml`.  At the root of your new repo. This will be the central place for all of your Pants
configuration.

```
# pants.toml
[GLOBAL]
pants_version = "2.8.0rc5"
backend_packages = ["pants.backend.python"]

[python-infer]
string_imports = true
```

This sets up Pants for a python project and also enables the string import feature which is necessary for Pants to play
nicely with Django settings files.

Now you are ready to download the `pants` script. This will be committed to your repo and will be the entrypoint for
Pants going forward.

```
curl -L -O https://static.pantsbuild.org/setup/pants && chmod +x ./pants
./pants --version
```


## Step 3: Setup 3rdparty dependencies.
One of the features of Pants is that it can manage all of your third party dependencies. This allows you to enforce
consistent third party dependency versions across all of your applications.  The biggest advantage, though is it can
build all third party dependencies into standalone pex files (python executable) which can be used to easily deploy
your code in a consistent way to production.

Pants is incredibly flexible with how you configure your repo but in our case we are going to create a subdirectory
called `3rdparty` where we will keep a list of all our requirements.  We will then create another directory called
`src` where we will keep our application code.  In each of these folders we will create BUILD files which provide
targets for Pants to use when building your project.

Create the following files with the contents indicated.

```
# 3rdparty/requirements.txt
Django==3.2
```

```
# 3rdparty/BUILD
python_requirements()
```

```
# src/BUILD
pex_binary(
    name="django-admin",
    dependencies=[
        "3rdparty:Django",
    ],
    script="django-admin",
)
```

The combination of these files ends up making an executable target called `django-admin` which will run the
django-admin script in the Django pypi module. That means that by running `./pants run src:django-admin`, Pants will
download Django from pypi, install it into an ephemeral virtual environment and run the django-admin script.

# Step 4: Setup the Django Project

Part 1 of the Django tutorial instructs you how to setup a django project.  We can roughly follow along,
however, because we are using Pants there are a few differences.

When the tutorial says to run `django-admin startproject mysite`, you will need to run the following:

```
./pants run src:django-admin -- startproject mysite
```

This causes Pants to invoke the django-admin binary target you created in the previous step.  The arguments after `--`
are passed as arguments to `django-admin`.

In our case, we want to keep all our source code nested under the `src/` prefix, so we will want to move the results of
the `startproject` call.
```
mv mysite/* src
```

Start project creates a `manage.py` script which becomes the entry point for many django tasks.  To run this with Pants
we will need to create a target for it. Add the following to `src/BUILD`

```
# src/BUILD
…
python_sources()

pex_binary(
    name="manage",
    entry_point="manage.py",
    dependencies=[
        "src/mysite:mysite",
    ],
)
```

This creates another binary target that can be invoked using `./pants run`.  You’ll notice that we are specifying
`src/mysite:mysite` as a dependency. This is telling Pants that when it builds this script to run, it needs to include
`src/mysite` in order for it to work correctly.

The only issue is, currently, Pants will not find any targets in `src/mysite`, because we haven’t created any.
To fix this, create `src/mysite/BUILD` as follows.

```
# src/mysite/BUILD
python_sources()
```

We should now be able to call manage.py like this:

```
./pants run src:manage
```

This should output the available commands from `manage.py`.

## Step 5: Create the Polls App

We can now use this `manage` binary target to create the polls app as directed in the tutorial. Just as before, we will
need to move it into the src directory after we create it.

```
./pants run src:manage -- startapp polls
mv polls src
```

We also need to create a BUILD file like we did with the `mysite` directory.  While we can do this manually, this is a
common task so Pants has a helper for this.

```
./pants tailor
```

This simple command will introspect your project and create BUILD files in all directories that it thinks need them with
the correct targets. Don’t worry, it won’t modify any of the BUILD files you have already.

You can now follow the “Write your first view” step verbatim by creating/modifying the following files:

```
# src/polls/views.py
from django.http import HttpResponse


def index(request):
    return HttpResponse("Hello, world. You're at the polls index.")
```

```
# src/polls/urls.py
from django.urls import path

from . import views

urlpatterns = [
    path('', views.index, name='index'),
]
```

```
# src/mysite/urls.py
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path('polls/', include('polls.urls')),
    path('admin/', admin.site.urls),
]
```

At this point in the tutorial, we are instructed to run the development server. We can do this with the following
command.

```
./pants run src:manage -- runserver --noreload
```

If you are following along, at this point you are probably looking at `ModuleNotFoundError: No module named ‘polls’`.
This is because, when we originally made the `manage` target, we did not specify `polls` as a dependency. This means
that when Pants builds this target, it completely excludes that directory. This might seem annoying in a small project
like this, but when you have a massive project with hundreds of thousands of lines of code, including every file in
every build becomes a big issue.

To fix this, we need to amend the `manage` target in `src/BUILD`.

```
# src/BUILD
pex_binary(
    name="manage",
    entry_point="manage.py",
    dependencies=[
        "src/mysite:mysite",
        "src/polls:polls",
    ],
)
```

Now we can run the server.  We are going to turn off autoreloading for now - in the next part of this tutorial we will
cover how to get this working with Pants.

```
./pants run src:manage -- runserver --noreload
```

Visit `http://localhost:8000/polls` in our browser and you should be able to see the view we wrote.

# Step 6: Models and Migrations

At this point, we are ready to start Part 2 of the Django tutorial. One of the features of Pants is that it builds and
executes targets in isolation. This means when your files are running, they will actually be running out of short lived
temporary directories.  This creates some slight wrinkles with commands that are trying to modify your source tree,
such as the django migration commands.

The first step to mitigate this is to hardcode the path to the SQLite database in settings.py (currently it relies on
`__file__` which will result in a SQLite DB being made in the temporary directory for the Pants run and then
immediately being torn down.

```
# settings.py
...
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": "/path/to/your/repo/db.sqlite3",
    }
}
...
```

When prompted to run migrate, use the following Pants equivalent

```
./pants run src:manage -- migrate
```

You should see `db.sqlite3` created at the root of your project.

Now, follow the instructions to create a new model and add the polls app to settings.py.

```
# src/polls/models.py
from django.db import models


class Question(models.Model):
    question_text = models.CharField(max_length=200)
    pub_date = models.DateTimeField('date published')


class Choice(models.Model):
    question = models.ForeignKey(Question, on_delete=models.CASCADE)
    choice_text = models.CharField(max_length=200)
    votes = models.IntegerField(default=0)
```

```
# src/mysite/settings.py
…
INSTALLED_APPS = [
    'polls.apps.PollsConfig',
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]
```

At this point we need to run `makemigrations`. Unfortunately there isn’t a great path for this on Pants as of writing.
What I found worked best was the following.

```
./pants run src:manage -- makemigrations --dry-run --verbosity 3 polls
```

This command prints the proposed migration files to stdout. You then need to write this output to
`src/polls/migrations/0001_initial.py`.

We also need to run `tailor` again to create a BUILD file in the `src/polls/migrations` directory so Pants can find it.
```
./pants tailor
```

And, because these migrations are not referenced anywhere in our code, we need to create an explicit dependency for the
`poll` library on `polls/migrations`.

```
# src/polls/BUILD
python_sources(
  dependencies=[
      "src/polls/migrations:migrations",
  ],
)
```
You should now be able to apply your new migrations.
```
./pants run src:manage -- migrate
```

## Step 7: Complete Django Tutorial Parts 2, 3 and 4

We can now complete the rest of Part 2 as well as Parts 3 and 4 with pretty much no adjustment.  Any time you are asked
to run `manage.py xyz` you will instead run `./pants run src:manage -- xyz`.

When the tutorial prompts you to create templates, create `polls/templates/polls/index.html`. If you run the server
at this point and try to access `http://127.0.0.1:8000/polls/` you will get an error: `TemplateDoesNotExist at /polls/`.
Once again, Pants is not bundling our new templates directory when running the manage target.  To prove this, we can
run the `./pants dependencies` command to see what is being included.
```
./pants dependencies src:managed
```

In the output, you will not see the templates directory. To solve this, we need to create a target for the
templates directory and then include it as a dependency of the `polls` target.

Normally in Pants, each directory represents a single target. This establishes a nice balance between the boilerplate
of BUILD files and keeping targets narrowly scoped (you could theoretically have one target that includes all the files
in your app but it would kind of defeat the whole purpose of Pants).  With templates, the benefits of narrow modules
are reduced (they are just static files that don't get evaluated unless explicitly used).  As a result, we are just
going to make a single target at the top of the templates directory to include all html files.

```
# src/polls/templates/BUILD
resources(
    sources=[
      "**/*.html",
    ],
)
```

Now we can add this as a dependency of the `polls` target.
```
# src/polls/BUILD
python_sources(
  dependencies=[
      "src/polls/migrations:migrations",
      "src/polls/templates:templates",
  ],
)
...
```

At this point, you should be able to run the server and visit `http://127.0.0.1:8000/polls/` successfully.

You can now complete the rest of Parts 3 and 4 of the tutorial without any adjustment.

## Step 8: A few quick wins from Pants

If you've been following along with these steps you may now be wondering why it's worth all this effort. So,
before we stop for the day, let's take a moment and take advantage of some of the best practices that Pants makes
trivial to implement: syntax formatting, dependency ordering and linting.

To do this, we just need to add some more backend packages to our pants config.

```
# pants.toml
[GLOBAL]
...
backend_packages = [
  "pants.backend.python",
  "pants.backend.python.lint.flake8",
  "pants.backend.python.lint.isort",
  "pants.backend.python.lint.black",
]
```

Now we can run the `lint` goal to see whether we are failing any of these new checks.
```
./pants lint ::
```

If you want to enforce these rules in CI, you simply need to add a step to run this goal.  In a large project,
you can get great performance on this by using the `--changed-since` flag to only lint files that have changed
since the provided gitref.
```
./pants lint --changed-since=main
```

In my case, `flake8` was very mad that all my lines were too long. I like longer lines so I added the following to my
`pants.toml`.

```
# pants.toml
...
[flake8]
args=[
  "--max-line-length=120"
]
```

To actually fix the problems being found by lint, we can use the `fmt` goal.
```
./pants fmt ::
```

This will format all your syntax with `black` and order your dependencies with `isort`.  Unfortunately you have to
solve your own `flake8` errors!

## That's it for this time.

At this point, we have completed the first four parts of the seven part Django tutorial. The next part deals with
testing. This is another area where Pants can really take things up a notch, giving us parallelized test runs, test
result caching and smart target selection for free.  I'll also make sure to cover how to get autoreload
working on the development server.  [Follow me on Twitter](https://twitter.com/gordon_cassie) for notifications about
the next post (I keep it pretty quiet otherwise).

And, as always, we are hiring at iManage Closing Folders. If you want to help us figure out how to build high
performing software development teams [check out the job postings](https://imanage.com/about/careers/) for our Toronto
office (remote friendly!).

Thanks to Benjy Weinberger for reviewing drafts of this post.
