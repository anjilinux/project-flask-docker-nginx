Flask by Example – Setting up Postgres, SQLAlchemy, and Alembic
by Real Python 114 Comments databases devops flask web-dev
Tweet Share Email

Table of Contents

    Install Requirements
    Update Configuration
    Data Model
    Local Migration
    Remote Migration
    Conclusion

Remove ads

In this part we’re going to set up a Postgres database to store the results of our word counts as well as SQLAlchemy, an Object Relational Mapper, and Alembic to handle database migrations.

Free Bonus: Click here to get access to a free Flask + Python video tutorial that shows you how to build Flask web app, step-by-step.

Updates:

    02/09/2020: Upgraded to Python version 3.8.1 as well as the latest versions of Psycopg2, Flask-SQLAlchemy, and Flask-Migrate. See below for details. Explicitly install and use Flask-Script due to change of Flask-Migrate internal interface.
    03/22/2016: Upgraded to Python version 3.5.1 as well as the latest versions of Psycopg2, Flask-SQLAlchemy, and Flask-Migrate. See below for details.
    02/22/2015: Added Python 3 support.

Remember: Here’s what we’re building - A Flask app that calculates word-frequency pairs based on the text from a given URL.

    Part One: Set up a local development environment and then deploy both a staging and a production environment on Heroku.
    Part Two: Set up a PostgreSQL database along with SQLAlchemy and Alembic to handle migrations. (current)
    Part Three: Add in the back-end logic to scrape and then process the word counts from a webpage using the requests, BeautifulSoup, and Natural Language Toolkit (NLTK) libraries.
    Part Four: Implement a Redis task queue to handle the text processing.
    Part Five: Set up Angular on the front-end to continuously poll the back-end to see if the request is done processing.
    Part Six: Push to the staging server on Heroku - setting up Redis and detailing how to run two processes (web and worker) on a single Dyno.
    Part Seven: Update the front-end to make it more user-friendly.
    Part Eight: Create a custom Angular Directive to display a frequency distribution chart using JavaScript and D3.

Need the code? Grab it from the repo.
Install Requirements

Tools used in this part:

    PostgreSQL (11.6)
    Psycopg2 (2.8.4) - a Python adapter for Postgres
    Flask-SQLAlchemy (2.4.1) - Flask extension that provides SQLAlchemy support
    Flask-Migrate (2.5.2) - extension that supports SQLAlchemy database migrations via Alembic

To get started, install Postgres on your local computer, if you don’t have it already. Since Heroku uses Postgres, it will be good for us to develop locally on the same database. If you don’t have Postgres installed, Postgres.app is an easy way to get up and running for Mac OS X users. Consult the download page for more info.

Once you have Postgres installed and running, create a database called wordcount_dev to use as our local development database:

$ psql
# create database wordcount_dev;
CREATE DATABASE
# \q

In order to use our newly created database within the Flask app we to need to install a few things:

$ cd flask-by-example

    cding into the directory should activate the virtual environment and set the environment variables found in the .env file via autoenv, which we set up in part 1.

$ python -m pip install psycopg2==2.8.4 Flask-SQLAlchemy===2.4.1 Flask-Migrate==2.5.2
$ python -m pip freeze > requirements.txt

    If you’re on OS X and having trouble installing psycopg2 check out this Stack Overflow article.

    You may need to install psycopg2-binary instead of psycopg2 if your installation fails.

Remove ads
Update Configuration

Add SQLALCHEMY_DATABASE_URI field to the Config() class in your config.py file to set your app to use the newly created database in development (local), staging, and production:

import os

class Config(object):
    ...
    SQLALCHEMY_DATABASE_URI = os.environ['DATABASE_URL']

Your config.py file should now look like this:

import os
basedir = os.path.abspath(os.path.dirname(__file__))


class Config(object):
    DEBUG = False
    TESTING = False
    CSRF_ENABLED = True
    SECRET_KEY = 'this-really-needs-to-be-changed'
    SQLALCHEMY_DATABASE_URI = os.environ['DATABASE_URL']


class ProductionConfig(Config):
    DEBUG = False


class StagingConfig(Config):
    DEVELOPMENT = True
    DEBUG = True


class DevelopmentConfig(Config):
    DEVELOPMENT = True
    DEBUG = True


class TestingConfig(Config):
    TESTING = True

Now when our config is loaded into our app the appropriate database will be connected to it as well.

Similar to how we added an environment variable in the last post, we are going to add a DATABASE_URL variable. Run this in the terminal:

