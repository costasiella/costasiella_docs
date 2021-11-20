Setup development environment
=============================

This page will explain how to setup a development environment for Costasiella.
A Linux development environment based on Ubuntu 20.04 is presumed in this guide.

Preparation
--------------

Sign up for reCAPTCHA and create keys that apply to the localhost domain. https://www.google.com/recaptcha/about/

Required packages
-----------------

Install required packages

.. code-block:: bash

    sudo apt install curl git mysql-server python3-mysqldb libmysqlclient-dev redis-server redis-tools python-pip libffi-dev
    sudo pip install virtualenvwrapper


MySQL database & Redis server
-----------------------------

.. code-block:: bash

    sudo mysql
    CREATE DATABASE costasiella CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
    CREATE DATABASE vault CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
    create user 'user'@'localhost' identified by 'password';
    grant all privileges on costasiella.* to 'user'@'localhost';
    grant all privileges on vault.* to 'user'@'localhost';
    flush privileges;

Vault setup
-----------

Download Vault from https://www.vaultproject.io

Create an empty file called vault_config_dev.hcl (or anything to your liking) and put the following in it:

.. code-block::

    ui = true
    disable_mlock = true

    storage "mysql" {
    username = "user"
    password = "password"
    database = "vault"
    }

    listener "tcp" {
    address     = "127.0.0.1:8200"
    tls_disable = true
    }

Edit the storage section with your database information.

Start the Vault server using

.. code-block::

    vault server -config=/path/to/config/config_dev.hcl


Open a webbrowser and go to http://localhost:8200/ui to perform an initial setup for Vault.
For key shares and key threshold, fill in a 1. Though note for production something like 5 and 3 would be more appropriate.

**Important**
    Save the root token and key somewhere safe. When these keys are lost, you lose all access to Vault and the encrypted data.

Continue by clicking the “Unseal” button and enter the key from the previous step (not the root token).
Then sign in to vault using the root token.

**Set up the transit engine**
Click “Enable new engine” and choose Transit under generic. Enter a recognizable name for the Path, in this document the default name “transit” will be used for the transit engine path. Click enable engine on the bottom of the page.

Click “Create encryption key” and enter a recognizable name. Then click the Create entryption key button. 
In this document “costasiella” will be used as the name for the encryption key.

In case you’d like to do further vault setup (at some point) using the terminal; add the following lines to the .bashrc or .zshrc file in your home directory.

.. code-block:: bash

    # Vault config
    complete -C /usr/local/bin/vault vault
    export VAULT_ADDR=http://127.0.0.1:8200


Fetch source from git
----------------------

Go to the folder where you want to keep the Costasiella source files

.. code-block:: bash

    git clone https://github.com/costasiella/costasiella.git
    git clone https://github.com/costasiella/frontend.git

NPM
----

.. code-block:: bash

    curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
    sudo apt install nodejs
    cd frontend
    # install node modules
    npm install

Python virtual environment
---------------------------

Add the following lines to the .bashrc or .zshrc file in your home directory

.. code-block:: bash

    # virtualenvwrapper stuff
    export WORKON_HOME=$HOME/Development/virtualenvs
    export PROJECT_HOME=$HOME/Development
    export VIRTUALENVWRAPPER_SCRIPT=/usr/local/bin/virtualenvwrapper.sh
    source /usr/local/bin/virtualenvwrapper_lazy.sh

Open a new terminal to start the development environment, to make sure .bashrc or .zshrc is reloaded.

Create a new virtual environment and install required python modules

.. code-block:: bash

    mkvirtualenv cs_dev -p /usr/bin/python3
    cd <your costasiella root dir>/
    pip install -r requirements.txt

Django settings
----------------

Go to your costasiella root dir/app/app and edit settings/common.py

* Edit the databases section as required
* Under the vault configuration section edit the following setting to reflect your environment

.. code-block:: bash
    
    ...
    
    VAULT_URL = ‘http://localhost:8200’
    VAULT_TOKEN = <Your root token here, definitely bad idea for production, but fine for development>
    VAULT_TRANSIT_KEY = “costasiella”

    ...

    RECAPTCHA_PUBLIC_KEY = '<Your site key here>'
    RECAPTCHA_PRIVATE_KEY = '<Your secret key here>'
    
    ...

Prepare for lift off
----------------------

Init database; create admin user; start django (back-end) development server.
Go to <your costasiella root dir>/app (this folder should contain a file called manage.py).

.. code-block:: bash
    
    ./manage.py migrate 
    ./manage.py createsuperuser
    # fill out questions to create initial super admin user
    ./manage.py loaddata costasiella/fixtures/*.json
    ./manage.py runserver

Start the npm development server;
Open a new terminal tab or window and go your costasiella frontend root dir.

.. code-block

    npm run start

A webbrowser will open to localhost:3000. There’s a proxy that’ll allow access to some django pages using the /d path in the address.
eg. http://localhost:3000/d/admin

The Django development server runs on port 8000 in case you'd like to access it directly.

**Apply any database updates that might be available**

Open a browser and go to http://localhost:3000/d/update

Create a user and log in
-------------------------

Open a webbrowser (tab) and go to <your domain>/d/admin. 
Log in using the initial superuser credentials created earlier.

*Create user account*

Click “add” next to accounts under the COSTASIELLA section.
Add a new account and enter the user’s names and an email address in the edit screen after saving. 
Add the account to the Admins group and click save.

*Create email address for account*

Click “Add” next to Email addresses under the ACCOUNTS section. 
Use the little looking glass next to “user” in the “Add email address” form to select the user just created. 
Then enter the same email address as entered when saving the user and check both the “Verified” and “Primary” boxes. 
Click Save.

Almost there, log out of the admin page by clicking LOG OUT in the top right corner. 

*Make the created user an employee to gain access to the backend*

Run the following code in a mysql terminal with a user that has permissions to modify your Costasiella database.

.. code-block:: bash

    use costasiella;
    update costasiella_account set employee=1 where id=2;

Now log in using the credentials your created.

In case a “CSRF Token Failed” error message shows, click the back button in the browser and try again. 
It might show up in some cases during the first login in the development environment. After a refresh/retry it shouldn’t show anymore.



GraphiQL
---------

The GraphiQL interface is available at http://localhost:8000/d/graphql
