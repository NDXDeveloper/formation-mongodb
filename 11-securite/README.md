🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 11 : Sécurité MongoDB

## Introduction

La sécurité est un pilier fondamental de toute infrastructure de base de données en production. MongoDB offre un ensemble complet de fonctionnalités de sécurité pour protéger vos données contre les accès non autorisés, les violations de confidentialité et les compromissions système. Ce chapitre explore en profondeur les mécanismes de sécurité disponibles dans MongoDB, avec un focus sur les configurations de production et les meilleures pratiques pour les environnements critiques.

## Importance de la Sécurité dans MongoDB

### Contexte de Sécurité

Dans l'écosystème moderne des bases de données distribuées, MongoDB peut être exposé à plusieurs vecteurs de menaces :

- **Accès non autorisés** : Tentatives d'accès par des acteurs malveillants
- **Élévation de privilèges** : Exploitation de permissions mal configurées
- **Interception de données** : Capture de données en transit sur le réseau
- **Compromission des données au repos** : Accès physique aux fichiers de données
- **Attaques par injection** : Exploitation de requêtes mal construites
- **Déni de service** : Surcharge intentionnelle des ressources
- **Menaces internes** : Accès abusif par des utilisateurs légitimes

### Impacts Potentiels

Une faille de sécurité peut avoir des conséquences graves :

- **Perte de données sensibles** : Exposition d'informations confidentielles
- **Violations réglementaires** : Non-conformité RGPD, HIPAA, PCI-DSS, etc.
- **Atteinte à la réputation** : Perte de confiance des clients et partenaires
- **Coûts financiers** : Amendes, remédiation, litiges
- **Interruption de service** : Impact sur la continuité d'activité

## Architecture de Sécurité MongoDB

MongoDB adopte une approche **defense in depth** (défense en profondeur) avec plusieurs couches de sécurité :

```
┌─────────────────────────────────────────────────────────┐
│                   RÉSEAU & FIREWALL                     │
│            (IP Whitelisting, Network Isolation)         │
├─────────────────────────────────────────────────────────┤
│                  CHIFFREMENT EN TRANSIT                 │
│                     (TLS/SSL 1.2+)                      │
├─────────────────────────────────────────────────────────┤
│                    AUTHENTIFICATION                     │
│         (SCRAM, x.509, LDAP, Kerberos, OIDC)            │
├─────────────────────────────────────────────────────────┤
│                     AUTORISATION                        │
│            (RBAC - Role-Based Access Control)           │
├─────────────────────────────────────────────────────────┤
│                  CHIFFREMENT AU REPOS                   │
│              (WiredTiger Encryption at Rest)            │
├─────────────────────────────────────────────────────────┤
│              CHIFFREMENT AU NIVEAU CHAMP                │
│           (CSFLE, Queryable Encryption)                 │
├─────────────────────────────────────────────────────────┤
│                        AUDIT                            │
│              (Audit Logs & Monitoring)                  │
└─────────────────────────────────────────────────────────┘
```

## Principes de Sécurité MongoDB

### 1. Principe du Moindre Privilège

Chaque utilisateur, application ou service ne devrait avoir que les permissions strictement nécessaires à son fonctionnement. Ce principe s'applique à tous les niveaux :

- **Accès base de données** : Limiter aux bases nécessaires
- **Accès collections** : Restreindre aux collections requises
- **Opérations** : N'autoriser que les actions indispensables
- **Accès réseau** : Limiter les sources IP autorisées

### 2. Authentification Obligatoire

**Par défaut, MongoDB n'active pas l'authentification**. Cette configuration est acceptable uniquement pour le développement local. En production, l'authentification doit être activée systématiquement.

```javascript
// Configuration minimale de sécurité
security:
  authorization: enabled
```

### 3. Chiffrement Multi-Niveaux

- **En transit** : TLS 1.2 minimum pour toutes les communications
- **Au repos** : Chiffrement des fichiers de données et journaux
- **Au niveau application** : CSFLE pour les données ultra-sensibles

### 4. Audit et Traçabilité

Maintenir une piste d'audit complète pour :
- Détecter les comportements anormaux
- Répondre aux exigences de conformité
- Faciliter les investigations post-incident
- Démontrer la conformité lors d'audits

### 5. Isolation Réseau

MongoDB ne devrait jamais être directement exposé sur Internet :
- Utiliser des réseaux privés virtuels (VPC)
- Implémenter des pare-feu au niveau réseau
- Utiliser des proxies inverses ou bastion hosts
- Activer l'IP Whitelisting

## Structure du Chapitre

