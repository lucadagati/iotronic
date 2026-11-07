=========
IoTronic
=========

IoTronic is an Internet of Things resource management service for OpenStack clouds. It lets a cloud operator register IoT boards (running the *Lightning-rod* agent), group them in fleets, inject plugins on them, expose their services and web services, and attach them to Neutron networks, all through an OpenStack-style REST API.

* Free software: Apache License 2.0
* Upstream source: https://opendev.org/openstack/iotronic (originally ``git.openstack.org/git/openstack/iotronic``)
* This repository: Python 3 / 3.12 port of the service, https://github.com/lucadagati/iotronic

.. contents:: Contents
   :local:
   :depth: 2

1. What it is and why it exists
===============================

IoT devices (sensors, actuators, single-board computers) are usually at the network edge, behind NAT, with intermittent connectivity. IoTronic treats them as cloud resources:

* boards connect *out* to a WAMP router (Crossbar) so they can be reached without inbound ports;
* the control plane (API, conductor, WAMP agent) keeps the inventory in a SQL database, authenticates users through Keystone and issues commands to boards as WAMP RPC calls;
* the operator manages boards, plugins (code run on the board), services (TCP ports tunnelled through the cloud), fleets, ports (Neutron VIFs) and web services (DNS names via Designate and an nginx proxy) with a uniform API.

This repository contains only the cloud side. The board side is `iotronic-lightning-rod <https://opendev.org/x/iotronic-lightning-rod>`_.

2. Architecture
===============

::

    user / CLI / dashboard
            |  HTTPS, Keystone token
            v
    +-------------------+  RPC (oslo.messaging, RabbitMQ)  +-----------------------+
    |  iotronic-api     | -------------------------------> |  iotronic-conductor   |
    |  (pecan + WSME,   |                                  |  business logic,      |
    |   Apache mod_wsgi |                                  |  periodic tasks       |
    |   on :8812)       |                                  +----------+------------+
    +---------+---------+                                             |  RPC
              |  SQLAlchemy                                           v
              v                                              +-----------------------+   WAMP (autobahn)
        MySQL / MariaDB  <----------------------------------|  iotronic-wamp-agent  |<----------------+
        database "iotronic"                                  |  registration agent   |                 |
                                                             +-----------+-----------+          +------+------+
                                                                         | WAMP                 |  Crossbar   |
                                                                         +--------------------->|  router     |
                                                                                                |  realm s4t  |
                                                                                                +------+------+
                                                                                                       | WebSocket :8181
                                                                                                +------v------+
                                                                                                | Lightning-  |
                                                                                                | rod on board|
                                                                                                +-------------+

Components and entry points (``setup.cfg``):

.. csv-table::
   :header-rows: 1
   :widths: auto

   "Process","Entry point","Role"
   "API","``iotronic/wsgi/app.wsgi`` (WSGI)","REST API ``/`` and ``/v1`` (boards, plugins, services, fleets, ports, webservices, requests, results)"
   "Conductor","``iotronic-conductor``","Executes API requests, talks to Neutron/Designate, schedules board actions"
   "WAMP agent","``iotronic-wamp-agent``","Connects to Crossbar, handles board registration and session heartbeat"
   "DB migrations","``iotronic-dbsync``","Alembic migrations (``upgrade``, ``version``, ``stamp``, ``revision``, ``create_schema``)"

Database tables (created by the migrations): ``conductors``, ``wampagents``, ``boards``, ``locations``, ``sessions``, ``plugins``, ``injection_plugins``, ``services``, ``exposed_services``, ``ports_on_boards``, ``fleets``, ``webservices``, ``enabled_webservices``, ``requests``, ``results``.

3. Requirements
===============

* Linux (Ubuntu is the reference platform); Python 3.12 is the declared target (``tox.ini``, ``setup.cfg``). The checks described in section 8 were run with Python 3.11.
* A working OpenStack control plane: SQL database (MySQL/MariaDB), RabbitMQ, Memcached and Keystone. Neutron and Designate only for the port and web-service features.
* Crossbar (WAMP router) reachable by the agent and by the boards.
* Python packages from ``requirements.txt``: pbr, eventlet, oslo.config/log/concurrency/policy/messaging/db, paramiko, SQLAlchemy (``<2``), keystonemiddleware, autobahn, python-neutronclient, python-designateclient, pecan, PyMySQL, osprofiler, WSME. Two upper bounds were added after testing: ``oslo.db<15`` (newer releases removed ``EngineFacade``, which the DB layer uses) and ``SQLAlchemy<2``.
* Apache 2 with ``mod_wsgi`` for the API (a plain WSGI server also works for experiments, see section 6), and optionally nginx for the web-service proxy.

