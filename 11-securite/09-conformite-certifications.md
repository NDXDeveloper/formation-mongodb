🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.9 Conformité et Certifications

## Introduction

La conformité réglementaire est devenue un impératif pour toute organisation manipulant des données sensibles. MongoDB, en tant que base de données critique, doit répondre à de nombreux standards de sécurité et de conformité selon l'industrie et la géographie.

**Pourquoi la conformité est critique** :

```
Aspects légaux
├─ Éviter les amendes (jusqu'à 4% du CA global pour RGPD)
├─ Éviter les poursuites judiciaires
└─ Respecter les obligations contractuelles

Aspects business
├─ Gagner la confiance des clients
├─ Accéder à certains marchés (gouvernement, santé, finance)
├─ Différenciation compétitive
└─ Protection de la réputation

Aspects techniques
├─ Améliorer la posture de sécurité globale
├─ Standardiser les pratiques
├─ Faciliter les audits
└─ Réduire les risques
```

### Certifications supportées par MongoDB

MongoDB (particulièrement MongoDB Atlas) supporte les certifications suivantes :

```
Standards internationaux
├─ ISO/IEC 27001:2013 (Information Security Management)
├─ ISO/IEC 27017:2015 (Cloud Security)
├─ ISO/IEC 27018:2019 (Cloud Privacy)
└─ ISO/IEC 27701:2019 (Privacy Information Management)

Standards américains
├─ SOC 2 Type II (Service Organization Control)
├─ SOC 3
├─ FedRAMP (Federal Risk and Authorization Management Program)
└─ HIPAA/HITECH (Health Insurance Portability and Accountability Act)

Standards financiers
├─ PCI DSS (Payment Card Industry Data Security Standard)
└─ SOX (Sarbanes-Oxley Act)

Standards européens
├─ RGPD/GDPR (Règlement Général sur la Protection des Données)
└─ C5 (Cloud Computing Compliance Controls Catalogue - Allemagne)

Standards sectoriels
├─ HITRUST CSF (Health Information Trust Alliance)
├─ FERPA (Family Educational Rights and Privacy Act - Éducation)
└─ COPPA (Children's Online Privacy Protection Act)
```

## PCI-DSS (Payment Card Industry)

### Vue d'ensemble

**Applicable à** : Toute organisation qui stocke, traite ou transmet des données de cartes bancaires.

**Niveaux** :
- Niveau 1 : >6 millions transactions/an
- Niveau 2 : 1-6 millions transactions/an
- Niveau 3 : 20,000-1 million transactions/an (e-commerce)
- Niveau 4 : <20,000 transactions/an

### 12 exigences PCI-DSS

#### Exigence 1 & 2 : Firewall et Configuration

**Contrôles MongoDB** :

```yaml
# mongod.conf - Configuration PCI-DSS
net:
  port: 27017
  bindIp: 10.0.2.10  # Jamais 0.0.0.0
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb/server.pem
    CAFile: /etc/ssl/mongodb/ca.pem
    allowInvalidCertificates: false  # Critique pour PCI

security:
  authorization: enabled
  clusterAuthMode: x509

# Firewall doit être activé
# Seules les IPs applicatives autorisées
```

**Documentation requise** :
- Diagramme réseau montrant la segmentation
- Liste des flux autorisés
- Configuration du firewall
- Justification de chaque règle

#### Exigence 3 & 4 : Protection et Transmission des données

**Chiffrement des données de cartes** :

```javascript
// CSFLE pour numéros de cartes - OBLIGATOIRE
const clientEncryption = new ClientEncryption(keyVaultClient, {
  keyVaultNamespace: 'encryption.__keyVault',
  kmsProviders: {
    aws: {
      accessKeyId: process.env.AWS_ACCESS_KEY_ID,
      secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY
    }
  }
});

// Créer une DEK pour les cartes
const dataKeyId = await clientEncryption.createDataKey('aws', {
  masterKey: {
    key: process.env.AWS_KMS_KEY_ARN,
    region: 'us-east-1'
  },
  keyAltNames: ['creditCardKey']
});

// Insérer avec chiffrement
await collection.insertOne({
  cardholder: "John Doe",
  cardNumber: await clientEncryption.encrypt(
    "4111111111111111",
    {
      keyId: dataKeyId,
      algorithm: "AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic"
    }
  ),
  expiryDate: "12/25",
  cvv: await clientEncryption.encrypt(
    "123",
    {
      keyId: dataKeyId,
      algorithm: "AEAD_AES_256_CBC_HMAC_SHA_512-Random"
    }
  )
});
```

**Exigences spécifiques** :

```
☐ PAN (Primary Account Number) DOIT être chiffré
☐ CVV/CVC JAMAIS stocké (même chiffré)
☐ PIN JAMAIS stocké
☐ Full Track Data JAMAIS stocké après autorisation
☐ TLS 1.2 minimum pour transmission
☐ Certificats avec validité < 825 jours
☐ Strong cryptography (AES-256, RSA 2048+)
```

**Script de validation PCI-DSS** :

```python
#!/usr/bin/env python3
# pci-dss-validator.py
# Valide la conformité PCI-DSS pour MongoDB

from pymongo import MongoClient
from pymongo.encryption import ClientEncryption
import re
import sys

def validate_pci_compliance(connection_string, database_name):
    """Valide la conformité PCI-DSS"""

    issues = []

    # Connexion
    client = MongoClient(connection_string)
    db = client[database_name]

    print("=== PCI-DSS Compliance Validation ===\n")

    # 1. Vérifier TLS
    print("[1] Checking TLS configuration...")
    server_status = client.admin.command('serverStatus')
    if 'security' not in server_status or 'SSLServerHasCertificateAuthority' not in server_status['security']:
        issues.append("FAIL: TLS/SSL not properly configured")
        print("    ❌ TLS not configured")
    else:
        print("    ✓ TLS is configured")

    # 2. Vérifier authentification
    print("[2] Checking authentication...")
    if not client.admin.command('connectionStatus')['authInfo']['authenticatedUsers']:
        issues.append("FAIL: Authentication not enforced")
        print("    ❌ Authentication not enforced")
    else:
        print("    ✓ Authentication enforced")

    # 3. Scanner les collections pour PANs en clair
    print("[3] Scanning for unencrypted PANs...")
    pan_pattern = re.compile(r'\b[0-9]{13,19}\b')

    for collection_name in db.list_collection_names():
        collection = db[collection_name]

        # Échantillonner 100 documents
        sample = collection.aggregate([{ "$sample": { "size": 100 } }])

        for doc in sample:
            doc_str = str(doc)
            if pan_pattern.search(doc_str):
                # Vérifier si c'est du Binary (chiffré)
                if not any(isinstance(v, bytes) for v in doc.values()):
                    issues.append(f"CRITICAL: Potential unencrypted PAN in {collection_name}")
                    print(f"    ❌ Potential unencrypted PAN in {collection_name}")
                    break

    if not any('unencrypted PAN' in i for i in issues):
        print("    ✓ No unencrypted PANs found in sample")

    # 4. Vérifier audit logging
    print("[4] Checking audit configuration...")
    # Note: Nécessite Enterprise
    try:
        audit_config = client.admin.command('getParameter', 1, auditAuthorizationSuccess=1)
        print("    ✓ Audit is configured")
    except:
        issues.append("FAIL: Audit logging not configured (Enterprise required)")
        print("    ❌ Audit not configured")

    # 5. Vérifier network isolation
    print("[5] Checking network configuration...")
    cmd_line = client.admin.command('getCmdLineOpts')
    bind_ip = cmd_line.get('parsed', {}).get('net', {}).get('bindIp', '0.0.0.0')

    if bind_ip == '0.0.0.0':
        issues.append("CRITICAL: MongoDB bound to 0.0.0.0 (exposed)")
        print("    ❌ MongoDB bound to 0.0.0.0")
    else:
        print(f"    ✓ MongoDB bound to {bind_ip}")

    # Summary
    print("\n=== Summary ===")
    if not issues:
        print("✓ All PCI-DSS checks passed")
        return 0
    else:
        print(f"❌ {len(issues)} issue(s) found:\n")
        for issue in issues:
            print(f"  - {issue}")
        return 1

if __name__ == '__main__':
    if len(sys.argv) < 3:
        print("Usage: python3 pci-dss-validator.py <connection_string> <database>")
        sys.exit(1)

    sys.exit(validate_pci_compliance(sys.argv[1], sys.argv[2]))
```

