🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.7 Network Security et IP Whitelisting

## Introduction

La sécurité réseau constitue la première ligne de défense pour protéger une infrastructure MongoDB. Avant même que TLS/SSL ou l'authentification n'entrent en jeu, la configuration réseau détermine qui peut atteindre votre serveur MongoDB. Une mauvaise configuration réseau peut exposer votre base de données à Internet, rendant inefficaces toutes les autres mesures de sécurité.

**Principe fondamental** : **Defense in Depth** (défense en profondeur)

```
┌─────────────────────────────────────────────────────────────────┐
│  Couche 1 : Network Security (cette section)                    │
│  • Firewall                                                     │
│  • IP Whitelisting                                              │
│  • Network Segmentation                                         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Couche 2 : Transport Security                                  │
│  • TLS/SSL (section 11.5.1)                                     │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Couche 3 : Authentication & Authorization                      │
│  • SCRAM-SHA-256, x509, LDAP                                    │
│  • RBAC                                                         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Couche 4 : Encryption                                          │
│  • Encryption at Rest (section 11.5.2)                          │
│  • CSFLE (section 11.5.3)                                       │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Couche 5 : Audit                                               │
│  • Audit logging (section 11.6)                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Vecteurs d'attaque réseau

Sans sécurité réseau appropriée :

```
Exposition directe à Internet (bindIp: 0.0.0.0)
├─ Port scanning automatisé
├─ Tentatives de brute force
├─ Exploits de vulnérabilités connues
├─ DDoS attacks
└─ Compromission totale du serveur

Accès depuis réseau interne non segmenté
├─ Mouvement latéral après compromission
├─ Accès depuis postes de travail infectés
└─ Exfiltration de données

Absence de contrôle d'accès réseau
├─ Impossible de tracer les sources d'accès
├─ Tous les serveurs internes peuvent se connecter
└─ Pas de séparation dev/staging/prod
```

## Configuration de base MongoDB

### Paramètre bindIp

Le paramètre `bindIp` définit les interfaces réseau sur lesquelles MongoDB écoute.

#### Configuration par défaut (insécure)

```yaml
# ❌ DANGEREUX : Écoute sur toutes les interfaces
net:
  port: 27017
  bindIp: 0.0.0.0  # Accessible depuis n'importe quelle interface
```

**Conséquences** :
- MongoDB accessible depuis Internet si le serveur a une IP publique
- Exposition maximale
- **Ne JAMAIS utiliser en production**

#### Configuration sécurisée (localhost uniquement)

```yaml
# ✅ Sécurisé : Localhost uniquement (application sur même serveur)
net:
  port: 27017
  bindIp: 127.0.0.1
```

**Usage** :
- Application et MongoDB sur le même serveur
- Développement local
- Conteneurs Docker sur même host

#### Configuration production (IP privée)

```yaml
# ✅ Production : IP privée spécifique
net:
  port: 27017
  bindIp: 10.0.1.10  # IP privée du serveur
```

**Usage** :
- Architecture multi-serveurs
- Replica Set
- Sharded Cluster

#### Configuration multi-interfaces

```yaml
# ✅ Multiple interfaces (avec prudence)
net:
  port: 27017
  bindIp: 127.0.0.1,10.0.1.10,10.0.2.10
  # localhost + 2 interfaces privées
```

**Usage** :
- Serveur avec plusieurs interfaces réseau
- Séparation trafic applicatif / réplication
- Multi-datacenter

### Vérification de la configuration

```bash
# Vérifier sur quelles interfaces MongoDB écoute
netstat -tuln | grep 27017

# Sortie attendue (production) :
# tcp  0  0  10.0.1.10:27017  0.0.0.0:*  LISTEN

# ⚠️ Sortie dangereuse :
# tcp  0  0  0.0.0.0:27017    0.0.0.0:*  LISTEN  # Accessible partout !

# Alternative avec ss (plus moderne)
ss -tuln | grep 27017

# Tester l'accessibilité depuis l'extérieur
telnet <server-ip> 27017

# Si aucune réponse : ✅ Bon signe (firewall bloque)
# Si connexion établie : ⚠️ Vérifier la configuration
```

### Ports MongoDB à contrôler

```
Ports standards MongoDB :
├─ 27017 : mongod (port par défaut)
├─ 27018 : mongod (shard server dans un cluster)
├─ 27019 : mongod (config server dans un cluster)
├─ 27020 : mongocryptd (CSFLE)
└─ 28017 : Web Status Page (obsolète, désactivé par défaut)

Ports Replica Set :
└─ Utilise le même port (27017) pour réplication

Ports mongos (router) :
└─ 27017 (ou personnalisé)

⚠️ AUCUN de ces ports ne doit être exposé à Internet
```

## Firewall Linux

### iptables (traditionnel)

#### Configuration de base

```bash
#!/bin/bash
# mongodb-firewall-iptables.sh

# Politique par défaut : TOUT BLOQUER
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Autoriser loopback
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# Autoriser connexions établies
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# SSH (administration)
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# MongoDB : Autoriser uniquement depuis les serveurs applicatifs
# Serveur app 1
iptables -A INPUT -p tcp -s 10.0.1.20 --dport 27017 -j ACCEPT
# Serveur app 2
iptables -A INPUT -p tcp -s 10.0.1.21 --dport 27017 -j ACCEPT
# Serveur app 3
iptables -A INPUT -p tcp -s 10.0.1.22 --dport 27017 -j ACCEPT

