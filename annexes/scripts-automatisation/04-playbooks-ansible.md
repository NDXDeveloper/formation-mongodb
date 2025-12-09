🔝 Retour au [Sommaire](/SOMMAIRE.md)

# G.4 Playbooks Ansible

## Introduction

**Ansible** permet d'automatiser le déploiement, la configuration et la maintenance de MongoDB à grande échelle. Cette section fournit des playbooks prêts à l'emploi pour toutes les tâches d'infrastructure MongoDB.

## Avantages d'Ansible pour MongoDB

| Avantage | Description |
|----------|-------------|
| **Idempotence** | Exécution sûre et répétable |
| **Sans agent** | Communication via SSH |
| **Déclaratif** | Infrastructure as Code |
| **Réutilisable** | Rôles modulaires |
| **Versionnable** | Historique Git complet |
| **Testable** | Environnements reproductibles |

## Structure de projet recommandée

```
mongodb-ansible/
├── ansible.cfg                 # Configuration Ansible
├── inventory/
│   ├── production/
│   │   ├── hosts.yml          # Inventaire production
│   │   └── group_vars/
│   │       ├── all.yml        # Variables globales
│   │       ├── mongodb.yml    # Variables MongoDB
│   │       └── vault.yml      # Secrets chiffrés
│   └── staging/
│       ├── hosts.yml
│       └── group_vars/
├── playbooks/
│   ├── install-mongodb.yml
│   ├── configure-replica-set.yml
│   ├── configure-sharding.yml
│   ├── backup.yml
│   ├── maintenance.yml
│   └── security.yml
├── roles/
│   ├── mongodb-common/
│   ├── mongodb-standalone/
│   ├── mongodb-replica-set/
│   ├── mongodb-sharding/
│   ├── mongodb-backup/
│   └── mongodb-monitoring/
└── group_vars/
    └── all.yml
```

## Configuration Ansible

### ansible.cfg

```ini
[defaults]
inventory = inventory/production/hosts.yml
remote_user = ansible
host_key_checking = False
retry_files_enabled = False
roles_path = roles
forks = 10
timeout = 30
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
control_path = /tmp/ansible-ssh-%%h-%%p-%%r
```

## Inventaires

### inventory/production/hosts.yml

```yaml
---
all:
  children:
    mongodb_standalone:
      hosts:
        mongo-standalone-01:
          ansible_host: 192.168.1.10

    mongodb_replica_set:
      children:
        mongodb_primary:
          hosts:
            mongo-rs-01:
              ansible_host: 192.168.1.20
              mongodb_replica_set_role: primary

        mongodb_secondary:
          hosts:
            mongo-rs-02:
              ansible_host: 192.168.1.21
              mongodb_replica_set_role: secondary
            mongo-rs-03:
              ansible_host: 192.168.1.22
              mongodb_replica_set_role: secondary

    mongodb_sharding:
      children:
        mongodb_config_servers:
          hosts:
            mongo-config-01:
              ansible_host: 192.168.1.30
            mongo-config-02:
              ansible_host: 192.168.1.31
            mongo-config-03:
              ansible_host: 192.168.1.32

        mongodb_shards:
          hosts:
            mongo-shard1-01:
              ansible_host: 192.168.1.40
              shard_name: shard1
            mongo-shard2-01:
              ansible_host: 192.168.1.50
              shard_name: shard2

        mongodb_mongos:
          hosts:
            mongo-mongos-01:
              ansible_host: 192.168.1.60
            mongo-mongos-02:
              ansible_host: 192.168.1.61

    mongodb_backup_servers:
      hosts:
        mongo-backup-01:
          ansible_host: 192.168.1.70

  vars:
    ansible_user: ansible
    ansible_python_interpreter: /usr/bin/python3
```

### inventory/production/group_vars/all.yml

