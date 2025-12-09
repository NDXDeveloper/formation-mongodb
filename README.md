# 🍃 Formation MongoDB

![License](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-blue.svg)
![MongoDB](https://img.shields.io/badge/MongoDB-6.x%20%2F%207.x%20%2F%208.x-green.svg)
![Chapitres](https://img.shields.io/badge/Chapitres-23-orange.svg)
![Annexes](https://img.shields.io/badge/Annexes-7-yellow.svg)
![Langue](https://img.shields.io/badge/Langue-Français-blue.svg)

**Vous voulez maîtriser MongoDB ? Vous êtes au bon endroit !**

Du premier `db.collection.find()` jusqu'au cluster shardé en production — cette formation vous accompagne pas à pas, que vous soyez développeur ou DevOps.

<p align="center">
  <img src="https://www.mongodb.com/assets/images/global/leaf.png" alt="MongoDB Leaf" width="120">
</p>

---

## ✨ Ce qui vous attend

- 📚 **23 chapitres progressifs** — on commence doucement, on finit expert
- 🏗️ **10 parties thématiques** — du CRUD basique au sharding multi-régions
- 🎯 **Patterns de modélisation** — les 9 patterns qui changent tout
- ⚡ **Agrégation en profondeur** — pipelines, $lookup, $graphLookup...
- 🔒 **Sécurité complète** — authentification, chiffrement, audit
- ☁️ **MongoDB Atlas** — l'offre cloud décortiquée
- 🐳 **Docker & Kubernetes** — déploiements prêts à l'emploi
- 📖 **7 annexes de référence** — glossaire, commandes, configurations

**⏱️ Durée estimée :** 50-70 heures • **📊 Niveau :** Tous niveaux

---

## 🚀 Prêt à commencer ?

👉 **[Consultez le sommaire complet](/SOMMAIRE.md)** pour découvrir tous les chapitres

Ou choisissez votre point d'entrée :

| Vous êtes... | Commencez par | Lien direct |
|--------------|---------------|-------------|
| 🌱 **Débutant complet** | Partie 1 : Les fondamentaux | [Chapitre 1 : Introduction](/partie-01-introduction-concepts-fondamentaux/01-introduction-a-mongodb/README.md) |
| 🔧 **Dev qui connaît le CRUD** | Partie 2 : Modélisation | [Chapitre 4 : Patterns](/partie-02-modelisation-conception/04-modelisation-des-donnees/README.md) |
| ⚙️ **DevOps / SRE** | Partie 4 : Architecture distribuée | [Chapitre 9 : Réplication](/partie-04-architecture-distribuee/09-replication/README.md) |
| 🎯 **Pressé** | L'essentiel en 10h | [Parcours express](#%EF%B8%8F-parcours-sugg%C3%A9r%C3%A9s) |

---

## 📚 Aperçu du contenu

### Les 10 parties

| # | Partie | Vous apprendrez à... |
|---|--------|---------------------|
| 1 | **Fondamentaux** | Installer, configurer, faire vos premières requêtes |
| 2 | **Modélisation** | Concevoir des schémas efficaces, indexer intelligemment |
| 3 | **Transactions** | Gérer ACID et la concurrence |
| 4 | **Architecture distribuée** | Mettre en place Replica Sets et Sharding |
| 5 | **Sécurité & Admin** | Sécuriser, sauvegarder, monitorer |
| 6 | **Atlas** | Utiliser MongoDB en cloud managé |
| 7 | **Développement** | Intégrer MongoDB dans vos applications |
| 8 | **Production** | Optimiser, déployer, scaler |
| 9 | **Bonnes pratiques** | Éviter les pièges, résoudre les problèmes |
| 10 | **Perspectives** | Suivre les évolutions, aller plus loin |

### Les annexes qui vous sauveront la vie

| Annexe | Quand l'utiliser |
|--------|------------------|
| 📗 **[Glossaire](/annexes/glossaire/README.md)** | "C'est quoi un chunk déjà ?" |
| 📘 **[Commandes mongosh](/annexes/commandes-mongosh/README.md)** | Aide-mémoire quotidien |
| 📙 **[Requêtes de référence](/annexes/requetes-reference/README.md)** | Copier-coller les classiques |
| 📕 **[Configurations](/annexes/configuration-reference/README.md)** | Templates prêts à l'emploi |
| 📓 **[Checklist performance](/annexes/checklist-performance/README.md)** | Avant la mise en prod |
| 🐳 **[Docker Compose](/annexes/docker-compose/README.md)** | Démarrer en 30 secondes |
| 🔧 **[Scripts](/annexes/scripts-automatisation/README.md)** | Automatiser la maintenance |

👉 **[Voir le sommaire détaillé](/SOMMAIRE.md)**

---

## 🏃 Démarrage rapide

### 1. Lancez MongoDB
```bash
# Avec Docker (30 secondes chrono)
docker run -d --name mongodb -p 27017:27017 mongo:7

# Connectez-vous
docker exec -it mongodb mongosh
```

### 2. Testez vos premières commandes
```javascript
// Créer une base
use maFormation

// Insérer un document
db.utilisateurs.insertOne({ nom: "Alice", niveau: "débutant" })

// Le retrouver
db.utilisateurs.find()

// 🎉 Vous venez de faire du MongoDB !
```

### 3. Suivez la formation
```bash
# Clonez le dépôt
git clone https://github.com/NDXDeveloper/formation-mongodb.git
cd formation-mongodb

# Ouvrez le sommaire et c'est parti !
```

👉 **[Accéder au sommaire](/SOMMAIRE.md)**

---

## 🗺️ Parcours suggérés

| Votre objectif | Chapitres | Durée | Ce que vous saurez faire |
|----------------|-----------|-------|--------------------------|
| 🌱 **Découvrir MongoDB** | 1 → 3 | ~8h | CRUD, requêtes, filtres |
| 🌿 **Devenir autonome** | 1 → 7 | ~25h | Modéliser, indexer, agréger |
| 🌳 **Maîtriser en profondeur** | 1 → 23 | ~60h | Tout, de A à Z |
| ⚡ **Parcours express** | 1-3, 5-6, 9, 14 | ~15h | L'essentiel pour être opérationnel |
| 🔧 **Focus DevOps** | 9-13, 17-18 | ~20h | Déployer et opérer en production |

> 💡 **Conseil :** Gardez un terminal `mongosh` ouvert pendant votre lecture. Testez chaque concept immédiatement !

---

## 📁 Structure du projet
```
formation-mongodb/
│
├── 📄 README.md ............... Vous êtes ici !
├── 📄 SOMMAIRE.md ............. Table des matières complète
├── 📄 LICENSE ................. CC BY-NC-SA 4.0
│
├── 📁 partie-01 à 10/ ......... Les 23 chapitres
│   └── 📁 01-introduction/
│       ├── 📄 README.md
│       ├── 📄 01-quest-ce-que-mongodb.md
│       └── ...
│
└── 📁 annexes/ ................ Références et ressources
    ├── 📁 glossaire/
    ├── 📁 commandes-mongosh/
    ├── 📁 docker-compose/
    └── ...
```

---

## ❓ Questions fréquentes

**Q : Dois-je suivre l'ordre des chapitres ?**
> Oui si vous débutez, non si vous avez déjà des bases. Le [sommaire](/SOMMAIRE.md) vous aide à naviguer.

**Q : Combien de temps pour tout parcourir ?**
> 50-70h au total. Comptez 2-3 mois à raison de 30min-1h par jour.

**Q : Il y a des exercices pratiques ?**
> Cette formation est théorique. Je recommande de pratiquer avec [MongoDB University](https://university.mongodb.com/) en parallèle.

**Q : C'est à jour avec MongoDB 8 ?**
> Oui, la formation couvre les versions 6.x, 7.x et 8.x.

**Q : Je peux l'utiliser pour enseigner ?**
> Absolument ! Licence CC BY-NC-SA 4.0 — citez la source et gardez la même licence.

---

## 📝 Licence

**Creative Commons BY-NC-SA 4.0**

- ✅ Utiliser et partager librement
- ✅ Modifier et adapter
- ✅ Attribution requise
- ❌ Pas d'usage commercial
- 🔄 Partage dans les mêmes conditions

Voir [LICENSE](/LICENSE) pour les détails.

---

## 👨‍💻 Auteur

**Nicolas DEOUX**

Cette formation est née de mes notes personnelles et de mon envie de partager. Elle n'est certainement pas parfaite — mais j'espère qu'elle vous sera utile !

- 📧 [NDXDev@gmail.com](mailto:NDXDev@gmail.com)
- 💼 [LinkedIn](https://www.linkedin.com/in/nicolas-deoux-ab295980/)
- 🐙 [GitHub](https://github.com/NDXDeveloper)

---

## 🙏 Ressources qui m'ont inspiré

- 📖 [Documentation officielle MongoDB](https://www.mongodb.com/docs/) — La référence absolue
- 🎓 [MongoDB University](https://university.mongodb.com/) — Cours gratuits excellents
- 📚 [Practical MongoDB Aggregations](https://www.practical-mongodb-aggregations.com/) — Pour devenir un pro de l'agrégation

---

<div align="center">

## 🍃 Prêt à plonger dans MongoDB ?

**[📚 Ouvrir le sommaire](/SOMMAIRE.md)** | **[🚀 Commencer par le chapitre 1](/partie-01-introduction-concepts-fondamentaux/01-introduction-a-mongodb/README.md)**

---

*Cette formation évolue. N'hésitez pas à me contacter si vous repérez des erreurs ou avez des suggestions.*

*Dernière mise à jour : Novembre 2025*

**[⬆ Retour en haut](#-formation-complète-mongodb)**

</div>
