🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 1 : Introduction à MongoDB

## Bienvenue dans ce tutoriel MongoDB !

Ce premier chapitre vous introduit au monde de MongoDB, une des bases de données NoSQL les plus populaires au monde. Que vous soyez développeur, administrateur système, data analyst ou simplement curieux, ce chapitre vous donnera les bases nécessaires pour comprendre ce qu'est MongoDB et pourquoi il est devenu incontournable.

---

## Objectifs de ce chapitre

À la fin de ce chapitre, vous serez capable de :

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Ce que vous apprendrez                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ✅ Comprendre ce qu'est MongoDB et sa philosophie                 │
│                                                                     │
│   ✅ Connaître l'histoire et l'évolution de MongoDB                 │
│                                                                     │
│   ✅ Différencier les bases NoSQL des bases SQL traditionnelles     │
│                                                                     │
│   ✅ Maîtriser les fondements théoriques (CAP, ACID, cohérence)     │
│                                                                     │
│   ✅ Identifier les cas d'usage appropriés pour MongoDB             │
│                                                                     │
│   ✅ Comprendre l'architecture générale de MongoDB                  │
│                                                                     │
│   ✅ Maîtriser la terminologie (documents, collections, bases)      │
│                                                                     │
│   ✅ Installer MongoDB sur votre système                            │
│                                                                     │
│   ✅ Utiliser les outils essentiels (mongosh, Compass, Atlas)       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## À qui s'adresse ce chapitre ?

Ce chapitre est conçu pour être **accessible aux débutants complets**. Aucune connaissance préalable de MongoDB n'est requise.

| Profil | Ce que vous y trouverez |
|--------|-------------------------|
| **Débutant complet** | Toutes les bases pour démarrer de zéro |
| **Développeur SQL** | Comparaisons SQL/NoSQL et équivalences |
| **Développeur expérimenté** | Fondements théoriques et bonnes pratiques |
| **Administrateur système** | Installation, configuration et architecture |

---

## Structure du chapitre

Ce chapitre est organisé en **10 sections** progressives, allant des concepts de base jusqu'à l'installation et la prise en main des outils.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Parcours de ce chapitre                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   CONCEPTS                                                          │
│   ────────                                                          │
│   1.1 Qu'est-ce que MongoDB ?                                       │
│    │                                                                │
│    ▼                                                                │
│   1.2 Historique et évolution                                       │
│    │                                                                │
│    ▼                                                                │
│   1.3 NoSQL vs SQL                                                  │
│    │                                                                │
│    ▼                                                                │
│   1.4 Fondements théoriques ──┬── 1.4.1 Théorème CAP                │
│    │                          ├── 1.4.2 MongoDB et le CAP           │
│    │                          └── 1.4.3 Eventual vs Strong          │
│    ▼                                                                │
│   1.5 Cas d'usage et critères de choix                              │
│    │                                                                │
│    ▼                                                                │
│   1.6 Architecture générale                                         │
│    │                                                                │
│    ▼                                                                │
│   1.7 Terminologie                                                  │
│                                                                     │
│   PRATIQUE                                                          │
│   ────────                                                          │
│   1.8 Installation (Windows, Linux, macOS)                          │
│    │                                                                │
│    ▼                                                                │
│   1.9 Installation via Docker                                       │
│    │                                                                │
│    ▼                                                                │
│   1.10 Outils : mongosh, Compass, Atlas                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Sommaire détaillé

### Concepts fondamentaux