#### Exigence 6 : Sécurité des applications

```
☐ Patch management process documenté
☐ Updates de sécurité MongoDB appliqués < 30 jours
☐ Code review incluant sécurité
☐ Injection MongoDB (NoSQL injection) prévenue
☐ OWASP Top 10 adressé
☐ Tests de sécurité avant production
```

**Prévention NoSQL Injection** :

```javascript
// ❌ MAUVAIS : Vulnérable à NoSQL injection
app.post('/login', async (req, res) => {
  const user = await db.collection('users').findOne({
    username: req.body.username,  // Danger !
    password: req.body.password   // Danger !
  });
});

// Attaque possible :
// POST /login
// { "username": {"$ne": null}, "password": {"$ne": null} }
// → Retourne le premier utilisateur !

// ✅ BON : Paramètres validés et typés
app.post('/login', async (req, res) => {
  // Validation stricte
  if (typeof req.body.username !== 'string' ||
      typeof req.body.password !== 'string') {
    return res.status(400).json({ error: 'Invalid input' });
  }

  const user = await db.collection('users').findOne({
    username: req.body.username,
    password: hashPassword(req.body.password)
  });
});
```

#### Exigence 8 & 9 : Accès et Sécurité physique

```
☐ Authentification unique par utilisateur (pas de comptes partagés)
☐ MFA pour accès administratif
☐ Politique de mots de passe forte (16+ caractères)
☐ Comptes inactifs désactivés après 90 jours
☐ Session timeout configuré (15 min d'inactivité)
☐ Accès physique aux serveurs contrôlé et loggé
☐ Destruction sécurisée des médias
```

**Configuration session timeout** :

```javascript
// Middleware Express pour session timeout
const sessionTimeout = 15 * 60 * 1000; // 15 minutes

app.use((req, res, next) => {
  if (req.session.lastActivity &&
      (Date.now() - req.session.lastActivity > sessionTimeout)) {
    req.session.destroy();
    return res.status(401).json({ error: 'Session expired' });
  }
  req.session.lastActivity = Date.now();
  next();
});
```

#### Exigence 10 : Monitoring et Audit

**Configuration audit PCI-DSS** :

```yaml
# mongod.conf - Audit PCI-DSS
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit-pci.json
  filter: |
    {
      $or: [
        { atype: "authenticate" },
        { atype: { $in: ["createUser", "updateUser", "dropUser"] } },
        { atype: { $in: ["grantRolesToUser", "revokeRolesFromUser"] } },
        { atype: { $in: ["dropDatabase", "dropCollection"] } },
        {
          atype: "authCheck",
          "param.ns": { $regex: "^payment\\.(cards|transactions)" }
        },
        { atype: "shutdown" }
      ]
    }
```

**Rétention** :
- 3 mois immédiatement disponibles (online)
- 12 mois archivés (offline, accessible sous 24h)

#### Exigence 11 : Tests de sécurité

```
☐ Vulnerability scanning trimestriel (Nessus, Qualys, etc.)
☐ Penetration testing annuel
☐ Tests après changements significatifs
☐ Rapport ASV (Approved Scanning Vendor) trimestriel
☐ Tests des contrôles applicatifs
```

**Automated Vulnerability Scanning** :

```bash
#!/bin/bash
# mongodb-vulnerability-scan.sh

# Utiliser mongodb-scanner (exemple)
docker run --rm \
  -v $(pwd)/reports:/reports \
  mongo-scanner:latest \
  --target mongodb://mongo.example.com:27017 \
  --output /reports/scan-$(date +%Y%m%d).json

# Parser les résultats
jq '.vulnerabilities[] | select(.severity == "HIGH" or .severity == "CRITICAL")' \
  reports/scan-$(date +%Y%m%d).json > critical-vulns.json

# Alerter si vulnérabilités critiques
if [ $(jq length critical-vulns.json) -gt 0 ]; then
  echo "ALERT: Critical vulnerabilities found" | \
    mail -s "PCI Vulnerability Alert" security@company.com
fi
```

#### Exigence 12 : Politique de sécurité

```
☐ Politique de sécurité documentée et approuvée
☐ Revue annuelle de la politique
☐ Formation annuelle du personnel
☐ Gestion des vendors (BAA, SLA)
☐ Incident response plan
☐ Tests annuels du DRP
```

### Checklist PCI-DSS complète

```markdown
## PCI-DSS v4.0 Checklist MongoDB

### Requirement 1: Network Security Controls
- [ ] 1.1.1 Processes documented
- [ ] 1.2.1 Configuration standards defined
- [ ] 1.2.2 Only necessary ports/protocols enabled
- [ ] 1.2.3 Restrict inbound Internet to CDE only
- [ ] 1.3.1 Inbound traffic restricted
- [ ] 1.4.1 Network connections between CDE and untrusted networks controlled
- [ ] 1.4.2 Outbound traffic from CDE restricted

### Requirement 2: Secure Configurations
- [ ] 2.1.1 Configuration standards implemented
- [ ] 2.2.1 Vendor default passwords changed
- [ ] 2.2.2 Unnecessary services disabled
- [ ] 2.2.3 Security features configured
- [ ] 2.2.4 System components hardened

### Requirement 3: Protect Stored Account Data
- [ ] 3.3.1 PAN masked when displayed
- [ ] 3.3.2 PAN rendered unreadable with strong cryptography
- [ ] 3.4.1 PAN unreadable at rest
- [ ] 3.5.1 Encryption keys protected
- [ ] 3.6.1 Key management procedures defined

### Requirement 4: Protect Cardholder Data with Strong Cryptography
- [ ] 4.2.1 Strong cryptography for transmission
- [ ] 4.2.1.1 TLS 1.2 minimum
- [ ] 4.2.1.2 Strong cipher suites only

### Requirement 8: Identify Users and Authenticate Access
- [ ] 8.2.1 Unique ID for each user
- [ ] 8.2.2 MFA for admin access
- [ ] 8.3.1 Strong passwords enforced
- [ ] 8.3.6 Password history (4 generations)

### Requirement 10: Log and Monitor All Access
- [ ] 10.2.1 Audit trails for all users
- [ ] 10.2.2 Automated audit trails
- [ ] 10.3.1 Logs retained 1 year (3 months online)
```

