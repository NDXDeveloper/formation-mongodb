🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.5 Cas d'usage et quand choisir MongoDB

## Introduction

Choisir la bonne base de données est une décision cruciale pour tout projet. MongoDB excelle dans de nombreux contextes, mais n'est pas la solution universelle à tous les problèmes de stockage de données. Cette section vous aidera à identifier les situations où MongoDB est le choix optimal et celles où d'autres solutions pourraient être plus appropriées.

---

## Les forces de MongoDB

Avant d'explorer les cas d'usage, rappelons les caractéristiques qui font la force de MongoDB :

| Force | Description |
|-------|-------------|
| **Schéma flexible** | Adaptez votre modèle de données sans migrations complexes |
| **Documents riches** | Stockez des structures de données complexes naturellement |
| **Scalabilité horizontale** | Distribuez vos données sur plusieurs serveurs |
| **Haute disponibilité** | Réplication automatique avec basculement |
| **Performance** | Excellentes performances en lecture/écriture |
| **Requêtes expressives** | Filtres puissants, agrégations, recherche full-text |
| **Écosystème riche** | Drivers, outils, cloud (Atlas) |

---

## Cas d'usage principaux

### 1. Applications web et mobiles

MongoDB est particulièrement adapté aux applications web et mobiles modernes.

#### Pourquoi MongoDB excelle ici ?

- **Profils utilisateurs flexibles** : Chaque utilisateur peut avoir des attributs différents
- **Sessions et préférences** : Stockage naturel en documents
- **Évolution rapide** : Les fonctionnalités changent fréquemment
- **Scalabilité** : Gestion de millions d'utilisateurs

#### Exemple : Profil utilisateur

```json
{
  "_id": ObjectId("..."),
  "username": "marie_dev",
  "email": "marie@example.com",
  "profile": {
    "firstName": "Marie",
    "lastName": "Dupont",
    "avatar": "https://cdn.example.com/avatars/marie.jpg",
    "bio": "Développeuse passionnée"
  },
  "preferences": {
    "theme": "dark",
    "language": "fr",
    "notifications": {
      "email": true,
      "push": false,
      "sms": false
    }
  },
  "socialLinks": [
    { "platform": "twitter", "url": "https://twitter.com/marie_dev" },
    { "platform": "github", "url": "https://github.com/mariedev" }
  ],
  "lastLogin": ISODate("2024-11-28T14:30:00Z"),
  "createdAt": ISODate("2023-06-15T10:00:00Z")
}
```

#### Entreprises utilisant MongoDB pour le web/mobile

- **Adobe** : Gestion des profils Creative Cloud
- **Uber** : Données utilisateurs et géolocalisation
- **Lyft** : Application de covoiturage
- **Coinbase** : Plateforme de cryptomonnaies

---

### 2. Gestion de contenu (CMS)

Les systèmes de gestion de contenu bénéficient grandement de la flexibilité de MongoDB.

#### Pourquoi MongoDB excelle ici ?

- **Types de contenu variés** : Articles, vidéos, podcasts, galeries
- **Métadonnées flexibles** : Chaque contenu peut avoir ses propres attributs
- **Hiérarchie naturelle** : Catégories, tags, relations imbriquées
- **Versioning** : Historique des modifications dans le document

#### Exemple : Article de blog