```yaml
---
# Configuration générale
mongodb_version: "7.0"
mongodb_package: "mongodb-org"

# Chemins
mongodb_data_dir: "/var/lib/mongodb"
mongodb_log_dir: "/var/log/mongodb"
mongodb_conf_dir: "/etc/mongod.conf"

# Réseau
mongodb_bind_ip: "0.0.0.0"
mongodb_port: 27017

# Sécurité
mongodb_enable_auth: true
mongodb_enable_encryption: true

# Performance
mongodb_wiredtiger_cache_size_gb: 2

# Backup
mongodb_backup_enabled: true
mongodb_backup_retention_days: 30
```

### inventory/production/group_vars/vault.yml (chiffré)

```yaml
---
# Utiliser: ansible-vault create vault.yml
# Éditer: ansible-vault edit vault.yml
# Voir: ansible-vault view vault.yml

# Credentials MongoDB
mongodb_admin_user: "admin"
mongodb_admin_password: "SuperSecurePassword123!"
mongodb_backup_user: "backup"
mongodb_backup_password: "BackupPassword456!"
mongodb_monitoring_user: "monitor"
mongodb_monitoring_password: "MonitorPassword789!"

# Clés de chiffrement
mongodb_keyfile_content: |
  YourVeryLongAndSecureKeyFileContentHere
  MustBeAtLeast1024BytesLong...
```

## Playbook 1 : Installation MongoDB

### playbooks/install-mongodb.yml

```yaml
---
- name: Install MongoDB
  hosts: "{{ target_hosts | default('all') }}"
  become: true

  vars:
    mongodb_version: "7.0"

  tasks:
    - name: Import MongoDB GPG key (Debian/Ubuntu)
      apt_key:
        url: "https://www.mongodb.org/static/pgp/server-{{ mongodb_version }}.asc"
        state: present
      when: ansible_os_family == "Debian"

    - name: Add MongoDB repository (Debian/Ubuntu)
      apt_repository:
        repo: "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu {{ ansible_distribution_release }}/mongodb-org/{{ mongodb_version }} multiverse"
        state: present
        filename: mongodb-org
      when: ansible_os_family == "Debian"

    - name: Add MongoDB repository (RedHat/CentOS)
      yum_repository:
        name: mongodb-org-{{ mongodb_version }}
        description: MongoDB Repository
        baseurl: "https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/{{ mongodb_version }}/x86_64/"
        gpgcheck: yes
        enabled: yes
        gpgkey: "https://www.mongodb.org/static/pgp/server-{{ mongodb_version }}.asc"
      when: ansible_os_family == "RedHat"

    - name: Install MongoDB packages
      package:
        name:
          - mongodb-org
          - mongodb-org-server
          - mongodb-org-shell
          - mongodb-org-mongos
          - mongodb-org-tools
        state: present

    - name: Install pymongo (for Ansible modules)
      pip:
        name: pymongo
        state: present

    - name: Create MongoDB data directory
      file:
        path: "{{ mongodb_data_dir }}"
        state: directory
        owner: mongodb
        group: mongodb
        mode: '0755'

    - name: Create MongoDB log directory
      file:
        path: "{{ mongodb_log_dir }}"
        state: directory
        owner: mongodb
        group: mongodb
        mode: '0755'

    - name: Configure MongoDB
      template:
        src: templates/mongod.conf.j2
        dest: /etc/mongod.conf
        owner: mongodb
        group: mongodb
        mode: '0644'
      notify: restart mongodb

    - name: Enable and start MongoDB service
      systemd:
        name: mongod
        enabled: yes
        state: started

  handlers:
    - name: restart mongodb
      systemd:
        name: mongod
        state: restarted
```

### templates/mongod.conf.j2

```jinja2
# MongoDB configuration file

# Storage
storage:
  dbPath: {{ mongodb_data_dir }}
  journal:
    enabled: true
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: {{ mongodb_wiredtiger_cache_size_gb }}
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true

# System log
systemLog:
  destination: file
  logAppend: true
  path: {{ mongodb_log_dir }}/mongod.log

# Network
net:
  port: {{ mongodb_port }}
  bindIp: {{ mongodb_bind_ip }}
{% if mongodb_enable_encryption %}
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb.pem
    CAFile: /etc/ssl/ca.pem
{% endif %}

# Process management
processManagement:
  timeZoneInfo: /usr/share/zoneinfo

{% if mongodb_enable_auth %}
# Security
security:
  authorization: enabled
  keyFile: /etc/mongodb-keyfile
{% endif %}

{% if mongodb_replica_set_name is defined %}
# Replication
replication:
  replSetName: {{ mongodb_replica_set_name }}
{% endif %}

{% if mongodb_sharding_role is defined %}
# Sharding
sharding:
  clusterRole: {{ mongodb_sharding_role }}
{% endif %}
```

