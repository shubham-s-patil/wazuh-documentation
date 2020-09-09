.. Copyright (C) 2020 Wazuh, Inc.

.. _wazuh-container:

Wazuh Docker deployment
=======================

- `Requirements`_
- `Usage`_
- `Exposed ports`_

Requirements
------------

- `Container memory`_
- `Increase max_map_count on your host (Linux)`_
- `Increase max_map_count on your host (Windows)`_
- `SELinux`_
- `Docker for OSX`_

Container memory
^^^^^^^^^^^^^^^^

It is recommended to set Docker host preferences to give at least **6GB** memory for the host that created the containers (this doesn't necessarily mean they all will use it, but **Elasticsearch** requires them to work properly).

Increase max_map_count on your host (Linux)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. You need to increase ``max_map_count`` on your Docker host:

    .. code-block:: console

      # sysctl -w vm.max_map_count=262144

#. To set this value permanently, update the vm.max_map_count setting in ``/etc/sysctl.conf``. To verify after rebooting, run "sysctl vm.max_map_count".

    .. warning::

      If you don't set the **max_map_count** on your host, Elasticsearch will probably NOT work.

Increase max_map_count on your host (Windows)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. You need to increase ``max_map_count`` on your Docker host:

    .. code-block:: console

      $ docker-machine ssh default
      # sysctl -w vm.max_map_count=262144
      # exit

#. To set this value permanently, update the vm.max_map_count setting in ``/var/lib/boot2docker/profile``:

  2.1. Open the file ``/var/lib/boot2docker/bootlocal.sh`` for edition:

     .. code-block:: console

      $ docker-machine ssh default
      # vi /var/lib/boot2docker/bootlocal.sh

  2.2 Add the following line into the profile file:

     .. code-block:: console

      sysctl -w vm.max_map_count=262144

  2.3. Make the script runnable:

     .. code-block:: console

      # chmod +x /var/lib/boot2docker/bootlocal.sh

  2.4. To verify after rebooting, run "sysctl vm.max_map_count".

    .. warning::

      If you don't set the **max_map_count** on your host, Elasticsearch will probably NOT work.

SELinux
^^^^^^^

On distributions with SELinux enabled out-of-the-box, you will need to either re-context the files or put SELinux into Permissive mode for docker-elk to start properly. For example, on Red Hat and CentOS the following command will apply the proper context:

.. code-block:: console

  # chcon -R system_u:object_r:admin_home_t:s0 docker-elk/

Docker for OSX
^^^^^^^^^^^^^^

In Docker for OSX, there is a default memory limit of 2GB, so in order to run `docker-compose up` successfully you have to change default memory settings from 2GB to at least 4 or 5GB. To do so, click on the Docker icon in the menu bar, then on "Preferences...", go to the "Advanced" tab and set 5GB of memory, and finally click on "Apply & Restart" and run `docker-compose up`.

Usage
-----

Demo deployment
^^^^^^^^^^^^^^^

This configuration is provided as-is for testing purposes only, it is designed for maximum ease of use to get you started fast with Wazuh. For real production environments see `Production deployment`_.


1. Cone the Wazuh repository to your system:

  .. code-block:: console

    $ git clone https://github.com/wazuh/wazuh-docker.git -b v|WAZUH_LATEST_DOCKER|_|ELASTICSEARCH_LATEST_DOCKER| --depth=1

2. Start Wazuh and Elastic Stack using `docker-compose`. From the directory where you have the ``docker-compose.yml`` file:

   a) Foreground::

      $ docker-compose up

   b) Background::

      $ docker-compose up -d

.. note::
  Wazuh-kibana container will run multiple queries to Elasticsearch API using curl, to learn when Elasticsearch is up. It is expected to see several ``Failed to connect to elasticsearch port 9200`` log messages, until Elasticsearch is started. Then the set up process will continue normally.


Production deployment
^^^^^^^^^^^^^^^^^^^^^

These are the steps to deploy a production grade Wazuh cluster using the "Open Distro for Elasticsearch" Docker images.


1. Clone the Wazuh repository to your system:

  .. code-block:: console

    $ git clone https://github.com/wazuh/wazuh-docker.git -b v|WAZUH_LATEST_DOCKER|_|ELASTICSEARCH_LATEST_DOCKER| --depth=1

  We'll use ``production-cluster.yml`` as base for this deployment, all code snippets on this section come from this file.

