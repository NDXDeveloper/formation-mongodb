🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.6 Ansible pour MongoDB

## Introduction

Ansible est un outil d'automatisation agentless basé sur SSH, idéal pour déployer et gérer des infrastructures MongoDB de manière déclarative et reproductible. Contrairement aux outils basés sur des agents, Ansible ne nécessite que Python et SSH sur les cibles, simplifiant considérablement la gestion de flottes MongoDB.

```
┌────────────────────────────────────────────────────────────────────┐
│              Ansible MongoDB Architecture                          │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │               Control Node (Ansible)                         │  │
│  │                                                              │  │
│  │  ┌────────────────────────────────────────────────────────┐  │  │
│  │  │            Ansible Core                                │  │  │
│  │  │                                                        │  │  │
│  │  │  • Inventory Management                                │  │  │
│  │  │  • Playbook Execution                                  │  │  │
│  │  │  • Module Library                                      │  │  │
│  │  │  • Variable Management                                 │  │  │
│  │  │  • Fact Gathering                                      │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  │                                                              │  │
│  │  Project Structure:                                          │  │
│  │  ansible-mongodb/                                            │  │
│  │  ├── inventory/                                              │  │
│  │  │   ├── production                                          │  │
│  │  │   ├── staging                                             │  │
│  │  │   └── group_vars/                                         │  │
│  │  ├── roles/                                                  │  │
│  │  │   ├── mongodb-common/                                     │  │
│  │  │   ├── mongodb-replicaset/                                 │  │
│  │  │   └── mongodb-sharded/                                    │  │
│  │  ├── playbooks/                                              │  │
│  │  └── ansible.cfg                                             │  │
│  └──────────────────────────┬───────────────────────────────────┘  │
│                             │ SSH                                  │
│                             ▼                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              Managed Nodes (MongoDB Servers)                 │  │
│  │                                                              │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │  │
│  │  │  mongo-01    │  │  mongo-02    │  │  mongo-03    │        │  │
│  │  │              │  │              │  │              │        │  │
│  │  │  • Python    │  │  • Python    │  │  • Python    │        │  │
│  │  │  • SSH       │  │  • SSH       │  │  • SSH       │        │  │
│  │  │  • MongoDB   │  │  • MongoDB   │  │  • MongoDB   │        │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘        │  │
│  │                                                              │  │
│  │  Tasks Executed:                                             │  │
│  │  • System configuration                                      │  │
│  │  • MongoDB installation                                      │  │
│  │  • Replica Set setup                                         │  │
│  │  • User management                                           │  │
│  │  • Monitoring configuration                                  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Avantages d'Ansible pour MongoDB

```yaml
# ansible-benefits.yaml
---
agentless:
  description: "Pas d'agent à installer sur les serveurs"
  benefits:
    - "Seulement SSH et Python nécessaires"
    - "Réduction de la surface d'attaque"
    - "Simplification de la maintenance"

declarative:
  description: "Configuration déclarative de l'infrastructure"
  benefits:
    - "Infrastructure as Code"
    - "État désiré vs état actuel"
    - "Idempotence garantie"

modular:
  description: "Architecture modulaire avec rôles réutilisables"
  benefits:
    - "Réutilisation du code"
    - "Séparation des préoccupations"
    - "Collections Galaxy disponibles"

scalable:
  description: "Gestion de flottes MongoDB"
  benefits:
    - "Déploiement parallèle"
    - "Inventaire dynamique"
    - "Tags et groupes flexibles"

ecosystem:
  description: "Écosystème riche et communauté active"
  benefits:
    - "Modules officiels MongoDB"
    - "Collections communautaires"
    - "Intégration avec Ansible Tower/AWX"
```

---

## Configuration de Base

### Installation d'Ansible

```bash
# Installation sur le control node
# Ubuntu/Debian
sudo apt update
sudo apt install -y ansible python3-pip

# RHEL/CentOS
sudo yum install -y epel-release
sudo yum install -y ansible

# Via pip (recommandé pour la dernière version)
pip3 install --user ansible

# Vérifier l'installation
ansible --version

# Installer les collections MongoDB
ansible-galaxy collection install community.mongodb
ansible-galaxy collection install community.general
```

### Structure du Projet

```bash
# Créer la structure du projet
mkdir -p ansible-mongodb/{inventory,roles,playbooks,group_vars,host_vars,files,templates}

# Structure complète
tree ansible-mongodb/
```

```
ansible-mongodb/
├── ansible.cfg                    # Configuration Ansible
├── inventory/                     # Inventaires
│   ├── production                 # Inventaire production
│   ├── staging                    # Inventaire staging
│   └── dev                        # Inventaire développement
├── group_vars/                    # Variables par groupe
│   ├── all.yml                    # Variables globales
│   ├── mongodb_replicaset.yml     # Variables replica set
│   └── mongodb_sharded.yml        # Variables sharded cluster
├── host_vars/                     # Variables par host
│   ├── mongo-01.yml
│   └── mongo-02.yml
├── roles/                         # Rôles Ansible
│   ├── mongodb-common/            # Rôle commun
│   │   ├── defaults/
│   │   ├── files/
│   │   ├── handlers/
│   │   ├── meta/
│   │   ├── tasks/
│   │   ├── templates/
│   │   └── vars/
│   ├── mongodb-replicaset/        # Rôle replica set
│   └── mongodb-sharded/           # Rôle sharded cluster
├── playbooks/                     # Playbooks
│   ├── deploy-mongodb.yml
│   ├── configure-replicaset.yml
│   ├── backup-mongodb.yml
│   └── upgrade-mongodb.yml
├── files/                         # Fichiers statiques
└── templates/                     # Templates Jinja2
    ├── mongod.conf.j2
    └── mongodb.service.j2
```

### ansible.cfg - Configuration

```ini
# ansible-mongodb/ansible.cfg
[defaults]
# Inventaire par défaut
inventory = ./inventory/production

# Rôles
roles_path = ./roles:~/.ansible/roles:/usr/share/ansible/roles