```json
{
  "_id": ObjectId("..."),
  "type": "article",
  "title": "Introduction à MongoDB",
  "slug": "introduction-mongodb",
  "status": "published",
  "author": {
    "id": ObjectId("..."),
    "name": "Jean Martin",
    "avatar": "https://cdn.example.com/authors/jean.jpg"
  },
  "content": {
    "excerpt": "Découvrez les bases de MongoDB...",
    "body": "# Introduction\n\nMongoDB est une base de données...",
    "format": "markdown"
  },
  "media": [
    {
      "type": "image",
      "url": "https://cdn.example.com/images/mongodb-logo.png",
      "alt": "Logo MongoDB",
      "position": "header"
    }
  ],
  "categories": ["Bases de données", "NoSQL", "Tutoriels"],
  "tags": ["mongodb", "nosql", "débutant"],
  "seo": {
    "metaTitle": "Introduction à MongoDB - Guide complet",
    "metaDescription": "Apprenez les fondamentaux de MongoDB...",
    "canonicalUrl": "https://blog.example.com/introduction-mongodb"
  },
  "stats": {
    "views": 15420,
    "likes": 342,
    "shares": 89
  },
  "publishedAt": ISODate("2024-11-01T09:00:00Z"),
  "updatedAt": ISODate("2024-11-15T14:30:00Z")
}
```

#### Exemple : Contenu multimédia

```json
{
  "_id": ObjectId("..."),
  "type": "video",
  "title": "Tutoriel MongoDB en 10 minutes",
  "duration": 623,
  "resolution": "1080p",
  "sources": [
    { "quality": "1080p", "url": "https://cdn.example.com/videos/tuto-hd.mp4" },
    { "quality": "720p", "url": "https://cdn.example.com/videos/tuto-sd.mp4" }
  ],
  "thumbnails": {
    "small": "https://cdn.example.com/thumbs/tuto-sm.jpg",
    "large": "https://cdn.example.com/thumbs/tuto-lg.jpg"
  },
  "subtitles": [
    { "language": "fr", "url": "https://cdn.example.com/subs/tuto-fr.vtt" },
    { "language": "en", "url": "https://cdn.example.com/subs/tuto-en.vtt" }
  ],
  "chapters": [
    { "title": "Introduction", "startTime": 0 },
    { "title": "Installation", "startTime": 45 },
    { "title": "Premier document", "startTime": 180 }
  ]
}
```

---

### 3. Catalogue produits (E-commerce)

Les catalogues e-commerce sont un cas d'usage classique pour MongoDB.

#### Pourquoi MongoDB excelle ici ?

- **Attributs variables** : Chaque catégorie de produit a ses propres caractéristiques
- **Recherche avancée** : Filtres dynamiques sur n'importe quel attribut
- **Variantes** : Tailles, couleurs, options facilement gérées
- **Performance** : Requêtes rapides sur des millions de produits

#### Exemple : Produits avec attributs variables

```json
// Produit électronique
{
  "_id": ObjectId("..."),
  "sku": "LAPTOP-PRO-2024",
  "name": "Laptop Pro 15 pouces",
  "category": ["Électronique", "Ordinateurs", "Laptops"],
  "brand": "TechBrand",
  "price": {
    "amount": 1299.99,
    "currency": "EUR",
    "discount": {
      "percentage": 10,
      "validUntil": ISODate("2024-12-31")
    }
  },
  "specs": {
    "processor": "Intel Core i7-12700H",
    "ram": "16 GB DDR5",
    "storage": "512 GB NVMe SSD",
    "display": {
      "size": "15.6 pouces",
      "resolution": "2560x1440",
      "type": "IPS",
      "refreshRate": 165
    },
    "graphics": "NVIDIA RTX 4060",
    "battery": "72 Wh",
    "weight": "1.8 kg"
  },
  "inventory": {
    "quantity": 45,
    "warehouse": "Paris-Est",
    "restockDate": ISODate("2024-12-15")
  },
  "images": [
    { "url": "https://cdn.shop.com/laptop-front.jpg", "main": true },
    { "url": "https://cdn.shop.com/laptop-side.jpg", "main": false }
  ],
  "ratings": {
    "average": 4.5,
    "count": 234
  }
}

// Vêtement (structure différente)
{
  "_id": ObjectId("..."),
  "sku": "TSHIRT-BIO-BLU-M",
  "name": "T-shirt Bio Coton",
  "category": ["Mode", "Homme", "T-shirts"],
  "brand": "EcoWear",
  "price": {
    "amount": 29.99,
    "currency": "EUR"
  },
  "specs": {
    "material": "100% coton bio",
    "care": ["Lavage 30°", "Pas de sèche-linge"],
    "origin": "Portugal"
  },
  "variants": [
    { "color": "Bleu", "size": "S", "sku": "TSHIRT-BIO-BLU-S", "stock": 12 },
    { "color": "Bleu", "size": "M", "sku": "TSHIRT-BIO-BLU-M", "stock": 25 },
    { "color": "Bleu", "size": "L", "sku": "TSHIRT-BIO-BLU-L", "stock": 18 },
    { "color": "Vert", "size": "M", "sku": "TSHIRT-BIO-GRN-M", "stock": 8 }
  ],
  "sizing": {
    "fit": "Regular",
    "sizeGuide": "https://shop.com/size-guide"
  }
}
```