Ce chapitre couvre les aspects suivants de la sécurité MongoDB :

### 11.1 Vue d'Ensemble de la Sécurité MongoDB
Introduction aux concepts de sécurité, modèle de menaces et architecture de sécurité globale.

### 11.2 Authentification
Mécanismes d'authentification disponibles :
- **SCRAM** : Mécanisme par défaut basé sur mots de passe
- **x.509** : Authentification par certificats
- **LDAP** : Intégration avec Active Directory
- **Kerberos** : Pour les environnements enterprise

### 11.3 Autorisation et Rôles
Système de contrôle d'accès basé sur les rôles (RBAC) :
- Rôles intégrés et leur utilisation appropriée
- Création de rôles personnalisés
- Gestion granulaire des privilèges

### 11.4 Gestion des Utilisateurs
Opérations de gestion des utilisateurs :
- Création et suppression d'utilisateurs
- Modification des rôles et permissions
- Rotation des credentials
- Gestion des sessions

### 11.5 Chiffrement
Protection des données à différents niveaux :
- **TLS/SSL** : Chiffrement des communications réseau
- **Encryption at Rest** : Protection des données stockées
- **CSFLE** : Chiffrement au niveau champ côté client
- **Queryable Encryption** : Recherche sur données chiffrées

### 11.6 Audit
Configuration et exploitation des journaux d'audit :
- Types d'événements audités
- Configuration des filtres d'audit
- Intégration avec SIEM
- Analyse des logs d'audit

### 11.7 Network Security et IP Whitelisting
Sécurisation au niveau réseau :
- Configuration des interfaces réseau
- IP Whitelisting et blacklisting
- Intégration VPC et sous-réseaux
- Configuration des pare-feu

### 11.8 Security Checklist
Liste de contrôle complète pour sécuriser MongoDB en production.

### 11.9 Conformité et Certifications
Respect des normes et réglementations :
- RGPD (GDPR)
- HIPAA
- PCI-DSS
- SOC 2
- ISO 27001

## Recommandations Générales de Production

### Configuration Minimale de Sécurité

Toute instance MongoDB en production doit impérativement avoir :

```yaml
# mongod.conf - Configuration minimale de sécurité
security:
  authorization: enabled

net:
  bindIp: 127.0.0.1,<adresse_ip_privée>
  port: 27017
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb.pem
    CAFile: /etc/ssl/ca.pem

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true

auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.log
```

### Séparation des Environnements

Maintenir une séparation stricte entre :

- **Développement** : Accès large, données de test
- **Recette/Staging** : Configuration proche de la production
- **Production** : Sécurité maximale, accès restreint

Chaque environnement doit avoir :
- Ses propres credentials
- Ses propres certificats
- Son propre réseau isolé
- Sa propre configuration de monitoring

### Gestion des Secrets

Ne jamais stocker de secrets en clair :

- Utiliser des gestionnaires de secrets (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault)
- Chiffrer les fichiers de configuration sensibles
- Implémenter la rotation automatique des credentials
- Utiliser des variables d'environnement pour l'injection de secrets

### Monitoring de Sécurité

Surveiller en continu :

- Tentatives de connexion échouées
- Élévations de privilèges
- Opérations administratives
- Accès à des données sensibles
- Modifications de configuration de sécurité
- Anomalies de trafic réseau

### Mises à Jour de Sécurité

- Abonner à la MongoDB Security Mailing List
- Appliquer les correctifs de sécurité rapidement
- Tester les mises à jour en pré-production
- Maintenir un processus de patch management

## MongoDB Atlas : Sécurité Managée

MongoDB Atlas simplifie considérablement la gestion de la sécurité en fournissant :

- **Chiffrement automatique** : En transit et au repos par défaut
- **Authentification intégrée** : SCRAM activé automatiquement
- **IP Whitelisting** : Interface de gestion simplifiée
- **Audit logs** : Disponibles pour tous les tiers
- **Conformité** : Certifications maintenues par MongoDB
- **Private endpoints** : VPC peering, AWS PrivateLink, Azure Private Link
- **Gestion des clés** : Intégration avec les KMS cloud

Cependant, même avec Atlas, la responsabilité partagée s'applique :

| Responsabilité | MongoDB Atlas | Client |
|----------------|---------------|--------|
| Infrastructure physique | ✅ | ❌ |
| Système d'exploitation | ✅ | ❌ |
| MongoDB software | ✅ | ❌ |
| Chiffrement au repos | ✅ | ❌ |
| Chiffrement en transit | ✅ | ✅ Vérifier la configuration |
| Authentification | ✅ Mécanisme | ✅ Gestion des utilisateurs |
| Autorisation | ✅ Fonctionnalité | ✅ Configuration des rôles |
| Gestion des credentials | ❌ | ✅ |
| Contrôle d'accès applicatif | ❌ | ✅ |
| Sécurité du code | ❌ | ✅ |