# MongoDB : Autoriser depuis autres membres du Replica Set
# Primary
iptables -A INPUT -p tcp -s 10.0.2.10 --dport 27017 -j ACCEPT
# Secondary 1
iptables -A INPUT -p tcp -s 10.0.2.11 --dport 27017 -j ACCEPT
# Secondary 2
iptables -A INPUT -p tcp -s 10.0.2.12 --dport 27017 -j ACCEPT

# Bloquer tout le reste (déjà défini par politique par défaut)
# Mais on peut logger les tentatives
iptables -A INPUT -p tcp --dport 27017 -j LOG --log-prefix "MongoDB-BLOCKED: "
iptables -A INPUT -p tcp --dport 27017 -j DROP

# Sauvegarder les règles
iptables-save > /etc/iptables/rules.v4

# Pour Ubuntu/Debian, installer iptables-persistent
apt-get install -y iptables-persistent
```

#### Configuration avancée (rate limiting)

```bash
#!/bin/bash
# mongodb-firewall-iptables-advanced.sh

# Protection contre SYN flood
iptables -A INPUT -p tcp --syn -m limit --limit 1/s --limit-burst 3 -j ACCEPT
iptables -A INPUT -p tcp --syn -j DROP

# Rate limiting MongoDB (max 100 nouvelles connexions/min par IP)
iptables -A INPUT -p tcp --dport 27017 -m state --state NEW \
  -m recent --set --name MONGO_CONN

iptables -A INPUT -p tcp --dport 27017 -m state --state NEW \
  -m recent --update --seconds 60 --hitcount 100 --name MONGO_CONN \
  -j LOG --log-prefix "MongoDB-RATE-LIMIT: "

iptables -A INPUT -p tcp --dport 27017 -m state --state NEW \
  -m recent --update --seconds 60 --hitcount 100 --name MONGO_CONN \
  -j DROP

# Protection contre port scanning
iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP
iptables -A INPUT -p tcp --tcp-flags ALL ALL -j DROP
```

#### Script de gestion

```bash
#!/bin/bash
# /usr/local/bin/mongodb-firewall.sh

case "$1" in
  start)
    echo "Starting MongoDB firewall..."
    /etc/iptables/mongodb-rules.sh
    ;;
  stop)
    echo "Stopping MongoDB firewall..."
    iptables -F
    iptables -P INPUT ACCEPT
    iptables -P FORWARD ACCEPT
    iptables -P OUTPUT ACCEPT
    ;;
  restart)
    $0 stop
    sleep 1
    $0 start
    ;;
  status)
    iptables -L -n -v | grep 27017
    ;;
  *)
    echo "Usage: $0 {start|stop|restart|status}"
    exit 1
esac
```

### firewalld (RHEL/CentOS/Fedora)

#### Configuration de base

```bash
# Démarrer firewalld
systemctl start firewalld
systemctl enable firewalld

# Créer une zone dédiée pour MongoDB
firewall-cmd --permanent --new-zone=mongodb
firewall-cmd --reload

# Ajouter les sources autorisées (serveurs applicatifs)
firewall-cmd --permanent --zone=mongodb --add-source=10.0.1.20/32
firewall-cmd --permanent --zone=mongodb --add-source=10.0.1.21/32
firewall-cmd --permanent --zone=mongodb --add-source=10.0.1.22/32

# Ajouter les membres du Replica Set
firewall-cmd --permanent --zone=mongodb --add-source=10.0.2.10/32
firewall-cmd --permanent --zone=mongodb --add-source=10.0.2.11/32
firewall-cmd --permanent --zone=mongodb --add-source=10.0.2.12/32

# Autoriser le port MongoDB
firewall-cmd --permanent --zone=mongodb --add-port=27017/tcp

# Appliquer les règles
firewall-cmd --reload

# Vérifier la configuration
firewall-cmd --zone=mongodb --list-all
```

#### Configuration avancée (rich rules)

```bash
# Rich rule pour logging
firewall-cmd --permanent --zone=mongodb --add-rich-rule='
  rule family="ipv4"
  source address="10.0.1.0/24"
  port protocol="tcp" port="27017"
  log prefix="MongoDB-Access: " level="info"
  accept
'

# Rich rule avec rate limiting
firewall-cmd --permanent --zone=mongodb --add-rich-rule='
  rule family="ipv4"
  source address="10.0.1.0/24"
  port protocol="tcp" port="27017"
  limit value="100/m"
  accept
'

# Bloquer une IP spécifique
firewall-cmd --permanent --zone=mongodb --add-rich-rule='
  rule family="ipv4"
  source address="192.168.1.100"
  port protocol="tcp" port="27017"
  reject
'

# Recharger
firewall-cmd --reload
```

#### Service personnalisé

```bash
# Créer un service MongoDB personnalisé
cat > /etc/firewalld/services/mongodb.xml <<EOF
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>MongoDB</short>
  <description>MongoDB NoSQL Database Server</description>
  <port protocol="tcp" port="27017"/>