## HIPAA (Health Insurance Portability)

### Vue d'ensemble

**Applicable à** : Organisations manipulant des PHI (Protected Health Information).

**Types de données PHI** :
- Nom, adresse, dates (naissance, admission, décès)
- Numéros de téléphone, fax, email
- Numéro de sécurité sociale
- Numéro de dossier médical
- Plan de santé bénéficiaire
- Identifiants biométriques

### HIPAA Security Rule

#### Administrative Safeguards

**Risk Analysis** :

```python
#!/usr/bin/env python3
# hipaa-risk-analysis.py

import json
from datetime import datetime

class HIPAARiskAnalysis:
    def __init__(self):
        self.risks = []

    def assess_mongodb_deployment(self, config):
        """Évalue les risques HIPAA d'un déploiement MongoDB"""

        print("=== HIPAA Risk Analysis ===\n")

        # 1. Encryption at rest
        if not config.get('encryption_at_rest'):
            self.risks.append({
                'category': 'Technical Safeguard',
                'requirement': 'Encryption',
                'risk': 'PHI stored unencrypted on disk',
                'likelihood': 'Medium',
                'impact': 'High',
                'risk_level': 'HIGH',
                'mitigation': 'Enable MongoDB Encryption at Rest'
            })

        # 2. Encryption in transit
        if not config.get('tls_enabled'):
            self.risks.append({
                'category': 'Technical Safeguard',
                'requirement': 'Transmission Security',
                'risk': 'PHI transmitted in cleartext',
                'likelihood': 'High',
                'impact': 'High',
                'risk_level': 'CRITICAL',
                'mitigation': 'Enable TLS/SSL'
            })

        # 3. Access controls
        if not config.get('authentication_enabled'):
            self.risks.append({
                'category': 'Technical Safeguard',
                'requirement': 'Access Control',
                'risk': 'Unauthenticated access to PHI',
                'likelihood': 'Medium',
                'impact': 'Critical',
                'risk_level': 'CRITICAL',
                'mitigation': 'Enable authentication and RBAC'
            })

        # 4. Audit controls
        if not config.get('audit_enabled'):
            self.risks.append({
                'category': 'Technical Safeguard',
                'requirement': 'Audit Controls',
                'risk': 'No audit trail of PHI access',
                'likelihood': 'High',
                'impact': 'High',
                'risk_level': 'HIGH',
                'mitigation': 'Enable audit logging'
            })

        # 5. Backup
        if not config.get('backup_enabled'):
            self.risks.append({
                'category': 'Physical Safeguard',
                'requirement': 'Contingency Plan',
                'risk': 'PHI loss in case of disaster',
                'likelihood': 'Low',
                'impact': 'Critical',
                'risk_level': 'HIGH',
                'mitigation': 'Implement automated backups'
            })

        return self.risks

    def generate_report(self):
        """Génère un rapport de risque HIPAA"""

        report = {
            'date': datetime.now().isoformat(),
            'total_risks': len(self.risks),
            'critical_risks': len([r for r in self.risks if r['risk_level'] == 'CRITICAL']),
            'high_risks': len([r for r in self.risks if r['risk_level'] == 'HIGH']),
            'risks': self.risks
        }

        return report

# Exemple d'utilisation
config = {
    'encryption_at_rest': False,
    'tls_enabled': True,
    'authentication_enabled': True,
    'audit_enabled': False,
    'backup_enabled': True
}

analyzer = HIPAARiskAnalysis()
risks = analyzer.assess_mongodb_deployment(config)
report = analyzer.generate_report()

print(json.dumps(report, indent=2))
```

#### Technical Safeguards

**1. Access Control**

```javascript
// Rôle HIPAA pour personnel médical (lecture seule)
db.createRole({
  role: "hipaaReadOnlyRole",
  privileges: [
    {
      resource: { db: "hospital", collection: "patients" },
      actions: ["find"]
    }
  ],
  roles: []
});

// Rôle pour médecins (read/write)
db.createRole({
  role: "hipaaPhysicianRole",
  privileges: [
    {
      resource: { db: "hospital", collection: "patients" },
      actions: ["find", "insert", "update"]
    },
    {
      resource: { db: "hospital", collection: "treatments" },
      actions: ["find", "insert", "update"]
    }
  ],
  roles: []
});

// Créer utilisateur avec minimum necessary access
db.createUser({
  user: "dr.smith",
  pwd: passwordPrompt(),
  roles: [
    { role: "hipaaPhysicianRole", db: "hospital" }
  ]
});
```

**2. Audit Controls**

```yaml
# mongod.conf - Audit HIPAA
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit-hipaa.json
  filter: |
    {
      $or: [
        { atype: "authenticate" },
        {
          atype: "authCheck",
          "param.ns": { $regex: "^hospital\\.(patients|treatments|medical_records)" }
        },
        { atype: { $in: ["createUser", "dropUser", "grantRolesToUser"] } },
        { atype: { $in: ["dropDatabase", "dropCollection"] } }
      ]
    }
```

**Rétention** : 6 ans minimum.

**3. Transmission Security**

```yaml
# mongod.conf - TLS HIPAA
net:
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb/server.pem
    CAFile: /etc/ssl/mongodb/ca.pem
    allowInvalidCertificates: false
    # TLS 1.2 minimum
    disabledProtocols: TLS1_0,TLS1_1
```

**4. Encryption at Rest**

```yaml
# mongod.conf - Encryption HIPAA
security:
  enableEncryption: true
  encryptionKeyFile: /secure/mongodb-keyfile
  # Ou mieux : KMS externe
  kmip:
    serverName: kmip.hospital.com
    port: 5696
    clientCertificateFile: /etc/ssl/mongodb/client.pem
```

**5. CSFLE pour PHI spécifiques**

```javascript
// Chiffrer SSN, numéro de dossier, etc.
const encryptedFields = {
  fields: [
    {
      path: "ssn",
      bsonType: "string",
      keyId: ssnKeyId
    },
    {
      path: "medicalRecordNumber",
      bsonType: "string",
      keyId: mrnKeyId
    }
  ]
};

// Insertion avec CSFLE
await collection.insertOne({
  firstName: "John",
  lastName: "Doe",
  ssn: "123-45-6789",  // Sera chiffré automatiquement
  medicalRecordNumber: "MRN-123456"  // Sera chiffré automatiquement
});
```

#### Physical Safeguards

```
☐ Facility access controls (datacenter badge access)
☐ Workstation security (screen locks, encrypted laptops)
☐ Device and media controls (secure disposal)
☐ Environmental controls (fire suppression, temperature)
```

### Breach Notification Rule

**En cas de breach PHI** :

