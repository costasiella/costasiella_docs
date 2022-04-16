Setup production environment
=============================

*A quick note: This guide should be mostly correct and complete, but hasn't passed final review yet. In case you run into a problem, please create an issue on github so it can be checked."*

Prerequisites for this guide
----------------------------


- A DNS record pointing to the server should be in place, eg. demo.costasiella.com. This is not supported: costasiella.com/demo
- A Ubuntu 20.04 server 
  (Another flavour of Linux is also ok, but you might have to change some package names and take into account that not every distro build their packages with the same parameters).
- At least 2 (v)CPU cores with performance equal to a 2nd generation Intel i5 desktop processor or greater.
- At least 4 GB of RAM
- Sufficient storage to hold the infrastructure components (5GB of free space should easily suffice, but more is recommended to have space for media in the application).

Target audience
---------------

This guide assumes some experience in Linux system administration. 
In case you're not comfortable with or knowledgeable about any of the following items, it might be best to ask an experienced Linux system administrator for help.
Although you might be able to follow this guide by simply copying commands, there's a chance you create security issues exposing your customer data if you don't actually understand what you're copying and why.

- Configuring DNS records
- Running commands on a Linux terminal
- Managing Nginx
- Managing MySQL
- Managinx Docker
- Managing SSL certificates
- Configuring backups on a Linux server

Introduction
----------------

This guide is one possible way of setting up a production environment for Costasiella.
Some components have been put into containers, while others are deployed in a more traditional way on a Linux server. 
Basically these choices come down to personal preference. This guide keeps some things on the host so they can be easily backed up using traditional methods.
Costasiella code and required modules are packed in docker containers, to ensure compatibility and prevent messing with (perhaps incompatible) modules on a server.  
Of course, you're completely free to deploy everything directly on a server or containerize everything. 
You've got access to the source code, so build the environment that suits you.

The production setup example in this guide consists of the following components:

A Ubuntu 20.04 server (Host) with the following components:

- Nginx (Host)
- MySQL server (Host)
- Hashicorp Vault (Host)
- Costasiella frontend (Host)
- Redis (Docker)
- Costasiella backend (Docker)
- Costasiella celery worker (Docker)
- Costasiella celery beat (Docker)

Quick start
-----------

In case you're looking to evaluate Costasiella, the guide on setting up a local development environment might be a bit quicker.
However, never use that setup on a production system. It's not secure enough, doesn't scale well when it comes to performance and will be hard to maintain.

For a production system, there are just a number some components that need to be installed and configured to get Costasiella up and running.
No way around it unfortunately.

Preparation
--------------

Sign up for reCAPTCHA and create keys that apply to your domain, eg. demo.costasiella.com. 
https://www.google.com/recaptcha/about/

Install required packages
-------------------------

.. code-block:: bash
    
    sudo apt install docker docker-compose git mysql-server nginx-full

Set server time to UTC
----------------------

.. code-block:: bash

    sudo timedatectl set-timezone UTC

Find docker interface IP address
---------------------------------

Use the command "ip a s" to show all network interfaces and look for the one called "docker0"

.. code-block:: bash

    $ ip a s
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host 
        valid_lft forever preferred_lft forever
    2: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
        link/ether 52:54:00:11:41:6d brd ff:ff:ff:ff:ff:ff
        inet 192.168.122.233/24 brd 192.168.122.255 scope global dynamic noprefixroute enp3s0
        valid_lft 3581sec preferred_lft 3581sec
        inet6 fe80::3607:3352:99b6:b438/64 scope link noprefixroute 
        valid_lft forever preferred_lft forever
    3: br-a80887c9ec37: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
        link/ether 02:42:fa:8e:cc:22 brd ff:ff:ff:ff:ff:ff
        inet 172.18.0.1/16 brd 172.18.255.255 scope global br-a80887c9ec37
        valid_lft forever preferred_lft forever
        inet6 fe80::42:faff:fe8e:cc22/64 scope link 
        valid_lft forever preferred_lft forever
    4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
        link/ether 02:42:50:7f:0c:07 brd ff:ff:ff:ff:ff:ff
        inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
        valid_lft forever preferred_lft forever

In this case the IP address of the docker interface is 172.17.0.1. This address will be used a few times in this guide. 
Check which address your docker interface is configured to use and make a note somewhere it's easy to reference when you need it.


MySQL configuration
-------------------

Edit mysql server config in /etc/mysql/mysql.conf.d/mysqld.cnf.
Set the bind address to localhost (127.0.0.1) and the docker interface address (172.17.0.1 in this example).
Restart the mysql service after chaning the configuration

.. code-block:: bash

    bind-address            = 127.0.0.1,172.17.0.1