## Playbook 2 : Configuration Replica Set

### playbooks/configure-replica-set.yml

```yaml
---
- name: Configure MongoDB Replica Set
  hosts: mongodb_replica_set
  become: true

  vars:
    mongodb_replica_set_name: "rs0"

  pre_tasks:
    - name: Ensure MongoDB is installed
      package:
        name: mongodb-org
        state: present

  tasks:
    - name: Generate keyfile for replica set
      copy:
        content: "{{ mongodb_keyfile_content }}"
        dest: /etc/mongodb-keyfile
        owner: mongodb
        group: mongodb
        mode: '0400'
      when: mongodb_enable_auth

    - name: Configure MongoDB for replica set
      template:
        src: templates/mongod-replica.conf.j2
        dest: /etc/mongod.conf
        owner: mongodb
        group: mongodb
        mode: '0644'
      notify: restart mongodb

    - name: Restart MongoDB
      meta: flush_handlers

    - name: Wait for MongoDB to start
      wait_for:
        port: "{{ mongodb_port }}"
        delay: 5
        timeout: 60

- name: Initialize Replica Set
  hosts: mongodb_primary[0]
  become: true

  tasks:
    - name: Check if replica set is already initialized
      mongodb_shell:
        login_port: "{{ mongodb_port }}"
        eval: "rs.status()"
      register: rs_status
      ignore_errors: yes

    - name: Initialize replica set
      mongodb_shell:
        login_port: "{{ mongodb_port }}"
        eval: |
          rs.initiate({
            _id: "{{ mongodb_replica_set_name }}",
            members: [
              { _id: 0, host: "{{ hostvars[groups['mongodb_primary'][0]]['ansible_host'] }}:{{ mongodb_port }}", priority: 2 }
            ]
          })
      when: rs_status.failed

    - name: Wait for primary election
      wait_for:
        timeout: 30
      when: rs_status.failed

    - name: Add secondary members
      mongodb_shell:
        login_port: "{{ mongodb_port }}"
        eval: |
          rs.add("{{ hostvars[item]['ansible_host'] }}:{{ mongodb_port }}")
      loop: "{{ groups['mongodb_secondary'] }}"
      when: rs_status.failed

- name: Create admin user
  hosts: mongodb_primary[0]
  become: true

  tasks:
    - name: Create admin user
      mongodb_user:
        login_port: "{{ mongodb_port }}"
        database: admin
        name: "{{ mongodb_admin_user }}"
        password: "{{ mongodb_admin_password }}"
        roles:
          - role: root
            db: admin
        state: present
      when: mongodb_enable_auth

    - name: Create backup user
      mongodb_user:
        login_user: "{{ mongodb_admin_user }}"
        login_password: "{{ mongodb_admin_password }}"
        login_port: "{{ mongodb_port }}"
        database: admin
        name: "{{ mongodb_backup_user }}"
        password: "{{ mongodb_backup_password }}"
        roles:
          - role: backup
            db: admin
          - role: restore
            db: admin
        state: present
      when: mongodb_enable_auth
```

## Playbook 3 : Configuration Sharding

### playbooks/configure-sharding.yml

