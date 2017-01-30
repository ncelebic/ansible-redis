# ansible-redis

 - Requires Ansible 1.6.3+
 - Compatible with RHEL/CentOS 7.x

## Contents

 1. [Getting Started](#getting-started)
  1. [Single Redis node](#single-redis-node)
  2. [Master-Slave Replication](#master-slave-replication)
  3. [Redis Sentinel](#redis-sentinel)
 2. [Installing Redis](#installing-redis)
 3. [Role Variables](#role-variables)

## Getting started

Below are a few example playbooks and configurations for deploying a variety of Redis architectures.

This role expects to be run as root or as a user with sudo privileges.

### Single Redis node

Deploying a single Redis server node is pretty trivial; just add the role to your playbook and go. Here's an example which we'll make a little more exciting by setting the bind address to 127.0.0.1:

``` yml
---
- hosts: redis01.example.com
  vars:
    - redis_bind: 127.0.0.1
  roles:
    - ansible-redis
```

That's it! You'll now have a Redis server listening on 127.0.0.1 on redis01.example.com. Redis binary locations are defined by the package that gets installed.

### Master-Slave replication

Configuring [replication](http://redis.io/topics/replication) in Redis is accomplished by deploying multiple nodes, and setting the `redis_slaveof` variable on the slave nodes, just as you would in the redis.conf. In the example that follows, we'll deploy a Redis master with three slaves.

In this example, we're going to use groups to separate the master and slave nodes. Let's start with the inventory file:

``` ini
[redis-master]
redis-master.example.com

[redis-slave]
redis-slave0[1:3].example.com
```

And here's the playbook:

``` yml
---
- name: configure the master redis server
  hosts: redis-master
  roles:
    - ansible-redis

- name: configure redis slaves
  hosts: redis-slave
  vars:
    - redis_slaveof: redis-master.example.com 6379
  roles:
    - ansible-redis
```

In this case, I'm assuming you have DNS records set up for redis-master.example.com, but that's not always the case. You can pretty much go crazy with whatever you need this to be set to. In many cases, I tell Ansible to use the eth1 IP address for the master. Here's a more flexible value for the sake of posterity:

``` yml
redis_slaveof: "{{ hostvars['redis-master.example.com'].ansible_eth1.ipv4.address }} {{ redis_port }}"
```

Now you're cooking with gas! Running this playbook should have you ready to go with a Redis master and three slaves.

You can also specify redis_slaveof in the host_vars in the playbook.  That way, there's no division of hosts.

### Redis Sentinel

#### Introduction

Using Master-Slave replication is great for durability and distributing reads and writes, but not so much for high availability. If the master node fails, a slave must be manually promoted to master, and connections will need to be redirected to the new master. The solution for this problem is [Redis Sentinel](http://redis.io/topics/sentinel), a distributed system which uses Redis itself to communicate and handle automatic failover in a Redis cluster.

Sentinel itself uses the same redis-server binary that Redis uses, but runs with the `--sentinel` flag and with a different configuration file. All of this, of course, is abstracted with this Ansible role, but it's still good to know.

#### Configuration

To add a Sentinel node to an existing deployment, assign this same `redis` role to it, and set the variable `redis_sentinel` to True on that particular host. This can be done in any number of ways, and for the purposes of this example I'll extend on the inventory file used above in the Master/Slave configuration:

``` ini
[redis-master]
redis-master.example.com

[redis-slave]
redis-slave0[1:3].example.com

[redis-sentinel]
redis-sentinel0[1:3].example.com redis_sentinel=True
```

Above, we've added three more hosts in the **redis-sentinel** group (though this group serves no purpose within the role, it's merely an identifier), and set the `redis_sentinel` variable inline within the inventory file.

Now, all we need to do is set the `redis_sentinel_master_ip` variable to define the Redis master which Sentinel should monitor. In this case, I'm going to do this within the playbook:

``` yml
- name: configure the master redis server
  hosts: redis-master
  roles:
    - ansible-redis

- name: configure redis slaves
  hosts: redis-slave
  vars:
    - redis_slaveof: redis-master.example.com 6379
  roles:
    - ansible-redis

- name: configure redis sentinel nodes
  hosts: redis-sentinel
  vars:
    - redis_sentinel_master_ip: 192.168.22.33
  roles:
    - ansible-redis
```

This will configure the Sentinel nodes to monitor the master we created above using the identifier `master01`. By default, Sentinel will use a quorum of 2, which means that at least 2 Sentinels must agree that a master is down in order for a failover to take place. This value can be overridden by setting the `quorum` key within your monitor definition. See the [Sentinel docs](http://redis.io/topics/sentinel) for more details.

Along with the variables listed above, Sentinel has a number of its own configurables just as Redis server does. These are prefixed with `redis_sentinel_`, and are enumerated in the **Role Variables** section below.


## Installing Redis

A Redis package must be installable by yum on the server.  Making a package available to the node is beyond the scope of this role.  EPEL and Remi are two different repositories that offer Redis RPMs.

Do not forget to set the version variable if you care about the version that you want installed.  The default is `latest` which will install the latest default version available.

For example (file was stored in same folder as the playbook that included the redis role):
```yml
vars:
  - redis_version: latest
```

## Role Variables

Here is a list of all the default variables for this role, which are also available in defaults/main.yml. One of these days I'll format these into a table or something.

``` yml
---
## Installation options
redis_server: true

# Used when redis_installation_source is 'src'
redis_version: latest
redis_user: root
redis_group: "{{ redis_user }}"
redis_dir: /var/lib/redis/{{ redis_service_name }}
# The open file limit for Redis/Sentinel
redis_nofile_limit: 16384


## Role options
# Configure Redis as a service
# This creates the init scripts for Redis and ensures the process is running
# Also applies for Redis Sentinel
redis_as_service: true
# Add local facts to /etc/ansible/facts.d for Redis
redis_local_facts: true
# Service name
redis_service_name: "{{ redis_port }}"
redis_systemd_service_name: "redis_{{ redis_service_name }}"
redis_config_filename: "redis_{{ redis_service_name }}.conf"
redis_config_dir: "/etc/redis"
redis_config: "{{ redis_config_dir }}/{{ redis_config_filename }}"

## Networking/connection options
redis_bind: 0.0.0.0
redis_port: 6379
redis_password: false
# Slave replication options
redis_min_slaves_to_write: 0
redis_min_slaves_max_lag: 10
redis_tcp_backlog: 511
redis_tcp_keepalive: 0
# Max connected clients at a time
redis_maxclients: 10000
redis_timeout: 0
# Socket options
# Set socket_path to the desired path to the socket. E.g. /var/run/redis/{{ redis_port }}.sock
redis_socket_path: false
redis_socket_perm: 755

## Replication options
# Set slaveof just as you would in redis.conf. (e.g. "redis01 6379")
redis_slaveof: false
# Make slaves read-only. "yes" or "no"
redis_slave_read_only: "yes"
redis_slave_priority: 100
redis_repl_backlog_size: false

## Logging
redis_logfile: "/var/log/redis/{{ redis_service_name }}.log"
# Enable syslog. "yes" or "no"
redis_syslog_enabled: "no"
redis_syslog_ident: "{{ redis_service_name }}"
# Syslog facility. Must be USER or LOCAL0-LOCAL7
redis_syslog_facility: USER

## General configuration
redis_daemonize: "yes"
redis_pidfile: /var/run/redis/{{ redis_port }}.pid
# Number of databases to allow
redis_databases: 16
redis_loglevel: notice
# Log queries slower than this many milliseconds. -1 to disable
redis_slowlog_log_slower_than: 10000
# Maximum number of slow queries to save
redis_slowlog_max_len: 128
# Redis memory limit (e.g. 4294967296, 4096mb, 4gb)
redis_maxmemory: false
redis_maxmemory_policy: allkeys-lru
redis_rename_commands: []
# How frequently to snapshot the database to disk
# e.g. "900 1" => 900 seconds if at least 1 key changed
redis_save:
  - 900 1
  - 300 10
  - 60 10000
redis_appendonly: "no"
redis_appendfilename: "appendonly.aof"
redis_appendfsync: "everysec"
redis_no_appendfsync_on_rewrite: "no"
redis_auto_aof_rewrite_percentage: "100"
redis_auto_aof_rewrite_min_size: "64mb"

## Redis sentinel configs
# Set this to true on a host to configure it as a Sentinel
redis_sentinel: false
redis_sentinel_dir: /var/lib/redis/sentinel_{{ redis_service_name }}
redis_sentinel_systemd_service_name: sentinel_{{ redis_service_name }}
redis_sentinel_config_filename: sentinel_{{ redis_service_name }}.conf
redis_sentinel_config: "{{ redis_config_dir }}/{{ redis_sentinel_config_filename }}"
redis_sentinel_bind: 0.0.0.0
redis_sentinel_port: 26379
redis_sentinel_pidfile: /var/run/redis/sentinel_{{ redis_service_name }}.pid
redis_sentinel_logfile: /var/log/redis/sentinel_{{ redis_service_name }}.log
redis_sentinel_syslog_ident: sentinel_{{ redis_service_name }}
redis_sentinel_master_ip: localhost
redis_sentinel_monitors:
  - name: master01
    host: "{{ redis_sentinel_master_ip }}"
    port: 6379
    quorum: 2
    down_after_milliseconds: 500
    parallel_syncs: 1
    failover_timeout: 60000
    notification_script: false
    client_reconfig_script: false

```

## Facts

The following facts are accessible in your inventory or tasks outside of this role.

- `{{ ansible_local.redis.bind }}`
- `{{ ansible_local.redis.port }}`
- `{{ ansible_local.redis.sentinel_bind }}`
- `{{ ansible_local.redis.sentinel_port }}`
- `{{ ansible_local.redis.sentinel_monitors }}`

To disable these facts, set `redis_local_facts` to a false value.