```
Timeline :
├─ Découverte de la breach
├─ Investigation (< 60 jours pour notifier)
├─ Notification aux individus affectés (< 60 jours)
├─ Notification HHS (Department of Health and Human Services)
└─ Notification médias si > 500 personnes

Procédure :
1. Arrêter la breach immédiatement
2. Préserver les preuves (logs, snapshots)
3. Investiguer l'étendue
4. Identifier les PHI compromises
5. Notifier selon les délais légaux
6. Documenter toutes les actions
7. Implémenter des correctifs
8. Revue post-incident
```

**Script de détection de breach** :

```python
#!/usr/bin/env python3
# hipaa-breach-detector.py

import re
from datetime import datetime, timedelta

def analyze_audit_logs(log_file):
    """Analyse les logs d'audit pour détecter des accès suspects"""

    suspicious_activities = []

    # Patterns suspects
    patterns = {
        'bulk_access': r'"find".*"patients".*"batchSize":\s*(\d+)',
        'unauthorized_access': r'"authCheck".*"result":\s*(?!0)',
        'after_hours': r'"ts":\{"\$date":"(\d{4}-\d{2}-\d{2}T(\d{2}):\d{2}:\d{2})',
    }

    with open(log_file, 'r') as f:
        for line in f:
            # Détection accès en masse
            if 'batchSize' in line:
                match = re.search(patterns['bulk_access'], line)
                if match and int(match.group(1)) > 100:
                    suspicious_activities.append({
                        'type': 'BULK_ACCESS',
                        'severity': 'HIGH',
                        'description': f'Bulk access detected: {match.group(1)} records',
                        'line': line
                    })

            # Détection accès hors heures
            match = re.search(patterns['after_hours'], line)
            if match:
                hour = int(match.group(2))
                if hour < 6 or hour > 22:  # Hors 6h-22h
                    suspicious_activities.append({
                        'type': 'AFTER_HOURS_ACCESS',
                        'severity': 'MEDIUM',
                        'description': f'Access at {hour}h',
                        'line': line
                    })

    return suspicious_activities

# Analyse
activities = analyze_audit_logs('/var/log/mongodb/audit-hipaa.json')

if activities:
    print(f"⚠️  {len(activities)} suspicious activities detected")

    # Alerter si activités suspectes
    with open('/var/log/hipaa-alerts.log', 'a') as f:
        f.write(f"\n=== Alert {datetime.now()} ===\n")
        for activity in activities:
            f.write(f"{activity['severity']}: {activity['description']}\n")
```

### Business Associate Agreement (BAA)

Pour utiliser MongoDB Atlas avec données HIPAA :

```
☐ Signer BAA avec MongoDB Inc.
☐ Configurer cluster MongoDB Atlas dédié
☐ Activer encryption at rest
☐ Activer audit logging
☐ Configurer VPC Peering ou PrivateLink
☐ Documenter tous les contrôles
☐ Réaliser HIPAA Security Risk Assessment
```

## SOX (Sarbanes-Oxley Act)

### Vue d'ensemble

**Applicable à** : Sociétés cotées en bourse aux États-Unis.

**Focus** : Intégrité des données financières, contrôles IT, séparation des rôles.

### Contrôles SOX pour MongoDB

#### 1. Séparation des rôles (Segregation of Duties)

```javascript
// Rôle développeur (staging uniquement)
db.createRole({
  role: "developerRole",
  privileges: [
    {
      resource: { db: "staging_finance", collection: "" },
      actions: ["find", "insert", "update", "remove"]
    }
  ],
  roles: []
});

// Rôle production (lecture seule)
db.createRole({
  role: "productionReadRole",
  privileges: [
    {
      resource: { db: "production_finance", collection: "" },
      actions: ["find"]
    }
  ],
  roles: []
});

// Rôle DBA (admin mais pas accès aux données financières)
db.createRole({
  role: "dbaRole",
  privileges: [
    { resource: { cluster: true }, actions: ["hostManager", "clusterMonitor"] },
    { resource: { db: "", collection: "" }, actions: ["dbAdmin", "dbAdminAnyDatabase"] }
  ],
  roles: [],
  // Explicitement SANS readWrite sur production_finance
  authenticationRestrictions: [
    {
      clientSource: ["10.0.10.0/24"]  // Bastion uniquement
    }
  ]
});

// Aucun utilisateur ne doit avoir les 3 :
// 1. Développement de code
// 2. Déploiement en production
// 3. Modification de données financières
```

#### 2. Change Management

```bash
#!/bin/bash
# sox-change-management.sh
# Tous les changements doivent être approuvés et tracés

CHANGE_ID=$1
APPROVER=$2
CHANGE_DESCRIPTION=$3

if [ -z "$CHANGE_ID" ] || [ -z "$APPROVER" ] || [ -z "$CHANGE_DESCRIPTION" ]; then
  echo "Usage: $0 <change_id> <approver> <description>"
  exit 1
fi

# Logger le changement
cat >> /var/log/mongodb-changes.log <<EOF
{
  "timestamp": "$(date -Iseconds)",
  "change_id": "$CHANGE_ID",
  "approver": "$APPROVER",
  "description": "$CHANGE_DESCRIPTION",
  "performed_by": "$USER",
  "hostname": "$(hostname)"
}
EOF

# Exécuter le changement uniquement si approuvé
echo "Change $CHANGE_ID logged and ready for execution"
```

#### 3. Audit des modifications de données financières

```yaml
# mongod.conf - Audit SOX
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit-sox.json
  filter: |
    {
      $or: [
        {
          atype: "authCheck",
          "param.ns": { $regex: "^finance\\.(transactions|accounts|ledger)" },
          "param.command": { $in: ["insert", "update", "delete"] }
        },
        {
          atype: "authCheck",
          "param.command": { $in: ["dropDatabase", "dropCollection"] }
        },
        { atype: { $in: ["createUser", "dropUser", "grantRolesToUser"] } }
      ]
    }
```

**Rétention** : 7 ans minimum.

#### 4. Contrôle des accès

```javascript
// Audit trimestriel des accès
db.getUsers().forEach(function(user) {
  print("User: " + user.user);
  print("Roles: " + JSON.stringify(user.roles));
  print("Last login: " + user.customData?.lastLogin);
  print("---");
});

// Script pour désactiver comptes inactifs
db.getUsers().forEach(function(user) {
  var lastLogin = user.customData?.lastLogin;
  var ninetyDaysAgo = new Date();
  ninetyDaysAgo.setDate(ninetyDaysAgo.getDate() - 90);

  if (!lastLogin || new Date(lastLogin) < ninetyDaysAgo) {
    print("WARNING: User " + user.user + " inactive for >90 days");
    // db.updateUser(user.user, { roles: [] });  // Désactiver
  }
});
```

#### 5. Backup et Recovery

```
☐ Backups quotidiens automatisés
☐ Tests de restauration trimestriels DOCUMENTÉS
☐ Offsite backup (différent datacenter)
☐ Backup retention : 7 ans minimum
☐ Backup integrity validation mensuelle
☐ RTO/RPO documentés et testés
```

**Script de test de restauration SOX** :

