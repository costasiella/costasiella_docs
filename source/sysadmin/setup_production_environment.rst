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

- MySQL (Host)
- Nginx (Host)
- Hashicorp Vault (Host)
- Costasiella frontend (Host)
- Redis (Docker)
- Costasiella backend (Docker)
- Costasiella celery worker (Docker)
- Costasiella celery beat (Docker)

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

**Configure server to listen to localhost and docker network interface**

**Create databases**

**Create users**


Install Hashicorp Vault
-------------------------

Please visit the following URL and follow the setup steps of your chosen method. For this guide using the package manager (apt) is assumed.
https://learn.hashicorp.com/tutorials/vault/getting-started-install

After installing vault, make it start at boot and configure it to use a MySQL database for it's storage instead of files. 

.. code-block:: bash

    sudo systemctl enable vault



- Craete & set a new django secret key (50 characters of random stuff will do)s