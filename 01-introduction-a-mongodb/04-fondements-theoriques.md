🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.4 Fondements théoriques

## Introduction

Avant de plonger dans l'utilisation pratique de MongoDB, il est essentiel de comprendre les **fondements théoriques** qui sous-tendent les bases de données distribuées. Ces concepts vous aideront à comprendre pourquoi MongoDB fonctionne comme il le fait, et à faire des choix éclairés lors de la conception de vos applications.

Cette section aborde les principes fondamentaux qui gouvernent tous les systèmes de bases de données distribuées, dont MongoDB fait partie.

---

## Pourquoi ces concepts sont-ils importants ?

```
┌─────────────────────────────────────────────────────────────────────┐
│          Pourquoi comprendre les fondements théoriques ?            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   🎯 FAIRE DES CHOIX ÉCLAIRÉS                                       │
│   ───────────────────────────                                       │
│   Comprendre les compromis inhérents aux systèmes distribués        │
│   vous permet de configurer MongoDB de manière optimale pour        │
│   votre cas d'usage spécifique.                                     │
│                                                                     │
│   🔧 DIAGNOSTIQUER LES PROBLÈMES                                    │
│   ────────────────────────────                                      │
│   Quand un comportement vous semble étrange (données "en retard",   │
│   écritures refusées...), la théorie vous aide à comprendre         │
│   pourquoi et comment résoudre le problème.                         │
│                                                                     │
│   📐 CONCEVOIR DES ARCHITECTURES ROBUSTES                           │
│   ────────────────────────────────────────                          │
│   Savoir ce qui est possible et ce qui ne l'est pas vous évite      │
│   de concevoir des systèmes qui ne peuvent pas fonctionner          │
│   comme prévu.                                                      │
│                                                                     │
│   💬 COMMUNIQUER EFFICACEMENT                                       │
│   ────────────────────────────                                      │
│   Maîtriser le vocabulaire (CAP, ACID, cohérence éventuelle...)     │
│   vous permet de dialoguer avec d'autres professionnels             │
│   et de comprendre la documentation.                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Les défis des systèmes distribués

MongoDB, comme toute base de données moderne conçue pour la scalabilité, est un **système distribué**. Cela signifie que vos données peuvent être réparties sur plusieurs machines, dans plusieurs datacenters, voire sur plusieurs continents.

Cette distribution apporte de nombreux avantages :

| Avantage | Description |
|----------|-------------|
| **Haute disponibilité** | Le système reste opérationnel même si des serveurs tombent en panne |
| **Scalabilité** | Possibilité d'ajouter des serveurs pour gérer plus de données et de requêtes |
| **Performance** | Répartition de la charge et proximité géographique avec les utilisateurs |
| **Résilience** | Protection contre la perte de données |

Mais cette distribution introduit également des **défis fondamentaux** :

```
┌─────────────────────────────────────────────────────────────────────┐
│               Les défis des systèmes distribués                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌─────────┐         ┌─────────┐         ┌─────────┐               │
│   │ Serveur │◄───?───►│ Serveur │◄───?───►│ Serveur │               │
│   │    A    │         │    B    │         │    C    │               │
│   │ Paris   │         │  Lyon   │         │Marseille│               │
│   └─────────┘         └─────────┘         └─────────┘               │
│                                                                     │
│   Questions fondamentales :                                         │
│                                                                     │
│   ❓ Comment garantir que tous les serveurs ont les mêmes données ? │
│                                                                     │
│   ❓ Que se passe-t-il si la connexion entre serveurs est coupée ?  │
│                                                                     │
│   ❓ Comment éviter que deux clients modifient la même donnée       │
│      simultanément de manière conflictuelle ?                       │
│                                                                     │
│   ❓ Comment garantir qu'une transaction est soit complètement      │
│      effectuée, soit pas du tout ?                                  │
│                                                                     │
│   ❓ Faut-il privilégier la rapidité ou la sécurité des données ?   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

Ces questions n'ont pas de réponses simples. La théorie des systèmes distribués nous enseigne qu'il existe des **compromis inévitables** et que le choix dépend de vos besoins spécifiques.

---

## Les concepts que nous allons explorer

Cette section est divisée en plusieurs parties qui couvrent les fondements théoriques essentiels :

### 1.4.1 Le théorème CAP

Le **théorème CAP** (Consistency, Availability, Partition tolerance) est un résultat fondamental qui établit qu'un système distribué ne peut garantir simultanément que deux des trois propriétés suivantes :

- **C**onsistency (Cohérence) : Tous les nœuds voient les mêmes données
- **A**vailability (Disponibilité) : Le système répond toujours aux requêtes
- **P**artition tolerance (Tolérance au partitionnement) : Le système fonctionne malgré les coupures réseau

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Aperçu du théorème CAP                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                         Consistency                                 │
│                             /\                                      │
│                            /  \                                     │
│                           /    \                                    │
│                          / Choi-\                                   │
│                         /  sissez \                                 │
│                        /    deux    \                               │
│                       /──────────────\                              │
│                      /                \                             │
│                     /                  \                            │
│                    /                    \                           │
│           Partition ──────────────────── Availability               │
│           tolerance                                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

→ Nous verrons en détail ce que signifie ce théorème et ses implications pratiques.

### 1.4.2 Positionnement de MongoDB dans le théorème CAP