```bash
#!/bin/bash
# sox-backup-test.sh
# Test de restauration trimestriel SOX

TEST_DATE=$(date +%Y%m%d)
BACKUP_FILE="/backup/mongodb/latest.gz"
RESTORE_DIR="/tmp/sox-restore-test-$TEST_DATE"

echo "=== SOX Backup Test - $TEST_DATE ===" | tee -a /var/log/sox-backup-tests.log

# 1. Restaurer dans environnement test
mkdir -p "$RESTORE_DIR"
mongorestore --gzip --archive="$BACKUP_FILE" --dir="$RESTORE_DIR" >> /var/log/sox-backup-tests.log 2>&1

if [ $? -eq 0 ]; then
  echo "✓ Restore successful" | tee -a /var/log/sox-backup-tests.log
else
  echo "❌ Restore FAILED" | tee -a /var/log/sox-backup-tests.log
  exit 1
fi

# 2. Valider l'intégrité des données
mongosh --eval "
  use finance;
  var count = db.transactions.countDocuments();
  print('Transactions count: ' + count);

  if (count > 0) {
    print('✓ Data integrity validated');
  } else {
    print('❌ Data integrity FAILED');
  }
" >> /var/log/sox-backup-tests.log

# 3. Nettoyer
rm -rf "$RESTORE_DIR"

echo "=== Test complete ===" | tee -a /var/log/sox-backup-tests.log

# Notifier
mail -s "SOX Backup Test - $TEST_DATE" compliance@company.com < /var/log/sox-backup-tests.log
```

### Documentation SOX requise

```
☐ System documentation
  ├─ Architecture diagrams
  ├─ Data flow diagrams
  └─ Network diagrams

☐ Security controls documentation
  ├─ Access control matrix
  ├─ RBAC definitions
  └─ Segregation of duties matrix

☐ Change management documentation
  ├─ Change request forms
  ├─ Approval records
  ├─ Implementation logs
  └─ Rollback procedures

☐ Backup and recovery documentation
  ├─ Backup procedures
  ├─ Recovery procedures
  ├─ Test results (quarterly)
  └─ RTO/RPO statements

☐ Audit logs and reports
  ├─ Access logs (7 years)
  ├─ Change logs (7 years)
  ├─ Quarterly access reviews
  └─ Annual security assessments
```

## RGPD (Règlement Général sur la Protection des Données)

### Principes RGPD

```
1. Licéité, loyauté, transparence
2. Limitation des finalités
3. Minimisation des données
4. Exactitude
5. Limitation de la conservation
6. Intégrité et confidentialité
7. Responsabilité (accountability)
```

### Implémentation MongoDB pour RGPD

#### 1. Droit à l'oubli (Right to be Forgotten)

```javascript
// Script pour supprimer toutes les données d'un individu
async function deleteUserData(userId) {
  const session = client.startSession();

  try {
    await session.withTransaction(async () => {
      // Supprimer de toutes les collections
      await db.users.deleteOne({ userId }, { session });
      await db.orders.deleteMany({ userId }, { session });
      await db.preferences.deleteOne({ userId }, { session });
      await db.analytics.deleteMany({ userId }, { session });

      // Logger la suppression (RGPD exige traçabilité)
      await db.gdpr_requests.insertOne({
        type: 'deletion',
        userId,
        timestamp: new Date(),
        status: 'completed'
      }, { session });
    });

    console.log(`User ${userId} data deleted successfully`);
  } finally {
    await session.endSession();
  }
}

// Anonymisation (alternative à suppression)
async function anonymizeUserData(userId) {
  await db.users.updateOne(
    { userId },
    {
      $set: {
        name: "ANONYMIZED",
        email: `anon-${userId}@anonymized.local`,
        phone: null,
        address: null,
        gdprAnonymized: true,
        anonymizedAt: new Date()
      },
      $unset: {
        ssn: "",
        birthDate: "",
        preferences: ""
      }
    }
  );
}
```

#### 2. Droit à la portabilité

```javascript
// Export des données d'un utilisateur au format JSON
async function exportUserData(userId) {
  const userData = {
    user: await db.users.findOne({ userId }),
    orders: await db.orders.find({ userId }).toArray(),
    preferences: await db.preferences.findOne({ userId }),
    exportDate: new Date(),
    format: 'JSON',
    gdprRequest: true
  };

  // Générer un fichier ZIP
  const archive = archiver('zip');
  archive.append(JSON.stringify(userData, null, 2), { name: 'user-data.json' });

  return archive;
}
```

#### 3. Consentement et traçabilité

```javascript
// Schéma pour tracer le consentement
const consentSchema = {
  userId: ObjectId,
  consents: [
    {
      type: "marketing_emails",  // Type de consentement
      granted: true,
      timestamp: ISODate(),
      ipAddress: "192.168.1.100",
      userAgent: "Mozilla/5.0...",
      source: "signup_form",  // Où le consentement a été donné
      version: "1.0"  // Version de la politique
    }
  ],
  withdrawals: [
    {
      type: "marketing_emails",
      timestamp: ISODate(),
      reason: "user_request"
    }
  ]
};

// Vérifier le consentement avant traitement
async function hasConsent(userId, consentType) {
  const consent = await db.consents.findOne({
    userId,
    "consents.type": consentType,
    "consents.granted": true
  });

  // Vérifier qu'il n'a pas été retiré
  if (consent) {
    const withdrawal = consent.withdrawals?.find(w => w.type === consentType);
    if (withdrawal) {
      return withdrawal.timestamp < consent.consents.find(c => c.type === consentType).timestamp;
    }
    return true;
  }

  return false;
}
```

#### 4. Limitation de la rétention

```javascript
// TTL indexes pour suppression automatique
db.sessions.createIndex(
  { "createdAt": 1 },
  { expireAfterSeconds: 30 * 24 * 60 * 60 }  // 30 jours
);

db.logs.createIndex(
  { "timestamp": 1 },
  { expireAfterSeconds: 365 * 24 * 60 * 60 }  // 1 an
);

db.analytics.createIndex(
  { "eventDate": 1 },
  { expireAfterSeconds: 90 * 24 * 60 * 60 }  // 90 jours
);

// Script de purge manuel pour données plus anciennes
db.orders.deleteMany({
  status: "completed",
  completedAt: { $lt: new Date(Date.now() - 7 * 365 * 24 * 60 * 60 * 1000) }  // 7 ans
});
```

#### 5. Sécurité et confidentialité by design

```javascript
// Pseudonymisation automatique à l'insertion
db.users.insertOne({
  userId: generateUserId(),  // ID pseudonyme
  name: encrypt(userData.name),  // Chiffré
  email: hash(userData.email),  // Hashé
  preferences: userData.preferences,
  createdAt: new Date()
});

// Minimisation des données
// Ne collecter QUE ce qui est nécessaire
const userSchema = {
  userId: { type: String, required: true },
  name: { type: String, required: true },
  email: { type: String, required: true },
  // PAS de données inutiles comme : race, religion, orientation sexuelle
};
```

#### 6. Transferts hors UE

```
☐ Mécanisme de transfert approprié :
  ├─ Clauses contractuelles types (SCC)
  ├─ Binding Corporate Rules (BCR)
  ├─ Privacy Shield (invalide depuis 2020)
  └─ Dérogations spécifiques

☐ Évaluation de l'adéquation du pays tiers
☐ Mesures supplémentaires si nécessaire (chiffrement)
☐ Documentation du transfert
```