#### Entreprises utilisant MongoDB pour l'e-commerce

- **eBay** : Recherche et catalogue
- **Bosch** : Catalogue de pièces industrielles
- **OTTO** : E-commerce allemand majeur

---

### 4. Internet des Objets (IoT)

MongoDB est parfaitement adapté à la collecte et l'analyse de données IoT.

#### Pourquoi MongoDB excelle ici ?

- **Volume de données** : Gestion de milliards de points de données
- **Time Series** : Collections optimisées pour les données temporelles
- **Schéma variable** : Différents types de capteurs
- **Agrégations** : Analyse en temps réel

#### Exemple : Données de capteurs

```json
// Données d'un capteur environnemental
{
  "_id": ObjectId("..."),
  "deviceId": "SENSOR-ENV-001",
  "location": {
    "type": "Point",
    "coordinates": [2.3522, 48.8566],
    "building": "Entrepôt A",
    "zone": "Zone de stockage"
  },
  "timestamp": ISODate("2024-11-28T14:35:22.456Z"),
  "readings": {
    "temperature": 22.5,
    "humidity": 45.2,
    "co2": 412,
    "pressure": 1013.25
  },
  "status": "normal",
  "battery": 87
}

// Données d'un véhicule connecté
{
  "_id": ObjectId("..."),
  "vehicleId": "VH-2024-00456",
  "timestamp": ISODate("2024-11-28T14:35:22.456Z"),
  "position": {
    "type": "Point",
    "coordinates": [2.2945, 48.8584]
  },
  "speed": 45.5,
  "heading": 127,
  "engine": {
    "rpm": 2400,
    "temperature": 92,
    "fuelLevel": 68
  },
  "diagnostics": {
    "codes": [],
    "tirePressure": [2.2, 2.2, 2.1, 2.1]
  }
}
```

#### Collections Time Series (MongoDB 5.0+)

```javascript
// Création d'une collection time series optimisée
db.createCollection("sensorData", {
  timeseries: {
    timeField: "timestamp",
    metaField: "deviceId",
    granularity: "seconds"
  },
  expireAfterSeconds: 2592000  // 30 jours
})
```

---

### 5. Applications de gaming

L'industrie du jeu vidéo utilise massivement MongoDB.

#### Pourquoi MongoDB excelle ici ?

- **Profils joueurs** : Progression, inventaire, achievements
- **Données en temps réel** : Leaderboards, statistiques
- **Haute disponibilité** : Joueurs connectés 24/7
- **Scalabilité** : Pics de charge lors des événements

#### Exemple : Profil de joueur