```yaml
---
- name: Deploy Config Servers
  hosts: mongodb_config_servers
  become: true

  vars:
    mongodb_replica_set_name: "configReplSet"
    mongodb_sharding_role: "configsvr"

  tasks:
    - name: Configure config servers
      template:
        src: templates/mongod-config.conf.j2
        dest: /etc/mongod.conf
        mode: '0644'
      notify: restart mongodb

    - name: Restart MongoDB
      meta: flush_handlers

- name: Initialize Config Server Replica Set
  hosts: mongodb_config_servers[0]
  become: true

  tasks:
    - name: Initialize config server replica set
      mongodb_shell:
        login_port: 27019
        eval: |
          rs.initiate({
            _id: "configReplSet",
            configsvr: true,
            members: [
              {% for host in groups['mongodb_config_servers'] %}
              { _id: {{ loop.index0 }}, host: "{{ hostvars[host]['ansible_host'] }}:27019" }{{ "," if not loop.last else "" }}
              {% endfor %}
            ]
          })

- name: Deploy Shards
  hosts: mongodb_shards
  become: true

  vars:
    mongodb_sharding_role: "shardsvr"

  tasks:
    - name: Configure shard servers
      template:
        src: templates/mongod-shard.conf.j2
        dest: /etc/mongod.conf
        mode: '0644'
      notify: restart mongodb

    - name: Restart MongoDB
      meta: flush_handlers

- name: Deploy mongos routers
  hosts: mongodb_mongos
  become: true

  tasks:
    - name: Configure mongos
      template:
        src: templates/mongos.conf.j2
        dest: /etc/mongos.conf
        mode: '0644'

    - name: Create mongos systemd service
      template:
        src: templates/mongos.service.j2
        dest: /etc/systemd/system/mongos.service
        mode: '0644'

    - name: Start mongos service
      systemd:
        name: mongos
        enabled: yes
        state: started
        daemon_reload: yes

- name: Configure Sharded Cluster
  hosts: mongodb_mongos[0]
  become: true

  tasks:
    - name: Add shards to cluster
      mongodb_shard:
        login_host: localhost
        login_port: 27017
        shard: "{{ hostvars[item]['shard_name'] }}/{{ hostvars[item]['ansible_host'] }}:27018"
        state: present
      loop: "{{ groups['mongodb_shards'] }}"
```

### templates/mongos.conf.j2

```jinja2
# Mongos configuration

# System log
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongos.log

# Network
net:
  port: 27017
  bindIp: {{ mongodb_bind_ip }}

# Sharding
sharding:
  configDB: configReplSet/{% for host in groups['mongodb_config_servers'] %}{{ hostvars[host]['ansible_host'] }}:27019{{ "," if not loop.last else "" }}{% endfor %}

{% if mongodb_enable_auth %}
# Security
security:
  keyFile: /etc/mongodb-keyfile
{% endif %}
```

## Playbook 4 : Gestion des utilisateurs

### playbooks/manage-users.yml

```yaml
---
- name: Manage MongoDB Users
  hosts: mongodb_primary[0]
  become: true

  vars:
    mongodb_users:
      - name: app_user
        password: "AppPassword123"
        database: myapp
        roles:
          - role: readWrite
            db: myapp
      - name: readonly_user
        password: "ReadOnly456"
        database: myapp
        roles:
          - role: read
            db: myapp
      - name: monitoring_user
        password: "{{ mongodb_monitoring_password }}"
        database: admin
        roles:
          - role: clusterMonitor
            db: admin
          - role: read
            db: local

  tasks:
    - name: Create MongoDB users
      mongodb_user:
        login_user: "{{ mongodb_admin_user }}"
        login_password: "{{ mongodb_admin_password }}"
        login_port: "{{ mongodb_port }}"
        database: "{{ item.database }}"
        name: "{{ item.name }}"
        password: "{{ item.password }}"
        roles: "{{ item.roles }}"
        state: present
      loop: "{{ mongodb_users }}"
      no_log: true  # Ne pas logger les mots de passe
```

## Playbook 5 : Backup automatisé

### playbooks/backup.yml