**Configuration Atlas multi-région** :

```hcl
# terraform/mongodb-atlas-eu.tf
# Garantir que les données restent en UE

resource "mongodbatlas_cluster" "gdpr_compliant" {
  project_id = var.project_id
  name       = "gdpr-cluster"

  # Région UE uniquement
  provider_name               = "AWS"
  provider_region_name        = "EU_WEST_1"
  provider_instance_size_name = "M30"

  # Backup région UE
  backup_enabled               = true
  provider_backup_enabled      = true
  pit_enabled                  = true

  # Réplication UE uniquement
  replication_specs {
    num_shards = 1
    regions_config {
      region_name     = "EU_WEST_1"
      electable_nodes = 3
      priority        = 7
      read_only_nodes = 0
    }
  }

  # Encryption
  encryption_at_rest_provider = "AWS"

  # Labels RGPD
  labels = {
    compliance = "GDPR"
    region     = "EU"
  }
}
```

### Documentation RGPD requise

```
☐ Registre des traitements (Article 30)
  ├─ Finalités du traitement
  ├─ Catégories de données
  ├─ Catégories de personnes concernées
  ├─ Destinataires des données
  ├─ Transferts hors UE
  ├─ Délais de suppression
  └─ Mesures de sécurité

☐ Privacy Impact Assessment (PIA/DPIA)
  ├─ Description du traitement
  ├─ Nécessité et proportionnalité
  ├─ Risques pour les droits et libertés
  └─ Mesures pour traiter les risques

☐ Politique de confidentialité
☐ Procédures d'exercice des droits
☐ Procédure de notification de violation (72h)
☐ Contrats avec sous-traitants (DPA)
```

## ISO 27001

### Vue d'ensemble

**ISO/IEC 27001** : Standard international pour la gestion de la sécurité de l'information.

**Annexe A** : 114 contrôles organisés en 14 domaines.

### Contrôles ISO 27001 pertinents pour MongoDB

#### A.9 : Access Control

```javascript
// A.9.2.1 : User registration and de-registration
function createUserISO27001(username, role, businessJustification) {
  // Enregistrement formalisé
  const request = {
    requestId: generateRequestId(),
    username,
    role,
    requestedBy: currentUser,
    businessJustification,
    approvedBy: null,
    status: 'pending',
    createdAt: new Date()
  };

  db.accessRequests.insertOne(request);

  // Workflow d'approbation
  notifyApprover(request);
}

// A.9.2.6 : Removal of access rights
function deregisterUser(username) {
  // Désactivation immédiate
  db.runCommand({
    updateUser: username,
    roles: []  // Retirer tous les rôles
  });

  // Logger
  db.accessLogs.insertOne({
    action: 'deregistration',
    username,
    performedBy: currentUser,
    timestamp: new Date()
  });
}
```

#### A.10 : Cryptography

```
☐ A.10.1.1 : Policy on the use of cryptographic controls
  ├─ TLS 1.2+ pour transit
  ├─ AES-256 pour repos
  ├─ RSA 2048+ pour échange de clés
  └─ SHA-256+ pour hashing

☐ A.10.1.2 : Key management
  ├─ KMS externe (AWS KMS, Azure Key Vault)
  ├─ Rotation annuelle des Master Keys
  ├─ Séparation des responsabilités
  └─ Backup sécurisé des clés
```

#### A.12 : Operations Security

```bash
#!/bin/bash
# A.12.1.2 : Change management procedures

# Tous les changements passent par ce script
CHANGE_TYPE=$1  # patch, config, upgrade
CHANGE_DESC=$2
APPROVAL_TICKET=$3

# Validation
if [ -z "$APPROVAL_TICKET" ]; then
  echo "ERROR: Approval ticket required"
  exit 1
fi

# Vérifier que le ticket est approuvé
APPROVED=$(curl -s "https://tickets.company.com/api/tickets/$APPROVAL_TICKET" | jq -r '.status')
if [ "$APPROVED" != "approved" ]; then
  echo "ERROR: Change not approved"
  exit 1
fi

# Logger le changement
echo "$(date -Iseconds) - $CHANGE_TYPE - $CHANGE_DESC - $APPROVAL_TICKET - $USER" >> /var/log/iso27001-changes.log

# Backup pre-change
mongodump --archive=/backup/pre-change-$(date +%Y%m%d-%H%M%S).gz --gzip

# Exécuter le changement
echo "Change approved and logged. Proceed with implementation."
```

#### A.12.3 : Backup

```bash
#!/bin/bash
# A.12.3.1 : Information backup

# Backup quotidien avec vérification d'intégrité
BACKUP_DIR="/backup/mongodb"
DATE=$(date +%Y%m%d)
BACKUP_FILE="$BACKUP_DIR/mongodb-backup-$DATE.gz"

# Backup
mongodump --archive="$BACKUP_FILE" --gzip --oplog

# Vérification d'intégrité
if [ $? -eq 0 ]; then
  # Calculer checksum
  sha256sum "$BACKUP_FILE" > "$BACKUP_FILE.sha256"

  # Tester le backup
  mongorestore --archive="$BACKUP_FILE" --gzip --dryRun

  if [ $? -eq 0 ]; then
    echo "✓ Backup successful and verified" >> /var/log/iso27001-backup.log
  else
    echo "❌ Backup verification FAILED" >> /var/log/iso27001-backup.log
    # Alerter
    mail -s "ALERT: Backup verification failed" security@company.com
  fi
else
  echo "❌ Backup FAILED" >> /var/log/iso27001-backup.log
fi

# A.12.3.1 : Tester la restauration (mensuel)
if [ $(date +%d) -eq 01 ]; then
  # Premier jour du mois : test de restauration
  RESTORE_DIR="/tmp/iso27001-restore-test-$DATE"
  mkdir -p "$RESTORE_DIR"

  mongorestore --archive="$BACKUP_FILE" --gzip --dir="$RESTORE_DIR" --nsInclude="test.*"

  if [ $? -eq 0 ]; then
    echo "✓ Restore test successful" >> /var/log/iso27001-backup.log
    rm -rf "$RESTORE_DIR"
  else
    echo "❌ Restore test FAILED" >> /var/log/iso27001-backup.log
    # Alerter
    mail -s "CRITICAL: Restore test failed" security@company.com
  fi
fi
```

#### A.12.4 : Logging and Monitoring

```yaml
# Configuration logging ISO 27001
auditLog:
  destination: syslog
  format: JSON
  filter: |
    {
      $or: [
        { atype: "authenticate" },
        { atype: { $in: ["createUser", "updateUser", "dropUser"] } },
        { atype: { $in: ["createRole", "updateRole", "dropRole"] } },
        { atype: { $in: ["grantRolesToUser", "revokeRolesFromUser"] } },
        { atype: "authCheck", result: { $ne: 0 } },
        { atype: { $in: ["dropDatabase", "dropCollection"] } },
        { atype: "shutdown" }
      ]
    }
```

#### A.18 : Compliance