| Section | Titre | Description |
|---------|-------|-------------|
| 1.1 | [Qu'est-ce que MongoDB ?](./01-quest-ce-que-mongodb.md) | Définition, caractéristiques et philosophie |
| 1.2 | [Historique et évolution](./02-historique-et-evolution.md) | De 2007 à aujourd'hui, les grandes étapes |
| 1.3 | [NoSQL vs SQL](./03-nosql-vs-sql.md) | Comparaison détaillée, avantages et inconvénients |
| 1.4 | [Fondements théoriques](./04-fondements-theoriques.md) | Concepts clés des bases distribuées |
| ↳ 1.4.1 | [Le théorème CAP](./04.1-theoreme-cap.md) | Consistency, Availability, Partition tolerance |
| ↳ 1.4.2 | [MongoDB et le CAP](./04.2-positionnement-mongodb-cap.md) | Comment MongoDB gère les compromis |
| ↳ 1.4.3 | [Eventual vs Strong Consistency](./04.3-eventual-vs-strong-consistency.md) | Choisir le bon niveau de cohérence |
| 1.5 | [Cas d'usage et choix](./05-cas-usage-et-choix.md) | Quand utiliser MongoDB ? |
| 1.6 | [Architecture générale](./06-architecture-generale.md) | Standalone, Replica Set, Sharded Cluster |
| 1.7 | [Terminologie](./07-terminologie.md) | Documents, Collections, Bases de données |

### Installation et outils

| Section | Titre | Description |
|---------|-------|-------------|
| 1.8 | [Installation native](./08-installation-mongodb.md) | Windows, Linux (Ubuntu, CentOS), macOS |
| 1.9 | [Installation Docker](./09-installation-docker.md) | Docker, Docker Compose, bonnes pratiques |
| 1.10 | [Présentation des outils](./10-presentation-outils.md) | mongosh, MongoDB Compass, MongoDB Atlas |

---

## Temps de lecture estimé

| Section | Durée estimée |
|---------|---------------|
| 1.1 - 1.3 | ~30 minutes |
| 1.4 (Fondements théoriques) | ~45 minutes |
| 1.5 - 1.7 | ~30 minutes |
| 1.8 - 1.10 (Installation) | ~45 minutes |
| **Total chapitre 1** | **~2h30** |

> 💡 **Conseil** : Vous pouvez lire les sections conceptuelles (1.1 à 1.7) d'une traite, puis passer à l'installation quand vous êtes prêt à pratiquer.

---

## Prérequis

### Connaissances requises

- **Aucune** connaissance préalable de MongoDB n'est nécessaire
- Des notions de base en informatique sont utiles mais pas obligatoires
- Une familiarité avec JSON est un plus (nous l'expliquerons si besoin)

### Pour les sections pratiques (1.8 - 1.10)

Pour suivre les sections d'installation, vous aurez besoin de :

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Prérequis techniques                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   💻 MATÉRIEL                                                       │
│   • Ordinateur avec au moins 4 Go de RAM                            │
│   • 10 Go d'espace disque disponible                                │
│   • Processeur 64 bits                                              │
│                                                                     │
│   🖥️ SYSTÈME D'EXPLOITATION (au choix)                              │
│   • Windows 10 ou 11                                                │
│   • macOS 11 (Big Sur) ou ultérieur                                 │
│   • Ubuntu 20.04+ / Debian 11+ / CentOS 7+                          │
│                                                                     │
│   🔧 OPTIONNEL (pour Docker)                                        │
│   • Docker Desktop (Windows/macOS)                                  │
│   • Docker Engine (Linux)                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Comment utiliser ce chapitre ?

### Lecture recommandée

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Parcours recommandés                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   🚀 PARCOURS RAPIDE (pressé)                                       │
│   ────────────────────────────                                      │
│   1.1 → 1.3 → 1.7 → 1.8 ou 1.9 → 1.10                               │
│   (Sautez les fondements théoriques, revenez-y plus tard)           │
│                                                                     │
│   📚 PARCOURS COMPLET (recommandé)                                  │
│   ─────────────────────────────────                                 │
│   1.1 → 1.2 → 1.3 → 1.4 → 1.5 → 1.6 → 1.7 → 1.8/1.9 → 1.10          │
│   (Suivez l'ordre pour une compréhension complète)                  │
│                                                                     │
│   🎯 PARCOURS PRATIQUE (développeur)                                │
│   ──────────────────────────────────                                │
│   1.1 → 1.3 → 1.9 (Docker) → 1.10 → 1.7                             │
│   (Installation rapide puis pratique immédiate)                     │
│                                                                     │
│   🏗️ PARCOURS ARCHITECTURE (ops/admin)                              │
│   ────────────────────────────────────                              │
│   1.1 → 1.4 → 1.6 → 1.8 → 1.10                                      │
│   (Focus sur la théorie et l'architecture)                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Conventions utilisées

Tout au long de ce tutoriel, vous rencontrerez les conventions suivantes :

| Élément | Signification |
|---------|---------------|
| `code` | Commandes, noms de fichiers, code |
| **gras** | Termes importants, concepts clés |
| *italique* | Termes anglais, emphase légère |
| 💡 | Conseil ou astuce |
| ⚠️ | Attention, point important |
| ❌ | À éviter, erreur courante |
| ✅ | Bonne pratique |

### Schémas ASCII

Nous utilisons des schémas en caractères ASCII pour illustrer les concepts :

```
┌─────────────┐
│   Exemple   │      ← Boîte de titre
├─────────────┤
│             │
│   Contenu   │      ← Zone de contenu
│             │
└─────────────┘

    │
    ▼               ← Flèche de flux

──────────────      ← Séparateur
```

---

## Après ce chapitre

Une fois ce chapitre terminé, vous serez prêt à passer au **Chapitre 2 : Fondamentaux de MongoDB**, où vous apprendrez à :

- Effectuer des opérations CRUD (Create, Read, Update, Delete)
- Utiliser les opérateurs de requête
- Travailler avec les types de données
- Maîtriser les bases de l'indexation

---

## Ressources complémentaires

En complément de ce tutoriel, vous pouvez consulter :

| Ressource | URL | Description |
|-----------|-----|-------------|
| Documentation officielle | [docs.mongodb.com](https://docs.mongodb.com) | Référence complète |
| MongoDB University | [learn.mongodb.com](https://learn.mongodb.com) | Cours gratuits en ligne |
| MongoDB Community Forums | [community.mongodb.com](https://community.mongodb.com) | Entraide communautaire |

---

## C'est parti !

Vous êtes prêt à commencer votre apprentissage de MongoDB. Rendez-vous dans la première section pour découvrir ce qu'est MongoDB et pourquoi cette base de données a révolutionné le monde du stockage de données.

---


⏭️ [Qu'est-ce que MongoDB ?](/01-introduction-a-mongodb/01-quest-ce-que-mongodb.md)