```yaml
---
- name: MongoDB Backup
  hosts: mongodb_backup_servers
  become: true

  vars:
    backup_dir: "/backup/mongodb"
    backup_retention_days: "{{ mongodb_backup_retention_days }}"
    mongodb_uri: "mongodb://{{ mongodb_backup_user }}:{{ mongodb_backup_password }}@{{ hostvars[groups['mongodb_primary'][0]]['ansible_host'] }}:{{ mongodb_port }}/admin"

  tasks:
    - name: Ensure backup directory exists
      file:
        path: "{{ backup_dir }}"
        state: directory
        owner: mongodb
        group: mongodb
        mode: '0755'

    - name: Install MongoDB Database Tools
      package:
        name: mongodb-database-tools
        state: present

    - name: Create backup script
      template:
        src: templates/backup-script.sh.j2
        dest: /usr/local/bin/mongodb-backup.sh
        mode: '0755'

    - name: Setup backup cron job
      cron:
        name: "MongoDB daily backup"
        minute: "0"
        hour: "2"
        job: "/usr/local/bin/mongodb-backup.sh >> /var/log/mongodb/backup.log 2>&1"
        user: mongodb

    - name: Create cleanup script for old backups
      template:
        src: templates/cleanup-backups.sh.j2
        dest: /usr/local/bin/cleanup-old-backups.sh
        mode: '0755'

    - name: Setup cleanup cron job
      cron:
        name: "Cleanup old MongoDB backups"
        minute: "0"
        hour: "3"
        job: "/usr/local/bin/cleanup-old-backups.sh >> /var/log/mongodb/cleanup.log 2>&1"
        user: mongodb
```

### templates/backup-script.sh.j2

```bash
#!/bin/bash
set -euo pipefail

BACKUP_DIR="{{ backup_dir }}"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="backup_${TIMESTAMP}"
BACKUP_PATH="${BACKUP_DIR}/${BACKUP_NAME}"

echo "[$(date)] Starting backup..."

mongodump \
    --uri="{{ mongodb_uri }}" \
    --out="${BACKUP_PATH}" \
    --gzip \
    --oplog

if [ $? -eq 0 ]; then
    echo "[$(date)] Backup completed: ${BACKUP_PATH}"
else
    echo "[$(date)] Backup failed!"
    exit 1
fi
```

### templates/cleanup-backups.sh.j2

```bash
#!/bin/bash
set -euo pipefail

BACKUP_DIR="{{ backup_dir }}"
RETENTION_DAYS={{ backup_retention_days }}

echo "[$(date)] Cleaning up backups older than ${RETENTION_DAYS} days..."

find "${BACKUP_DIR}" -type d -mtime +${RETENTION_DAYS} -exec rm -rf {} +

echo "[$(date)] Cleanup completed"
```

## Playbook 6 : Maintenance

### playbooks/maintenance.yml

```yaml
---
- name: MongoDB Maintenance
  hosts: "{{ target_hosts | default('mongodb_secondary') }}"
  become: true
  serial: 1  # Un serveur à la fois

  vars:
    maintenance_tasks:
      - compact
      - reindex
      - validate

  tasks:
    - name: Check if host is secondary
      mongodb_shell:
        login_user: "{{ mongodb_admin_user }}"
        login_password: "{{ mongodb_admin_password }}"
        login_port: "{{ mongodb_port }}"
        eval: "rs.isMaster().secondary"
      register: is_secondary

    - name: Fail if host is primary
      fail:
        msg: "Cannot run maintenance on PRIMARY node"
      when: not is_secondary.transformed_output | bool

    - name: Get list of databases
      mongodb_shell:
        login_user: "{{ mongodb_admin_user }}"
        login_password: "{{ mongodb_admin_password }}"
        login_port: "{{ mongodb_port }}"
        eval: |
          db.adminCommand('listDatabases').databases
            .filter(d => !['admin', 'config', 'local'].includes(d.name))
            .map(d => d.name)
      register: databases

    - name: Compact collections
      mongodb_shell:
        login_user: "{{ mongodb_admin_user }}"
        login_password: "{{ mongodb_admin_password }}"
        login_port: "{{ mongodb_port }}"
        database: "{{ item }}"
        eval: |
          db.getCollectionNames().forEach(function(coll) {
            print("Compacting " + coll);
            db.runCommand({compact: coll, force: true});
          });
      loop: "{{ databases.transformed_output }}"
      when: "'compact' in maintenance_tasks"

    - name: Reindex collections
      mongodb_shell:
        login_user: "{{ mongodb_admin_user }}"
        login_password: "{{ mongodb_admin_password }}"
        login_port: "{{ mongodb_port }}"
        database: "{{ item }}"
        eval: |
          db.getCollectionNames().forEach(function(coll) {
            print("Reindexing " + coll);
            db[coll].reIndex();
          });
      loop: "{{ databases.transformed_output }}"
      when: "'reindex' in maintenance_tasks"

    - name: Validate collections
      mongodb_shell:
        login_user: "{{ mongodb_admin_user }}"
        login_password: "{{ mongodb_admin_password }}"
        login_port: "{{ mongodb_port }}"
        database: "{{ item }}"
        eval: |
          db.getCollectionNames().forEach(function(coll) {
            print("Validating " + coll);
            var result = db[coll].validate({full: true});
            if (!result.valid) {
              print("INVALID: " + coll);
            }
          });
      loop: "{{ databases.transformed_output }}"
      when: "'validate' in maintenance_tasks"
      register: validation_results

    - name: Report validation errors
      debug:
        msg: "Validation errors found: {{ validation_results }}"
      when:
        - "'validate' in maintenance_tasks"
        - validation_results.failed is defined and validation_results.failed
```