</service>
EOF

# Recharger les services
firewall-cmd --reload

# Utiliser le service
firewall-cmd --permanent --zone=mongodb --add-service=mongodb
firewall-cmd --reload
```

### UFW (Ubuntu/Debian)

```bash
# Activer UFW
ufw --force enable

# Politique par défaut
ufw default deny incoming
ufw default allow outgoing

# SSH (toujours autoriser en premier !)
ufw allow 22/tcp

# MongoDB : Autoriser depuis serveurs applicatifs
ufw allow from 10.0.1.20 to any port 27017 proto tcp
ufw allow from 10.0.1.21 to any port 27017 proto tcp
ufw allow from 10.0.1.22 to any port 27017 proto tcp

# MongoDB : Autoriser depuis Replica Set members
ufw allow from 10.0.2.10 to any port 27017 proto tcp
ufw allow from 10.0.2.11 to any port 27017 proto tcp
ufw allow from 10.0.2.12 to any port 27017 proto tcp

# Rate limiting (optionnel)
ufw limit 27017/tcp

# Vérifier le statut
ufw status numbered

# Logs
ufw logging on
tail -f /var/log/ufw.log
```

## IP Whitelisting

### Configuration au niveau application

#### Implémentation avec MongoDB driver (Node.js)

```javascript
// mongodb-ip-whitelist.js
const { MongoClient } = require('mongodb');
const net = require('net');

const ALLOWED_IPS = [
  '10.0.1.20',
  '10.0.1.21',
  '10.0.1.22',
  '192.168.1.0/24'  // Subnet
];

function isIPAllowed(ip) {
  // Vérification simple pour IPs
  if (ALLOWED_IPS.includes(ip)) {
    return true;
  }

  // Vérification pour subnets (simplifiée)
  for (const allowed of ALLOWED_IPS) {
    if (allowed.includes('/')) {
      // Parse subnet (utiliser un package comme 'ip' pour production)
      const [subnet, mask] = allowed.split('/');
      // Vérification simplifiée
      if (ip.startsWith(subnet.split('.').slice(0, -1).join('.'))) {
        return true;
      }
    }
  }

  return false;
}

// Wrapper de connexion avec validation IP
async function createSecureConnection(uri, clientIP) {
  if (!isIPAllowed(clientIP)) {
    throw new Error(`Access denied from IP: ${clientIP}`);
  }

  const client = new MongoClient(uri);
  await client.connect();
  return client;
}

// Usage
const clientIP = '10.0.1.20';  // À obtenir depuis la requête
const client = await createSecureConnection(
  'mongodb://mongodb.internal:27017',
  clientIP
);
```

#### Middleware Express avec whitelist

```javascript
// middleware/mongodb-ip-whitelist.js
const ipRangeCheck = require('ip-range-check');

const ALLOWED_IPS = [
  '10.0.1.20',
  '10.0.1.21',
  '192.168.1.0/24'
];

function mongoDBIPWhitelist(req, res, next) {
  const clientIP = req.ip ||
                   req.headers['x-forwarded-for'] ||
                   req.connection.remoteAddress;

  // Nettoyer l'IP (enlever ::ffff: si IPv6-mapped IPv4)
  const cleanIP = clientIP.replace(/^::ffff:/, '');

  if (!ipRangeCheck(cleanIP, ALLOWED_IPS)) {
    console.error(`Blocked MongoDB access from: ${cleanIP}`);
    return res.status(403).json({
      error: 'Access denied',
      message: 'Your IP is not authorized to access this resource'
    });
  }

  next();
}

// Usage dans Express
app.use('/api/data', mongoDBIPWhitelist, dataRoutes);
```

### Configuration avec Proxy (nginx)

```nginx
# /etc/nginx/conf.d/mongodb-proxy.conf
# Proxy TCP pour MongoDB avec IP whitelist

stream {
    # Log format
    log_format mongodb_access '$remote_addr [$time_local] '
                              '$protocol $status $bytes_sent $bytes_received '
                              '$session_time "$upstream_addr"';

    # Whitelist map
    geo $whitelist {
        default 0;
        10.0.1.20 1;
        10.0.1.21 1;
        10.0.1.22 1;
        10.0.2.0/24 1;
    }

    # MongoDB upstream
    upstream mongodb_backend {
        server 10.0.3.10:27017 max_fails=3 fail_timeout=30s;
        server 10.0.3.11:27017 max_fails=3 fail_timeout=30s backup;
    }

    # MongoDB proxy server
    server {
        listen 27017;
        proxy_pass mongodb_backend;
        proxy_connect_timeout 10s;
        proxy_timeout 300s;

        # Logging
        access_log /var/log/nginx/mongodb-access.log mongodb_access;
        error_log /var/log/nginx/mongodb-error.log;

        # IP whitelist check (nécessite nginx-plus ou module externe)
        # allow 10.0.1.20;
        # allow 10.0.1.21;
        # deny all;
    }
}
```

### Configuration avec HAProxy

```
# /etc/haproxy/haproxy.cfg
global
    log /dev/log local0
    maxconn 4096
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    timeout connect 10s
    timeout client 300s
    timeout server 300s