```
☐ A.18.1.1 : Identification of applicable legislation
  ├─ RGPD (si données UE)
  ├─ PCI-DSS (si paiements)
  ├─ HIPAA (si santé US)
  └─ Lois locales

☐ A.18.1.2 : Intellectual property rights
  ├─ Licences MongoDB respectées
  └─ Pas de reverse engineering

☐ A.18.1.3 : Protection of records
  ├─ Rétention selon obligations légales
  └─ Destruction sécurisée

☐ A.18.1.4 : Privacy and protection of PII
  ├─ Conformité RGPD
  └─ Privacy by design

☐ A.18.1.5 : Regulation of cryptographic controls
  ├─ Export controls respectés
  └─ Algorithmes conformes
```

### Certification ISO 27001

**Processus** :

```
1. Gap Analysis
   ├─ Évaluer l'état actuel
   ├─ Identifier les écarts
   └─ Prioriser les actions

2. ISMS Implementation
   ├─ Définir le scope
   ├─ Politique de sécurité
   ├─ Risk assessment
   ├─ Statement of Applicability (SoA)
   └─ Implémenter les contrôles

3. Internal Audit
   ├─ Auditer les contrôles
   ├─ Vérifier la documentation
   └─ Management Review

4. Certification Audit
   ├─ Stage 1 : Documentation review
   ├─ Stage 2 : Implementation audit
   └─ Certification (si succès)

5. Surveillance Audits
   ├─ Annuel : Surveillance audit
   └─ 3 ans : Re-certification
```

## SOC 2 Type II

### Vue d'ensemble

**SOC 2** : Service Organization Control 2
**Trust Service Criteria** :

```
1. Security (mandatory)
2. Availability (optional)
3. Processing Integrity (optional)
4. Confidentiality (optional)
5. Privacy (optional)
```

### Security Criteria (CC)

#### CC6.1 : Logical and Physical Access Controls

**Contrôles MongoDB** :

```javascript
// Authentification multifacteur pour admins
db.runCommand({
  createUser: "admin",
  pwd: passwordPrompt(),
  roles: ["root"],
  mechanisms: ["SCRAM-SHA-256"],
  customData: {
    mfaEnabled: true,
    mfaMethod: "TOTP"
  }
});

// IP Whitelisting
// Via firewall ou MongoDB Atlas IP Access List
```

#### CC6.6 : Logical Access - Removal

```javascript
// Procédure de dé-provisioning
async function deprovisionUser(username, terminationDate) {
  // 1. Logger la demande
  await db.audit.insertOne({
    action: 'user_deprovision',
    username,
    terminationDate,
    requestedBy: currentUser,
    requestedAt: new Date()
  });

  // 2. Désactiver immédiatement
  await db.runCommand({
    updateUser: username,
    roles: []
  });

  // 3. Supprimer après 30 jours (rétention)
  await db.audit.insertOne({
    action: 'user_deletion_scheduled',
    username,
    scheduledFor: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000)
  });
}
```

#### CC7.2 : System Monitoring

```python
#!/usr/bin/env python3
# soc2-monitoring.py
# Monitoring SOC 2 des événements de sécurité

import pymongo
from datetime import datetime, timedelta

def monitor_security_events():
    """Monitor des événements critiques SOC 2"""

    client = pymongo.MongoClient("mongodb://localhost:27017")

    # Événements des dernières 24h
    since = datetime.now() - timedelta(hours=24)

    # 1. Échecs d'authentification
    auth_failures = client.admin.command('aggregate', 'system.profile', pipeline=[
        {'$match': {'ts': {'$gte': since}, 'op': 'authenticate', 'result': {'$ne': 0}}},
        {'$group': {'_id': '$remote', 'count': {'$sum': 1}}},
        {'$match': {'count': {'$gte': 5}}}  # 5+ échecs
    ])

    if auth_failures:
        print(f"⚠️  Multiple auth failures detected from {len(auth_failures)} IPs")
        # Alerter

    # 2. Changements de privilèges
    privilege_changes = client.admin.command('aggregate', 'system.profile', pipeline=[
        {'$match': {'ts': {'$gte': since}, 'command.grantRolesToUser': {'$exists': True}}}
    ])

    if privilege_changes:
        print(f"ℹ️  {len(privilege_changes)} privilege changes in last 24h")

    # 3. Accès hors heures
    after_hours = client.admin.command('aggregate', 'system.profile', pipeline=[
        {'$match': {
            'ts': {'$gte': since},
            '$expr': {
                '$or': [
                    {'$lt': [{'$hour': '$ts'}, 6]},
                    {'$gt': [{'$hour': '$ts'}, 22]}
                ]
            }
        }}
    ])

    if after_hours:
        print(f"⚠️  {len(after_hours)} after-hours accesses detected")

if __name__ == '__main__':
    monitor_security_events()
```

### Rapport SOC 2 Type II

**Contenu** :

```
1. Management's Description of the System
   ├─ System overview
   ├─ Infrastructure
   ├─ Software
   ├─ People
   ├─ Procedures
   └─ Data

2. Independent Service Auditor's Report
   ├─ Opinion on management's description
   ├─ Opinion on control design
   └─ Opinion on operating effectiveness

3. Control Objectives and Related Controls
   ├─ Pour chaque Trust Service Criteria
   ├─ Description du contrôle
   ├─ Tests performed
   └─ Results of tests

4. Other Information Provided by Management
   ├─ Complementary controls (user entity)
   └─ Subservice organizations
```

**Timeline** :

```
Period : Minimum 6 mois (12 mois recommandé)
├─ Préparation : 2-4 mois
├─ Audit période : 6-12 mois
├─ Fieldwork : 2-4 semaines
└─ Rapport : 2-4 semaines

Total : ~12-18 mois pour première certification
```

## Outils et automatisation

### Outil d'audit multi-conformité