4. Installation
===============

The scenario below uses three hosts: a *controller* (MySQL, Keystone, RabbitMQ), an *IoTronic host* (conductor, WAMP agent, API, Crossbar) and the boards. Everything except the board can share one machine. **A complete OpenStack plus IoTronic installation was not verified in this environment**; the steps follow the repository files, and the database, import and API steps were exercised as described in section 8.

4.1 Controller host
-------------------

Install the SQL database, message queue, Memcached and Keystone following the `OpenStack documentation <https://docs.openstack.org/>`_.

Create the database::

    CREATE DATABASE iotronic;
    GRANT ALL PRIVILEGES ON iotronic.* TO iotronic@'localhost' IDENTIFIED BY '<IOTRONIC_DBPASS>';
    GRANT ALL PRIVILEGES ON iotronic.* TO iotronic@'%' IDENTIFIED BY '<IOTRONIC_DBPASS>';

Register the service, user, roles and endpoints in Keystone::

    openstack service create iot --name Iotronic
    openstack user create --password-prompt iotronic
    openstack role add --project service --user iotronic admin
    openstack role create admin_iot_project
    openstack role create manager_iot_project
    openstack role create user_iot
    openstack role add --project service --user iotronic admin_iot_project

    openstack endpoint create --region RegionOne iot public   http://<IP_IOTRONIC>:8812
    openstack endpoint create --region RegionOne iot internal http://<IP_IOTRONIC>:8812
    openstack endpoint create --region RegionOne iot admin    http://<IP_IOTRONIC>:8812

4.2 Crossbar (WAMP router)
--------------------------

::

    apt install python3-pip libsnappy-dev libssl-dev libffi-dev python3-dev
    pip install python-snappy crossbar
    mkdir /etc/crossbar

Create ``/etc/crossbar/config.json`` with one router worker, realm ``s4t`` and a WebSocket transport on TCP 8181::

    {
       "version": 2,
       "controller": {},
       "workers": [
          {
             "type": "router",
             "realms": [
                {
                   "name": "s4t",
                   "roles": [
                      {
                         "name": "anonymous",
                         "permissions": [
                            { "uri": "*",
                              "allow": { "publish": true, "subscribe": true,
                                         "call": true, "register": true } }
                         ]
                      }
                   ]
                }
             ],
             "transports": [
                {
                    "type": "websocket",
                    "endpoint": { "type": "tcp", "port": 8181 },
                    "debug": true,
                    "options": {
                      "enable_webstatus": true,
                      "fail_by_drop": true,
                      "open_handshake_timeout": 2500,
                      "close_handshake_timeout": 1000,
                      "auto_ping_interval": 10000,
                      "auto_ping_timeout": 5000,
                      "auto_ping_size": 4
                    }
                }
             ]
          }
       ]
    }

Note that this configuration lets any anonymous client publish, subscribe, call and register on every URI. It is suitable for a lab only; see section 10.

Create ``/etc/systemd/system/crossbar.service``::

    [Unit]
    Description=Crossbar.io
    After=network.target

    [Service]
    Type=simple
    User=root
    Group=root
    StandardInput=null
    StandardOutput=journal
    StandardError=journal
    ExecStart=/usr/local/bin/crossbar start --cbdir=/etc/crossbar/
    ExecStop=/usr/local/bin/crossbar stop --cbdir=/etc/crossbar/
    Restart=on-abort

    [Install]
    WantedBy=multi-user.target

Then ``systemctl daemon-reload && systemctl enable --now crossbar``.

4.3 IoTronic
------------

::

    useradd -m -d /var/lib/iotronic iotronic
    git clone https://github.com/lucadagati/iotronic.git
    cd iotronic
    python3 -m venv /opt/iotronic-venv      # optional; the systemd units below assume /usr/local/bin
    pip install -r requirements.txt
    pip install .
    mkdir -p /var/log/iotronic
    chown -R iotronic:iotronic /var/log/iotronic

