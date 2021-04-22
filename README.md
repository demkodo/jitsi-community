# jitsi-community

This a project to allow users to sign up to use jitsi. It repurposes Django's
admin console to provide a web UI for managing uses. And uses the
regisitration-redux module to allow self sign up. The requirements.txt
includes a dependency for gunicorn to serve it along with systemd unit files
for a socket and service.

**IMPORTANT WARNING: There is currently no
automated way to migrate your users from the prosody user database into the
Django database** . we had a small number of users that we migrated manually
by recreating the users in Django admin console.

## Licenses

The solution reuses code from various projects and as such those parts fall
under following licences:
Code reused from Django - [BSD-3-Clause](https://choosealicense.com/licenses/bsd-3-clause/)
Code reused from Prosody and the Prosody community modules - [MIT](https://choosealicense.com/licenses/mit/)

All code not falling under the above is dual licensed under 
[Apache-2.0](https://choosealicense.com/licenses/apache-2.0/) and
[MIT](https://opensource.org/licenses/MIT).

## Folder structure

|meet-accountmanager|a Django application for user sign up| |prosody|contains
files for prosody| |systemd|unit files for services and socket|
|config|example configuration files|

## Installation

These instructions are for installation on Ubuntu 20.04.  They
assume that you already have a working jitsi meet installation and mariadb.
We followed these Digital Ocean community tutorials to set them up: [How To
Install Jitsi Meet on Ubuntu 20.04 By Elliot
Cooper](https://www.digitalocean.com/community/tutorials/how-to-install-jitsi-meet-on-ubuntu-20-04)
[How To Install MariaDB on Ubuntu 20.04 By Brian Boucheron and Mark
Drake](https://www.digitalocean.com/community/tutorials/how-to-install-mariadb-on-ubuntu-20-04)

Create a MariaDB database and users for our services.  Open mariadb client:
```sh
mariadb
```
In the following change _<replace with password>_ for the accountmanager and
prosody database users and run it to create the database:
```mysql
CREATE DATABASE accountmanager CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_520_ci;

CREATE USER 'accountmanager'@'localhost' IDENTIFIED BY '<replace with password>';
GRANT CREATE, ALTER, INDEX, SELECT, UPDATE, INSERT, DELETE, REFERENCES ON accountmanager.* TO 'accountmanager'@'localhost';

CREATE USER 'prosody'@'localhost' IDENTIFIED BY '<replace with password>';
GRANT SELECT ON accountmanager.* TO 'accountmanager'@'localhost';
```


Creating a system user and group for the meet-accountmanager service to run
as:
```sh
sudo adduser adduser --quiet --system --home "/var/lib/meet-accountmanager" --group meet-accountmanager
```

Creating home, configuration and logging directories for the
meet-accountmanager service to use. The logging and home directories should be
writable by the service.

```sh
sudo mkdir -p /{etc/meet-accountmanager,/var/{lib,log}/meet-accountmanager}
chown -R meet-accountmanager:meet-accountmanager /var/{lib,log}/meet-accountmanager
```

Unpack the meet account manager into /opt/meet-accountmanager
```sh
sudo tar -xJf meet-accountmanager.tar.xz -C /opt
```

Configure Django's database conection by copying
```sh
cp /opt/meet-accountmanager/example-configuration/* /etc/meet-accountmananger/
```

Configure Django's email server password by placing it in the file `/etc/meet-accountmanager/email_password`.

Generate a secret key for session and cookie encryption
```sh
cd /etc/meet-accountmanager/
python3 /opt/meet-accountmanager/create_key.py key
```

configure the email accounts that will receive notifications for approvals.
edit `accountmanager/settings.py`. Update the line with the emails:
```python
REGISTRATION_ADMINS = [('<change to name>', '<change to email address>')]
```

Activate the Python virtual environment and use Django's mange.py the
initialize the database.
```sh
cd /opt/meet-accountmanager
source venv/bin/activate
python manage.py makemigrations
python manage.py migrate
```

Add a Django admin user
```sh
python manage.py createsuperuser
```

Add the socket and service
```sh
sudo cp
systemd/meet-accountmanager.{service,socket} /etc/systemd/system/
```

Restart the socket and service
```sh
sudo systemctl enable --now
meet-accountmanager.socket
```

Test the Django socket.
```sh
sudo -u www-data curl --unix-socket /run/gunicorn.sock http
```
The Gunicorn service should be automatically
started and you should see some HTML from your server in the terminal.

Nginx configuration add the following to your Nginx configuration for the Jitsi Meet site.
The file is located in /etc/nginx/sites-available and is probably
named `_<your site address>_.conf`.

Add the following before the first `server` block:
```
upstream accountmanager {
    # fail_timeout=0 means we always retry an upstream even if it failed
    # to return a good HTTP response
    server unix:/run/meet-accountmanager.sock fail_timeout=0;
}
```

Add the following block after the `location = /external_api.js` block:
```
    location ~ ^/static2/(.*)$ {
        add_header 'Access-Control-Allow-Origin' '*';
        alias /opt/meet-accountmanager/static2/$1;
	# try_files $uri =404;
        # cache all versioned files
        if ($arg_v) {
          expires 1y;
        }
    }
```
Add the following block after the `location = /xmpp-websocket` block:
```
    location ^~ /accountmanager/ {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;
        # we don't want nginx trying to do something clever with
        # redirects, we set the Host: header above already.
        proxy_set_header SCRIPT_NAME /accountmanager;
        proxy_redirect off;
        proxy_pass http://accountmanager;
    }
```


Unpack the 
Prosody setup Adding the auth module Adding the hashes.so Udating the Prosody
configuration.
We are using a version of hashes.so taken from a more recent version prosody
because we need SHA-256 support.
```sh
mv /usr/lib/prosody/util/hashes.so /usr/lib/prosody/util/hashes.so.bak
cp hashes.so /usr/lib/prosody/util/
cp mod_auth_sql_hashed.lua /usr/lib/prosody/modules/
```

Edit the prosody configuration for the jitsi meet instance.
This involves setting the prosody instance to use the auth_sql_hashed and adding an auth_sql block to the credentials to the prosody sql user you created earlier.
In the configuration block for the Prosody host used by your Jitsi instance.
```lua
        authentication = "sql_hashed"
        auth_sql = { driver = "MySQL", database = "accountmanager", username = "prosody", password = "<prosody sql user password>", host = "localhost"
```
Restart the prosody instance.


Test that a user that is added in Django can log into jitsi-meet.

## References
The following sources were consulted to create the installation guide:
[Django documentation](https://docs.djangoproject.com/en/3.2/)
[Gunicorn documentation on deployment](https://docs.gunicorn.org/en/latest/deploy.html)
[django-registration-redux 2.9 documentation](https://django-registration-redux.readthedocs.io/en/latest/)