```python
#!/usr/bin/env python3
# compliance-audit-tool.py
# Outil d'audit multi-conformité pour MongoDB

import pymongo
import json
from datetime import datetime
from enum import Enum

class ComplianceStandard(Enum):
    PCI_DSS = "PCI-DSS"
    HIPAA = "HIPAA"
    SOX = "SOX"
    GDPR = "GDPR"
    ISO27001 = "ISO 27001"
    SOC2 = "SOC 2"

class ComplianceAuditor:
    def __init__(self, connection_string):
        self.client = pymongo.MongoClient(connection_string)
        self.results = {}

    def audit(self, standard):
        """Exécute l'audit pour un standard donné"""

        print(f"\n=== {standard.value} Compliance Audit ===\n")

        if standard == ComplianceStandard.PCI_DSS:
            return self.audit_pci_dss()
        elif standard == ComplianceStandard.HIPAA:
            return self.audit_hipaa()
        elif standard == ComplianceStandard.SOX:
            return self.audit_sox()
        elif standard == ComplianceStandard.GDPR:
            return self.audit_gdpr()
        elif standard == ComplianceStandard.ISO27001:
            return self.audit_iso27001()
        elif standard == ComplianceStandard.SOC2:
            return self.audit_soc2()

    def audit_pci_dss(self):
        """Audit PCI-DSS"""
        checks = []

        # Requirement 2: Secure configurations
        checks.append(self._check_authentication())
        checks.append(self._check_tls())

        # Requirement 3: Protect stored data
        checks.append(self._check_encryption_at_rest())
        checks.append(self._check_field_level_encryption())

        # Requirement 10: Track and monitor
        checks.append(self._check_audit_enabled())

        return self._generate_report(ComplianceStandard.PCI_DSS, checks)

    def audit_hipaa(self):
        """Audit HIPAA"""
        checks = []

        # Technical Safeguards
        checks.append(self._check_access_control())
        checks.append(self._check_audit_enabled())
        checks.append(self._check_encryption_at_rest())
        checks.append(self._check_tls())

        # Retention
        checks.append(self._check_audit_retention(years=6))

        return self._generate_report(ComplianceStandard.HIPAA, checks)

    def _check_authentication(self):
        """Vérifie que l'authentification est activée"""
        try:
            # Tenter une connexion sans auth
            test_client = pymongo.MongoClient(
                self.client.address,
                serverSelectionTimeoutMS=2000
            )
            test_client.admin.command('ping')
            return {
                'check': 'Authentication',
                'status': 'FAIL',
                'message': 'Authentication is not enforced'
            }
        except pymongo.errors.OperationFailure:
            return {
                'check': 'Authentication',
                'status': 'PASS',
                'message': 'Authentication is enforced'
            }

    def _check_tls(self):
        """Vérifie que TLS est activé"""
        server_status = self.client.admin.command('serverStatus')

        if 'security' in server_status and 'SSLServerHasCertificateAuthority' in server_status['security']:
            return {
                'check': 'TLS/SSL',
                'status': 'PASS',
                'message': 'TLS is configured'
            }
        else:
            return {
                'check': 'TLS/SSL',
                'status': 'FAIL',
                'message': 'TLS is not configured'
            }

    def _check_encryption_at_rest(self):
        """Vérifie le chiffrement au repos"""
        server_status = self.client.admin.command('serverStatus')

        if 'security' in server_status and 'encryptionAtRest' in server_status['security']:
            return {
                'check': 'Encryption at Rest',
                'status': 'PASS',
                'message': 'Encryption at Rest is enabled'
            }
        else:
            return {
                'check': 'Encryption at Rest',
                'status': 'FAIL',
                'message': 'Encryption at Rest is not enabled (Enterprise required)'
            }

    def _check_audit_enabled(self):
        """Vérifie que l'audit est activé"""
        try:
            self.client.admin.command('getParameter', 1, auditAuthorizationSuccess=1)
            return {
                'check': 'Audit Logging',
                'status': 'PASS',
                'message': 'Audit is configured'
            }
        except:
            return {
                'check': 'Audit Logging',
                'status': 'FAIL',
                'message': 'Audit is not configured (Enterprise required)'
            }

    def _generate_report(self, standard, checks):
        """Génère un rapport d'audit"""

        passed = len([c for c in checks if c['status'] == 'PASS'])
        failed = len([c for c in checks if c['status'] == 'FAIL'])

        report = {
            'standard': standard.value,
            'date': datetime.now().isoformat(),
            'summary': {
                'total_checks': len(checks),
                'passed': passed,
                'failed': failed,
                'compliance_rate': f"{(passed/len(checks)*100):.1f}%"
            },
            'checks': checks
        }

        # Afficher le résumé
        print(f"Total checks: {len(checks)}")
        print(f"Passed: {passed}")
        print(f"Failed: {failed}")
        print(f"Compliance rate: {report['summary']['compliance_rate']}\n")

        # Détails
        for check in checks:
            status_icon = "✓" if check['status'] == 'PASS' else "❌"
            print(f"{status_icon} {check['check']}: {check['message']}")

        return report

# Utilisation
if __name__ == '__main__':
    auditor = ComplianceAuditor("mongodb://localhost:27017")

    # Auditer PCI-DSS
    pci_report = auditor.audit(ComplianceStandard.PCI_DSS)

    # Sauvegarder le rapport
    with open(f'compliance-report-{datetime.now().strftime("%Y%m%d")}.json', 'w') as f:
        json.dump(pci_report, f, indent=2)
```

### Dashboard de conformité

```python
#!/usr/bin/env python3
# compliance-dashboard.py
# Dashboard temps réel de conformité

from flask import Flask, render_template, jsonify
from compliance_audit_tool import ComplianceAuditor, ComplianceStandard
import schedule
import time
import threading

app = Flask(__name__)

# État global de conformité
compliance_state = {}

def update_compliance_status():
    """Met à jour l'état de conformité"""
    auditor = ComplianceAuditor("mongodb://localhost:27017")

    compliance_state['pci_dss'] = auditor.audit(ComplianceStandard.PCI_DSS)
    compliance_state['hipaa'] = auditor.audit(ComplianceStandard.HIPAA)
    compliance_state['sox'] = auditor.audit(ComplianceStandard.SOX)
    compliance_state['gdpr'] = auditor.audit(ComplianceStandard.GDPR)
    compliance_state['last_update'] = datetime.now().isoformat()

@app.route('/')
def dashboard():
    return render_template('compliance_dashboard.html')

@app.route('/api/compliance/status')
def compliance_status():
    return jsonify(compliance_state)

def schedule_audits():
    """Planifie les audits automatiques"""
    # Audit quotidien
    schedule.every().day.at("02:00").do(update_compliance_status)

    while True:
        schedule.run_pending()
        time.sleep(60)

if __name__ == '__main__':
    # Audit initial
    update_compliance_status()

    # Démarrer le scheduler en background
    scheduler_thread = threading.Thread(target=schedule_audits, daemon=True)
    scheduler_thread.start()

    # Démarrer l'app Flask
    app.run(host='0.0.0.0', port=5000)
```

## Conclusion

La conformité n'est pas une destination mais un processus continu. MongoDB fournit les outils techniques nécessaires, mais la conformité requiert également :

**Organisation** :
- Politique de sécurité documentée
- Procédures opérationnelles claires
- Formation du personnel
- Audits réguliers

**Technique** :
- Configuration sécurisée (cette documentation)
- Monitoring continu
- Automatisation des contrôles
- Documentation technique

**Juridique** :
- Conformité aux lois applicables
- Contrats avec vendors (BAA, DPA)
- Privacy notices
- Procédures de réponse aux incidents

**Checklist finale** :

```
☐ Identifier les standards applicables
☐ Réaliser un gap analysis
☐ Implémenter les contrôles techniques
☐ Documenter toutes les configurations
☐ Former l'équipe
☐ Mettre en place le monitoring
☐ Planifier les audits réguliers
☐ Tester les procédures
☐ Maintenir la documentation à jour
☐ Améliorer continuellement
```

**Ressources** :
- [MongoDB Compliance Documentation](https://www.mongodb.com/collateral/mongodb-security-architecture-white-paper)
- [MongoDB Atlas Certifications](https://www.mongodb.com/cloud/trust)
- Standards officiels (PCI-DSS, HIPAA, ISO 27001, etc.)
- Consultants spécialisés en conformité

La conformité est un investissement qui protège votre organisation, vos clients, et vos données. Prenez-la au sérieux ! 🔐

⏭️ [Sauvegarde et Restauration](/12-sauvegarde-restauration/README.md)