``setup.cfg`` installs the configuration into ``/etc/iotronic`` (``iotronic.conf``, ``policy.json``), the WSGI script into ``/var/www/cgi-bin/iotronic/app.wsgi`` and the two systemd units into ``/etc/systemd/system``. The units run ``/usr/local/bin/iotronic-conductor`` and ``/usr/local/bin/iotronic-wamp-agent``: if you install into a virtual environment, edit ``ExecStart`` accordingly. ``pbr`` derives the package version from git, so install from a clone, not from a plain tarball (or export ``PBR_VERSION``).

Edit ``/etc/iotronic/iotronic.conf`` (section 5). On the host that runs the registration agent set ``register_agent = True`` in ``[wamp]``: there must be exactly one such agent.

Create the schema::

    iotronic-dbsync upgrade

4.4 API (Apache)
----------------

::

    apt-get install apache2 libapache2-mod-wsgi-py3
    touch /var/log/iotronic/iotronic-api_error.log /var/log/iotronic/iotronic-api_access.log
    chown -R iotronic:iotronic /var/log/iotronic
    cp etc/apache2/iotronic.conf /etc/apache2/sites-available/iotronic.conf
    a2ensite iotronic
    systemctl restart apache2

The virtual host listens on port 8812 and runs the WSGI app as user ``iotronic``. ``etc/apache2/iotronic-ssl.conf`` is the TLS variant.

4.5 Start the services
----------------------

::

    systemctl enable --now iotronic-wamp-agent
    systemctl enable --now iotronic-conductor

4.6 Boards
----------

Follow the `iotronic-lightning-rod README <https://opendev.org/x/iotronic-lightning-rod>`_ and point the board to the Crossbar URL and realm ``s4t``.

5. Configuration
================

``/etc/iotronic/iotronic.conf`` (template: ``etc/iotronic/iotronic.conf``). Placeholders in angle brackets must be replaced. Keys in the template and defaults from the code:

.. csv-table::
   :header-rows: 1
   :widths: auto

   "Section / key","Template value","Code default","Meaning"
   "DEFAULT transport_url","rabbit://<user>:<password>@<host>:5672/","none","RabbitMQ URL for RPC between API, conductor and agent"
   "DEFAULT debug","True","oslo default","Verbose logging (set False in production)"
   "DEFAULT proxy","nginx","see iotronic/wamp/agent.py","Proxy module in iotronic/wamp/proxies"
   "DEFAULT auth_strategy","keystone","keystone","keystone or noauth (never use noauth in production)"
   "DEFAULT pecan_debug","commented","false","Pecan debug mode, insecure"
   "conductor service_port_min / service_port_max","50000 / 60000","50000 / 60000","Port range used for exposed board services"
   "conductor heartbeat_timeout","not in template","60","Seconds before a board is considered lost"
   "conductor api_url","not in template","unset","URL of the API; otherwise taken from the Keystone catalog"
   "wamp wamp_transport_url","ws://<host>:<port>/","ws://localhost:8181/","Crossbar WebSocket URL"
   "wamp wamp_realm","s4t","s4t","WAMP realm"
   "wamp register_agent","commented","False","Run the registration agent (exactly one per deployment)"
   "wamp skip_cert_verify","commented","False","Skip TLS verification of wss:// URLs"
   "wamp autoPingInterval / autoPingTimeout","not in template","2 / 2","WebSocket ping settings"
   "wamp service_allow_list / service_allow_list_path","not in template","False / (/var/lib/wstun/allowlist)","wstun allow-list integration"
   "database connection","mysql+pymysql://<user>:<password>@<host>/iotronic","none","SQLAlchemy URL (MySQL/MariaDB only, see section 10)"
   "keystone_authtoken","www_authenticate_uri, auth_url, username=iotronic, project_name=service, ...","none","keystonemiddleware settings"
   "neutron / designate","URL, credentials, region","localhost defaults","Needed for ports and web services"