# Collections
collections_paths = ./collections:~/.ansible/collections:/usr/share/ansible/collections

# Callbacks
callbacks_enabled = timer, profile_tasks, profile_roles

# Output
stdout_callback = yaml
bin_ansible_callbacks = True

# SSH
host_key_checking = False
remote_user = ansible
private_key_file = ~/.ssh/id_rsa

# Performance
forks = 10
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600

# Retry
retry_files_enabled = True
retry_files_save_path = ~/.ansible-retry

# Logs
log_path = ./ansible.log

# Privilege escalation
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[inventory]
enable_plugins = host_list, script, auto, yaml, ini, toml

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no
control_path = /tmp/ansible-ssh-%%h-%%p-%%r
pipelining = True
```

---

## Inventaires

### Inventaire Statique - Production

```ini
# inventory/production
[all:vars]
ansible_user=ansible
ansible_python_interpreter=/usr/bin/python3

# MongoDB Replica Set
[mongodb_replicaset]
mongo-rs-01 ansible_host=10.0.1.10
mongo-rs-02 ansible_host=10.0.1.11
mongo-rs-03 ansible_host=10.0.1.12

[mongodb_replicaset:vars]
mongodb_replicaset_name=rs0
mongodb_port=27017

# MongoDB Sharded Cluster - Config Servers
[mongodb_config_servers]
mongo-cfg-01 ansible_host=10.0.2.10
mongo-cfg-02 ansible_host=10.0.2.11
mongo-cfg-03 ansible_host=10.0.2.12

[mongodb_config_servers:vars]
mongodb_port=27019
mongodb_replicaset_name=configReplSet

# MongoDB Sharded Cluster - Shard 0
[mongodb_shard_0]
mongo-sh0-01 ansible_host=10.0.3.10
mongo-sh0-02 ansible_host=10.0.3.11
mongo-sh0-03 ansible_host=10.0.3.12

[mongodb_shard_0:vars]
mongodb_port=27018
mongodb_replicaset_name=shard0
mongodb_shard_name=shard0

# MongoDB Sharded Cluster - Shard 1
[mongodb_shard_1]
mongo-sh1-01 ansible_host=10.0.4.10
mongo-sh1-02 ansible_host=10.0.4.11
mongo-sh1-03 ansible_host=10.0.4.12

[mongodb_shard_1:vars]
mongodb_port=27018
mongodb_replicaset_name=shard1
mongodb_shard_name=shard1

# MongoDB Sharded Cluster - Mongos
[mongodb_mongos]
mongo-mongos-01 ansible_host=10.0.5.10
mongo-mongos-02 ansible_host=10.0.5.11

[mongodb_mongos:vars]
mongodb_port=27017

# Groupes logiques
[mongodb_shards:children]
mongodb_shard_0
mongodb_shard_1

[mongodb_sharded_cluster:children]
mongodb_config_servers
mongodb_shards
mongodb_mongos

[mongodb:children]
mongodb_replicaset
mongodb_sharded_cluster

# Variables globales pour toutes les instances MongoDB
[mongodb:vars]
mongodb_version=7.0
mongodb_edition=org
mongodb_data_dir=/var/lib/mongodb
mongodb_log_dir=/var/log/mongodb
mongodb_user=mongodb
mongodb_group=mongodb
```

### Inventaire Dynamique - AWS

```python
#!/usr/bin/env python3
# inventory/aws_ec2_inventory.py
"""
Inventaire dynamique pour instances MongoDB sur AWS EC2
"""

import json
import boto3
from typing import Dict, List

def get_inventory() -> Dict:
    """
    Récupère l'inventaire depuis AWS EC2 basé sur les tags
    """
    ec2 = boto3.client('ec2', region_name='eu-west-3')

    # Récupérer les instances avec le tag mongodb
    response = ec2.describe_instances(
        Filters=[
            {'Name': 'tag:Application', 'Values': ['mongodb']},
            {'Name': 'instance-state-name', 'Values': ['running']}
        ]
    )

    inventory = {
        '_meta': {
            'hostvars': {}
        },
        'all': {
            'children': ['ungrouped']
        }
    }

    # Parser les instances
    for reservation in response['Reservations']:
        for instance in reservation['Instances']:
            # Extraire les informations
            instance_id = instance['InstanceId']
            private_ip = instance.get('PrivateIpAddress', '')
            public_ip = instance.get('PublicIpAddress', '')

            # Extraire les tags
            tags = {tag['Key']: tag['Value'] for tag in instance.get('Tags', [])}

            # Déterminer le groupe
            role = tags.get('Role', 'ungrouped')
            environment = tags.get('Environment', 'unknown')

            # Créer le groupe si nécessaire
            if role not in inventory:
                inventory[role] = {
                    'hosts': [],
                    'vars': {}
                }

            # Ajouter l'host
            hostname = tags.get('Name', instance_id)
            inventory[role]['hosts'].append(hostname)

            # Variables de l'host
            inventory['_meta']['hostvars'][hostname] = {
                'ansible_host': public_ip or private_ip,
                'ansible_user': 'ubuntu',
                'instance_id': instance_id,
                'private_ip': private_ip,
                'public_ip': public_ip,
                'instance_type': instance['InstanceType'],
                'availability_zone': instance['Placement']['AvailabilityZone'],
                'environment': environment,
                'role': role,
                'tags': tags
            }

    return inventory

if __name__ == '__main__':
    print(json.dumps(get_inventory(), indent=2))
```

### Variables Globales

```yaml
# group_vars/all.yml
---
# Configuration système
timezone: "Europe/Paris"
ntp_servers:
  - 0.pool.ntp.org
  - 1.pool.ntp.org

# Utilisateurs
mongodb_admin_users:
  - username: ansible
    groups: sudo
    ssh_keys:
      - "ssh-rsa AAAAB3NzaC1yc2E... ansible@control"

# Packages système
system_packages:
  - vim
  - htop
  - curl
  - wget
  - net-tools
  - python3
  - python3-pip

# Python packages
python_packages:
  - pymongo
  - psutil

