..
    Copyright 2013 Rackspace Hosting

    Licensed under the Apache License, Version 2.0 (the "License"); you may
    not use this file except in compliance with the License. You may obtain
    a copy of the License at

        http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
    WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
    License for the specific language governing permissions and limitations
    under the License.

*********************************
Development Environment on Ubuntu
*********************************

Designate is comprised of four main components :ref:`designate-api`, :ref:`designate-central`,
:ref:`designate-mdns`, and :ref:`designate-pool-manager`, supported by a few
standard open source components. For more information see :ref:`architecture`.

There are many different options for customizing Designate, and two of these options
have a major impact on the installation process:

* The storage backend used (SQLite or MySQL)
* The DNS backend used (PowerDNS or BIND9)

This guide will walk you through setting up a typical development environment for Designate,
using BIND9 as the DNS backend and MySQL as the storage backend. For a more complete discussion on
installation & configuration options, please see :ref:`architecture` and :ref:`production-architecture`.

For this guide you will need access to an Ubuntu Server (14.04).

.. _Development Environment:

Development Environment
+++++++++++++++++++++++

Installing Designate
====================

.. index::
   double: install; designate

1. Install system package dependencies (Ubuntu)

::

   $ apt-get install python-pip python-virtualenv git
   $ apt-get build-dep python-lxml

2. Clone the Designate repo from GitHub

::

   $ cd /var/lib
   $ git clone https://github.com/openstack/designate.git
   $ cd designate


3. Setup virtualenv

.. note::
   This is an optional step, but will allow Designate's dependencies
   to be installed in a contained environment that can be easily deleted
   if you choose to start over or uninstall Designate.

::

   $ virtualenv --no-site-packages .venv
   $ . .venv/bin/activate


4. Install Designate and its dependencies

::

   $ pip install -r requirements.txt -r test-requirements.txt
   $ python setup.py develop


5. Change directories to the etc/designate folder.

.. note::
    Everything from here on out should take place in or below your designate/etc folder

::

   $ cd etc/designate


6. Create Designate's config files by copying the sample config files

::

   $ ls *.sample | while read f; do cp $f $(echo $f | sed "s/.sample$//g"); done


7. Make the directory for Designate’s log files

::

   $ mkdir /var/log/designate


Configuring Designate
======================

.. index::
    double: configure; designate

Open the designate.conf file for editing

::

  $ editor designate.conf


Copy or mirror the configuration from this sample file here:

.. literalinclude:: ../examples/basic-config-sample.conf
    :language: ini


Installing RabbitMQ
===================

.. note::

    Do the following commands as "root" or via sudo <command>

Install the RabbitMQ package

::

    $ apt-get install rabbitmq-server

Create a user:

::

    $ rabbitmqctl add_user designate designate

Give the user access to the / vhost:

::

    $ sudo rabbitmqctl set_permissions -p "/" designate ".*" ".*" ".*"


Installing MySQL
================

.. index::
    double: install; mysql

Install the MySQL server package

::

    $ apt-get install mysql-server-5.5


If you do not have MySQL previously installed, you will be prompted to change the root password.
By default, the MySQL root password for Designate is "password". You can:

* Change the root password to "password"
* If you want your own password, edit the designate.conf file and change any instance of
   "mysql://root:password@127.0.0.1/designate" to "mysql://root:YOUR_PASSWORD@127.0.0.1/designate"

You can change your MySQL password anytime with the following command::

    $ mysqladmin -u root -p password NEW_PASSWORD
    Enter password <enter your old password>

Create the Designate tables

::

    $ mysql -u root -p
    Enter password: <enter your password here>

    mysql> CREATE DATABASE `designate` CHARACTER SET utf8 COLLATE utf8_general_ci;
    mysql> CREATE DATABASE `designate_pool_manager` CHARACTER SET utf8 COLLATE utf8_general_ci;
    mysql> exit;


Install additional packages

::

    $ apt-get install libmysqlclient-dev
    $ pip install mysql-python


Installing BIND9
================

.. index::
    double: install; bind9

Install the DNS server, BIND9

::

      $ apt-get install bind9

      # Update the BIND9 Configuration
      $ editor /etc/bind/named.conf.options

      # Change the corresponding lines in the config file:
      options {
        directory "/var/cache/bind";
        dnssec-validation auto;
        auth-nxdomain no; # conform to RFC1035
        listen-on-v6 { any; };
        allow-new-zones yes;
        request-ixfr no;
        recursion no;
      };

      # Disable AppArmor for BIND9
      $ touch /etc/apparmor.d/disable/usr.sbin.named
      $ service apparmor reload

      # Restart BIND9:
      $ service bind9 restart


Initialize & Start the Central Service
======================================

.. index::
   double: install; central


If you intend to run Designate as a non-root user, then sudo permissions need to be granted

::

   $ echo "designate ALL=(ALL) NOPASSWD:ALL" | sudo tee -a /etc/sudoers.d/90-designate
   $ sudo chmod 0440 /etc/sudoers.d/90-designate


::

   # Sync the Designate database:
   $ designate-manage database sync

   # Start the central service:
   $ designate-central


You'll now be seeing the log from the central service.

Initialize & Start the API Service
==================================

.. index::
   double: install; api

Open up a new ssh window and log in to your server (or however you’re communicating with your server).

::

   $ cd /var/lib/designate

   # Make sure your virtualenv is sourced
   $ source .venv/bin/activate

   # Start the API Service
   $ designate-api

You’ll now be seeing the log from the API service.


Initialize & Start the Pool Manager Service
===========================================

.. index::
   double: install; pool-manager

Open up a new ssh window and log in to your server (or however you’re communicating with your server).

::

   # Sync the Pool Manager's cache:
   $ designate-manage pool-manager-cache sync

   # Start the pool manager service:
   $ designate-pool-manager


You'll now be seeing the log from the Pool Manager service.


Initialize & Start the MiniDNS Service
===========================================

.. index::
   double: install; minidns

Open up a new ssh window and log in to your server (or however you’re communicating with your server).

::

   # Start the minidns service:
   $ designate-mdns


You'll now be seeing the log from the MiniDNS service.

Exercising the API
==================

.. note:: If you have a firewall enabled, make sure to open port 53, as well as Designate's default port (9001).

Using a web browser, curl statement, or a REST client, calls can be made to the
Designate API using the following format where "api_version" is either v1 or v2
and "command" is any of the commands listed under the corresponding version at :ref:`rest`

::

   http://IP.Address:9001/api_version/command

You can find the IP Address of your server by running

::

   curl -s checkip.dyndns.org | sed -e 's/.*Current IP Address: //' -e 's/<.*$//'

A couple of notes on the API:

- Before Domains are created, you must create a server (/v1/servers).
