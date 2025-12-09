🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe D : Configuration de Référence par Cas d'Usage

## Vue d'ensemble

Cette annexe fournit des configurations MongoDB prêtes à l'emploi pour différents scénarios de déploiement. Chaque configuration est documentée avec :

- **Fichiers de configuration** : Paramètres mongod.conf/mongos.conf complets
- **Commandes de démarrage** : Scripts shell pour lancer les instances
- **Topologie réseau** : Schémas d'architecture
- **Prérequis système** : CPU, RAM, stockage recommandés
- **Commandes de vérification** : Tests post-déploiement

---

## Structure de l'annexe

### D.1 - Configuration Replica Set (3 nœuds)
Configuration standard de haute disponibilité avec 3 membres (1 Primary, 2 Secondary). Idéale pour la production avec besoin de failover automatique.

**Cas d'usage** : Applications critiques, environnements de production, besoin de lecture scalable

### D.2 - Configuration Sharded Cluster
Architecture complète de sharding avec config servers, mongos et shards. Pour les très grandes bases de données nécessitant une distribution horizontale.

**Cas d'usage** : Big Data, croissance massive, distribution géographique des données

### D.3 - Configuration Développement Local
Setup minimal pour le développement sur poste local. Instance standalone avec paramètres optimisés pour le debug.

**Cas d'usage** : Développement, tests unitaires, prototypage rapide

### D.4 - Configuration Haute Performance
Paramètres optimisés pour maximiser les performances en lecture/écriture. Tuning WiredTiger, cache, journaling.

**Cas d'usage** : Applications temps réel, IoT, analytics, forte charge transactionnelle

---

## Principe d'utilisation

Chaque configuration suit le même format :

```
1. PRÉSENTATION
   - Objectif et cas d'usage
   - Topologie et architecture
   - Prérequis matériels

2. FICHIERS DE CONFIGURATION
   - mongod.conf détaillé avec commentaires
   - mongos.conf si applicable
   - Paramètres kernel Linux recommandés

3. DÉPLOIEMENT
   - Scripts de démarrage
   - Commandes d'initialisation
   - Configuration réseau/sécurité

4. VÉRIFICATION
   - Commandes de test
   - Checks de santé
   - Métriques à surveiller

5. MAINTENANCE
   - Procédures de sauvegarde
   - Rolling upgrades
   - Troubleshooting commun
```

---

## Recommandations générales

### Sécurité de base (tous environnements)

```yaml
# À adapter dans chaque mongod.conf
security:
  authorization: enabled        # Activer l'authentification

net:
  bindIp: 127.0.0.1,<IP_PRIVÉE>  # Limiter les interfaces

systemLog:
  verbosity: 0                   # Log normal (0-5)
```

### Paramètres système Linux

```bash
# /etc/sysctl.conf - Pour tous les environnements de production
vm.swappiness = 1
net.ipv4.tcp_keepalive_time = 300
fs.file-max = 98000

# Limites ulimit - /etc/security/limits.conf
mongod soft nofile 64000
mongod hard nofile 64000
mongod soft nproc 32000
mongod hard nproc 32000
```

### Stockage recommandé

| Type d'usage | Système de fichiers | Options de montage |
|--------------|---------------------|-------------------|
| Production | XFS | `noatime,nobarrier` |
| Développement | ext4 | `defaults` |
| Haute performance | XFS | `noatime,nobarrier,allocsize=16M` |

---

## Dimensionnement mémoire

### Règle générale WiredTiger

```
Cache WiredTiger = max(50% RAM - 1 GB, 256 MB)
```

**Exemples** :
- Serveur 16 GB RAM → Cache 7 GB
- Serveur 32 GB RAM → Cache 15 GB
- Serveur 64 GB RAM → Cache 31 GB

### Répartition mémoire type

Sur un serveur avec 32 GB RAM :

| Composant | Allocation | Description |
|-----------|-----------|-------------|
| WiredTiger Cache | 15 GB | Cache des données |
| Système d'exploitation | 4 GB | OS + buffers |
| Index MongoDB | 8 GB | Index chargés en RAM |
| Connexions | 2 GB | Pool de connexions |
| Overhead | 3 GB | Marge de sécurité |

---

## Checklist pré-déploiement

Avant d'appliquer une configuration :