# MongoDB configuration globale
mongodb:
  version: "7.0"
  edition: "org"  # org ou enterprise

  # Paths
  data_dir: "/var/lib/mongodb"
  log_dir: "/var/log/mongodb"
  config_file: "/etc/mongod.conf"

  # User/Group
  user: "mongodb"
  group: "mongodb"

  # Network
  bind_ip: "0.0.0.0"
  port: 27017

  # Security
  auth_enabled: true
  keyfile_path: "/etc/mongodb-keyfile"

  # Storage
  storage_engine: "wiredTiger"
  cache_size_gb: 4

  # Replication
  oplog_size_mb: 10240

  # System
  ulimits:
    nofile: 64000
    nproc: 32000

# Monitoring
monitoring:
  enabled: true
  exporter_version: "0.40.0"
  exporter_port: 9216

# Backup
backup:
  enabled: true
  s3_bucket: "company-mongodb-backups"
  s3_region: "eu-west-3"
  retention_days: 30
```

### Variables Replica Set

```yaml
# group_vars/mongodb_replicaset.yml
---
# Configuration spécifique au Replica Set
mongodb_replicaset:
  name: "rs0"

  # Membres
  members:
    - host: "mongo-rs-01:27017"
      priority: 7
      votes: 1
      tags:
        datacenter: "dc1"
        usage: "production"

    - host: "mongo-rs-02:27017"
      priority: 6
      votes: 1
      tags:
        datacenter: "dc1"
        usage: "production"

    - host: "mongo-rs-03:27017"
      priority: 5
      votes: 1
      tags:
        datacenter: "dc2"
        usage: "backup"

  # Settings
  settings:
    electionTimeoutMillis: 10000
    heartbeatIntervalMillis: 2000
    heartbeatTimeoutSecs: 10
    catchUpTimeoutMillis: 60000

# Utilisateurs MongoDB
mongodb_users:
  - name: "admin"
    password: "{{ vault_mongodb_admin_password }}"
    database: "admin"
    roles:
      - db: "admin"
        role: "root"

  - name: "app-user"
    password: "{{ vault_mongodb_app_password }}"
    database: "app_db"
    roles:
      - db: "app_db"
        role: "readWrite"

  - name: "backup-user"
    password: "{{ vault_mongodb_backup_password }}"
    database: "admin"
    roles:
      - db: "admin"
        role: "backup"
      - db: "admin"
        role: "restore"

  - name: "monitoring-user"
    password: "{{ vault_mongodb_monitoring_password }}"
    database: "admin"
    roles:
      - db: "admin"
        role: "clusterMonitor"
      - db: "local"
        role: "read"

# Configuration mongod spécifique
mongodb_config:
  storage:
    wiredTiger:
      engineConfig:
        cacheSizeGB: 6
        journalCompressor: "snappy"
      collectionConfig:
        blockCompressor: "snappy"

  operationProfiling:
    mode: "slowOp"
    slowOpThresholdMs: 100

  setParameter:
    enableLocalhostAuthBypass: false
    transactionLifetimeLimitSeconds: 60
```

---

## Rôle MongoDB Common

### Structure du Rôle

```bash
# Créer la structure du rôle
ansible-galaxy init roles/mongodb-common
```

### defaults/main.yml - Variables par Défaut

```yaml
# roles/mongodb-common/defaults/main.yml
---
# Version MongoDB
mongodb_version: "7.0"
mongodb_edition: "org"

# Paths
mongodb_data_dir: "/var/lib/mongodb"
mongodb_log_dir: "/var/log/mongodb"
mongodb_config_dir: "/etc"
mongodb_config_file: "{{ mongodb_config_dir }}/mongod.conf"

# User and group
mongodb_user: "mongodb"
mongodb_group: "mongodb"
mongodb_uid: 1000
mongodb_gid: 1000

# Network
mongodb_bind_ip: "0.0.0.0"
mongodb_port: 27017
mongodb_max_connections: 65536

# Security
mongodb_auth_enabled: true
mongodb_keyfile_path: "/etc/mongodb-keyfile"
mongodb_keyfile_content: "{{ vault_mongodb_keyfile }}"

# Storage
mongodb_storage_engine: "wiredTiger"
mongodb_cache_size_gb: 4
mongodb_journal_enabled: true

# Replication
mongodb_replication_enabled: false
mongodb_replicaset_name: "rs0"
mongodb_oplog_size_mb: 10240

# System
mongodb_ulimits:
  nofile: 64000
  nproc: 32000

# Transparent Huge Pages
mongodb_disable_thp: true

# Monitoring
mongodb_exporter_enabled: false
mongodb_exporter_version: "0.40.0"
mongodb_exporter_port: 9216
```

### tasks/main.yml - Tâches Principales

```yaml
# roles/mongodb-common/tasks/main.yml
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_os_family }}.yml"

- name: System preparation
  include_tasks: system.yml
  tags: [system]

- name: Install MongoDB
  include_tasks: install.yml
  tags: [install]

- name: Configure MongoDB
  include_tasks: configure.yml
  tags: [configure]

- name: Setup security
  include_tasks: security.yml
  tags: [security]

- name: Setup monitoring
  include_tasks: monitoring.yml
  when: mongodb_exporter_enabled
  tags: [monitoring]

- name: Start MongoDB service
  include_tasks: service.yml
  tags: [service]
```

### tasks/system.yml - Préparation Système

```yaml
# roles/mongodb-common/tasks/system.yml
---
- name: Set timezone
  timezone:
    name: "{{ timezone }}"
  when: timezone is defined

- name: Install system packages
  package:
    name: "{{ system_packages }}"
    state: present
  register: result
  until: result is succeeded
  retries: 3
  delay: 5

- name: Create MongoDB group
  group:
    name: "{{ mongodb_group }}"
    gid: "{{ mongodb_gid }}"
    state: present

- name: Create MongoDB user
  user:
    name: "{{ mongodb_user }}"
    uid: "{{ mongodb_uid }}"
    group: "{{ mongodb_group }}"
    shell: /bin/false
    home: "{{ mongodb_data_dir }}"
    create_home: no
    state: present