# ACL pour IP whitelist
frontend mongodb_front
    bind *:27017
    mode tcp

    # Définir les ACLs
    acl network_allowed src 10.0.1.20
    acl network_allowed src 10.0.1.21
    acl network_allowed src 10.0.1.22
    acl network_allowed src 10.0.2.0/24

    # Bloquer si pas dans la whitelist
    tcp-request connection reject unless network_allowed

    # Logger les connexions
    log-format "%ci:%cp [%t] %ft %b/%s %Tw/%Tc/%Tt %B %ts %ac/%fc/%bc/%sc/%rc %sq/%bq"

    default_backend mongodb_back

backend mongodb_back
    mode tcp
    balance roundrobin
    option tcp-check

    # Health check MongoDB
    tcp-check send "ping\r\n"
    tcp-check expect string "pong"

    server mongodb1 10.0.3.10:27017 check inter 5s fall 3 rise 2
    server mongodb2 10.0.3.11:27017 check inter 5s fall 3 rise 2
    server mongodb3 10.0.3.12:27017 check inter 5s fall 3 rise 2
```

## Network Segmentation

### Architecture recommandée

```
┌─────────────────────────────────────────────────────────────────┐
│  Internet                                                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  DMZ (10.0.0.0/24)                                              │
│  ├─ Load Balancer / API Gateway                                 │
│  ├─ Firewall : Autoriser 80/443 → App Tier                      │
│  └─ Firewall : Bloquer tout vers DB Tier                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Application Tier (10.0.1.0/24)                                 │
│  ├─ Application Servers                                         │
│  ├─ Firewall : Autoriser depuis DMZ                             │
│  ├─ Firewall : Autoriser 27017 → DB Tier                        │
│  └─ PAS d'accès direct depuis Internet                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Database Tier (10.0.2.0/24)                                    │
│  ├─ MongoDB Replica Set                                         │
│  ├─ Firewall : Autoriser UNIQUEMENT depuis App Tier             │
│  ├─ Firewall : Autoriser entre membres Replica Set              │
│  └─ AUCUN accès depuis Internet ou DMZ                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Management Network (10.0.10.0/24)                              │
│  ├─ Bastion Host (Jump Server)                                  │
│  ├─ Monitoring (Prometheus, Grafana)                            │
│  ├─ Backup Server                                               │
│  └─ Accès SSH uniquement via VPN                                │
└─────────────────────────────────────────────────────────────────┘
```

### VPC et Subnets (AWS exemple)

```hcl
# terraform/vpc.tf
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "mongodb-vpc"
    Environment = "production"
  }
}

# Subnet public (DMZ)
resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "public-subnet-${count.index + 1}"
    Tier = "dmz"
  }
}

# Subnet application (privé)
resource "aws_subnet" "application" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 10}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "app-subnet-${count.index + 1}"
    Tier = "application"
  }
}

# Subnet database (privé isolé)
resource "aws_subnet" "database" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 20}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "db-subnet-${count.index + 1}"
    Tier = "database"
  }
}

# Internet Gateway (pour subnet public)
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "mongodb-igw"
  }
}

# NAT Gateway (pour subnets privés)
resource "aws_eip" "nat" {
  count  = 3
  domain = "vpc"

  tags = {
    Name = "nat-eip-${count.index + 1}"
  }
}

resource "aws_nat_gateway" "main" {
  count         = 3
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name = "nat-gateway-${count.index + 1}"
  }
}
```

### Security Groups (AWS)

```hcl
# terraform/security-groups.tf

# Security Group pour Application
resource "aws_security_group" "application" {
  name        = "mongodb-app-sg"
  description = "Security group for application servers"
  vpc_id      = aws_vpc.main.id

  # Entrée depuis Load Balancer
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
    description     = "HTTP from ALB"
  }

  # SSH depuis Bastion uniquement
  ingress {
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion.id]
    description     = "SSH from bastion"
  }

  # Sortie vers MongoDB
  egress {
    from_port       = 27017
    to_port         = 27017
    protocol        = "tcp"
    security_groups = [aws_security_group.mongodb.id]
    description     = "MongoDB access"
  }

  # Sortie Internet (pour updates, APIs externes)
  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS outbound"
  }

  tags = {
    Name = "mongodb-app-sg"
  }
}

# Security Group pour MongoDB
resource "aws_security_group" "mongodb" {
  name        = "mongodb-db-sg"
  description = "Security group for MongoDB database servers"
  vpc_id      = aws_vpc.main.id

  # Entrée depuis Application Tier UNIQUEMENT
  ingress {
    from_port       = 27017
    to_port         = 27017
    protocol        = "tcp"
    security_groups = [aws_security_group.application.id]
    description     = "MongoDB from application"
  }

  # Entrée depuis autres membres du Replica Set (self-referencing)
  ingress {
    from_port = 27017
    to_port   = 27017
    protocol  = "tcp"
    self      = true
    description = "MongoDB replication"
  }

  # Entrée depuis Bastion pour administration
  ingress {
    from_port       = 27017
    to_port         = 27017
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion.id]
    description     = "MongoDB admin from bastion"
  }

  # SSH depuis Bastion
  ingress {
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion.id]
    description     = "SSH from bastion"
  }

  # Sortie limitée (updates système uniquement)
  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS for updates"
  }

  # Sortie vers KMS (pour KMIP)
  egress {
    from_port   = 5696
    to_port     = 5696
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
    description = "KMIP to KMS"
  }

  tags = {
    Name = "mongodb-db-sg"
  }
}

