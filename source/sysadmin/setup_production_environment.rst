Setup production environment
=============================

Prerequisites for this guide
----------------------------


- A Ubuntu 20.04 server
- At least 2 (v)CPU cores with performance at least equal to a 2nd generation Intel i5 desktop processor
- At least 4 GB of RAM
- Sufficient storage to hold the infrastructure components (5GB of free space should easily suffice, but more is recommended to have space for media in the application).


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

Install required packages
-------------------------

.. code-block:: bash
    
    sudo apt install docker docker-compose git mysql-server nginx-full

Set server time to UTC
----------------------

.. code-block:: bash

    sudo timedatectl set-timezone UTC


MySQL configuration
-------------------

Edit mysql server config in /etc/mysql/mysql.conf.d/mysqld.cnf.
Set the bind address to localhost (127.0.0.1) and the docker interface address (172.18.0.1 in this example).
Restart the mysql service after chaning the configuration

.. code-block:: bash

    bind-address            = 127.0.0.1,172.18.0.1


Create database for Costasiella & Vault.
In this example a user with the username "user" and password "password" is created. 
This user can access the MySQL server from the 172.18.0.0/16 docker subnet.
Something more secure is hightly recommended.

.. code-block:: bash

    sudo mysql
    mysql> create database costasiella;
    mysql> create database vault;
    mysql> create user 'user'@'172.18.%' identified by 'password';
    mysql> grant all privileges on costasiella.* to 'user'@'172.18.%';
    mysql> grant all privileges on vault.* to 'user'@'172.18.%';
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

.. code-block::bash

    export VAULT_ADDR=http://127.0.0.1:8200


**Perform initial setup for Vault**

Create an SSH tunnel to map port 8200 on your Costasiella server to port 8200 on your device.
Port 8200 should not be reachable on the server from the word wide web, please firewall it.
Or a cleaner approach is to create multiple listeners. One for localhost and one for the docker interface. 
Have a look here at the Vault docs for more info:
https://www.vaultproject.io/docs/configuration/listener/tcp

For now we keep it simple in this guide. Vault will listen on all interfaces and we'll assume that you've firewalled the external interface of your Costasiella server.
Using this command on your computer (Linux or Mac) will allow you to access the Vault UI on the server from http://localhost:8200 on your computer.

.. code-block::bash

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
In this guide the key name "Costasiella" will be used. Add that to the name field and click *Create encryption key*.

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

.. code-block::bash

    vault login <root token>
    vault token create -policy=use_costasiella_transit -period=768h    

Note down this token for later use in this guide and note that the token expires in 32 days (768 hours).
For security reasons, Vault doesn't allow tokens to live longer than this by default. However, it's a periodic token so it can be renewed an unlimited number of times.

.. code-block::bash

    vault login <your created token>
    vault token renew

Don't forget to regularly renew your token to ensure Costasiella doesn't lose access to Vault.

Backend setup preparation
-------------------------

**Create directories to hold docker bind mounts**

.. code-block:: bash

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

.. code-block:: bash


- Craete & set a new django secret key (50 characters of random stuff will do)s