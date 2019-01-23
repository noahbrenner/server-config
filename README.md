Linux Server Configuration
==========================

This is an overview of steps taken to configure a web server following Udacity's specifications as part of their [Full Stack Web Developer][] Nanodegree program. Specifically, the setup of an [Apache][] server running a [Flask][] app hosted on an [Amazon Lightsail][] instance running [Ubuntu][].

Addresses
---------

Server URL: http://52.36.249.184/

SSH IP and port (for Udacity reviewer): `52.36.249.184:2200`

Software installed
------------------

### Via `apt-get`

* `apache2`
* `libapache2-mod-wsgi-py3`
* `postgresql`
* `python3-pip`
* `python3` (already installed)
* `git` (already installed)
* `unattended-upgrades` (already installed)

### Via `pip3`

* `pipenv`

Summary of configuration changes made
-------------------------------------

* Update `/etc/ssh/sshd_config` to use port 2200, and disallow root login (password authentication was already turned off).
* Configure `ufw` to only allow traffic on ports 2200 (SSH), 80 (HTTP), and 123 (NTP), and to only use TCP.
* Set up automatic upgrades for installed packages using `unattended-upgrades`.
* Create a user named "grader". Give them root access using a newly created file: `/etc/sudoers.d/grader`. Enter a generated public key for them in `/home/grader/.ssh/authorized_keys`.
* `git clone` my [Instrument Catalog][] app into `/var/www/wsgi-scripts/` and install its dependencies. Create an `.env` file in its top level directory to define environment variables for the app. Create an `app.wsgi` file for the app (since one wasn't needed for its Heroku deployment) and use dotenv (a dependency of Flask) withing that file to load environment variables before importing the Flask app. **Note that I set the environment to "development" instead of "production"; See the *"Set environment variables used by our app"* bullet point in the [Setup steps](#setup-steps) section below for an explanation.**
* Create a database owned by a postgres user named "catalog". This user is mapped to the OS user "www-data". The user can modify the database, but can not create new databases, can not add new postgres users, and is not a super user. Also create a user named "root" so that databases can be managed using `sudo`. Username mappings were set in `/etc/postgresql/9.5/main/pg_ident.conf`. Database permissions were set in `/etc/postgresql/9.5/main/pg_hba.conf`.
* Update `/etc/apache2/sites-enabled/000-default.conf` to serve static assets directly and to mount our WSGI app at the root server path and run in a daemon process. The path to the app directory was also specified here so that `instrument_catalog` could be imported from `app.wsgi`.

Resources referenced
--------------------

When upgrading `apt` packages, there was a prompt about a new version of a grub config file. I decided to keep the existing version based on a comment here:

* https://forums.aws.amazon.com/thread.jspa?threadID=96207

Setting the server timezone:

* https://askubuntu.com/a/800675

Configuring automatic package upgrades

* https://help.ubuntu.com/community/AutomaticSecurityUpdates
* https://medium.com/@Alibaba_Cloud/automatic-security-upgrades-with-unattended-upgrades-package-8ba19d4607fe
    - Also includes instructions for having upgrade reports sent by email.

Configuring mod_wsgi:

* https://modwsgi.readthedocs.io/en/develop/user-guides/quick-configuration-guide.html
* http://flask.pocoo.org/docs/1.0/deploying/mod_wsgi/
* http://blog.dscpl.com.au/2014/09/python-module-search-path-and-modwsgi.html

Installing dependencies with Pipenv in production:

* https://medium.com/@DJetelina/pipenv-review-after-using-in-production-a05e7176f3f0

Configuring PostgreSQL:

* https://www.postgresql.org/docs/current/libpq-connect.html#LIBPQ-CONNSTRING

Setup steps
-----------

These steps aren't necessarily in the order I took them. They're in an order that I thought flowed decently well for reading.

* Create a Lightsail instance from <https://lightsail.aws.amazon.com/> and create a static IP for it in the **Networking** tab of the web dashboard (or do something similar with a different provider).
* Add a barebones `.vimrc` for convenience.
* SSH in and change the ssh public key to my own (`vim ~/.ssh/authorized_keys`)
* Set timezone to UTC (this was already set in my case)
    - `sudo timedatectl set-timezone UTC`
* Update packages
    - `sudo apt-get update && sudo apt-get upgrade && sudo apt-get autoremove`
* Change the SSH port
    - `sudo vim /etc/ssh/sshd_config` and change the line "`Port 22`" (to "`Port 2200`" in this case)
    - `sudo service ssh restart` to apply the changes
    - In the AWS dashboard (Networking tab), edit the firewall rules to add the new port for TCP connections and (optionally) delete the default port 22 rule.
* After confirming that non-root login works using public/private key and allows the use of `sudo`, disable root login and password login.
    - `sudo vim /etc/ssh/sshd_config`
        - Change appropriate line to `PermitRootLogin no`
        - Change appropriate line to `PasswordAuthentication no` (this was already set)
    - `sudo service ssh restart` to apply the changes
* Configure firewall (for a simple Lightsail server like this one, this step could be considered redundant since the same settings must also be made in the AWS dashboard, whose firewall is hit before ever reaching our server, but it's always good to make sure our server secures itself anyway).
    - Verify that firewall is *not* active yet before continuing: `sudo ufw status`
    - `sudo ufw default deny incoming && sudo ufw default allow outgoing`
    - `sudo ufw allow 2200/tcp` (or whatever port we set up for SSH)
    - `sudo ufw allow 80/tcp`
    - `sudo ufw allow 123/tcp` (for NTP; also add this in the AWS dashboard)
    - `sudo ufw enable`
* Create the "grader" user and give them `sudo` access
    - `sudo adduser grader`
    - `sudo EDITOR=vim visudo -f /etc/sudoers.d/90-cloud-init-users` (or copy the file and change the copy)
    - Create `/home/grader/.ssh/` and `/home/grader/.ssh/authorized_keys`, making sure each of them has "grader" as the owner and their permissions appropriately restricted
    - Create a key pair locally using `ssh-keygen` and add the public key to `/home/grader/.ssh/authorized_keys`
* Install new packages
    - `sudo apt-get install $PACKAGES` where `$PACKAGES` is a list of packages shown in the **"Via `apt-get`"** section of [Software installed](#software-installed).
    - `sudo pip3 install pipenv`
* Set up unattended packages upgrades
    - `sudo dpkg-reconfigure --priority=low unattended-upgrades` and select the following:
        - Automatically download and install stable updates? **Yes**
        - Origins-Pattern that packages must match to be upgraded: **(accept the default)**
    - `sudo vim /etc/apt/apt.conf.d/50unattended-upgrades`
        - Verify that security upgrades will be applied (that line is not commented out in the `Unattended-Upgrade::Allowed-Origins` section).
        - Decide if you want other categories of upgrades automatically applied (likely not for stability, though choosing not to may require more manual maintenance).
    - You can also check out the `/etc/apt/apt.conf.d/50unattended-upgrades` file to check or change the enabled status and frequency of package list updates and package upgrades.
* Configure mod_wsgi
    - `sudo mkdir /var/www/wsgi-scripts`
    - `sudo vim /var/www/wsgi-scripts/myapp.wsgi` Make test script:
      ```python
      def application(environ, start_response):
          status = '200 OK'
          output = b'Hello again, World!'

          response_headers = [('Content-type', 'text/plain'),
                              ('Content-Length', str(len(output)))]

          start_response(status, response_headers)

          return [output]
      ```
    - `sudo vim /etc/apache2/sites-enabled/000-default.conf` in `<VirtualHost>`, add:

      ```apache
      # Mount WSGI app
      WSGIScriptAlias /myapp /var/www/wsgi-scripts/myapp.wsgi
      ```
    - `sudo apache2ctl restart`, then visit the `/myapp` path at the server URL in a browser.
* Install Python app
    - `cd /var/www/wsgi-scripts && sudo git clone https://github.com/noahbrenner/instrument-catalog-py.git`
    - `cd instrument-catalog-py && sudo pipenv install --system --deploy --sequential`
* Set up database
    - `sudo -u postgres createdb catalog`
    - `sudo -u postgres createuser -DRS catalog` ("catalog" user can not create databases or roles and is not a superuser)
    - `sudo -u postgres createuser -DRs root` ("root" user can not create databases or roles but *is* a superuser)
    - `sudo -u postgres createdb -O catalog catalog` (create "catalog" database owned by postgres user "catalog")
    - `sudo vim /etc/postgresql/9.5/main/pg_ident.conf`
        - Add the line `catalog-app     www-data    catalog` (mapname, system-username, pg-username)
    - `sudo vim /etc/postgresql/9.5/main/pg_hba.conf`
        - Add the line `local   catalog     catalog     peer map=catalog-app` (type, database, user, method, option)
    - `sudo service postgresql restart` (restart so that config file changes are applied)
* Set environment variables used by our app
    - `sudo vim /var/www/wsgi-scripts/instrument-catalog-py/.env`
      ```bash
      FLASK_ENV=development # See below for why this isn't "production"
      SECRET_KEY=[some random key]
      DATABASE_URL='postgresql:///catalog?user=catalog' # (dbname?user=pg_username)
      GOOGLE_CLIENT_ID='' # Not used, see below
      GOOGLE_CLIENT_SECRET='' # Not used, see below
      ```
      Note that for this demo we're setting the environment to "development", which isn't great but is necessary in order to meet the requirement of this Udacity project that the app only be served over port 80 (no `https`).
      - The Instrument Catalog app was designed to always redirect to `https` in a production environment, so it would not be able to run with the port 80 restriction if it were not set to "development".
      - OAuth login also requires `https`, so even with a workaround for the point above, the login functionality (and thus database operations other that reading) would not be usable for this demo. However, the app includes a dev mode login feature, which will now be usable due to the "development" environment setting. Since the app won't be served securely, we're not even going to supply a real Google client id or secret.
    - `sudo chown www-data /var/www/wsgi-scripts/instrument-catalog-py/.env` (make sure the web server can read it)
    - `sudo chmod 400 /var/www/wsgi-scripts/instrument-catalog-py/.env` (make sure no one else can read it)
    - `sudo vim /var/www/wsgi-scripts/instrument-catalog-py/app.wsgi` (this Python app was designed for deployment on Heroku and didn't need a WSGI file for that, so we're creating one now)
      ```python
      from dotenv import load_dotenv
      load_dotenv('/var/www/wsgi-scripts/instrument-catalog-py/.env')
      from instrument_catalog import app as application
      ```
* Initialize app database
    - `cd /var/www/wsgi-scripts/instrument-catalog-py`
    - `sudo flask db upgrade`
* Update mod_wsgi configuration
    - `sudo vim /etc/apache2/sites-enabled/000-default.conf`
      ```apache
      # Serve static assets
      Alias /static/ /var/www/wsgi-scripts/instrument-catalog-py/instrument_catalog/static/

      # Mount WSGI app
      WSGIDaemonProcess catalog python-path=/var/www/wsgi-scripts/instrument-catalog-py
      WSGIProcessGroup catalog
      WSGIScriptAlias / /var/www/wsgi-scripts/instrument_catalog-py/app.wsgi
      ```
    - `sudo apache2ctl restart` and visit the working web app!


[Amazon Lightsail]: https://aws.amazon.com/lightsail/
[Apache]: https://httpd.apache.org/
[Flask]: http://flask.pocoo.org/
[Full Stack Web Developer]: https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004
[Instrument Catalog]: https://github.com/noahbrenner/instrument-catalog-py
[Ubuntu]: https://www.ubuntu.com/