Create database for Costasiella & Vault.
In this example a user with the username "user" and password "password" is created. 
This user can access the MySQL server from the 172.17.0.0/16 docker subnet.
Something more secure is hightly recommended.

.. code-block:: bash

    sudo mysql
    mysql> create database costasiella;
    mysql> create database vault;
    mysql> create user 'user'@'172.17.%' identified by 'password';
    mysql> grant all privileges on costasiella.* to 'user'@'172.17.%';
    mysql> grant all privileges on vault.* to 'user'@'172.17.%';
    mysql> flush privileges;


Install Hashicorp Vault
-------------------------

Please visit the following URL and follow the setup steps of your chosen method. For this guide using the package manager (apt) is assumed.
https://learn.hashicorp.com/tutorials/vault/getting-started-install

After installing vault, make it start at boot and configure it to use a MySQL database for it's storage instead of files. 

.. code-block:: bash

    sudo systemctl enable vault

Configure Vault to use MySQL storage and don't use TLS for this example guide to keep things simple. 
In production it's recommended to configure this.

Open the Vault configuration file at /etc/vault.d/vault.hcl with your favorite editor.

- Comment out the file storage section
- Configure MySQL storage
- Disable TLS

.. code-block:: bash

    # Full configuration options can be found at https://www.vaultproject.io/docs/configuration

    ui = true

    #mlock = true
    #disable_mlock = true

    #storage "file" {
    #  path = "/opt/vault/data"
    #}
    #

    storage "mysql" {
    username = "user"
    password = "password"
    database = "vault"
    }

    #storage "consul" {
    #  address = "127.0.0.1:8500"
    #  path    = "vault"
    #}

    # HTTP listener
    #listener "tcp" {
    #  address = "127.0.0.1:8200"
    #  tls_disable = 1
    #}

    # HTTPS listener
    listener "tcp" {
    address       = "0.0.0.0:8200"
    tls_disable   = true
    #  tls_cert_file = "/opt/vault/tls/tls.crt"
    #  tls_key_file  = "/opt/vault/tls/tls.key"
    }

    # Enterprise license_path
    # This will be required for enterprise as of v1.8
    #license_path = "/etc/vault.d/vault.hclic"

    # Example AWS KMS auto unseal
    #seal "awskms" {
    #  region = "us-east-1"
    #  kms_key_id = "REPLACE-ME"
    #}

    # Example HSM auto unseal
    #seal "pkcs11" {
    #  lib            = "/usr/vault/lib/libCryptoki2_64.so"
    #  slot           = "0"
    #  pin            = "AAAA-BBBB-CCCC-DDDD"
    #  key_label      = "vault-hsm-key"
    #  hmac_key_label = "vault-hsm-hmac-key"

Restart the vault service to reload the configuration file.

Add the following to your .bashrc or .zshrc or whatever file your shell uses.

.. code-block:: bash

    export VAULT_ADDR=http://127.0.0.1:8200


**Perform initial setup for Vault**

Create an SSH tunnel to map port 8200 on your Costasiella server to port 8200 on your device.
Port 8200 should not be reachable on the server from the word wide web, please firewall it.
Or a cleaner approach is to create multiple listeners. One for localhost and one for the docker interface. 
Have a look here at the Vault docs for more info:
https://www.vaultproject.io/docs/configuration/listener/tcp

For now we keep it simple in this guide. Vault will listen on all interfaces and we'll assume that you've firewalled the external interface of your Costasiella server.
Using this command on your computer (Linux or Mac) will allow you to access the Vault UI on the server from http://localhost:8200 on your computer.

.. code-block:: bash

    ssh -C -L 8200:127.0.0.1:8200 -N <IP of your Costasiella server>


**Add a transit key**

Open a browser and open the Vault web UI at http://localhost:8200 to do the initial setup.
Set for example 5 key shares, with a threshold of 3 and click Initialize.

Download the keys and store them somewhere secure (eg. encrypted in a password manager database). You'll need them everytime Vault starts to unseal it and you'll need the root token for administration.
*Continue to unseal*

Add 3 of the 5 keys, one by one, to unseal.
Log in using the root token.

Go to Secrets and choose *Enable new engine*. 
Choose transit and click Next.
Accept the default path called transit and click *Enable engine*.

Create an encryption key for Costasiella by clicking *Create encryption key*. 
In this guide the key name *costasiella* will be used. Add that to the name field and click *Create encryption key*.

**Create a policy**

To avoid having to use the root token in Costasiella, we'll create a new token to which we'll assign a policy that's limited to using the Costasiella transit key and no other functionality withing vault.

Click *Policies* in the main menu.
Click *Create ACL policy*. 