# Security Group pour Bastion
resource "aws_security_group" "bastion" {
  name        = "mongodb-bastion-sg"
  description = "Security group for bastion host"
  vpc_id      = aws_vpc.main.id

  # SSH depuis IPs autorisées (bureau, VPN)
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = var.admin_ips  # ["1.2.3.4/32", "5.6.7.8/32"]
    description = "SSH from authorized IPs"
  }

  # Sortie illimitée (pour atteindre autres SGs)
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "mongodb-bastion-sg"
  }
}
```

### Network ACLs (couche supplémentaire)

```hcl
# terraform/network-acls.tf

resource "aws_network_acl" "database" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.database[*].id

  # Autoriser MongoDB depuis Application Tier
  ingress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = "10.0.10.0/24"  # Application subnet
    from_port  = 27017
    to_port    = 27017
  }

  # Autoriser réponses
  ingress {
    protocol   = "tcp"
    rule_no    = 110
    action     = "allow"
    cidr_block = "10.0.10.0/24"
    from_port  = 1024
    to_port    = 65535
  }

  # Autoriser intra-subnet (réplication)
  ingress {
    protocol   = "tcp"
    rule_no    = 120
    action     = "allow"
    cidr_block = "10.0.20.0/24"  # Database subnet
    from_port  = 27017
    to_port    = 27017
  }

  # Bloquer tout le reste
  ingress {
    protocol   = -1
    rule_no    = 200
    action     = "deny"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }

  # Sortie : Autoriser vers Application Tier
  egress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = "10.0.10.0/24"
    from_port  = 1024
    to_port    = 65535
  }

  # Sortie : Autoriser intra-subnet
  egress {
    protocol   = "tcp"
    rule_no    = 110
    action     = "allow"
    cidr_block = "10.0.20.0/24"
    from_port  = 27017
    to_port    = 27017
  }

  # Sortie : HTTPS pour updates
  egress {
    protocol   = "tcp"
    rule_no    = 120
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 443
    to_port    = 443
  }

  tags = {
    Name = "mongodb-database-nacl"
  }
}
```

## VPN et Bastion Host

### Bastion Host (Jump Server)

```bash
#!/bin/bash
# setup-bastion.sh - Configuration du bastion host

# 1. Durcissement SSH
cat > /etc/ssh/sshd_config.d/99-hardening.conf <<EOF
# Désactiver l'authentification par mot de passe
PasswordAuthentication no
ChallengeResponseAuthentication no

# Autoriser uniquement les clés
PubkeyAuthentication yes

# Désactiver root login
PermitRootLogin no

# Limiter les users autorisés
AllowUsers admin devops

# Timeout de session
ClientAliveInterval 300
ClientAliveCountMax 2

# Désactiver forwarding X11
X11Forwarding no

# Logging strict
LogLevel VERBOSE

# Limiter les algorithmes faibles
Ciphers aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
KexAlgorithms curve25519-sha256,diffie-hellman-group16-sha512
EOF

systemctl restart sshd

# 2. Installer fail2ban
apt-get install -y fail2ban

cat > /etc/fail2ban/jail.local <<EOF
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3

[sshd]
enabled = true
port = 22
logpath = /var/log/auth.log
EOF

systemctl enable fail2ban
systemctl restart fail2ban

# 3. Audit logging
apt-get install -y auditd
systemctl enable auditd
systemctl start auditd

# 4. Logging de toutes les commandes
cat >> /etc/bash.bashrc <<EOF
# Log toutes les commandes
export PROMPT_COMMAND='RETRN_VAL=\$?;logger -p local6.debug "\$(whoami) [\$\$]: \$(history 1 | sed "s/^[ ]*[0-9]\+[ ]*//" )"'
EOF

# 5. Message d'avertissement
cat > /etc/motd <<EOF
================================================================================
                    AUTHORIZED ACCESS ONLY

This is a restricted bastion host. All activities are logged and monitored.
Unauthorized access attempts will be prosecuted to the fullest extent of law.

For MongoDB access, use:
  ssh -L 27017:mongodb-primary.internal:27017 bastion.company.com

Then connect to: mongodb://localhost:27017
================================================================================
EOF
```

### SSH Tunneling pour MongoDB

```bash
# Option 1 : Port forwarding simple
ssh -L 27017:mongodb-primary.internal:27017 user@bastion.company.com

# Puis dans un autre terminal
mongosh mongodb://localhost:27017

# Option 2 : Port forwarding avec configuration
# ~/.ssh/config
Host mongodb-tunnel
    HostName bastion.company.com
    User admin
    LocalForward 27017 mongodb-primary.internal:27017
    LocalForward 27018 mongodb-secondary1.internal:27017
    LocalForward 27019 mongodb-secondary2.internal:27017
    IdentityFile ~/.ssh/bastion_key
    ServerAliveInterval 60
    ServerAliveCountMax 3

# Utilisation
ssh mongodb-tunnel