```json
{
  "_id": ObjectId("..."),
  "gamertag": "DragonSlayer42",
  "account": {
    "email": "player@example.com",
    "created": ISODate("2023-01-15"),
    "lastLogin": ISODate("2024-11-28T20:30:00Z"),
    "totalPlaytime": 145200  // secondes
  },
  "character": {
    "name": "Aldric",
    "class": "Warrior",
    "level": 47,
    "experience": 125000,
    "stats": {
      "strength": 85,
      "agility": 42,
      "intelligence": 30,
      "vitality": 78
    },
    "equipment": {
      "weapon": { "id": "sword_legendary_001", "name": "Épée du Dragon", "damage": 450 },
      "armor": { "id": "armor_rare_015", "name": "Armure de Mithril", "defense": 280 },
      "accessories": [
        { "id": "ring_epic_003", "name": "Anneau de Force", "bonus": "+15 STR" }
      ]
    }
  },
  "inventory": [
    { "itemId": "potion_health", "quantity": 25 },
    { "itemId": "potion_mana", "quantity": 12 },
    { "itemId": "material_iron", "quantity": 150 }
  ],
  "achievements": [
    { "id": "first_boss", "name": "Tueur de boss", "unlockedAt": ISODate("2023-02-01") },
    { "id": "level_25", "name": "Aventurier confirmé", "unlockedAt": ISODate("2023-04-15") }
  ],
  "socialData": {
    "friends": [ObjectId("..."), ObjectId("...")],
    "guild": {
      "id": ObjectId("..."),
      "name": "Les Chevaliers de l'Aube",
      "rank": "Officer"
    }
  }
}
```

#### Exemple : Leaderboard

```json
{
  "_id": ObjectId("..."),
  "leaderboardId": "weekly_pvp_season12",
  "period": {
    "start": ISODate("2024-11-25"),
    "end": ISODate("2024-12-01")
  },
  "rankings": [
    { "rank": 1, "playerId": ObjectId("..."), "gamertag": "ProGamer99", "score": 15420, "wins": 89 },
    { "rank": 2, "playerId": ObjectId("..."), "gamertag": "DragonSlayer42", "score": 14850, "wins": 82 },
    { "rank": 3, "playerId": ObjectId("..."), "gamertag": "NightHunter", "score": 14200, "wins": 78 }
  ],
  "updatedAt": ISODate("2024-11-28T21:00:00Z")
}
```

#### Entreprises de gaming utilisant MongoDB

- **EA (Electronic Arts)** : Simpsons Tapped Out, autres jeux mobiles
- **Sega** : Infrastructure de jeux
- **Epic Games** : Fortnite (certaines données)

---

### 6. Analyse en temps réel et tableaux de bord

MongoDB est idéal pour alimenter des dashboards temps réel.

#### Pourquoi MongoDB excelle ici ?

- **Agrégations puissantes** : Calculs complexes côté base de données
- **Change Streams** : Notifications en temps réel
- **Vues matérialisées** : Données pré-calculées
- **Performance** : Réponses rapides pour les dashboards

#### Exemple : Métriques d'application

```json
{
  "_id": ObjectId("..."),
  "timestamp": ISODate("2024-11-28T14:00:00Z"),
  "granularity": "hourly",
  "service": "api-gateway",
  "metrics": {
    "requests": {
      "total": 125000,
      "success": 123500,
      "errors": 1500,
      "byEndpoint": {
        "/api/users": 45000,
        "/api/products": 38000,
        "/api/orders": 42000
      }
    },
    "latency": {
      "p50": 45,
      "p95": 180,
      "p99": 450,
      "max": 2300
    },
    "errorBreakdown": {
      "400": 800,
      "401": 150,
      "500": 450,
      "503": 100
    }
  },
  "resources": {
    "cpu": { "avg": 45.2, "max": 78.5 },
    "memory": { "avg": 62.1, "max": 71.3 },
    "connections": { "avg": 450, "max": 890 }
  }
}
```

---

### 7. Gestion des logs et événements

MongoDB est fréquemment utilisé pour centraliser les logs applicatifs.

#### Pourquoi MongoDB excelle ici ?

- **Schéma flexible** : Chaque type de log peut avoir sa structure
- **TTL automatique** : Expiration automatique des anciens logs
- **Recherche** : Requêtes puissantes sur les logs
- **Volume** : Gestion de gros volumes d'écriture