Name it something clear and easy to remember. In this guide "use_costasiella_transit" will be used for the policy name.
Add the following to the *Policy* field.

.. code-block::

    # Vault transit key policy
    path "transit/encrypt/costasiella" {
    capabilities = ["update" ] 
    }
    path "transit/decrypt/costasiella" {
    capabilities = ["update" ] 
    }

    # List existing keys in UI
    path "transit/keys" {
    capabilities = [ "list" ]
    }

    # Enable to select the orders key in UI
    path "transit/keys/costasiella" {
    capabilities = [ "read" ]
    }

Click *Create policy*

**Create a token for Costasiella**

In an SSH or console session on your server:

.. code-block:: bash

    vault login <root token>
    vault token create -policy=use_costasiella_transit -period=768h    

Note down this token for later use in this guide and note that the token expires in 32 days (768 hours).
For security reasons, Vault doesn't allow tokens to live longer than this by default. However, it's a periodic token so it can be renewed an unlimited number of times.

.. code-block:: bash

    vault login <your created token>
    vault token renew

Don't forget to regularly renew your token to ensure Costasiella doesn't lose access to Vault.

Backend setup preparation
-------------------------

**Create directories to hold docker bind mounts**

.. code-block:: bash

    mkdir -pv /opt/docker/mounts/costasiella/logs
    mkdir -pv /opt/docker/mounts/costasiella/media
    mkdir -pv /opt/docker/mounts/costasiella/sockets
    mkdir -pv /opt/docker/mounts/costasiella/static

**Fetch backend code from GitHub**

The settings directory is copied to a separate bind mount point so it can persist after an update.

.. code-block:: bash

    cd /opt/docker/mounts/costasiella
    git clone https://github.com/costasiella/costasiella.git
    cp -prv /opt/docker/mounts/costasiella/costasiella/app/app/settings /opt/docker/mounts/costasiella/settings

**Edit Django settings**

Edit /opt/docker/mounts/costasiella/settings/common.py

- Replace the SECRET_KEY value with a random string that's 50 characters long.
- Update the databases section to allow the backend to connect to the MySQL server running on the host.
- Find the vault section and update it with the settings created earlier. (Note that the address 172.17.0.1 is the address of the docker interface).

.. code-block:: bash
    
    ...

    else:
        DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.mysql',
            'NAME': 'costasiella',
            'USER': 'user',
            'PASSWORD': 'password',
            'HOST': '172.17.0.1',
            'PORT': 3306
        }
    }

    ...

    VAULT_URL = 'http://172.17.0.1:8200'
    VAULT_TOKEN = '<The token you created here>'
    VAULT_TRANSIT_KEY = 'costasiella'
    
    ...

    RECAPTCHA_PUBLIC_KEY = '<Your site key here>'
    RECAPTCHA_PRIVATE_KEY = '<Your secret key here>'
    
    ...

Save the settings file

**Configure email**

Edit /opt/docker/mounts/costasiella/settings/production.py
Change the values in the email configuration section to reflect your email infrastructure.

As a general suggestion (feel free to take it or leave it) it could be wise to set up a local postfix server and point your Costasiella to that.
This way there's a message queue that will hold the messages in case the SMTP server you're sending to isn't accepting email for any reason. 
Another benefit is simpler email configuration in your Costasiella installation. You can simply point it to the IP of the system holding your postfix server and port 25.

For example:

.. code-block:: bash
    
    ...
    # Email configuration
    EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
    EMAIL_HOST = '172.17.0.1'  # In case you run postfix on your docker host
    EMAIL_PORT = 25
    DEFAULT_FROM_EMAIL = 'My Name <my_from_email@domain.com>'
    ...


For a full list of email options, please refer to the `Django documentation <https://docs.djangoproject.com/en/3.2/ref/settings/#email-backend>`_


Backend setup
-------------

**Starting containers**

Now it's time to spin up the containers holding the backend code.
To do this, we're going into the folder holding the code and use docker-compose to bring the environment online.

.. code-block:: bash

    cd /opt/docker/mounts/costasiella/costasiella
    sudo docker-compose up

**Getting the environment ready**

Now a few commands need to be executed inside the backend container to:

- Load fixtures
- Create an initial super user