## Matrice de Sécurité par Environnement

| Fonctionnalité | Développement | Staging | Production |
|----------------|---------------|---------|------------|
| Authentification | Optionnelle | **Obligatoire** | **Obligatoire** |
| TLS/SSL | Optionnel | **Obligatoire** | **Obligatoire** |
| Chiffrement au repos | Non | Recommandé | **Obligatoire** |
| IP Whitelisting | Non | Oui | **Obligatoire** |
| Audit logs | Non | Oui | **Obligatoire** |
| Isolation réseau | Non | Oui | **Obligatoire** |
| Rotation des credentials | Non | Trimestrielle | Mensuelle |
| Monitoring sécurité | Basique | Avancé | **Temps réel** |
| Tests d'intrusion | Non | Annuel | **Trimestriel** |
| Revue des permissions | Non | Trimestrielle | **Mensuelle** |

## Standards de Conformité

### RGPD (Règlement Général sur la Protection des Données)

MongoDB doit être configuré pour respecter :

- **Droit à l'oubli** : Capacité de supprimer les données personnelles
- **Portabilité des données** : Export des données dans un format standard
- **Chiffrement** : Protection des données personnelles
- **Audit** : Traçabilité des accès aux données personnelles
- **Pseudonymisation** : Séparation des données identifiantes

### HIPAA (Health Insurance Portability and Accountability Act)

Pour les données de santé :

- Chiffrement obligatoire en transit et au repos
- Audit détaillé de tous les accès
- Contrôles d'accès stricts
- Gestion des violations de données
- Formation du personnel

### PCI-DSS (Payment Card Industry Data Security Standard)

Pour les données de paiement :

- Segmentation du réseau
- Chiffrement des données de cartes
- Contrôles d'accès stricts
- Monitoring et logging
- Tests de sécurité réguliers

## Outils et Ressources

### Outils de Sécurité MongoDB

- **MongoDB Security Checklist** : Guide officiel de sécurisation
- **MongoDB Compass** : Audit visuel des permissions
- **MongoDB Atlas Security** : Dashboard de sécurité centralisé
- **mongo-express** : Interface web avec authentification

### Outils Tiers

- **Nessus** : Scanner de vulnérabilités
- **Qualys** : Évaluation de sécurité
- **OWASP ZAP** : Tests de sécurité applicative
- **Splunk/ELK** : Analyse des logs d'audit
- **HashiCorp Vault** : Gestion des secrets

### Documentation Officielle

- MongoDB Security Manual : https://docs.mongodb.com/manual/security/
- MongoDB Security Architecture : https://docs.mongodb.com/manual/core/security-architecture/
- Security Checklist : https://docs.mongodb.com/manual/administration/security-checklist/

## Conclusion

La sécurité MongoDB n'est pas une fonctionnalité à activer, mais un **processus continu** qui nécessite :

1. **Planification** : Définir les exigences de sécurité
2. **Implémentation** : Configurer les mécanismes de sécurité
3. **Monitoring** : Surveiller les menaces et anomalies
4. **Audit** : Vérifier régulièrement la conformité
5. **Amélioration** : Adapter les mesures aux nouvelles menaces

Les sections suivantes détaillent chaque aspect de la sécurité MongoDB avec des configurations concrètes, des exemples de production et des recommandations basées sur les meilleures pratiques de l'industrie.

---

**Points Clés à Retenir** :
- ✅ La sécurité par défaut de MongoDB nécessite une configuration active
- ✅ Adopter une approche defense-in-depth avec plusieurs couches
- ✅ L'authentification et le chiffrement sont obligatoires en production
- ✅ Appliquer le principe du moindre privilège systématiquement
- ✅ Maintenir des audits et logs pour la conformité et le forensic
- ✅ Automatiser la gestion de la sécurité via IaC et CI/CD

**Prochaines Étapes** :
- Commencer par la section 11.1 pour une vue d'ensemble détaillée
- Configurer l'authentification selon votre environnement (section 11.2)
- Implémenter un système de rôles adapté (section 11.3)
- Activer le chiffrement multi-niveaux (section 11.5)
- Valider votre configuration avec la Security Checklist (section 11.8)

⏭️ [Vue d'ensemble de la sécurité MongoDB](/11-securite/01-vue-ensemble-securite.md)