#### Exemple : Log applicatif

```json
{
  "_id": ObjectId("..."),
  "timestamp": ISODate("2024-11-28T14:35:22.456Z"),
  "level": "error",
  "service": "payment-service",
  "environment": "production",
  "host": "prod-payment-03",
  "message": "Payment processing failed",
  "context": {
    "orderId": "ORD-2024-123456",
    "userId": ObjectId("..."),
    "amount": 149.99,
    "currency": "EUR",
    "paymentMethod": "credit_card"
  },
  "error": {
    "type": "PaymentGatewayError",
    "code": "INSUFFICIENT_FUNDS",
    "message": "Card declined: insufficient funds",
    "stack": "PaymentGatewayError: Card declined...\n    at processPayment..."
  },
  "request": {
    "id": "req-abc123",
    "method": "POST",
    "path": "/api/payments",
    "duration": 2340
  }
}
```

#### Configuration TTL pour les logs

```javascript
// Les logs expirent après 30 jours
db.logs.createIndex(
  { "timestamp": 1 },
  { expireAfterSeconds: 2592000 }
)
```

---

### 8. Applications financières et fintech

Depuis l'arrivée des transactions ACID, MongoDB est adapté aux applications financières.

#### Pourquoi MongoDB excelle ici ?

- **Transactions ACID** : Garanties de cohérence depuis MongoDB 4.0
- **Audit trail** : Historique complet des opérations
- **Performance** : Traitement rapide des transactions
- **Flexibilité** : Produits financiers variés

#### Exemple : Transaction financière

```json
{
  "_id": ObjectId("..."),
  "transactionId": "TXN-2024-789456",
  "type": "transfer",
  "status": "completed",
  "timestamp": ISODate("2024-11-28T14:35:22.456Z"),
  "parties": {
    "sender": {
      "accountId": "ACC-001234",
      "name": "Jean Dupont",
      "iban": "FR76..."
    },
    "receiver": {
      "accountId": "ACC-005678",
      "name": "Marie Martin",
      "iban": "FR76..."
    }
  },
  "amount": {
    "value": 500.00,
    "currency": "EUR"
  },
  "fees": {
    "value": 0.50,
    "currency": "EUR"
  },
  "reference": "Remboursement dîner",
  "audit": {
    "initiatedBy": "mobile-app",
    "ipAddress": "192.168.1.100",
    "deviceId": "device-abc123",
    "approvedAt": ISODate("2024-11-28T14:35:25.000Z")
  }
}
```

#### Entreprises fintech utilisant MongoDB

- **Morgan Stanley** : Gestion de données financières
- **Barclays** : Applications bancaires
- **Stripe** : Infrastructure de paiement
- **Square** : Solutions de paiement

---

### 9. Personnalisation et recommandations

MongoDB est excellent pour les systèmes de recommandation.

#### Pourquoi MongoDB excelle ici ?

- **Profils riches** : Historique utilisateur complet
- **Requêtes flexibles** : Filtres multiples pour les recommandations
- **Agrégations** : Calcul de similarités et scores
- **Temps réel** : Mise à jour instantanée des préférences

#### Exemple : Profil de préférences utilisateur

```json
{
  "_id": ObjectId("..."),
  "userId": ObjectId("..."),
  "preferences": {
    "categories": [
      { "name": "Science-Fiction", "score": 0.85 },
      { "name": "Thriller", "score": 0.72 },
      { "name": "Documentaire", "score": 0.65 }
    ],
    "actors": [
      { "name": "Tom Hanks", "score": 0.90 },
      { "name": "Meryl Streep", "score": 0.78 }
    ],
    "directors": [
      { "name": "Christopher Nolan", "score": 0.95 }
    ]
  },
  "history": {
    "watched": [
      { "contentId": ObjectId("..."), "watchedAt": ISODate("..."), "completion": 1.0, "rating": 5 },
      { "contentId": ObjectId("..."), "watchedAt": ISODate("..."), "completion": 0.7, "rating": null }
    ],
    "searches": ["space movies", "thriller 2024", "best documentaries"],
    "wishlisted": [ObjectId("..."), ObjectId("...")]
  },
  "updatedAt": ISODate("2024-11-28T14:00:00Z")
}
```