$ export DATABASE_URL="postgresql:///wordcount_dev"

And then add that line into your .env file.

In your app.py file import SQLAlchemy and connect to the database:

from flask import Flask
from flask_sqlalchemy import SQLAlchemy
import os


app = Flask(__name__)
app.config.from_object(os.environ['APP_SETTINGS'])
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

from models import Result


@app.route('/')
def hello():
    return "Hello World!"


@app.route('/<name>')
def hello_name(name):
    return "Hello {}!".format(name)


if __name__ == '__main__':
    app.run()

Data Model

Set up a basic model by adding a models.py file:

from app import db
from sqlalchemy.dialects.postgresql import JSON


class Result(db.Model):
    __tablename__ = 'results'

    id = db.Column(db.Integer, primary_key=True)
    url = db.Column(db.String())
    result_all = db.Column(JSON)
    result_no_stop_words = db.Column(JSON)

    def __init__(self, url, result_all, result_no_stop_words):
        self.url = url
        self.result_all = result_all
        self.result_no_stop_words = result_no_stop_words

    def __repr__(self):
        return '<id {}>'.format(self.id)

Here we created a table to store the results of the word counts.

We first import the database connection that we created in our app.py file as well as JSON from SQLAlchemy’s PostgreSQL dialects. JSON columns are fairly new to Postgres and are not available in every database supported by SQLAlchemy so we need to import it specifically.

Next we created a Result() class and assigned it a table name of results. We then set the attributes that we want to store for a result-

    the id of the result we stored
    the url that we counted the words from
    a full list of words that we counted
    a list of words that we counted minus stop words (more on this later)

We then created an __init__() method that will run the first time we create a new result and, finally, a __repr__() method to represent the object when we query for it.
Local Migration

We are going to use Alembic, which is part of Flask-Migrate, to manage database migrations to update a database’s schema.

    Note: Flask-Migrate makes use of Flasks new CLI tool. However, this article uses the interface provided by Flask-Script, which was used before by Flask-Migrate. In order to use it, you need to install it via:

    $ python -m pip install Flask-Script==2.0.6
    $ python -m pip freeze > requirements.txt

Create a new file called manage.py:

import os
from flask_script import Manager
from flask_migrate import Migrate, MigrateCommand

from app import app, db


app.config.from_object(os.environ['APP_SETTINGS'])

migrate = Migrate(app, db)
manager = Manager(app)

manager.add_command('db', MigrateCommand)


if __name__ == '__main__':
    manager.run()

In order to use Flask-Migrate we imported Manager as well as Migrate and MigrateCommand to our manage.py file. We also imported app and db so we have access to them from within the script.

First, we set our config to get our environment - based on the environment variable - created a migrate instance, with app and db as the arguments, and set up a manager command to initialize a Manager instance for our app. Finally, we added the db command to the manager so that we can run the migrations from the command line.

In order to run the migrations initialize Alembic:

$ python manage.py db init
  Creating directory /flask-by-example/migrations ... done
  Creating directory /flask-by-example/migrations/versions ... done
  Generating /flask-by-example/migrations/alembic.ini ... done
  Generating /flask-by-example/migrations/env.py ... done
  Generating /flask-by-example/migrations/README ... done
  Generating /flask-by-example/migrations/script.py.mako ... done
  Please edit configuration/connection/logging settings in
  '/flask-by-example/migrations/alembic.ini' before proceeding.

After you run the database initialization you will see a new folder called “migrations” in the project. This holds the setup necessary for Alembic to run migrations against the project. Inside of “migrations” you will see that it has a folder called “versions”, which will contain the migration scripts as they are created.

Let’s create our first migration by running the migrate command.

$ python manage.py db migrate
  INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
  INFO  [alembic.runtime.migration] Will assume transactional DDL.
  INFO  [alembic.autogenerate.compare] Detected added table 'results'
    Generating /flask-by-example/migrations/versions/63dba2060f71_.py
    ... done

Now you’ll notice in your “versions” folder there is a migration file. This file is auto-generated by Alembic based on the model. You could generate (or edit) this file yourself; however, for most cases the auto-generated file will do.

Now we’ll apply the upgrades to the database using the db upgrade command:

$ python manage.py db upgrade
  INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
  INFO  [alembic.runtime.migration] Will assume transactional DDL.
  INFO  [alembic.runtime.migration] Running upgrade  -> 63dba2060f71, empty message

The database is now ready for us to use in our app:

$ psql
# \c wordcount_dev
You are now connected to database "wordcount_dev" as user "michaelherman".
# \dt

                List of relations
 Schema |      Name       | Type  |     Owner
--------+-----------------+-------+---------------
 public | alembic_version | table | michaelherman
 public | results         | table | michaelherman
(2 rows)

# \d results
                                     Table "public.results"
        Column        |       Type        |                      Modifiers
----------------------+-------------------+------------------------------------------------------
 id                   | integer           | not null default nextval('results_id_seq'::regclass)
 url                  | character varying |
 result_all           | json              |
 result_no_stop_words | json              |
Indexes:
    "results_pkey" PRIMARY KEY, btree (id)

Remove ads
Remote Migration

Finally, let’s apply the migrations to the databases on Heroku. First, though, we need to add the details of the staging and production databases to the config.py file.

To check if we have a database set up on the staging server run:

$ heroku config --app wordcount-stage
=== wordcount-stage Config Vars
APP_SETTINGS: config.StagingConfig

    Make sure to replace wordcount-stage with the name of your staging app.

Since we don’t see a database environment variable, we need to add the Postgres addon to the staging server. To do so, run the following command:

$ heroku addons:create heroku-postgresql:hobby-dev --app wordcount-stage
  Creating postgresql-cubic-86416... done, (free)
  Adding postgresql-cubic-86416 to wordcount-stage... done
  Setting DATABASE_URL and restarting wordcount-stage... done, v8
  Database has been created and is available
   ! This database is empty. If upgrading, you can transfer
   ! data from another database with pg:copy
  Use `heroku addons:docs heroku-postgresql` to view documentation.

    hobby-dev is the free tier of the Heroku Postgres addon.

Now when we run heroku config --app wordcount-stage again we should see the connection settings for the database:

=== wordcount-stage Config Vars
APP_SETTINGS: config.StagingConfig
DATABASE_URL: postgres://azrqiefezenfrg:Zti5fjSyeyFgoc-U-yXnPrXHQv@ec2-54-225-151-64.compute-1.amazonaws.com:5432/d2kio2ubc804p7

Next we need to commit the changes that you’ve made to git and push to your staging server:

$ git push stage master

Run the migrations that we created to migrate our staging database by using the heroku run command:

$ heroku run python manage.py db upgrade --app wordcount-stage
  Running python manage.py db upgrade on wordcount-stage... up, run.5677
  INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
  INFO  [alembic.runtime.migration] Will assume transactional DDL.
  INFO  [alembic.runtime.migration] Running upgrade  -> 63dba2060f71, empty message

    Notice how we only ran the upgrade, not the init or migrate commands like before. We already have our migration file set up and ready to go; we just need to apply it against the Heroku database.

Let’s now do the same for production.

    Set up a database for your production app on Heroku, just like you did for staging: heroku addons:create heroku-postgresql:hobby-dev --app wordcount-pro
    Push your changes to your production site: git push pro master Notice how you don’t have to make any changes to the config file - it’s setting the database based on the newly created DATABASE_URL environment variable.
    Apply the migrations: heroku run python manage.py db upgrade --app wordcount-pro

Now both our staging and production sites have their databases set up and are migrated - and ready to go!




=============================================================

**Author: Facundo Padilla**

**Social networks:**

- Facebook: https://www.facebook.com/facundodeveloper

- Instagram: https://www.instagram.com/facundodeveloper

- LinkedIn: https://www.linkedin.com/in/facundopadilla

- Udemy: https://www.udemy.com/user/facundo-padilla/

# **Version:**

- 1.0: 
	- 01/02/2021 (DD-MM-YYYY)
	-  02/01/2021 (MM-DD-YYYY)
	-  2021/02/01 (YYYY-MM-DDDD)

# What is this?:

- This Flask project is reusable and also an example of how to merge *Flask, Docker, Nginx, Gunicorn, MySQL, new: Flask-RESTX,  Factory Method design pattern,* and other optional dependencies such as D*ynaconf, Marshmallow, SQLAlchemy, Faker, PyMySQL, Pytest, etc...* which are installed inside the virtual environment "*env_flask*".

# How to use it?

- Easy, if you have Docker Compose installed, just run the following command inside the project directory: *docker-compose up*

- You can also use the commands created inside the "Makefile" file, simply by executing the "make" command; make [OPTION] - Example: *make full-start*

- Once you execute any of these commands, it starts to create the containers to work, they are already linked so you should have no problems to get it to work
- Once you have finished running and creating the containers, simply go to http://localhost:80/