## Playbook 7 : Monitoring

### playbooks/deploy-monitoring.yml

```yaml
---
- name: Deploy MongoDB Monitoring
  hosts: mongodb_replica_set:mongodb_sharding
  become: true

  tasks:
    - name: Install MongoDB Exporter for Prometheus
      get_url:
        url: "https://github.com/percona/mongodb_exporter/releases/download/v0.40.0/mongodb_exporter-0.40.0.linux-amd64.tar.gz"
        dest: /tmp/mongodb_exporter.tar.gz

    - name: Extract MongoDB Exporter
      unarchive:
        src: /tmp/mongodb_exporter.tar.gz
        dest: /opt/
        remote_src: yes

    - name: Create MongoDB Exporter systemd service
      template:
        src: templates/mongodb-exporter.service.j2
        dest: /etc/systemd/system/mongodb-exporter.service
        mode: '0644'

    - name: Start MongoDB Exporter
      systemd:
        name: mongodb-exporter
        enabled: yes
        state: started
        daemon_reload: yes

    - name: Deploy monitoring scripts
      copy:
        src: "{{ item }}"
        dest: /usr/local/bin/
        mode: '0755'
      with_fileglob:
        - "../scripts/monitoring/*.sh"

    - name: Setup health check cron
      cron:
        name: "MongoDB health check"
        minute: "*/5"
        job: "/usr/local/bin/health-check.sh"
```

### templates/mongodb-exporter.service.j2

```ini
[Unit]
Description=MongoDB Exporter
After=network.target

[Service]
Type=simple
User=mongodb
Environment="MONGODB_URI=mongodb://{{ mongodb_monitoring_user }}:{{ mongodb_monitoring_password }}@localhost:{{ mongodb_port }}"
ExecStart=/opt/mongodb_exporter/mongodb_exporter
Restart=always

[Install]
WantedBy=multi-user.target
```

## Playbook 8 : Sécurité

### playbooks/security-hardening.yml