- name: Create MongoDB directories
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ mongodb_user }}"
    group: "{{ mongodb_group }}"
    mode: '0750'
  loop:
    - "{{ mongodb_data_dir }}"
    - "{{ mongodb_log_dir }}"

- name: Configure system limits
  pam_limits:
    domain: "{{ mongodb_user }}"
    limit_type: "{{ item.limit_type }}"
    limit_item: "{{ item.limit_item }}"
    value: "{{ item.value }}"
  loop:
    - { limit_type: 'soft', limit_item: 'nofile', value: "{{ mongodb_ulimits.nofile }}" }
    - { limit_type: 'hard', limit_item: 'nofile', value: "{{ mongodb_ulimits.nofile }}" }
    - { limit_type: 'soft', limit_item: 'nproc', value: "{{ mongodb_ulimits.nproc }}" }
    - { limit_type: 'hard', limit_item: 'nproc', value: "{{ mongodb_ulimits.nproc }}" }

- name: Disable Transparent Huge Pages
  when: mongodb_disable_thp
  block:
    - name: Create THP disable script
      copy:
        dest: /etc/init.d/disable-transparent-hugepages
        mode: '0755'
        content: |
          #!/bin/bash
          ### BEGIN INIT INFO
          # Provides:          disable-transparent-hugepages
          # Required-Start:    $local_fs
          # Required-Stop:
          # Default-Start:     2 3 4 5
          # Default-Stop:      0 1 6
          # Short-Description: Disable THP
          ### END INIT INFO

          case $1 in
            start)
              if [ -d /sys/kernel/mm/transparent_hugepage ]; then
                echo never > /sys/kernel/mm/transparent_hugepage/enabled
                echo never > /sys/kernel/mm/transparent_hugepage/defrag
              fi
              ;;
          esac

    - name: Enable THP disable service
      systemd:
        name: disable-transparent-hugepages
        enabled: yes
      when: ansible_service_mgr == "systemd"

    - name: Disable THP immediately
      shell: |
        echo never > /sys/kernel/mm/transparent_hugepage/enabled
        echo never > /sys/kernel/mm/transparent_hugepage/defrag
      changed_when: false

- name: Configure kernel parameters
  sysctl:
    name: "{{ item.name }}"
    value: "{{ item.value }}"
    state: present
    reload: yes
  loop:
    - { name: 'vm.swappiness', value: '1' }
    - { name: 'net.core.somaxconn', value: '4096' }
    - { name: 'net.ipv4.tcp_keepalive_time', value: '300' }
```

### tasks/install.yml - Installation MongoDB

```yaml
# roles/mongodb-common/tasks/install.yml
---
- name: Add MongoDB repository key (Debian/Ubuntu)
  when: ansible_os_family == "Debian"
  apt_key:
    url: "https://www.mongodb.org/static/pgp/server-{{ mongodb_version }}.asc"
    state: present

- name: Add MongoDB repository (Debian/Ubuntu)
  when: ansible_os_family == "Debian"
  apt_repository:
    repo: "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu {{ ansible_distribution_release }}/mongodb-org/{{ mongodb_version }} multiverse"
    state: present
    filename: mongodb-org-{{ mongodb_version }}

- name: Add MongoDB repository (RedHat/CentOS)
  when: ansible_os_family == "RedHat"
  yum_repository:
    name: mongodb-org-{{ mongodb_version }}
    description: MongoDB Repository
    baseurl: "https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/{{ mongodb_version }}/x86_64/"
    gpgcheck: yes
    enabled: yes
    gpgkey: "https://www.mongodb.org/static/pgp/server-{{ mongodb_version }}.asc"

- name: Update apt cache (Debian/Ubuntu)
  when: ansible_os_family == "Debian"
  apt:
    update_cache: yes
    cache_valid_time: 3600

- name: Install MongoDB packages
  package:
    name:
      - mongodb-org={{ mongodb_version }}.*
      - mongodb-org-server={{ mongodb_version }}.*
      - mongodb-org-shell={{ mongodb_version }}.*
      - mongodb-org-mongos={{ mongodb_version }}.*
      - mongodb-org-tools={{ mongodb_version }}.*
    state: present
  register: mongodb_install
  until: mongodb_install is succeeded
  retries: 3
  delay: 10

- name: Hold MongoDB packages (prevent auto-upgrade)
  when: ansible_os_family == "Debian"
  dpkg_selections:
    name: "{{ item }}"
    selection: hold
  loop:
    - mongodb-org
    - mongodb-org-server
    - mongodb-org-shell
    - mongodb-org-mongos
    - mongodb-org-tools

- name: Install Python MongoDB libraries
  pip:
    name:
      - pymongo>=4.0
      - psutil
    state: present
    executable: pip3
```

### tasks/configure.yml - Configuration

```yaml
# roles/mongodb-common/tasks/configure.yml
---
- name: Generate mongod.conf from template
  template:
    src: mongod.conf.j2
    dest: "{{ mongodb_config_file }}"
    owner: root
    group: root
    mode: '0644'
    backup: yes
  notify: restart mongodb

- name: Create systemd service override directory
  file:
    path: /etc/systemd/system/mongod.service.d
    state: directory
    mode: '0755'

- name: Configure systemd service override
  template:
    src: mongod-override.conf.j2
    dest: /etc/systemd/system/mongod.service.d/override.conf
    mode: '0644'
  notify:
    - reload systemd
    - restart mongodb

- name: Enable and start MongoDB service
  systemd:
    name: mongod
    enabled: yes
    state: started
    daemon_reload: yes