# Option 3 : ProxyJump (SSH 7.3+)
ssh -J user@bastion.company.com admin@mongodb-primary.internal

# Option 4 : SOCKS proxy
ssh -D 8080 user@bastion.company.com

# Puis configurer le driver MongoDB pour utiliser le SOCKS proxy
mongosh mongodb://mongodb-primary.internal:27017 \
  --socksProxy localhost:8080
```

### VPN (WireGuard)

```bash
# /etc/wireguard/wg0.conf - Serveur VPN
[Interface]
Address = 10.0.100.1/24
ListenPort = 51820
PrivateKey = <server-private-key>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# Client 1 (Admin)
[Peer]
PublicKey = <client1-public-key>
AllowedIPs = 10.0.100.10/32

# Client 2 (DevOps)
[Peer]
PublicKey = <client2-public-key>
AllowedIPs = 10.0.100.11/32

# Démarrer le VPN
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0

# Firewall : Autoriser MongoDB depuis VPN
ufw allow from 10.0.100.0/24 to any port 27017
```

## MongoDB Atlas Network Security

### IP Access List (Whitelist)

```bash
# Via Atlas CLI
atlas accessLists create \
  --projectId 5e2211c17a3e5a48f5497de3 \
  --cidrBlock 10.0.1.0/24 \
  --comment "Application servers"

atlas accessLists create \
  --projectId 5e2211c17a3e5a48f5497de3 \
  --ipAddress 1.2.3.4 \
  --comment "Office IP"

# Lister les règles
atlas accessLists list --projectId 5e2211c17a3e5a48f5497de3

# Supprimer une règle
atlas accessLists delete \
  --projectId 5e2211c17a3e5a48f5497de3 \
  --entry 1.2.3.4
```

### VPC Peering (AWS)

```hcl
# terraform/atlas-vpc-peering.tf

# Récupérer le VPC Atlas
data "mongodbatlas_network_container" "atlas" {
  project_id = var.atlas_project_id

  provider_name = "AWS"
  region_name   = "US_EAST_1"
}

# Créer le peering
resource "mongodbatlas_network_peering" "aws" {
  accepter_region_name   = "us-east-1"
  project_id             = var.atlas_project_id
  container_id           = data.mongodbatlas_network_container.atlas.id
  provider_name          = "AWS"
  route_table_cidr_block = aws_vpc.main.cidr_block
  vpc_id                 = aws_vpc.main.id
  aws_account_id         = data.aws_caller_identity.current.account_id
}

# Accepter le peering côté AWS
resource "aws_vpc_peering_connection_accepter" "atlas" {
  vpc_peering_connection_id = mongodbatlas_network_peering.aws.connection_id
  auto_accept               = true

  tags = {
    Name = "mongodb-atlas-peering"
  }
}

# Ajouter les routes
resource "aws_route" "to_atlas" {
  count                     = length(aws_route_table.private)
  route_table_id            = aws_route_table.private[count.index].id
  destination_cidr_block    = data.mongodbatlas_network_container.atlas.atlas_cidr_block
  vpc_peering_connection_id = mongodbatlas_network_peering.aws.connection_id
}
```

### Private Endpoint (AWS PrivateLink)

```hcl
# terraform/atlas-private-endpoint.tf

resource "mongodbatlas_privatelink_endpoint" "atlas" {
  project_id    = var.atlas_project_id
  provider_name = "AWS"
  region        = "us-east-1"
}

resource "aws_vpc_endpoint" "atlas" {
  vpc_id             = aws_vpc.main.id
  service_name       = mongodbatlas_privatelink_endpoint.atlas.endpoint_service_name
  vpc_endpoint_type  = "Interface"

  subnet_ids         = aws_subnet.database[*].id
  security_group_ids = [aws_security_group.atlas_endpoint.id]

  private_dns_enabled = true

  tags = {
    Name = "mongodb-atlas-endpoint"
  }
}

resource "mongodbatlas_privatelink_endpoint_service" "atlas" {
  project_id          = var.atlas_project_id
  provider_name       = "AWS"
  private_link_id     = mongodbatlas_privatelink_endpoint.atlas.private_link_id
  endpoint_service_id = aws_vpc_endpoint.atlas.id
}

# Security Group pour endpoint
resource "aws_security_group" "atlas_endpoint" {
  name        = "mongodb-atlas-endpoint-sg"
  description = "Security group for MongoDB Atlas PrivateLink endpoint"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 27017
    to_port         = 27017
    protocol        = "tcp"
    security_groups = [aws_security_group.application.id]
    description     = "MongoDB from application"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "mongodb-atlas-endpoint-sg"
  }
}
```

## Monitoring et Alerting

### Monitoring des connexions réseau

```bash
#!/bin/bash
# /usr/local/bin/monitor-mongodb-connections.sh

# Surveiller les connexions actives
watch -n 5 'netstat -an | grep :27017 | grep ESTABLISHED | wc -l'

# Détail des connexions
netstat -antp | grep :27017 | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn

# Avec ss (plus rapide)
ss -tn state established '( dport = :27017 or sport = :27017 )' | \
  awk '{print $4}' | cut -d: -f1 | sort | uniq -c | sort -rn

# Script de monitoring complet
cat > /usr/local/bin/mongodb-network-monitor.sh <<'EOF'
#!/bin/bash

