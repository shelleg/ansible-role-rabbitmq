# ansible-role-rabbitmq

Ansible role for setting up and configuring a RabbitMQ server / cluster with or without ha.

This role is implemented as an example in the following github repo: [http://github.com/shelleg/ansible-playbook-rabbitmq](http://github.com/shelleg/ansible-playbook-rabbitmq)

Quickstart
----------

```
---
- hosts: rabbitmq_servers
  become: true
  vars_files:
    - "{{ playbook_dir }}/vars/rabbitmq-single.yml"
    - "{{ playbook_dir }}/vars/rabbitmq-ha.yml"
    # - "{{ playbook_dir }}/vars/rabbitmq-ha-shovels.yml"
  pre_tasks:
    # set hostname
    - name: main | changing hostname to match inventory_hostname_short
      hostname:
        name: "{{ inventory_hostname_short }}"
      when: ansible_hostname != inventory_hostname_short
    # when using vagrant eth1
    - set_fact:
        hosts_network_interface: eth1
      when: ansible_virtualization_type == "virtualbox"
  roles:
    - role: geerlingguy.pip
      pip_install_packages:
        - name: requests
    - role: ansible-role-hosts
    - role: ansible-role-rabbitmq
      # as cluster of n nodes (length of the rabbitmq_servers group)

```

Dependencies [ Optional ]
-------------------------

* `https://github.com/bertvv/ansible-role-hosts.git` - used for development purposes to set hostnames for cluster operation [ using master considering there was some issue with latest galaxy release ]
* `geerlingguy.pip` - used to install requests pip module required by rabbitmq ansible module

Variables
---------

**role switches:**

* `rabbitmq_purge: false` - **removes everything !**
* `rabbitmq_cleanup: false` = removes everything but data !
* `rabbitmq_backup: true` = used with purge and cleanup to backup first ... - needs more testing !

* `rabbitmq_setup_cluster: false` - setup cluster mode
* `rabbitmq_setup_ha: false` - setup ha mode

* `rabbitmq_guest_disable: false` - shoud the guest user be deleted or not
* `rabbitmq_management_plugin_enable: true` - enable management plugin for rabbitmqctl operations
```

**Other important variables:**
* `rabbitmq_inventory_group: rabbitmq_servers` inventory group to use for rabbitmq cluster
* `rabbitmq_master: []` - the first from the group will be elected as "master" / fist node of the cluster unless this variable is specified - should be a resolvable FQDN ...
* `rabbitmq_homedir` see Caveats section below.
* `rabbitmq_erlang_cookie` you want to change this, it's the key used by earlang to communicate between nodes


Caveats
-------
Changing the default data directory - although supported but quite delicate hence the default directory structure is preserved like so:

These vars create a rabbitmq-env.conf file used by rabbbitmq-server:
```
rabbitmq_homedir: /var/lib/rabbitmq
rabbitmq_config_dir: /etc/rabbitmq
rabbitmq_systemd_path: /etc/systemd/system/rabbitmq-server.service.d
# see -> https://www.rabbitmq.com/relocate.html
# rabbitmq_base e.g /data/rabbitmq
rabbitmq_base: "{{ rabbitmq_homedir }}"
rabbitmq_config_file: "{{ rabbitmq_config_dir }}/rabbitmq.conf"
rabbitmq_advanced_config_file: "{{ rabbitmq_config_dir }}/advanced.config"
rabbitmq_generated_config_dir: "{{ rabbitmq_config_dir }}/config/generated/"
rabbitmq_mnesia_base: "{{ rabbitmq_base }}/mnesia"
rabbitmq_mnesia_dir: "{{ rabbitmq_mnesia_base }}/rabbit@{{ ansible_hostname }}"
rabbitmq_schema_dir: "{{ rabbitmq_base }}/schema"
rabbitmq_log_base: /var/log/rabbitmq
rabbitmq_logs: "{{ rabbitmq_log_base }}/rabbit@{{ ansible_hostname }}.log"
rabbitmq_sasl_logs: "{{ rabbitmq_log_base }}/rabbit@{{ ansible_hostname }}-sasl.log"
rabbitmq_plugins_dir: "{{ rabbitmq_base }}/plugins"
rabbitmq_enabled_plugins_file: "{{ rabbitmq_config_dir }}/enabled_plugins"
rabbitmq_pid_file: "{{ rabbitmq_mnesia_dir }}.pid"

rabbitmq_erlang_cookie: 'SuperCaliFragilisticExpialiDocious'
rabbitmq_erlang_cookie_file: "{{ rabbitmq_base }}/.erlang.cookie"
```

Usage
-----

#### Setup a single server

**Example playbook:**
```
---
- hosts: rabbit1
  become: true
  become_user: root
  roles:
    - role: shelleg.rabbitmq

```

If you don't opt out on the default `rabbitmq_management` plugin installation
Once complete connect via http / amqp client to `http://<ip_address/dns_name>:15672` with custom users you may pass to the `rabbitmq_users` variable see adding users section below.

### Setup a cluster

* One of RabbitMQ's requirement (limitation) for clustering is that servers are reachable via DNS if you don't have DNS you can cheat by using hosts file (for the record I hate it but it's useful in some cases ...).
You can use `bertvv.ansible-role-hosts` role for this with any "fake info" you might want to push ... again for testing purposes only.

**Example playbook:**
```
---
- hosts: rabbitmq_servers
  become: true
  become_user: root
  roles:
    - role: shelleg.rabbitmq
      rabbitmq_setup_cluster: true
      rabbitmq_inventory_group: my_rabbits

```


### Setup a Highly Available Cluster
Requirements:

* DNS requirements apply here as described above.
* Enable HA by setting `rabbitmq_setup_ha: true`
3. Specify `rabbitmq_inventory_group` which defaults to: `rabbitmq_servers`

  This group will be scanned the first element will be elected as master and the rest of the group will be set as worker node / slave node.

  All Queues are replicated by default to all cluster members.

  This will set the /etc/rabbitmq/rabbit.conf file with the following info:

  `cluster_formation.classic_config.nodes.{{loop.index}} = rabbit@{{hostvars[host]['ansible_hostname']}}`

  e.g:
  ```
  cluster_formation.classic_config.nodes.1 = rabbit@rabbit1
  cluster_formation.classic_config.nodes.2 = rabbit@rabbit2
  cluster_formation.classic_config.nodes.3 = rabbit@rabbit3
  ```

**Example playbook:**
```
---
- hosts: rabbitmq_servers
  become: true
  become_user: root
  roles:
    - role: shelleg.rabbitmq
      rabbitmq_setup_cluster: true
      rabbitmq_setup_ha: true
      rabbitmq_inventory_group: my_rabbits

```


### Create RabbitMQ users like so:

See https://www.rabbitmq.com/management.html for more details:

```
rabbitmq_users:
  - name: 'rabbitmqadmin'
    password: 'rabbitmqadmin'
    vhost: '/'
    configure_priv: '.*'
    read_priv: '.*'
    write_priv: '.*'
    state: present
    tags: ['administrator']
```

### Create RabbitMQ Queues / Exchanges Bindings Shovels like so:

#### Queues
Queues is a straight forward list of variables ...
```
rabbitmq_queues:
  - name: foo
    max_length: 1000000 # optional
    vhost: 'my-vhost'
    durable: true # default
  - name: bar
```

these of course could be more complex as in:

```
rabbitmq_queues:
  - name: foo
  - name: my_queue
    message_ttl: 60000
    dead_letter_exchange: mqueue-receiver-aggregator-exchange
    dead_letter_routing_key: mqueue-receiver-aggregator-dead
```

This var is fed to the ##-queues.yml task file:

```
rabbitmq_queue:
  name: "{{ item.name }}"
  vhost: "{{ item.vhost | d('/') }}"
  durable: "{{ item.durable | d(true) }}"
  auto_delete: "{{ item.auto_delete | d(false) }}"
  arguments: "{{ item.arguments | d(omit) }}"
  state: "{{ item.state | d(omit) }}"
  message_ttl: "{{ item.message_ttl | d(omit) }}"
  dead_letter_exchange: "{{ item.dead_letter_exchange | d(omit) }}"
  dead_letter_routing_key: "{{ item.dead_letter_routing_key | d(omit) }}"
  max_length: "{{ item.max_length | d(omit) }}"
```

#### Exchanges
Exchanges are also a straight forward list of variables ... (see ##-queues.yml tasks file for how exactly ...)

```
rabbitmq_exchanges:
  - name: foo
    type: direct
  - name: bar
    type: fanout
```

#### Bindings
So are bindings ... (also see ##-queues.yml tasks file for how exactly ...)

```
rabbitmq_bindings:
  - source: foo
    destination: foo
    routing_key: ''
  - source: bar
    destination: bar
    routing_key: 'my-key'
```

#### Shovles

Shovels are a bit more complicated and require more info via variables such as:
* `rabbitmq_shovel_src` set to  the remote server you want to mirror queues from>
* `rabbitmq_shovel_username` the remote server's rabbitmq user name for authentication
* `rabbitmq_shovel_password` the remote server's password for authentication

An example var file:

please note a bug with rabbitmq_parameter module hence using shell with rabbitmqctl ...
```
---
rabbitmq_shovel_src: rabbit1
rabbitmq_shovel_username: rabbitmqadmin
rabbitmq_shovel_password: rabbitmqadmin
rabbitmq_shovels:
  - name: "foo"
    json: ' { "src-protocol": "amqp091", "src-uri": "amqp://{{ rabbitmq_shovel_username }}:{{ rabbitmq_shovel_password }}@{{ rabbitmq_shovel_src }}:{{ rabbitmq_listen_port }}", "src-queue": "foo", "dest-protocol": "amqp091", "dest-uri": "amqp://{{ rabbitmq_shovel_username }}:{{ rabbitmq_shovel_password }}@{{ ansible_hostname }}:{{ rabbitmq_listen_port }}", "dest-queue": "foo", "add-forward-headers": true, "ack-mode": "on-publish" }'
  - name: "bar"
    json: ' { "src-protocol": "amqp091", "src-uri": "amqp://{{ rabbitmq_shovel_username }}:{{ rabbitmq_shovel_password }}@{{ rabbitmq_shovel_src }}:{{ rabbitmq_listen_port }}", "src-queue": "bar", "dest-protocol": "amqp091", "dest-uri": "amqp://{{ rabbitmq_shovel_username }}:{{ rabbitmq_shovel_password }}@{{ ansible_hostname }}:{{ rabbitmq_listen_port }}", "dest-queue": "bar", "add-forward-headers": true, "ack-mode": "on-publish" }'
```

TODO / WIP (Pull request anyone ? ;)
------------------------------------
* Add tests
* Add SSL support

License
-------
[Apache 2](https://choosealicense.com/licenses/apache-2.0/)


Author Information
------------------
[Haggai Philip Zagury](http://www.tikalk.com/devops/haggai)