---

## Critères de décision

### Quand choisir MongoDB ✅

| Critère | Explication |
|---------|-------------|
| **Schéma évolutif** | Votre modèle de données change fréquemment |
| **Données semi-structurées** | Attributs variables selon les entités |
| **Documents autonomes** | Les données forment des unités logiques complètes |
| **Scalabilité horizontale** | Besoin de distribuer les données sur plusieurs serveurs |
| **Développement agile** | Itérations rapides, MVP, startups |
| **Haute disponibilité** | Tolérance aux pannes requise |
| **Données géospatiales** | Requêtes de localisation |
| **Time series** | Données IoT, métriques, logs |
| **Temps réel** | Change streams, notifications |
| **Big data** | Volumes importants de données |

### Quand hésiter ou éviter MongoDB ⚠️

| Critère | Explication | Alternative suggérée |
|---------|-------------|---------------------|
| **Relations complexes** | Nombreuses jointures many-to-many | PostgreSQL, MySQL |
| **Transactions complexes** | Transactions impliquant de nombreuses collections | Base relationnelle |
| **Reporting BI traditionnel** | Requêtes ad-hoc complexes, cubes OLAP | Data warehouse |
| **Données tabulaires simples** | Feuilles de calcul, données très structurées | PostgreSQL, SQLite |
| **Contraintes strictes** | Intégrité référentielle absolue | Base relationnelle |
| **Legacy SQL** | Intégration avec systèmes SQL existants | Garder SQL |
| **Graphes complexes** | Réseaux sociaux, recommandations avancées | Neo4j, Neptune |
| **Cache simple** | Données éphémères clé-valeur | Redis, Memcached |

---

## Arbre de décision simplifié

```
                    ┌─────────────────────────┐
                    │ Quel type de données ?  │
                    └───────────┬─────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
          ▼                     ▼                     ▼
    ┌───────────┐        ┌───────────┐        ┌───────────┐
    │ Tabulaires│        │ Documents │        │  Graphes  │
    │ rigides   │        │ flexibles │        │ relations │
    └─────┬─────┘        └─────┬─────┘        └─────┬─────┘
          │                    │                    │
          ▼                    ▼                    ▼
    ┌───────────┐        ┌───────────┐        ┌───────────┐
    │   SQL     │        │  MongoDB  │        │   Neo4j   │
    │PostgreSQL │        │           │        │           │
    └───────────┘        └─────┬─────┘        └───────────┘
                               │
                    ┌──────────┴──────────┐
                    │                     │
                    ▼                     ▼
            ┌─────────────┐       ┌─────────────┐
            │ Scalabilité │       │   Simple    │
            │  critique   │       │  instance   │
            └──────┬──────┘       └──────┬──────┘
                   │                     │
                   ▼                     ▼
            ┌─────────────┐       ┌─────────────┐
            │   MongoDB   │       │   MongoDB   │
            │   Sharded   │       │ Replica Set │
            └─────────────┘       └─────────────┘
```

---

## Études de cas réels

### Cas 1 : Startup SaaS B2B

**Contexte** : Application de gestion de projets pour entreprises

**Défis** :
- Fonctionnalités qui évoluent rapidement
- Chaque client a des besoins de personnalisation
- Croissance utilisateurs imprévisible

**Pourquoi MongoDB** :
- Schéma flexible pour les itérations rapides
- Documents imbriqués pour les projets/tâches/commentaires
- Scalabilité pour accompagner la croissance