THRESHOLD=1000
ALERT_EMAIL="ops@company.com"

# Compter les connexions
CONN_COUNT=$(ss -tn state established '( dport = :27017 )' | wc -l)

echo "MongoDB connections: $CONN_COUNT"

if [ $CONN_COUNT -gt $THRESHOLD ]; then
  echo "ALERT: MongoDB connections ($CONN_COUNT) exceed threshold ($THRESHOLD)" | \
    mail -s "MongoDB Connection Alert" $ALERT_EMAIL
fi

# Top IPs
echo "Top connecting IPs:"
ss -tn state established '( dport = :27017 )' | \
  awk '{print $4}' | cut -d: -f1 | sort | uniq -c | sort -rn | head -10
EOF

chmod +x /usr/local/bin/mongodb-network-monitor.sh

# Cron toutes les 5 minutes
*/5 * * * * /usr/local/bin/mongodb-network-monitor.sh
```

### Détection d'accès non autorisés

```python
#!/usr/bin/env python3
# /usr/local/bin/detect-unauthorized-access.py

import subprocess
import re
from collections import Counter

ALLOWED_IPS = [
    '10.0.1.20',
    '10.0.1.21',
    '10.0.1.22',
    '10.0.2.10',
    '10.0.2.11',
    '10.0.2.12'
]

def get_active_connections():
    """Récupère les connexions actives MongoDB"""
    result = subprocess.run(
        ['ss', '-tn', 'state', 'established', '( dport = :27017 )'],
        capture_output=True,
        text=True
    )

    connections = []
    for line in result.stdout.split('\n'):
        match = re.search(r'(\d+\.\d+\.\d+\.\d+):\d+', line)
        if match:
            connections.append(match.group(1))

    return connections

def check_unauthorized():
    """Vérifie les connexions non autorisées"""
    connections = get_active_connections()
    unauthorized = [ip for ip in connections if ip not in ALLOWED_IPS]

    if unauthorized:
        ip_counts = Counter(unauthorized)
        print("⚠️  UNAUTHORIZED CONNECTIONS DETECTED:")
        for ip, count in ip_counts.items():
            print(f"  - {ip}: {count} connections")

            # Bloquer l'IP
            subprocess.run(['ufw', 'deny', 'from', ip, 'to', 'any', 'port', '27017'])
            print(f"  ✓ Blocked {ip}")

        return True
    else:
        print("✓ All connections are authorized")
        return False

if __name__ == '__main__':
    import sys
    sys.exit(1 if check_unauthorized() else 0)
```

### Intégration avec Prometheus

```yaml
# /etc/prometheus/prometheus.yml
scrape_configs:
  - job_name: 'mongodb-network'
    static_configs:
      - targets: ['mongodb-exporter:9216']

    metric_relabel_configs:
      # Extraire l'IP source des connexions
      - source_labels: [__address__]
        target_label: instance
        replacement: '${1}'

# Alerting rules
# /etc/prometheus/rules/mongodb-network.yml
groups:
  - name: mongodb_network
    interval: 30s
    rules:
      - alert: UnauthorizedMongoDBAccess
        expr: |
          mongodb_connections{source_ip!~"10.0.1.*|10.0.2.*"} > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Unauthorized MongoDB access detected"
          description: "Connection from {{ $labels.source_ip }} to MongoDB"

      - alert: MongoDBConnectionSpike
        expr: |
          rate(mongodb_connections_total[5m]) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MongoDB connection spike detected"
          description: "Connection rate: {{ $value }} connections/sec"

      - alert: MongoDBPortScan
        expr: |
          increase(mongodb_connection_attempts_failed[1m]) > 50
        labels:
          severity: warning
        annotations:
          summary: "Possible port scan on MongoDB"
          description: "{{ $value }} failed connection attempts in 1 minute"
```

## Troubleshooting

### Connexion refusée

```bash
# 1. Vérifier que MongoDB écoute
netstat -tuln | grep 27017
ss -tuln | grep 27017

# 2. Vérifier bindIp dans la configuration
grep bindIp /etc/mongod.conf

# 3. Tester la connexion locale
mongosh mongodb://localhost:27017

# 4. Tester depuis le réseau
telnet <mongodb-ip> 27017

# 5. Vérifier le firewall
# iptables
iptables -L -n -v | grep 27017

# firewalld
firewall-cmd --list-all

# ufw
ufw status numbered

# 6. Vérifier les logs MongoDB
tail -f /var/log/mongodb/mongod.log | grep -i "connection\|bind"

# 7. Vérifier SELinux (RHEL/CentOS)
getenforce
ausearch -m avc -ts recent | grep mongod
```

### Connexion lente

```bash
# 1. Vérifier la latence réseau
ping <mongodb-ip>
mtr <mongodb-ip>

# 2. Vérifier les connexions actives
mongosh --eval "db.serverStatus().connections"

# 3. Identifier les slow queries réseau
tcpdump -i eth0 -nn -s0 -c 1000 port 27017 -w mongodb.pcap
wireshark mongodb.pcap

# 4. Vérifier la MTU
ip link show
ping -M do -s 1472 <mongodb-ip>