Comment MongoDB se positionne-t-il dans ce triangle ? Quels choix a fait MongoDB et comment pouvez-vous les ajuster ?

→ Nous explorerons les mécanismes de configuration (Write Concern, Read Concern, Read Preference) qui vous permettent d'ajuster le comportement de MongoDB.

### 1.4.3 Eventual Consistency vs Strong Consistency

La **cohérence** n'est pas binaire. Il existe un spectre allant de la cohérence forte (toutes les lectures voient la dernière écriture) à la cohérence éventuelle (les données finissent par converger).

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Spectre de la cohérence                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Cohérence                                            Cohérence    │
│   FORTE                                                ÉVENTUELLE   │
│                                                                     │
│   ◄────────────────────────────────────────────────────────────►    │
│                                                                     │
│   • Garanties maximales                    • Performance maximale   │
│   • Latence plus élevée                    • Disponibilité maximale │
│   • Idéal pour données critiques           • Idéal pour analytics   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

→ Nous comparerons ces approches et verrons quand utiliser chacune.

### 1.4.4 ACID et les transactions dans MongoDB

**ACID** est un acronyme décrivant les propriétés qui garantissent la fiabilité des transactions :

- **A**tomicity (Atomicité) : Tout ou rien
- **C**onsistency (Cohérence) : Respect des contraintes
- **I**solation : Transactions indépendantes
- **D**urability (Durabilité) : Persistance des données validées

→ Nous verrons comment MongoDB implémente ces garanties, notamment avec les transactions multi-documents introduites dans MongoDB 4.0.

---

## Prérequis pour cette section

Cette section est accessible aux débutants, mais il est utile d'avoir :

- Une compréhension basique de ce qu'est une base de données
- Une idée générale de ce que signifie "distribué" (données sur plusieurs machines)
- Lu les sections précédentes du tutoriel (recommandé mais pas obligatoire)

Ne vous inquiétez pas si ces concepts semblent abstraits au début. Nous les illustrerons avec de nombreux exemples concrets et des schémas visuels.

---

## Comment aborder cette section ?

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Conseils de lecture                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   📖 PREMIÈRE LECTURE                                               │
│   ───────────────────                                               │
│   Lisez l'ensemble pour avoir une vue d'ensemble.                   │
│   Ne vous attardez pas sur chaque détail.                           │
│   L'objectif est de comprendre les concepts généraux.               │
│                                                                     │
│   🔍 APPROFONDISSEMENT                                              │
│   ────────────────────                                              │
│   Revenez sur les sections pertinentes quand vous en aurez          │
│   besoin dans votre pratique quotidienne.                           │
│                                                                     │
│   💡 PRATIQUE                                                       │
│   ─────────                                                         │
│   Ces concepts prendront tout leur sens quand vous                  │
│   configurerez MongoDB et observerez son comportement.              │
│                                                                     │
│   🔗 RÉFÉRENCE                                                      │
│   ──────────                                                        │
│   Gardez cette section sous la main comme référence                 │
│   pour comprendre la documentation MongoDB.                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Vocabulaire clé

Avant de commencer, voici un mini-glossaire des termes que vous rencontrerez :

| Terme | Définition rapide |
|-------|-------------------|
| **Système distribué** | Système où les données sont réparties sur plusieurs machines |
| **Nœud** | Une machine (serveur) dans un système distribué |
| **Réplication** | Copie des données sur plusieurs nœuds |
| **Partition réseau** | Coupure de communication entre des nœuds |
| **Cohérence** | Fait que tous les nœuds voient les mêmes données |
| **Disponibilité** | Capacité du système à répondre aux requêtes |
| **Transaction** | Ensemble d'opérations exécutées comme une unité |
| **Latence** | Temps de réponse d'une opération |
| **Durabilité** | Garantie que les données persistées ne sont pas perdues |

---

## Ce que vous saurez à la fin de cette section

À la fin de cette section sur les fondements théoriques, vous serez capable de :

✅ Expliquer le théorème CAP et ses implications

✅ Comprendre pourquoi MongoDB est classé comme système "CP"

✅ Différencier cohérence forte et cohérence éventuelle

✅ Choisir le bon niveau de cohérence pour vos données

✅ Comprendre les garanties ACID et les transactions MongoDB

✅ Configurer MongoDB pour votre cas d'usage spécifique

✅ Dialoguer avec d'autres professionnels sur ces sujets

---

## Sommaire de la section

| Section | Titre | Description |
|---------|-------|-------------|
| 1.4.1 | [Le théorème CAP](./04.1-theoreme-cap.md) | Comprendre les compromis fondamentaux |
| 1.4.2 | [Positionnement de MongoDB dans le CAP](./04.2-positionnement-mongodb-cap.md) | Comment MongoDB gère ces compromis |
| 1.4.3 | [Eventual vs Strong Consistency](./04.3-eventual-vs-strong-consistency.md) | Choisir le bon niveau de cohérence |
| 1.4.4 | [ACID et les transactions](./04.4-acid-transactions.md) | Garanties transactionnelles dans MongoDB |

---

Commençons par explorer le théorème CAP, pierre angulaire de la théorie des systèmes distribués.

---


⏭️ [Le théorème CAP (Consistency, Availability, Partition tolerance)](/01-introduction-a-mongodb/04.1-theoreme-cap.md)