```

### templates/mongod.conf.j2 - Template Configuration

```jinja2
{# roles/mongodb-common/templates/mongod.conf.j2 #}
# MongoDB configuration file
# Generated by Ansible

# Storage
storage:
  dbPath: {{ mongodb_data_dir }}
  journal:
    enabled: {{ mongodb_journal_enabled | lower }}
  engine: {{ mongodb_storage_engine }}
{% if mongodb_storage_engine == "wiredTiger" %}
  wiredTiger:
    engineConfig:
      cacheSizeGB: {{ mongodb_cache_size_gb }}
      journalCompressor: snappy
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true
{% endif %}

# System Log
systemLog:
  destination: file
  path: {{ mongodb_log_dir }}/mongod.log
  logAppend: true
  verbosity: 0
  component:
    replication:
      verbosity: 1

# Network
net:
  port: {{ mongodb_port }}
  bindIp: {{ mongodb_bind_ip }}
  maxIncomingConnections: {{ mongodb_max_connections }}
  compression:
    compressors: snappy,zstd,zlib

{% if mongodb_replication_enabled %}
# Replication
replication:
  replSetName: {{ mongodb_replicaset_name }}
  oplogSizeMB: {{ mongodb_oplog_size_mb }}
{% endif %}

{% if mongodb_auth_enabled %}
# Security
security:
  authorization: enabled
{% if mongodb_replication_enabled %}
  keyFile: {{ mongodb_keyfile_path }}
{% endif %}
{% endif %}

# Operation Profiling
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100

# Process Management
processManagement:
  fork: false
  pidFilePath: /var/run/mongodb/mongod.pid

# Set Parameters
setParameter:
  enableLocalhostAuthBypass: false
  diagnosticDataCollectionEnabled: true
```

### tasks/security.yml - Sécurité

```yaml
# roles/mongodb-common/tasks/security.yml
---
- name: Create keyfile for replication
  when: mongodb_replication_enabled and mongodb_auth_enabled
  block:
    - name: Generate keyfile content
      set_fact:
        keyfile_content: "{{ mongodb_keyfile_content | default(lookup('password', '/dev/null length=756 chars=ascii_letters,digits')) }}"

    - name: Create keyfile
      copy:
        content: "{{ keyfile_content }}"
        dest: "{{ mongodb_keyfile_path }}"
        owner: "{{ mongodb_user }}"
        group: "{{ mongodb_group }}"
        mode: '0400'
      notify: restart mongodb

- name: Wait for MongoDB to start
  wait_for:
    port: "{{ mongodb_port }}"
    host: "{{ mongodb_bind_ip }}"
    timeout: 300

- name: Check if admin user exists
  community.mongodb.mongodb_user:
    login_host: localhost
    login_port: "{{ mongodb_port }}"
    database: admin
    name: "{{ mongodb_users[0].name }}"
    state: present
  ignore_errors: yes
  register: admin_check
  when: mongodb_users is defined

- name: Create MongoDB users
  community.mongodb.mongodb_user:
    login_host: localhost
    login_port: "{{ mongodb_port }}"
    database: "{{ item.database }}"
    name: "{{ item.name }}"
    password: "{{ item.password }}"
    roles: "{{ item.roles }}"
    state: present
  loop: "{{ mongodb_users }}"
  when:
    - mongodb_users is defined
    - mongodb_auth_enabled
  no_log: true
```

### handlers/main.yml - Handlers

```yaml
# roles/mongodb-common/handlers/main.yml
---
- name: reload systemd
  systemd:
    daemon_reload: yes

- name: restart mongodb
  systemd:
    name: mongod
    state: restarted

- name: reload mongodb
  systemd:
    name: mongod
    state: reloaded
```

---

## Rôle MongoDB Replica Set

### tasks/main.yml - Tâches Replica Set

```yaml
# roles/mongodb-replicaset/tasks/main.yml
---
- name: Ensure MongoDB is configured as replica set
  include_role:
    name: mongodb-common
  vars:
    mongodb_replication_enabled: true

- name: Initialize replica set
  include_tasks: initialize.yml
  when: inventory_hostname == groups['mongodb_replicaset'][0]
  tags: [initialize]

- name: Configure replica set members
  include_tasks: configure_members.yml
  when: inventory_hostname == groups['mongodb_replicaset'][0]
  tags: [configure]

- name: Wait for replica set to stabilize
  include_tasks: verify.yml
  tags: [verify]
```

### tasks/initialize.yml - Initialisation

```yaml
# roles/mongodb-replicaset/tasks/initialize.yml
---
- name: Check if replica set is already initialized
  community.mongodb.mongodb_status:
    login_host: localhost
    login_port: "{{ mongodb_port }}"
    login_user: "{{ mongodb_users[0].name | default(omit) }}"
    login_password: "{{ mongodb_users[0].password | default(omit) }}"
  register: rs_status
  ignore_errors: yes

- name: Initialize replica set
  when: rs_status.failed or rs_status.replicaset is not defined
  block:
    - name: Build replica set configuration
      set_fact:
        rs_config:
          _id: "{{ mongodb_replicaset.name }}"
          members: "{{ mongodb_replicaset.members | map('combine', {'_id': item.0}) | list }}"
      vars:
        item: "{{ mongodb_replicaset.members | zip(range(mongodb_replicaset.members | length)) | list }}"

    - name: Initialize the replica set
      community.mongodb.mongodb_replicaset:
        login_host: localhost
        login_port: "{{ mongodb_port }}"
        replica_set: "{{ mongodb_replicaset.name }}"
        members: "{{ mongodb_replicaset.members }}"
        validate: no
      register: rs_init

    - name: Wait for primary election
      community.mongodb.mongodb_status:
        login_host: localhost
        login_port: "{{ mongodb_port }}"
        poll: 10
        interval: 2
      register: rs_status
      until: rs_status.replicaset.primary is defined
      retries: 30
      delay: 5

    - name: Display replica set status
      debug:
        var: rs_status

- name: Configure replica set settings
  when: mongodb_replicaset.settings is defined
  community.mongodb.mongodb_shell:
    login_host: localhost
    login_port: "{{ mongodb_port }}"
    login_user: "{{ mongodb_users[0].name | default(omit) }}"
    login_password: "{{ mongodb_users[0].password | default(omit) }}"
    db: admin
    eval: |
      var config = rs.conf();
      config.settings = {{ mongodb_replicaset.settings | to_json }};
      rs.reconfig(config);
```

### tasks/verify.yml - Vérification

```yaml
# roles/mongodb-replicaset/tasks/verify.yml
---
- name: Get replica set status
  community.mongodb.mongodb_status:
    login_host: localhost
    login_port: "{{ mongodb_port }}"
    login_user: "{{ mongodb_users[0].name | default(omit) }}"
    login_password: "{{ mongodb_users[0].password | default(omit) }}"
  register: rs_status

- name: Verify all members are in the replica set
  assert:
    that:
      - rs_status.replicaset.members | length == mongodb_replicaset.members | length
      - rs_status.replicaset.primary is defined
    fail_msg: "Replica set is not properly configured"
    success_msg: "Replica set is healthy"

- name: Display replica set members
  debug:
    msg: "{{ item.name }} - State: {{ item.stateStr }} - Health: {{ item.health }}"
  loop: "{{ rs_status.replicaset.members }}"
```

---

## Playbooks

### deploy-mongodb.yml - Déploiement Complet

```yaml
# playbooks/deploy-mongodb.yml
---
- name: Deploy MongoDB Replica Set
  hosts: mongodb_replicaset
  become: yes

  pre_tasks:
    - name: Validate inventory
      assert:
        that:
          - groups['mongodb_replicaset'] | length >= 3
        fail_msg: "Replica set requires at least 3 members"

    - name: Gather facts
      setup:

  roles:
    - role: mongodb-common
      tags: [common]

    - role: mongodb-replicaset
      tags: [replicaset]

  post_tasks:
    - name: Display connection string
      debug:
        msg: "MongoDB Connection: mongodb://{% for host in groups['mongodb_replicaset'] %}{{ host }}:{{ mongodb_port }}{% if not loop.last %},{% endif %}{% endfor %}/admin?replicaSet={{ mongodb_replicaset.name }}"
      run_once: true
      tags: [always]
```

### backup-mongodb.yml - Backup

```yaml
# playbooks/backup-mongodb.yml
---
- name: Backup MongoDB
  hosts: mongodb_replicaset[0]
  become: yes

  vars:
    backup_dir: "/backup/mongodb"
    backup_timestamp: "{{ ansible_date_time.iso8601_basic_short }}"
    backup_name: "mongodb-backup-{{ backup_timestamp }}"

  tasks:
    - name: Create backup directory
      file:
        path: "{{ backup_dir }}"
        state: directory
        owner: "{{ mongodb_user }}"
        group: "{{ mongodb_group }}"
        mode: '0750'

    - name: Run mongodump
      command: >
        mongodump
        --host localhost:{{ mongodb_port }}
        --username {{ mongodb_users | selectattr('name', 'equalto', 'backup-user') | map(attribute='name') | first }}
        --password {{ mongodb_users | selectattr('name', 'equalto', 'backup-user') | map(attribute='password') | first }}
        --authenticationDatabase admin
        --out {{ backup_dir }}/{{ backup_name }}
        --gzip
        --oplog
      no_log: true
      register: mongodump_result

    - name: Display backup result
      debug:
        var: mongodump_result.stdout_lines

    - name: Create backup archive
      archive:
        path: "{{ backup_dir }}/{{ backup_name }}"
        dest: "{{ backup_dir }}/{{ backup_name }}.tar.gz"
        format: gz
      register: archive_result

    - name: Upload to S3
      when: backup.enabled and backup.s3_bucket is defined
      amazon.aws.aws_s3:
        bucket: "{{ backup.s3_bucket }}"
        object: "mongodb-backups/{{ backup_name }}.tar.gz"
        src: "{{ backup_dir }}/{{ backup_name }}.tar.gz"
        mode: put
        region: "{{ backup.s3_region }}"
      register: s3_upload

    - name: Remove local backup
      when: s3_upload is succeeded
      file:
        path: "{{ item }}"
        state: absent
      loop:
        - "{{ backup_dir }}/{{ backup_name }}"
        - "{{ backup_dir }}/{{ backup_name }}.tar.gz"

    - name: Cleanup old backups from S3
      when: backup.enabled
      amazon.aws.aws_s3:
        bucket: "{{ backup.s3_bucket }}"
        mode: list
        prefix: "mongodb-backups/"
        region: "{{ backup.s3_region }}"
      register: s3_objects

    - name: Delete old backups
      amazon.aws.aws_s3:
        bucket: "{{ backup.s3_bucket }}"
        object: "{{ item }}"
        mode: delobj
        region: "{{ backup.s3_region }}"
      loop: "{{ s3_objects.s3_keys | default([]) | select('match', '^mongodb-backups/.*\\.tar\\.gz$') | sort | reverse | list[backup.retention_days:] }}"
      when:
        - backup.enabled
        - s3_objects.s3_keys is defined
```

### upgrade-mongodb.yml - Mise à Jour

```yaml
# playbooks/upgrade-mongodb.yml
---
- name: Upgrade MongoDB
  hosts: mongodb_replicaset
  become: yes
  serial: 1

  vars:
    mongodb_new_version: "7.0"
    mongodb_backup_before_upgrade: true

  pre_tasks:
    - name: Verify current MongoDB version
      command: mongod --version
      register: current_version
      changed_when: false

    - name: Display current version
      debug:
        msg: "Current version: {{ current_version.stdout_lines[0] }}"

    - name: Create pre-upgrade backup
      when: mongodb_backup_before_upgrade and inventory_hostname == groups['mongodb_replicaset'][0]
      import_playbook: backup-mongodb.yml

  tasks:
    - name: Check if this is a secondary
      community.mongodb.mongodb_status:
        login_host: localhost
        login_port: "{{ mongodb_port }}"
        login_user: "{{ mongodb_users[0].name }}"
        login_password: "{{ mongodb_users[0].password }}"
      register: rs_status
      failed_when: false

    - name: Step down primary if needed
      when: rs_status.replicaset.ismaster | default(false)
      community.mongodb.mongodb_shell:
        login_host: localhost
        login_port: "{{ mongodb_port }}"
        login_user: "{{ mongodb_users[0].name }}"
        login_password: "{{ mongodb_users[0].password }}"
        db: admin
        eval: "db.adminCommand({ replSetStepDown: 60 })"

    - name: Stop MongoDB
      systemd:
        name: mongod
        state: stopped

    - name: Update MongoDB packages
      package:
        name:
          - mongodb-org={{ mongodb_new_version }}.*
          - mongodb-org-server={{ mongodb_new_version }}.*
          - mongodb-org-shell={{ mongodb_new_version }}.*
          - mongodb-org-mongos={{ mongodb_new_version }}.*
          - mongodb-org-tools={{ mongodb_new_version }}.*
        state: present
        update_cache: yes

    - name: Start MongoDB
      systemd:
        name: mongod
        state: started

    - name: Wait for MongoDB to be ready
      wait_for:
        port: "{{ mongodb_port }}"
        delay: 10
        timeout: 300

    - name: Verify new version
      command: mongod --version
      register: new_version
      changed_when: false

    - name: Display new version
      debug:
        msg: "New version: {{ new_version.stdout_lines[0] }}"

    - name: Wait for member to catch up
      pause:
        seconds: 30

    - name: Verify replica set health
      community.mongodb.mongodb_status:
        login_host: localhost
        login_port: "{{ mongodb_port }}"
        login_user: "{{ mongodb_users[0].name }}"
        login_password: "{{ mongodb_users[0].password }}"
      register: health_check
      until: health_check.replicaset.members | selectattr('name', 'equalto', inventory_hostname) | map(attribute='health') | first == 1
      retries: 10
      delay: 10

  post_tasks:
    - name: Final replica set status
      community.mongodb.mongodb_status:
        login_host: localhost
        login_port: "{{ mongodb_port }}"
        login_user: "{{ mongodb_users[0].name }}"
        login_password: "{{ mongodb_users[0].password }}"
      register: final_status
      run_once: true

    - name: Display final status
      debug:
        var: final_status.replicaset
      run_once: true
```

---

## Ansible Vault - Gestion des Secrets

### Création et Utilisation

```bash
# Créer un nouveau vault file
ansible-vault create group_vars/mongodb_replicaset/vault.yml

# Éditer un vault existant
ansible-vault edit group_vars/mongodb_replicaset/vault.yml

# Voir le contenu
ansible-vault view group_vars/mongodb_replicaset/vault.yml

# Changer le mot de passe
ansible-vault rekey group_vars/mongodb_replicaset/vault.yml

# Encrypter un fichier existant
ansible-vault encrypt group_vars/mongodb_replicaset/vault.yml

# Décrypter
ansible-vault decrypt group_vars/mongodb_replicaset/vault.yml
```

### vault.yml - Variables Sensibles

```yaml
# group_vars/mongodb_replicaset/vault.yml (encrypted)
---
vault_mongodb_admin_password: "SuperSecurePassword123!"
vault_mongodb_app_password: "AppPassword456!"
vault_mongodb_backup_password: "BackupPassword789!"
vault_mongodb_monitoring_password: "MonitoringPassword012!"
vault_mongodb_keyfile: "YXNkZmFzZGZhc2RmYXNkZmFzZGZhc2RmYXNkZmFzZGZhc2RmYXNkZmFzZGY="

# AWS credentials
vault_aws_access_key: "AKIAIOSFODNN7EXAMPLE"
vault_aws_secret_key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

### Utilisation avec Playbook

```bash
# Exécuter avec prompt pour le mot de passe
ansible-playbook playbooks/deploy-mongodb.yml --ask-vault-pass

# Utiliser un fichier de mot de passe
echo "vault_password" > ~/.vault_pass.txt
chmod 600 ~/.vault_pass.txt

ansible-playbook playbooks/deploy-mongodb.yml --vault-password-file ~/.vault_pass.txt

# Configurer dans ansible.cfg
[defaults]
vault_password_file = ~/.vault_pass.txt
```

---

## Ansible Tower / AWX Integration

### Job Template Configuration

```yaml
# tower_job_template.yml
---
name: "Deploy MongoDB Replica Set"
description: "Deploy and configure MongoDB Replica Set"
job_type: "run"
inventory: "Production"
project: "MongoDB Infrastructure"
playbook: "playbooks/deploy-mongodb.yml"
credential: "SSH Private Key"
vault_credential: "Vault Password"

extra_vars: |
  ---
  mongodb_version: "7.0"
  environment: production

survey_enabled: true
survey_spec:
  name: "MongoDB Deployment Survey"
  description: "Configuration options for MongoDB deployment"
  spec:
    - question_name: "MongoDB Version"
      question_description: "Select MongoDB version to deploy"
      required: true
      type: "multiplechoice"
      variable: "mongodb_version"
      choices:
        - "7.0"
        - "6.0"
        - "5.0"
      default: "7.0"

    - question_name: "Environment"
      question_description: "Target environment"
      required: true
      type: "multiplechoice"
      variable: "environment"
      choices:
        - "production"
        - "staging"
        - "development"
      default: "production"

    - question_name: "Enable Monitoring"
      question_description: "Install MongoDB exporter for Prometheus"
      required: true
      type: "boolean"
      variable: "mongodb_exporter_enabled"
      default: true

    - question_name: "Enable Backup"
      question_description: "Configure automated backups"
      required: true
      type: "boolean"
      variable: "backup_enabled"
      default: true
```

---

## Tests et Validation

### Playbook de Test

```yaml
# playbooks/test-mongodb.yml
---
- name: Test MongoDB Installation
  hosts: mongodb_replicaset
  become: yes

  tasks:
    - name: Check MongoDB service status
      systemd:
        name: mongod
      register: service_status
      failed_when: service_status.status.ActiveState != "active"

    - name: Test MongoDB port connectivity
      wait_for:
        port: "{{ mongodb_port }}"
        timeout: 10
      register: port_check

    - name: Test MongoDB authentication
      community.mongodb.mongodb_shell:
        login_host: localhost
        login_port: "{{ mongodb_port }}"
        login_user: "{{ mongodb_users[0].name }}"
        login_password: "{{ mongodb_users[0].password }}"
        db: admin
        eval: "db.adminCommand('ping')"
      register: auth_test

    - name: Test replica set status
      community.mongodb.mongodb_status:
        login_host: localhost
        login_port: "{{ mongodb_port }}"
        login_user: "{{ mongodb_users[0].name }}"
        login_password: "{{ mongodb_users[0].password }}"
      register: rs_status
      failed_when: rs_status.replicaset.members | selectattr('health', 'equalto', 0) | list | length > 0

    - name: Test write operation
      community.mongodb.mongodb_shell:
        login_host: localhost
        login_port: "{{ mongodb_port }}"
        login_user: "{{ mongodb_users[1].name }}"
        login_password: "{{ mongodb_users[1].password }}"
        db: "{{ mongodb_users[1].database }}"
        eval: |
          db.test.insertOne({
            test: "ansible-test",
            timestamp: new Date()
          });
          db.test.deleteMany({ test: "ansible-test" });
      register: write_test
      run_once: true

    - name: Generate test report
      template:
        src: test-report.j2
        dest: "/tmp/mongodb-test-report-{{ inventory_hostname }}.txt"
      delegate_to: localhost
```

---

## Best Practices

### Checklist Ansible MongoDB

```yaml
# ansible-best-practices.yaml
---
organization:
  - ✅ Structure de projet claire et cohérente
  - ✅ Séparation des environnements (dev/staging/prod)
  - ✅ Inventaires dynamiques pour le cloud
  - ✅ Variables par environnement avec group_vars
  - ✅ Secrets gérés avec Ansible Vault

roles:
  - ✅ Rôles modulaires et réutilisables
  - ✅ Variables avec defaults et vars
  - ✅ Handlers pour les redémarrages
  - ✅ Tags pour exécution sélective
  - ✅ Documentation dans README.md

playbooks:
  - ✅ Playbooks idempotents
  - ✅ Pre-tasks et post-tasks pour validation
  - ✅ Error handling avec blocks
  - ✅ Serial execution pour rolling updates
  - ✅ Dry-run testing avec --check

security:
  - ✅ Ansible Vault pour credentials
  - ✅ Rotation régulière des secrets
  - ✅ Principle of least privilege
  - ✅ SSH key authentication
  - ✅ Audit logs activés

performance:
  - ✅ Fact caching configuré
  - ✅ SSH pipelining activé
  - ✅ Parallel execution (forks)
  - ✅ Optimisation des loops
  - ✅ Async tasks pour opérations longues

testing:
  - ✅ Syntax check avec --syntax-check
  - ✅ Dry-run avec --check
  - ✅ Test playbooks dédiés
  - ✅ Molecule pour tests de rôles
  - ✅ CI/CD integration

documentation:
  - ✅ README.md pour chaque rôle
  - ✅ Commentaires dans les playbooks
  - ✅ Variables documentées
  - ✅ Exemples d'utilisation
  - ✅ Troubleshooting guide
```

### Makefile pour Simplifier l'Utilisation

```makefile
# Makefile
.PHONY: help deploy test backup upgrade clean

INVENTORY ?= inventory/production
PLAYBOOK_DIR = playbooks
VAULT_FILE = ~/.vault_pass.txt

help:
	@echo "Available targets:"
	@echo "  deploy      - Deploy MongoDB cluster"
	@echo "  test        - Test MongoDB installation"
	@echo "  backup      - Create MongoDB backup"
	@echo "  upgrade     - Upgrade MongoDB version"
	@echo "  clean       - Clean temporary files"

deploy:
	ansible-playbook $(PLAYBOOK_DIR)/deploy-mongodb.yml \
		-i $(INVENTORY) \
		--vault-password-file $(VAULT_FILE)

test:
	ansible-playbook $(PLAYBOOK_DIR)/test-mongodb.yml \
		-i $(INVENTORY) \
		--vault-password-file $(VAULT_FILE)

backup:
	ansible-playbook $(PLAYBOOK_DIR)/backup-mongodb.yml \
		-i $(INVENTORY) \
		--vault-password-file $(VAULT_FILE)

upgrade:
	ansible-playbook $(PLAYBOOK_DIR)/upgrade-mongodb.yml \
		-i $(INVENTORY) \
		--vault-password-file $(VAULT_FILE) \
		--ask-vars

syntax-check:
	ansible-playbook $(PLAYBOOK_DIR)/*.yml \
		-i $(INVENTORY) \
		--syntax-check

lint:
	ansible-lint $(PLAYBOOK_DIR)/*.yml roles/*/

clean:
	find . -name "*.retry" -delete
	find . -name "__pycache__" -type d -exec rm -rf {} +
	rm -f ansible.log
```

---

## Conclusion

Ansible fournit une solution puissante et flexible pour automatiser le déploiement et la gestion de MongoDB :

**Avantages clés :**
- **Agentless** : Pas d'agent à maintenir sur les serveurs
- **Idempotence** : Exécutions répétables sans effets secondaires
- **Déclaratif** : Description de l'état désiré plutôt que des commandes
- **Modulaire** : Rôles réutilisables et composables
- **Écosystème** : Collections community et Galaxy

**Use Cases principaux :**
- Déploiement initial de clusters MongoDB
- Configuration et reconfiguration
- Rolling updates et upgrades
- Backup et restore automatisés
- Opérations de maintenance
- Disaster recovery

**Intégration avec :**
- Terraform (provision + configuration)
- CI/CD pipelines (GitLab, Jenkins)
- Ansible Tower/AWX (orchestration centralisée)
- Monitoring (Prometheus, Grafana)

Ansible simplifie considérablement la gestion de flottes MongoDB en automatisant les tâches répétitives et en garantissant la cohérence des configurations entre environnements.

---


⏭️ [Terraform et MongoDB Atlas](/18-devops-deploiement/07-terraform-mongodb-atlas.md)