# 5. Vérifier les security groups (cloud)
# AWS
aws ec2 describe-security-groups --group-ids sg-xxxxx

# 6. Tracer la route
traceroute <mongodb-ip>
```

### IP bloquée par erreur

```bash
# iptables : Débloquer une IP
iptables -D INPUT -s <ip-address> -j DROP
iptables-save > /etc/iptables/rules.v4

# firewalld : Débloquer une IP
firewall-cmd --zone=mongodb --remove-source=<ip-address>
firewall-cmd --runtime-to-permanent

# ufw : Débloquer une IP
ufw status numbered
ufw delete <rule-number>

# fail2ban : Débloquer une IP
fail2ban-client set sshd unbanip <ip-address>
```

## Bonnes pratiques de production

### Checklist de sécurité réseau

```
☐ Configuration MongoDB
  ☐ bindIp configuré (jamais 0.0.0.0 en prod)
  ☐ Port par défaut changé (optionnel mais recommandé)
  ☐ TLS activé (section 11.5.1)
  ☐ Authentification activée

☐ Firewall
  ☐ Firewall activé sur tous les serveurs
  ☐ Règles restrictives (deny all par défaut)
  ☐ Whitelist explicite des IPs autorisées
  ☐ Rate limiting configuré

☐ Network Segmentation
  ☐ VPC/VNET configuré
  ☐ Subnets séparés (DMZ, App, DB, Management)
  ☐ Security Groups/NSGs configurés
  ☐ Network ACLs en place (optionnel)

☐ Accès Administration
  ☐ Bastion host configuré
  ☐ VPN pour accès distant
  ☐ SSH keys uniquement (pas de passwords)
  ☐ fail2ban ou équivalent activé

☐ Cloud (si applicable)
  ☐ VPC Peering ou PrivateLink configuré
  ☐ IP Access List configurée (Atlas)
  ☐ Private endpoints utilisés
  ☐ Pas d'exposition publique

☐ Monitoring
  ☐ Alertes sur connexions non autorisées
  ☐ Monitoring des connexions actives
  ☐ Logs réseau centralisés
  ☐ Tests réguliers de pénétration

☐ Documentation
  ☐ Liste des IPs autorisées maintenue
  ☐ Architecture réseau documentée
  ☐ Procédure d'ajout/suppression d'IP
  ☐ Runbook incident réseau
```

### Tests de validation

```bash
#!/bin/bash
# /usr/local/bin/validate-network-security.sh

echo "=== MongoDB Network Security Validation ==="

# Test 1 : Vérifier bindIp
echo -n "1. Checking bindIp configuration... "
BIND_IP=$(grep "bindIp:" /etc/mongod.conf | awk '{print $2}')
if [ "$BIND_IP" == "0.0.0.0" ]; then
  echo "❌ FAIL: bindIp is 0.0.0.0 (insecure)"
  exit 1
else
  echo "✓ PASS: bindIp is $BIND_IP"
fi

# Test 2 : Vérifier le firewall
echo -n "2. Checking firewall status... "
if systemctl is-active --quiet firewalld || systemctl is-active --quiet ufw; then
  echo "✓ PASS: Firewall is active"
else
  echo "❌ FAIL: No firewall detected"
  exit 1
fi

# Test 3 : Vérifier l'exposition publique
echo -n "3. Checking public exposure... "
PUBLIC_IP=$(curl -s ifconfig.me)
if timeout 5 bash -c "echo > /dev/tcp/$PUBLIC_IP/27017" 2>/dev/null; then
  echo "❌ FAIL: MongoDB is accessible from public IP"
  exit 1
else
  echo "✓ PASS: MongoDB not accessible from public IP"
fi

# Test 4 : Vérifier TLS
echo -n "4. Checking TLS configuration... "
if grep -q "mode: requireTLS" /etc/mongod.conf; then
  echo "✓ PASS: TLS is required"
else
  echo "⚠️  WARNING: TLS is not enforced"
fi

# Test 5 : Vérifier les règles firewall
echo "5. Checking firewall rules for port 27017..."
if command -v iptables &> /dev/null; then
  iptables -L INPUT -n | grep 27017
fi

echo ""
echo "=== Validation Complete ==="
```

## Conclusion

La sécurité réseau est la première ligne de défense de votre infrastructure MongoDB. Sans elle, toutes les autres mesures de sécurité (TLS, authentification, chiffrement) sont contournables.

**Principes clés** :

1. **Deny by default** : Bloquer tout, autoriser explicitement
2. **Network segmentation** : Isolation des tiers (DMZ, App, DB)
3. **IP Whitelisting** : Autoriser uniquement les sources connues
4. **Bastion/VPN** : Accès administratif sécurisé
5. **Monitoring** : Détection d'anomalies et alerting

**Règle d'or** : **MongoDB ne doit JAMAIS être exposé directement à Internet.**

Une configuration réseau bien pensée réduit considérablement la surface d'attaque et facilite la conformité réglementaire (PCI-DSS, HIPAA, SOC2).

---

**Ressources complémentaires** :
- MongoDB Security Checklist
- CIS Benchmark for MongoDB
- NIST Cybersecurity Framework
- OWASP Top 10 for APIs

⏭️ [Security Checklist](/11-securite/08-security-checklist.md)