.. code-block:: bash

    docker exec -it <costasiella backend container name> /bin/bash
    cd /opt/app
    # Load fixtures
    python manage.py loaddata costasiella/fixtures/*.json
    # Create super user
    ./manage.py createsuperuser


Frontend setup
--------------

**Fetch frontend code from GitHub and copy into the webserver directory**

.. code-block:: bash

    cd /opt
    git clone https://github.com/costasiella/frontend.git costasiella_frontend
    cp -prv /opt/costasiella_frontend/build/ /var/www/html/

**Configure Nginx**

Create a file representing your hostname in /etc/nginx/sites-available.
In this example the file *demo.costasiella.com* will be used.

.. code-block::bash

    # the upstream component nginx needs to connect to
    upstream django {
        server unix:///opt/docker/mounts/costasiella/sockets/app.sock; # for a file socket
        # TCP socket for easier setup, but it comes with some additional overhead.
        #server 127.0.0.1:8001; # for a web port socket (we'll use this first)
    }

    # Rate limiting zone
    limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;

    # configuration of the server
    server {
        # the port your site will be served on
        listen      80;
        # the domain name it will serve for
        server_name demo.costasiella.com;  # substitute for your domain name
        charset     utf-8;
        root        /var/www/html/build/;

        # max upload size
        client_max_body_size 10M;   # adjust to taste

        # Django media
        location /d/media  {
            alias /opt/docker/mounts/costasiella/media/;  # your Django project's media files - amend as required
        }

        # Django static
        location /d/static {
            alias /opt/docker/mounts/costasiella/static/; # your Django project's static files - amend as required
        }

        # Send all non-media requests to the Django backend
        # To read more about rate limiting: https://www.nginx.com/blog/rate-limiting-nginx/
        location /d {
            limit_req zone=mylimit burst=20 nodelay;
            uwsgi_pass  django;
            include     /etc/nginx/uwsgi_params; # the uwsgi_params file you installed
        }

        # React frontend app
        location / {
            alias /var/www/html/build/;
        }
    }

Restart the Nginx service

**Apply any database updates that might be available**

Open a browser and go to http://<your domain>/d/update


Configure the superuser account as a Costasiella admin
-------------------------------------------------------

Open a webbrowser (tab) and go to <your domain>/d/admin. 
Log in using the initial superuser credentials created earlier.

Navigate to Costasiella > Accounts and click the email address of the superuser. Now add the user to the Admins group and click save.

Run the following code in a mysql terminal with a user that has permissions to modify your Costasiella database.

.. code-block:: bash

    use costasiella;
    update costasiella_account set employee=1 where id=1;

This enables the superuser to sign in to the backend with admin privileges.

*Note: The superuser isn't created in the "regular" way. It doesn't have records in all the tables that regular accounts have.
It's highly recommened to use an account created under relations > accounts that's been granted admin privileges to manage Costasiella.*

Create additional admin accounts
---------------------------------

Creating at least one other admin account is always a good idea. 
This way you don't have to sign in for daily use with a user that has superuser status.

**Create additional account**

Log in using the superuser credentials your created on <your domain name> (eg. demo.costasiella.com).

Navigate to the backend and then to relations > accounts. Add a new account.
Edit the newly created account and set the Employee switch to on.

**Grant Admin privileges**

Go go to <your domain>/d/admin and sign in with your superuser account.

Navigate to the accounts list under the COSTASIELLA section. Click the email address of the account you just created to edit it.
Add the account to the Admins group and click save.

**Set password**

Again click the email address of the account you just created to edit it.
In the password field, click the small link "this form" to set a password.

After setting an initial password, the additional admin account is ready to be used.

**Note**

Preferably test it in another webbrowser. It's possible to sign in to <your domain>/d/admin and <your domain> using different accounts in the same browser.
This can give rise to unexpected behavour. In case Costasiella behaves strangely for you during these steps, sign out of both the <your domain> and <your domain>/d/admin and try again.


Setting the site name
----------------------

The set site name is used in some system email messages, eg. the password reset email.

Open a webbrowser (tab) and go to <your domain>/d/admin. 
Log in using the initial superuser credentials created earlier.

Navigate to the *Sites* section and click the default domain name *example.com*. 
Change the domain name and display name to reflect your installation and click *Save*.


Next steps
----------

Now the Costasiella environment is up and running, you can integrate it into your (hosting) infrastructure as required.
As these steps can differ greatly depending on the environment Costasiella is installed in, these won't be detailed in this guide.
Some common next steps can be:

- Publishing it through your load balancer / reverse proxy / Web application firewall
- Adding SSL certificates
- Configuring backups
- Adding the Costasiella url to your monitoring system to check if it's online and performing as it should


Troubleshooting
----------------

**Database wasn't reachable on backend container start**

In case a firewall or incorrect configuration prevented the backend from connecting to the database on startup, database migrations might not have run.
Symptoms might include messages that tables can't be found.

Resultion: Ensure the backend container can connect to the database and restart it. 

**Fixtures don;t load with error: Connection refused**

This error occurs when the backend container can't connect to your Vault instance. 

Resolution: Ensure the backend container can connect to your Vault instance.