2. Secure traffic by replacing demo certificates

  2.1 Generate certificates for each node of the cluster

    We made a Docker image to automate certificate generation using `SearchGuard TLS Tool <https://docs.search-guard.com/latest/offline-tls-tool>`_, modify the file ``ssl_certs/certs.yml`` and execute the following command to obtain the desired certificates::

      docker-compose -f generate-opendistro-certs.yml run --rm generator

    This will save the certificates into the **ssl_certs** directory as well as the config snippets for each one of the nodes.

  2.2 Setup SSL certificates for Elasticsearch on the directory **ssl_certs**. Check out the section `Docker Security <https://opendistro.github.io/for-elasticsearch-docs/docs/install/docker-security/>`_ from the Open Distro docs.

    .. code-block:: yaml

      - ./ssl_certs/root-ca.pem:/usr/share/elasticsearch/config/root-ca.pem
      - ./ssl_certs/node.key:/usr/share/elasticsearch/config/node.key
      - ./ssl_certs/node.pem:/usr/share/elasticsearch/config/node.pem
      - ./elastic_opendistro/custom-elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - ./elastic_opendistro/internal_users.yml:/usr/share/elasticsearch/plugins/opendistro_security/securityconfig/internal_users.yml


  2.3 Use a secure password for the admin user on Elasticsearch

    You have complete freedom to customize users on the Elasticsearch container by mounting your own ``internal_users.yml``::

      - ./elastic_opendistro/internal_users.yml:/usr/share/elasticsearch/plugins/opendistro_security/securityconfig/internal_users.yml

    It is possible to generate a hash using the same Docker image, type in any password when prompted and replace the hash on ``internal_users.yml``::

      docker run --rm -ti amazon/opendistro-for-elasticsearch:|OPENDISTRO_LATEST_DOCKER| bash /usr/share/elasticsearch/plugins/opendistro_security/tools/hash.sh


  2.4 Setup SSL certificates for filebeat on the Wazuh container

    .. code-block:: yaml

      environment:
        - FILEBEAT_SSL_VERIFICATION_MODE=full
        - SSL_CERTIFICATE_AUTHORITIES=/etc/filebeat/root-ca.pem
        - SSL_CERTIFICATE=/etc/filebeat/filebeat.pem
        - SSL_KEY=/etc/filebeat/filebeat.key
      volumes:
        - ./ssl_certs/root-ca.pem:/etc/filebeat/root-ca.pem
        - ./ssl_certs/filebeat.pem:/etc/filebeat/filebeat.pem
        - ./ssl_certs/filebeat.key:/etc/filebeat/filebeat.key

  2.4 Setup SSL certificates for Kibana

    Copy your own certificate into the ``kibana_od_ssl`` directory and set ``SERVER_SSL_ENABLED`` to **true**:

    .. code-block:: yaml

      environment:
        - SERVER_SSL_ENABLED=true
        - SERVER_SSL_CERTIFICATE=/usr/share/kibana/config/cert.pem
        - SERVER_SSL_KEY=/usr/share/kibana/config/key.pem
      volumes:
        - ./kibana_od_ssl/cert.pem:/usr/share/kibana/config/cert.pem
        - ./kibana_od_ssl/key.pem:/usr/share/kibana/config/key.pem


3. Start Wazuh and Elastic Stack using ``docker-compose``:

   a) Foreground::

      $ docker-compose -f production-cluster.yml up

   b) Background::

      $ docker-compose -f production-cluster.yml up -d




Exposed ports
-------------

By default, the stack exposes the following ports:

+-----------+-----------------------------+
| **1514**  | Wazuh TCP                   |
+-----------+-----------------------------+
| **1515**  | Wazuh TCP                   |
+-----------+-----------------------------+
| **514**   | Wazuh UDP                   |
+-----------+-----------------------------+
| **55000** | Wazuh API                   |
+-----------+-----------------------------+
| **9200**  | Elasticsearch HTTP          |
+-----------+-----------------------------+
| **443**   | Kibana HTTPS                |
+-----------+-----------------------------+

.. note::
  Configuration is not dynamically reloaded, so you will need to restart the stack after any change in the configuration of a component.
