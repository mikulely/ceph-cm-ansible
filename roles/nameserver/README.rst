nameserver
==========

This role is used to set up and configure a very basic **internal** BIND DNS master server.

This role has only been tested on CentOS 7.2 using BIND9.

It does the following:

- Installs and updates necessary packages
- Enables and configures firewalld
- Manages named.conf and BIND daemon config
- Manages forward and reverse DNS records

Prerequisites
+++++++++++++

- CentOS 7.2

Variables
+++++++++
Most variables are defined in ``roles/nameserver/defaults/main.yml`` and values are chosen to support our Sepia_ lab.  They can be overridden in the ``secrets`` repo.

+--------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------+
|Variable                                                |Description                                                                                                                |
+========================================================+===========================================================================================================================+
|``packages: []``                                        |A list of packages to install that is specific to the role.  The list is defined in ``roles/nameserver/vars/packages.yml`` |
+--------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------+
|``named_conf_dir: "/var/named"``                        |BIND main configuration directory.  Defined in ``roles/nameserver/defaults/main.yml``                                      |
+--------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------+
|``named_conf_file: "/etc/named.conf"``                  |BIND main configuration file.  This is the default CentOS path.  Defined in ``roles/nameserver/defaults/main.yml``         |
+--------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------+
|``named_conf_data_dir: "/var/named/data"``              |BIND data directory.  named debug output and statistics are stored here.  Defined in ``roles/nameserver/defaults/main.yml``|
+--------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------+
|``named_conf_listen_port: 53``                          |Port BIND should listen on.  Defined in ``roles/nameserver/defaults/main.yml``                                             |
+--------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------+
|::                                                      |                                                                                                                           |
|                                                        |                                                                                                                           |
|  named_conf_listen_iface:                              |Interface(s) BIND should listen on.  This defaults to listen on all IPv4 interfaces Ansible detects for the nameserver.    |
|    - 127.0.0.1                                         |Defined in ``roles/nameserver/defaults/main.yml``                                                                          |
|    - "{{ ansible_all_ipv4_addresses[0] }}"             |                                                                                                                           |
|                                                        |                                                                                                                           |
+--------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------+
|``named_conf_zones_path: "/var/named/zones"``           |Path to BIND zone files.  Defined in ``roles/nameserver/defaults/main.yml``                                                |
+--------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------+
|::                                                      |named daemon options.  Writes to ``/etc/sysconfig/named``.  Defined in ``roles/nameserver/defaults/main.yml``              |
|                                                        |                                                                                                                           |
|  named_conf_daemon_opts: []                            |                                                                                                                           |
|                                                        |                                                                                                                           |
|  # Example for IPv4 support only:                      |                                                                                                                           |
|   named_conf_daemon_opts: "-4"                         |                                                                                                                           |
|                                                        |                                                                                                                           |
+--------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------+
|::                                                      |Values used to populate corresponding settings in each zone file's SOA record                                              |
|                                                        |Defined in ``roles/nameserver/defaults/main.yml``                                                                          |
|  named_conf_soa_ttl: 3600                              |                                                                                                                           |
|  named_conf_soa_refresh: 3600                          |                                                                                                                           |
|  named_conf_soa_retry: 3600                            |                                                                                                                           |
|  named_conf_soa_expire: 604800                         |                                                                                                                           |
|                                                        |                                                                                                                           |
+--------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------+
|::                                                      |Desired primary nameserver and admin e-mail for each zone file.  Defined in the secrets repo                               |
|                                                        |                                                                                                                           |
|  named_conf_soa: []                                    |                                                                                                                           |
|                                                        |                                                                                                                           |
|  # Example:                                            |                                                                                                                           |
|  named_conf_soa: "ns1.example.com. admin.example.com." |                                                                                                                           |
|                                                        |                                                                                                                           |
+--------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------+

**named_domains: []**

The ``named_domains`` dictionary is the bread and butter of creating zone files.  It is in standard YAML syntax.  Each domain (key) must have ``forward`` and ``ipvar`` defined although ``ipvar`` can be set to ``NULL``.  Optional values include ``miscrecords`` and ``reverse``.

``forward``
  The domain of the forward lookup zone for each domain (key)

``ipvar``
  The variable assigned to a system in the Ansible inventory.  This allows systems to have multiple IPs assigned for a front and ipmi network, for example.  See **Inventory Example** below.

``miscrecords``
  Records to add to corresponding ``forward`` zone file.  This is a good place for CNAMEs and MX records and records for hosts you don't have in your Ansible inventory.  If your main nameserver is in a subdomain, you should create its glue record here.  See example.

``reverse``
  This should be a list of each reverse lookup IP C-Block address corresponding to the domain (key).  See example.

**Example**::

    named_domains:
      example.com:
        ipvar: NULL
        forward: example.com
        miscrecords:
          - www                 IN      A       8.8.8.8
          - www                 IN      TXT     "my www host"
          - ns1.private         IN      A       192.168.0.1
      private.example.com:
        ipvar: ip
        forward: private.example.com
        miscrecords:
          - mail                IN      MX      192.168.0.2
          - email               IN      CNAME   mail
        reverse:
          - 192.168.0.0
          - 192.168.1.0
          - 192.168.2.0
      mgmt.example.com:
        ipvar: mgmt
        forward: mgmt.example.com
        reverse:
          - 192.168.10.0
          - 192.168.11.0
          - 192.168.12.0
        
Inventory
+++++++++
This role will create forward and reverse DNS records for any host defined in your Ansible inventory when given an IP address assigned to a variable matching ``ipvar`` in ``named_domains``.

Using the ``named_domains`` example above and inventory below, forward *and reverse* records for ``ns1.private.example.com``, ``tester050.private.example.com``, and ``tester050.mgmt.example.com`` would be created.

**Example**::

    [nameserver]
    ns1.private.example.com ip=192.168.0.1

    [testnodes]
    tester050.private.example.com ip=192.168.1.50 mgmt=192.168.11.50

**Note:** Hosts in inventory with no IP address defined will not have records created and should be added to ``miscrecords`` in ``named_domains``.

Tags
++++

packages
    Install *and update* packages

config
    Configure and restart named service (if config changes)

firewall
    Enable firewalld and allow dns traffic

records
    Compiles and writes forward and reverse zone files using ``named_domains`` and Ansible inventory

Dependencies
++++++++++++

This role depends on the following roles:

secrets
    Provides a var, ``secrets_path``, containing the path of the secrets repository, a tree of Ansible variable files.

sudo
    Sets ``ansible_sudo: true`` for this role which causes all the plays in this role to execute with sudo.

To-Do
+++++

- Allow additional user-defined firewall rules
- DNSSEC
- Dynamic DNS

.. _Sepia: https://ceph.github.io/sepia/