**Résultat** : Time-to-market réduit de 40%, migrations de schéma simplifiées

---

### Cas 2 : Plateforme e-commerce

**Contexte** : Marketplace avec vendeurs tiers

**Défis** :
- Catalogue de millions de produits
- Attributs variables selon les catégories
- Recherche et filtres dynamiques

**Pourquoi MongoDB** :
- Schéma polymorphe pour les produits
- Index efficaces pour la recherche
- Performance en lecture

**Résultat** : Recherche 3x plus rapide, ajout de catégories sans migration

---

### Cas 3 : Application IoT industrielle

**Contexte** : Monitoring d'équipements dans des usines

**Défis** :
- 10 000 capteurs envoyant des données chaque seconde
- Rétention de données sur 1 an
- Tableaux de bord temps réel

**Pourquoi MongoDB** :
- Time Series Collections pour les métriques
- Sharding pour le volume
- Agrégations pour les dashboards

**Résultat** : 1 milliard de documents/mois, requêtes < 100ms

---

### Cas 4 : Application bancaire mobile

**Contexte** : Néobanque avec application mobile

**Défis** :
- Transactions financières critiques
- Audit et conformité réglementaire
- Haute disponibilité 24/7

**Pourquoi MongoDB** :
- Transactions ACID pour les opérations bancaires
- Documents pour l'historique complet
- Replica sets pour la haute disponibilité

**Résultat** : 99.99% de disponibilité, conformité PCI-DSS

---

## Checklist de décision

Avant de choisir MongoDB, posez-vous ces questions :

### ✅ MongoDB est probablement adapté si :

- [ ] Vos données ont une structure hiérarchique naturelle
- [ ] Le schéma va évoluer au fil du temps
- [ ] Vous avez besoin de scalabilité horizontale
- [ ] Vos données sont semi-structurées
- [ ] Vous développez une application web/mobile moderne
- [ ] Vous avez des pics de charge importants
- [ ] Vous travaillez avec des données géospatiales ou temporelles
- [ ] Vous préférez un modèle de données proche du code

### ⚠️ Reconsidérez si :

- [ ] Vos données sont purement tabulaires et rigides
- [ ] Vous avez de nombreuses relations many-to-many
- [ ] Vous avez un système legacy SQL à maintenir
- [ ] Vous avez besoin de reporting BI traditionnel
- [ ] Votre équipe n'a aucune expérience NoSQL
- [ ] Vous n'avez que des données simples clé-valeur

---

## Conclusion

MongoDB est un choix excellent pour une grande variété de cas d'usage modernes : applications web/mobile, CMS, e-commerce, IoT, gaming, analytics temps réel, et bien d'autres. Sa flexibilité, sa scalabilité et son écosystème riche en font un outil puissant pour les développeurs.

Cependant, comme tout outil, MongoDB n'est pas universel. L'important est de **choisir la base de données adaptée à votre problème**, et non de forcer votre problème à s'adapter à une technologie.

Dans de nombreux cas, une approche **polyglot** (utilisant plusieurs bases de données) peut être la meilleure solution, en combinant les forces de chaque technologie.

---

## Points clés à retenir

- MongoDB excelle pour les **applications web/mobile**, **CMS**, **e-commerce**, **IoT**, **gaming**
- La **flexibilité du schéma** est un avantage majeur pour les projets agiles
- Les **Time Series Collections** sont idéales pour l'IoT et les métriques
- Les **transactions ACID** rendent MongoDB viable pour les applications financières
- Évaluez vos **besoins réels** avant de choisir (schéma, scalabilité, relations)
- N'hésitez pas à combiner MongoDB avec d'autres bases (**polyglot persistence**)
- De nombreuses **grandes entreprises** utilisent MongoDB en production

---


⏭️ [Architecture générale de MongoDB](/01-introduction-a-mongodb/06-architecture-generale.md)
