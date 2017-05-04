installing airflow in a production environment (ubuntu) - a complete walkthrough
=======

author: Xia Wang

affiliation: Argo labs

date: 2017-04-21

## Introduction ##
Airflow is an open-source workflow management platform created by Maxime Beauchemin at airbnb in 2014. It provides a very intuitive way to establish dependency among different computing tasks, and to automatize the data pipeline activities. It is also shipped with a great webUI for manual control of the data pipeline, data profiling and other great features.

However, since airflow is still a relatively young software, there is still not many resources ([Tianlong](https://stlong0521.github.io/20161023%20-%20Airflow.html) and [Tianxia](http://blog.genesino.com/2016/05/airflow/) provided very good kickstart though) on the internet about how to install airflow on a production environment, and even less about the best practices in using it. This article aims at recording the problems and solutions I went through to successfully install and run airflow on an AWS EC2 ubuntu server. Without further ado, let's dive into the installation.
## First things first: install python and pip##
Assuming we have a clean slate ubuntu server. First we need to install `python` and the python package management tool `pip`.
Take installing python 2.7 for example:

`sudo apt-get install python-setuptools`

Then install pip:

`sudo apt-get install python-pip`

There 2 commands will give us the bare minimum to kickstart the airflow installation.

Note: if the default installed `pip` is not the up-to-date version, you might want to consider updating it:

`sudo pip install --upgrade pip`

## Assorted dishes: Install relational database (postgres) and configure the database##
Airflow is shipped with a sqlite database backend. But to be able to run the data pipeline on the webUI, we need to have a more powerful database backend, and configure the database so that airflow has access to it. We decided to install postgresql database for our purpose.

`sudo apt-get install postgresql postgresql-contrib`

So far as we know, the most recent versions of postgresql (8 and 9) don't have compatibility issues with airflow.

Now that we've installed the postgresql database, we need to create a database for airfow, and grant access to the EC2 user. To create a database for airflow, we need to access the postgresql command line tool `psql` as postgres' default superuser `postgres`:

`sudo -u postgres psql`

Then we will receive a psql prompt that looks like `postgres=#`.
We can type in sql queries to add a new user (ubuntu in our case), and grant it privileges to the database. 

	CREATE DATABASE airflow;
	CREATE ROLE ubuntu;
	GRANT ALL PRIVILEGES on database airflow to ubuntu;
	ALTER ROLE ubuntu SUPERUSER;
	ALTER ROLE ubuntu CREATEDB;
	GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO ubuntu;
	 

`psql -d airflow`

Type in

`\conninfo` 

will tell us the connection information.

One last thing we need to configure for the postgresql database is to change the settings in `pg_hba.conf`. Using the query:
`SHOW hba_file;`
will tell return the location of the `pg_hba.conf` file (it's likely in `/etc/postgresql/9.*/main/`). Open the file with a text editor (vi, emacs or nano), and change the ipv4 address to `0.0.0.0/0` and the ipv4 connection method from md5 (password) to `trust` if you don't want to use a password to connect to the database.
In the meantime, we also need to configure the `postgresql.conf` file to open the listen address to all ip addresses: 

`listen_addresses = '*'`.

## The main course: Install airflow, packages and dependencies; set up configurations##
Before installing, we can set up the default airflow home address:

`export AIRFLOW_HOME=~/airflow`

To install airflow and its packages is simply calling:

`sudo pip install airflow[foo,bar]`

 where foo, bar are package names separated by comma. Airflow also provide an option `all` to install all dependencies. We only chose to install the packages we thought useful (async, devel, celery, crypto, druid, gcp_api, jbdc, hdfs, hive, kerberos, ldap, password, postgres, qds, rabbitmq, s3, samba, slack), and ran into a few dependency issues. To be specific:

 - for the [cryptograph] package, we need to install `libssl-dev`: 

`sudo apt-get install libssl-dev` 

 - for the [kerbero] package, we need to install `libkrb5-dev`: 

`sudo apt-get install libkrb5-dev`

 - for the [hive] package, we need to install `libsasl2-dev`: 

`sudo apt-get install libsasl2-dev`

A general tip: if an error message arises during the installation, pay attention to which package failed the process, and try to install the dependency for that package, and try again.

After successfully installing airflow and packages, we should call 

`airflow initdb`

to set up the first-time configs.
An `airflow.cfg` file is generated in the airflow home directory. We should open it with a text editor, and change some configurations in the [core] section:

 - for the executor, we should use CeleryExecutor instead of SequentialExecutor if we want to run the pipeline in the webUI:

`executor = CeleryExecutor`

 - for the backend DB connection, we should pass along the connection info of the postgresql database `airflow` we just created:

`sql_alchemy_conn = postgresql+psycopg2://ubuntu@localhost:5432/airflow`

If you don't want the example dags to show up in the webUI, you can set the `load_examples` variable to `False`.
Save and quit.

And to prepare for the next steps, we also need to set up the `broker_url` and `celery_result_backend` in the [celery] section:

`broker_url = amqp://guest:guest@localhost:5672//`

`celery_result_backend = amqp://guest:guest@localhost:5672//` (can use the same one as `broker_url`)

For the configuration file to be loaded, we need to reset the database: 

`airflow initdb`

If the previous steps were followed correctly, we can now call the airflow webserver, and access the webUI:

`airflow webserver`

To access the webserver, configure the security group of your EC2 instance and make sure the port 8080 (default airflow webUI port) is open to your computer. Open a web browser, copy and paste your EC2 instance ipv4 address, followed by `:8080`, and the webUI should pop up.
However, we are still half way through. Close the browser and the airflow webserver.

## Exquisite dish: Rabbitmq and Celery##
Rabbitmq is the core component supporting airflow on distributed computing systems. To install the rabbitmq, run the following command:

`sudo apt-get install rabbitmq-server`

And change the configuration file `/etc/rabbitmq/rabbitmq-env.conf`:

`NODE_IP_ADDRESS=0.0.0.0`

And start a rabbitmq service:

`sudo service rabbitmq-server start`

Celery is the python api for rabbitmq. However, by the time this article is written, airflow 1.8 has compatibility issues with celery 4.0.2 due to the librabbitmq library. So make sure to install celery version 3 instead.

`sudo pip install 'celery>=3.1.17,<4.0'`

If you accidentally installed celery 4.0.2, you need to uninstall it before installing the lower version:

`sudo pip uninstall celery`

Otherwise, there will be a confusing error message when you call the `airflow worker`: `Received and deleted unknown message. Wrong destination?!?`

## The dessert: try out the airflow webUI##
Now airflow and the webUI is ready to flow. Let's see how to put dags (directed acyclic graphs: task workflow) in it and run them. We need to create a `dags` file in the airflow home directory:
`mkdir dags`. Write [some test dags](https://airflow.incubator.apache.org/tutorial.html#example-pipeline-definition) and put them in the `dags` directory. Reload the dags:

`airflow initdb

airflow webserver

airflow scheduler

airflow worker`

For the airflow webUI to work, we need to start a webserver and click the run button for a dag. Under the hood, the run button will trigger the `scheduler` to distribute the dag in a task queue (rabbitmq) and assign `workers` to carry out the task. So we need to have all the three airflow components (`webserver`, `scheduler` and `worker`) running. Since we installed the `scheduler` and the `worker` on the same EC2 instance, we had memory limitations and were not able to run all three components at once, we opened up the `airflow webserver` and `airflow scheduler` first, clicked the run button for the test dag, closed the `airflow webserver` and opened the `airflow worker`. The scheduler assigned the tasks in the queue to the workers, and the workers carried out the tasks. The scheduler and the workers recorded their activities in their respective logs in the airflow home directory. After the workers finished the task, we terminated the workers, and reopened the webserver. And the test dag in the webUI became marked successful.

A word about a few useful optional arguments:

 - `-D`: this argument can make the process run as a Daemon in the background
 - `-c`: this argument controls the maximum number of workers that can be triggered by `airflow worker`. The default number can also be set in the `airflow.cfg` file. It is handy when there is working memory limitation on the server.


## TODO##
There are 2 major improvements awaiting for our airflow setup.

 1. Our EC2 instance has the default working memory of 500mb, which is not sufficient for all the three components of airflow to run at the same time. We need to figure out a way to run them together, either by `swapon` memory or put the airflow scheduler and airflow worker on different instances.
 2. There is [discussion](https://www.mail-archive.com/dev@airflow.incubator.apache.org/msg00575.html) on the internet on that airflow webUI does not return error message even if a task fails. We need to look further into it.

## Conclusion##
Installing airflow can be a bit tricky, but understanding the underlying components and the way they interact with each other can help to make the installation task a bit easier.