Other files: ``etc/iotronic/policy.json`` (empty overrides; the defaults live in ``iotronic/common/policy.py``: roles ``admin``, ``admin_iot_project``, ``manager_iot_project``, ``user_iot``), ``etc/apache2/iotronic.conf`` (port 8812, 5 threads, log paths), ``etc/systemd/system/*.service``.

6. Usage and examples
=====================

With Keystone, request a token and call the API at the endpoint registered above::

    TOKEN=$(openstack token issue -f value -c id)
    curl -H "X-Auth-Token: $TOKEN" http://<IP_IOTRONIC>:8812/v1/boards

The API root is ``GET /`` (version document) and ``GET /v1`` (links to ``boards``, ``plugins``, ``services``, ``ports``, ``webservices``, ``fleets`` and ``requests``). Main resources, from ``iotronic/api/controllers/v1``:

* ``/v1/boards``: ``GET`` list (and ``/detail``), ``POST`` create (required body fields: ``name``, ``code``, ``location``), ``GET/PATCH/DELETE`` one board by uuid or name, ``POST /v1/boards/action`` and per-board sub-resources ``plugins``, ``services``, ``ports`` and ``webservices``;
* ``/v1/plugins``, ``/v1/services``, ``/v1/fleets``, ``/v1/webservices``, ``/v1/requests``, ``/v1/results`` with the same list/detail/create/update/delete pattern.

Authorization for every call is checked against the policy rules (for example ``iot:board:create``).

Experiment without Keystone (a lab setup that was exercised here, see section 8): set ``auth_strategy=noauth`` and run the WSGI application with any WSGI server, for example::

    # api.py
    from wsgiref.simple_server import make_server
    import oslo_i18n as i18n
    from iotronic.api import app
    from iotronic.common import service
    i18n.install('iotronic')
    service.prepare_service(['api', '--config-file', '/etc/iotronic/iotronic.conf'])
    make_server('127.0.0.1', 8812, app.VersionSelectorApplication()).serve_forever()

Output captured from that run (empty database)::

    $ curl http://127.0.0.1:8812/
    {"name": "OpenStack Iotronic API", "description": "Iotronic is an OpenStack project which aims to provision baremetal machines.", "versions": [{"id": "v1", ...}], "default_version": {...}}
    $ curl http://127.0.0.1:8812/v1/boards
    {"boards": []}
    $ curl http://127.0.0.1:8812/v1/plugins
    {"plugins": []}

Creating boards, injecting plugins and running actions additionally require the conductor, the WAMP agent, Crossbar and a connected Lightning-rod: **not verified in this environment**.

7. Repository layout
====================

.. csv-table::
   :header-rows: 1
   :widths: auto

   "Path","Content"
   "iotronic/api/","pecan application, hooks (context, db, rpc), Keystone middleware, controllers/v1/* (one module per resource)"
   "iotronic/conductor/","manager.py (service), endpoints.py (RPC methods), rpcapi.py (client), provisioner.py"
   "iotronic/wamp/","agent.py (WAMP agent), functions.py (RPC exposed to boards), proxies/ (nginx proxy for web services)"
   "iotronic/db/","DB API facade, SQLAlchemy models, Alembic migrations (sqlalchemy/alembic/versions)"
   "iotronic/objects/","Versioned objects (board, plugin, service, fleet, port, ...)"
   "iotronic/common/","config, context, policy, exceptions, Neutron/Designate/Keystone clients, utils"
   "iotronic/cmd/","conductor, wamp_agent, dbsync entry points"
   "iotronic/openstack/common/","Vendored oslo-incubator modules (service, periodic tasks, looping calls)"
   "etc/","iotronic.conf, policy.json, Apache and systemd files"
   "utils/","iotronic.sql, all.sql, loaddb, remove_board.sql, sample plugins in test_plugins/"
   "doc/source/","Sphinx sources (installation, usage and contributing stubs)"
   "setup.cfg, setup.py, tox.ini","pbr packaging, console scripts, tox environments (py312, pep8)"


8. Testing and verification
===========================

The repository has no unit-test suite (``test-requirements.txt`` only lists linting and runtime packages; ``tox -e pep8`` runs flake8). What was verified, in a throwaway virtual environment outside the repository (Python 3.11) and a throwaway MariaDB 11 container:

* ``python3 -I -m compileall iotronic`` succeeds.
* ``iotronic-dbsync upgrade`` applies all eight migrations (``df35e9cbeaff`` up to ``10460765f337``) on MariaDB, ``iotronic-dbsync version`` prints ``10460765f337`` and the 15 tables listed in section 2 are created. The same command **fails on SQLite** (``No support for ALTER of constraints``): the migrations require MySQL/MariaDB.
* With ``auth_strategy=noauth`` the API answered ``GET /``, ``/v1``, ``/v1/boards``, ``/v1/plugins`` and ``/v1/fleets`` with HTTP 200 (outputs in section 6) using ``oslo.db 14.1.1`` and ``SQLAlchemy 1.4.54``.

Bugs found and fixed while verifying:

* ``iotronic/common/context.py`` used attributes (``user``, ``tenant``, ``user_domain``) removed from recent ``oslo.context`` releases, so every API request failed with ``AttributeError: 'RequestContext' object has no attribute 'user'``. It now uses ``user_id``, ``project_id`` and ``user_domain_id``.
* With ``oslo.db >= 15`` or ``SQLAlchemy 2`` the DB layer fails with ``module 'oslo_db.sqlalchemy.session' has no attribute 'EngineFacade'``; ``requirements.txt`` now bounds ``oslo.db<15`` and ``SQLAlchemy<2``.

Not verified: the conductor and WAMP agent runtime, Keystone authentication, RabbitMQ RPC, Crossbar, Neutron, Designate, nginx proxy, Apache ``mod_wsgi`` deployment, systemd units, and any board interaction.

9. Troubleshooting
==================

.. csv-table::
   :header-rows: 1
   :widths: auto

   "Symptom","Cause and fix"
   "NoSuchOptError: no such option connection in group [database]","A module was imported before the DB options were registered, or the [database] section is missing: use the entry points and set connection."
   "iotronic-dbsync: No support for ALTER of constraints","SQLite is not supported; use MySQL/MariaDB."
   "AttributeError: 'RequestContext' object has no attribute 'user'","Old copy of context.py with a recent oslo.context; update."
   "module 'oslo_db.sqlalchemy.session' has no attribute 'EngineFacade'","Install with the pins in requirements.txt (oslo.db<15, SQLAlchemy<2)."
   "HTTP 401 on every call","Missing or invalid X-Auth-Token, or wrong [keystone_authtoken]."
   "HTTP 403 from the API","The user lacks one of the IoTronic roles (see section 5)."
   "Board stays offline","Crossbar unreachable from the board, wrong realm, or no WAMP agent with register_agent = True."
   "Apache serves nothing on 8812","mod_wsgi not enabled, wrong log directory ownership, or a2ensite iotronic not run."
   "pbr: Versioning for this project requires either an sdist tarball, or access to an upstream git repository","Install from a git clone or set PBR_VERSION."


10. Known limitations and security notes
========================================

* MySQL/MariaDB only for migrations (SQLite fails).
* ``DEFAULT debug=True`` is set in the template and ``noauth`` removes all authentication: use them only in a lab.
* The sample Crossbar realm grants anonymous clients every permission on every URI: restrict roles and add authentication before exposing the router.
* The template contains placeholders for passwords only; never commit a filled ``iotronic.conf``. The Apache virtual host and TLS variant are examples; terminate TLS in front of the API.
* ``iotronic/conductor/endpoints.py`` (``action_board``) contains ``except exception:`` where ``exception`` is a module, so an error during ``execute_on_board`` raises ``TypeError`` instead of the original error; it was not changed here.
* ``iotronic-dbsync online_data_migrations`` has no migrations registered and its version check is commented out.
* The code base is an older OpenStack project ported to Python 3.12 by changing syntax and packaging; it has not been validated against current OpenStack releases.

11. Extending IoTronic
======================

* **New API resource**: add an object in ``iotronic/objects``, a model and an Alembic revision (``iotronic-dbsync revision -m "msg" --autogenerate``), DB API methods in ``iotronic/db/sqlalchemy/api.py``, conductor RPC in ``iotronic/conductor/endpoints.py`` and ``rpcapi.py``, a controller in ``iotronic/api/controllers/v1`` mounted in ``v1/__init__.py``, and a policy rule in ``iotronic/common/policy.py``.
* **New board action**: add the action name to ``ACTIONS`` in ``iotronic/objects/board.py``; it is executed through ``action_board``.
* **Different web-service proxy**: add a module in ``iotronic/wamp/proxies`` implementing the ``proxy.Proxy`` interface and set ``proxy=<module>``.
* **Plugins for boards**: sample plugins are in ``utils/test_plugins``.
* **Style**: ``tox -e pep8`` (flake8; ignore list in ``tox.ini``). Contribution notes: ``CONTRIBUTING.rst``, ``HACKING.rst``.