- [ ] Vérifier la compatibilité de version MongoDB
- [ ] Valider les prérequis matériels
- [ ] Configurer les règles firewall
- [ ] Préparer les points de montage (volumes)
- [ ] Créer les utilisateurs système
- [ ] Configurer la résolution DNS
- [ ] Préparer les certificats SSL/TLS (production)
- [ ] Définir la stratégie de sauvegarde
- [ ] Planifier les fenêtres de maintenance
- [ ] Documenter l'architecture dans un wiki

---

## Évolution des configurations

### Migration progressive

```
Développement local → Replica Set (test) → Replica Set (prod) → Sharded Cluster
```

### Quand passer au niveau supérieur ?

| Indicateur | Standalone → Replica Set | Replica Set → Sharding |
|------------|-------------------------|------------------------|
| **Disponibilité** | Besoin HA > 99% | Multi-région requis |
| **Données** | > 100 GB | > 500 GB à 1 TB |
| **Charge lecture** | > 1000 ops/sec | > 10000 ops/sec |
| **Charge écriture** | > 500 ops/sec | > 5000 ops/sec |
| **Latence** | Dégradation visible | P95 > 100ms |

---

## Outils de déploiement complémentaires

Ces configurations peuvent être déployées avec :

### Docker / Docker Compose
```bash
# Voir Annexe F pour les configurations Docker Compose
docker-compose -f annexes/docker-compose/02-replica-set.yml up -d
```

### Kubernetes
```bash
# Via MongoDB Community Operator
kubectl apply -f mongodb-replica-set.yaml
```

### Ansible
```bash
# Voir Annexe G pour les playbooks
ansible-playbook -i inventory deploy-mongodb.yml
```

### Terraform (MongoDB Atlas)
```bash
# Pour les déploiements cloud
terraform apply -var-file="production.tfvars"
```

---

## Conventions de nommage

Dans toutes les configurations de cette annexe :

| Élément | Convention | Exemple |
|---------|-----------|---------|
| **Replica Set** | `rs-<env>-<app>` | `rs-prod-ecommerce` |
| **Sharded Cluster** | `sh-<env>-<app>` | `sh-prod-analytics` |
| **Base de données** | `<app>_<module>` | `ecommerce_catalog` |
| **Utilisateur** | `<app>_<role>` | `ecommerce_readwrite` |
| **Ports** | 27017 (primary), 27018+ | 27017, 27018, 27019 |
| **Hostnames** | `mongo-<type>-<num>` | `mongo-primary-01` |

---

## Support et ressources

### Documentation officielle
- [MongoDB Production Notes](https://docs.mongodb.com/manual/administration/production-notes/)
- [MongoDB Configuration File Options](https://docs.mongodb.com/manual/reference/configuration-options/)
- [MongoDB Hardware Requirements](https://docs.mongodb.com/manual/administration/production-checklist-operations/)

### Outils de validation
```bash
# Vérifier une configuration avant application
mongod --config mongod.conf --configExpand rest --outputConfig

# Tester la connectivité
mongosh "mongodb://host:27017/?replicaSet=rs0" --eval "db.runCommand({ping: 1})"
```

### Monitoring post-déploiement
```javascript
// Vérifier la santé du serveur
db.serverStatus()

// Vérifier la configuration active
db.adminCommand({getCmdLineOpts: 1})

// Statistiques de connexions
db.serverStatus().connections
```

---

## Notes importantes

⚠️ **Sécurité** : Les configurations fournies incluent des exemples de mots de passe et clés. **Ne jamais utiliser ces valeurs en production**. Générer des secrets forts et uniques.

⚠️ **Versions** : Ces configurations sont testées avec MongoDB 6.x, 7.x et 8.x. Vérifier la compatibilité des options selon votre version.

⚠️ **Sauvegarde** : Toujours effectuer une sauvegarde complète avant de modifier une configuration de production.

⚠️ **Tests** : Valider toute nouvelle configuration en environnement de test avant la production.

---

## Ordre de lecture recommandé

1. **Débutants** : Commencer par D.3 (Configuration développement local)
2. **Intermédiaires** : Passer à D.1 (Replica Set) pour comprendre la HA
3. **Avancés** : Explorer D.2 (Sharding) pour la scalabilité horizontale
4. **Experts** : Optimiser avec D.4 (Haute performance)

---


⏭️ [Configuration Replica Set (3 nœuds)](/annexes/configuration-reference/01-configuration-replica-set.md)