```yaml
---
- name: MongoDB Security Hardening
  hosts: all
  become: true

  tasks:
    - name: Configure firewall (UFW)
      ufw:
        rule: allow
        port: "{{ mongodb_port }}"
        proto: tcp
        from_ip: "{{ item }}"
      loop: "{{ mongodb_allowed_ips }}"
      when: ansible_os_family == "Debian"

    - name: Configure firewall (firewalld)
      firewalld:
        port: "{{ mongodb_port }}/tcp"
        permanent: yes
        state: enabled
        source: "{{ item }}"
      loop: "{{ mongodb_allowed_ips }}"
      when: ansible_os_family == "RedHat"

    - name: Generate TLS certificates
      command: |
        openssl req -newkey rsa:4096 -nodes -sha256 -x509 -days 365 \
          -out /etc/ssl/mongodb.pem \
          -keyout /etc/ssl/mongodb-key.pem \
          -subj "/C=US/ST=State/L=City/O=Organization/CN={{ ansible_hostname }}"
      args:
        creates: /etc/ssl/mongodb.pem
      when: mongodb_enable_encryption

    - name: Combine certificate and key
      shell: cat /etc/ssl/mongodb.pem /etc/ssl/mongodb-key.pem > /etc/ssl/mongodb-combined.pem
      args:
        creates: /etc/ssl/mongodb-combined.pem
      when: mongodb_enable_encryption

    - name: Set certificate permissions
      file:
        path: /etc/ssl/mongodb-combined.pem
        owner: mongodb
        group: mongodb
        mode: '0400'
      when: mongodb_enable_encryption

    - name: Enable audit logging
      lineinfile:
        path: /etc/mongod.conf
        insertafter: "^systemLog:"
        line: |
          auditLog:
            destination: file
            format: JSON
            path: /var/log/mongodb/audit.log
      notify: restart mongodb

    - name: Disable transparent huge pages
      template:
        src: templates/disable-thp.service.j2
        dest: /etc/systemd/system/disable-transparent-hugepages.service
        mode: '0644'

    - name: Enable and start THP disable service
      systemd:
        name: disable-transparent-hugepages
        enabled: yes
        state: started
        daemon_reload: yes

  handlers:
    - name: restart mongodb
      systemd:
        name: mongod
        state: restarted
```

## Rôle réutilisable : mongodb-common

### roles/mongodb-common/tasks/main.yml

```yaml
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_os_family }}.yml"

- name: Install MongoDB repository
  include_tasks: "install-repo-{{ ansible_os_family }}.yml"

- name: Install MongoDB packages
  package:
    name: "{{ mongodb_packages }}"
    state: present

- name: Create MongoDB directories
  file:
    path: "{{ item }}"
    state: directory
    owner: mongodb
    group: mongodb
    mode: '0755'
  loop:
    - "{{ mongodb_data_dir }}"
    - "{{ mongodb_log_dir }}"

- name: Configure system limits
  pam_limits:
    domain: mongodb
    limit_type: "{{ item.type }}"
    limit_item: "{{ item.item }}"
    value: "{{ item.value }}"
  loop:
    - { type: 'soft', item: 'nofile', value: '64000' }
    - { type: 'hard', item: 'nofile', value: '64000' }
    - { type: 'soft', item: 'nproc', value: '64000' }
    - { type: 'hard', item: 'nproc', value: '64000' }

- name: Configure MongoDB
  template:
    src: mongod.conf.j2
    dest: /etc/mongod.conf
    mode: '0644'
  notify: restart mongodb

- name: Enable MongoDB service
  systemd:
    name: mongod
    enabled: yes
    state: started
```

### roles/mongodb-common/handlers/main.yml

```yaml
---
- name: restart mongodb
  systemd:
    name: mongod
    state: restarted

- name: reload mongodb
  systemd:
    name: mongod
    state: reloaded
```

## Utilisation des playbooks

### Installation standalone

```bash
# Installation de base
ansible-playbook playbooks/install-mongodb.yml

# Installation sur des hôtes spécifiques
ansible-playbook playbooks/install-mongodb.yml --limit mongo-standalone-01

# Avec variables personnalisées
ansible-playbook playbooks/install-mongodb.yml -e "mongodb_version=6.0"
```

### Configuration Replica Set

```bash
# Configuration complète
ansible-playbook playbooks/configure-replica-set.yml

# Vérifier avant exécution
ansible-playbook playbooks/configure-replica-set.yml --check

# Mode verbeux
ansible-playbook playbooks/configure-replica-set.yml -vvv
```

### Gestion des secrets

```bash
# Créer un fichier vault
ansible-vault create inventory/production/group_vars/vault.yml

# Éditer le vault
ansible-vault edit inventory/production/group_vars/vault.yml

# Changer le mot de passe du vault
ansible-vault rekey inventory/production/group_vars/vault.yml

# Exécuter avec vault
ansible-playbook playbooks/install-mongodb.yml --ask-vault-pass

# Utiliser un fichier de mot de passe
echo "my_vault_password" > .vault_pass
chmod 600 .vault_pass
ansible-playbook playbooks/install-mongodb.yml --vault-password-file .vault_pass
```