# Initialize application:

- After you have executed the commands like *docker-compose up, make full-start* or whatever you have created in the Makefile or used, it automatically creates the tables and works without problems, the tables are created in the predefined database with the name "*flask_api*", if you want to change the name of this database just go to the file "docker-compose.yaml" and change the environment variable "*MYSQL_DATABASE*". 

- And that's it, once the tables are created, you only have to go to http://localhost:80 (Nginx)

# "Hot-Reloading":

- You could use Watchman to do hot-reaload, but it gives some problems/bugs, so you can directly work with the virtual environment "*env_flask*" and start working there to simulate a hot-reload, also, it is not recommended that a container contains integrated hot-realoading if it is going to be used for hot-reloading.

- **How to activate env_flask**:

	- **Linux / PowerShell**:
		
		1) Run: pip3 install virtualenv && python3 -m virtualenv env_flask
		2) Enter directory *../env_flask/bin/activate* and execute: **source activate** (in the terminal)
		3) Return to the path where **run_debug.py** is found, and run: pip install -r requirements.txt
		4) Run the following commands:
			- *export FLASK_APP=run_debug.py*
			- *flask run -h 0.0.0.0 -p 5001*(the host and port can be changed to the one you want)
			or
			- *python run_debug.py*

	- **Windows:**
		
		1) Run: pip3 install virtualenv && python3 -m virtualenv env_flask
		2) Enter directory ../env_flask/bin/activate and execute: **activate** in CMD
		3) Return to the path where **run_debug.py** is found, and run: pip install -r requirements.txt
		4) Run the following commands:
			- *set FLASK_APP=run_debug.py*
			- *flask run -h 0.0.0.0 -p 5001* (the host and port can be changed to the one you want)
			or
			- *python run_debug.py*
			
**ATTENTION:** in the Dynaconf configuration file (*settings.toml*), when running the *run_debug.py* , the MySQL connection points to *localhost* and port *3307*, if it is going to be uploaded to production, remove the "expose" option from the docker-file.yaml file.

- **Deactivate the virtual environment:**

	- Execute *deactivate* in the terminal or CMD

# Settings:

- **docker-compose.yaml:**

	- **MySQL (db)**:

		- MYSQL_USER: the user name you want to customize (the root user is default and cannot be deleted)

		- MYSQL_PASSWORD= the password of the user (not the root)

		- MYSQL_DATABASE= name of the database you want

		- MYSQL_ROOT_PASSWORD= root user password

		- More documentation: https://hub.docker.com/_/mysql

	- **Flask (flask_app)**:

		- PYTHONBUFFERED= by default leave it set to 1, it is used to display Python logs.

		- ENVVAR_PREFIX_FOR_DYNACONF= the name of our module

		- ENVVAR_FOR_DYNACONF= file name with extension ".toml", Dynaconf supports several others: https://dynaconf.readthedocs.io/en/docs_223/guides/examples.html

		- FLASK_APP= the name of the Python file to be executed by the server

		- FLASK_RUN_HOST= the host address of the application, defaulting to 0.0.0.0

		- FLASK_DEBUG= WARNING!, 1 enables debug, 0 for production.

		- command: Gunicorn command to execute

		- More documentation: https://hub.docker.com/_/python

	- **Nginx (nginx lol)**:

		- Read the docs: https://hub.docker.com/_/nginx

- **settings.toml (Dynaconf file):**
	- [default]
		- SQLALCHEMY_TRACK_MODIFICATIONS = "False" (is a setting that is no longer used, leave it at false)
	- [development]:
		> When you switch to debug mode, the db name becomes localhost and port 3307.
		- DEBUG = "True"
		- SQLALCHEMY_DATABASE_URI = "mysql+pymysql://root:password@localhost:port/db_name"
		
	- [production]:
		> When in production, mostly in the docker, it keeps running this configuration
		- DEBUG = "False"
		- SQLALCHEMY_DATABASE_URI = "mysql+pymysql://root:password@db_container_name/db_name"

# python run.py vs flask run:

- The difference between using Python to execute "run.py" and using "flask run", is that when you have the file "\_\_module__.py" in the project, Python automatically adds it to the "syspath", in the case of "flask run", this does not happen, even if there is the "\_\_init__.py" and the "\_\_module__.py" , so to avoid problems when working with flask run, in each \_\_init__.py file the following code fragment is added:

	    import sys
	    sys.path.append(".")