### Tags pour exécution sélective

```yaml
# Dans un playbook
tasks:
  - name: Install packages
    package:
      name: mongodb-org
      state: present
    tags:
      - install
      - packages

  - name: Configure MongoDB
    template:
      src: mongod.conf.j2
      dest: /etc/mongod.conf
    tags:
      - config
```

```bash
# Exécuter seulement les tâches avec tag 'install'
ansible-playbook playbooks/install-mongodb.yml --tags install

# Sauter les tâches avec tag 'config'
ansible-playbook playbooks/install-mongodb.yml --skip-tags config
```

## Tests et validation

### Playbook de test

```yaml
---
- name: Test MongoDB Installation
  hosts: all
  become: true

  tasks:
    - name: Check MongoDB is running
      systemd:
        name: mongod
        state: started
      check_mode: yes
      register: mongodb_status

    - name: Verify MongoDB version
      shell: mongod --version | head -1
      register: mongodb_version_output

    - name: Test MongoDB connection
      mongodb_shell:
        login_port: "{{ mongodb_port }}"
        eval: "db.adminCommand('ping')"
      register: ping_result

    - name: Display results
      debug:
        msg:
          - "MongoDB Status: {{ mongodb_status.status.ActiveState }}"
          - "MongoDB Version: {{ mongodb_version_output.stdout }}"
          - "Connection Test: {{ 'OK' if ping_result.transformed_output.ok == 1 else 'FAILED' }}"
```

### Molecule pour tests automatisés

```bash
# Installer Molecule
pip install molecule molecule-docker

# Initialiser un scénario de test
cd roles/mongodb-common
molecule init scenario

# Tester le rôle
molecule test
```

## Bonnes pratiques

### Organisation
- ✅ Utiliser des rôles réutilisables
- ✅ Séparer les variables par environnement
- ✅ Versionner dans Git avec `.gitignore` pour les secrets
- ✅ Documenter chaque playbook

### Sécurité
- ✅ Utiliser Ansible Vault pour les secrets
- ✅ Ne jamais committer les mots de passe
- ✅ Limiter les permissions SSH (principe du moindre privilège)
- ✅ Auditer régulièrement les playbooks

### Performance
- ✅ Utiliser `serial` pour les déploiements contrôlés
- ✅ Activer `pipelining` pour SSH
- ✅ Utiliser `async` pour les tâches longues
- ✅ Configurer le cache de facts

### Idempotence
- ✅ Tester avec `--check` avant exécution réelle
- ✅ Utiliser des modules Ansible plutôt que `shell`/`command`
- ✅ Vérifier l'état avant modification
- ✅ Utiliser `creates`/`removes` avec shell/command

### Maintenance
- ✅ Documenter les changements dans un CHANGELOG
- ✅ Tagger les versions stables
- ✅ Maintenir un environnement de test
- ✅ Automatiser les tests avec CI/CD

## Intégration CI/CD

### GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - lint
  - test
  - deploy

ansible-lint:
  stage: lint
  script:
    - ansible-lint playbooks/*.yml

molecule-test:
  stage: test
  script:
    - molecule test

deploy-staging:
  stage: deploy
  script:
    - ansible-playbook -i inventory/staging playbooks/install-mongodb.yml
  only:
    - develop

deploy-production:
  stage: deploy
  script:
    - ansible-playbook -i inventory/production playbooks/install-mongodb.yml
  only:
    - master
  when: manual
```

## Checklist de déploiement

- [ ] Inventaire créé et vérifié
- [ ] Variables configurées par environnement
- [ ] Secrets chiffrés avec Ansible Vault
- [ ] Connexion SSH testée vers tous les hôtes
- [ ] Playbooks testés en staging
- [ ] Sauvegarde effectuée avant modification
- [ ] Fenêtre de maintenance planifiée
- [ ] Équipe informée du déploiement
- [ ] Procédure de rollback documentée
- [ ] Monitoring activé post-déploiement

---


⏭️ Retour au [Sommaire](/SOMMAIRE.md)
