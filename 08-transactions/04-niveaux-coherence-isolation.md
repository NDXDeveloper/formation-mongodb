🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.4 Niveaux de cohérence et d'isolation

## Introduction

Dans un système distribué comme MongoDB, la gestion de la cohérence des données et de l'isolation des opérations concurrentes représente l'un des défis architecturaux les plus complexes. Contrairement aux bases de données relationnelles traditionnelles qui s'appuient sur des mécanismes d'isolation stricts (SERIALIZABLE, REPEATABLE READ, etc.), MongoDB adopte une approche plus flexible et configurable à travers deux concepts fondamentaux : **Read Concern** et **Write Concern**.

Cette flexibilité n'est pas un compromis technique, mais une réponse architecturale délibérée aux contraintes inhérentes aux systèmes distribués. Elle permet aux développeurs de faire des choix éclairés entre cohérence, disponibilité et performance selon les besoins spécifiques de chaque opération.

## Le défi de la cohérence dans les systèmes distribués

### Rappel du théorème CAP dans le contexte transactionnel

Le théorème CAP (Consistency, Availability, Partition tolerance) prend tout son sens lorsqu'on aborde les transactions dans MongoDB. Dans un Replica Set distribué géographiquement, chaque opération de lecture ou d'écriture doit naviguer dans cet espace de compromis :

**Cohérence forte (Strong Consistency)** : Garantit que toutes les lectures retournent la dernière écriture validée. Cela implique une synchronisation entre les nœuds, ce qui augmente la latence et réduit la disponibilité en cas de partition réseau.

**Cohérence éventuelle (Eventual Consistency)** : Accepte que les lectures puissent temporairement retourner des données obsolètes, en échange d'une latence réduite et d'une meilleure disponibilité.

**Exemple concret** :

Considérons une application bancaire avec un Replica Set déployé sur trois continents (Amérique du Nord, Europe, Asie). Lorsqu'un client à New York effectue un virement :

- Avec une cohérence forte, l'application doit attendre que l'opération soit répliquée sur la majorité des nœuds avant de confirmer. Si le nœud en Asie est temporairement indisponible, l'opération peut prendre 300-500ms.
- Avec une cohérence éventuelle, la confirmation est immédiate (10-20ms), mais un client consultant son compte depuis Tokyo pourrait voir son solde mis à jour avec quelques secondes de délai.

### La notion de "données sales" (Dirty Reads)

MongoDB permet, selon la configuration, différents niveaux d'isolation qui déterminent si une opération peut voir des données non encore committées ou non encore répliquées. Cette flexibilité est cruciale pour optimiser les performances, mais elle nécessite une compréhension approfondie des implications.

**Scénario réaliste** - Système de réservation de billets d'avion :

```
T0 : Transaction A réserve le siège 12A (écriture sur le Primary)
T1 : La réplication vers les Secondaires est en cours (délai réseau)
T2 : Transaction B lit depuis un Secondary pour afficher les sièges disponibles
```

Si Transaction B utilise un niveau d'isolation faible, elle pourrait voir le siège 12A comme encore disponible, créant une double réservation potentielle. Un niveau d'isolation plus strict forcerait Transaction B à attendre la réplication ou à lire depuis le Primary.

## Vue d'ensemble des mécanismes de cohérence MongoDB

MongoDB offre deux leviers principaux pour contrôler la cohérence et l'isolation :

### Read Concern : Contrôler ce qu'on lit

Le **Read Concern** détermine le niveau de garantie sur les données lues. Il répond à la question : "Quelle version des données suis-je autorisé à voir ?"

Les niveaux disponibles forment un spectre allant de la performance maximale à la cohérence maximale. Chaque niveau offre un équilibre différent entre :

- **Fraîcheur des données** : À quel point les données sont récentes
- **Durabilité** : Le risque que les données lues soient annulées (rollback)
- **Latence** : Le temps d'attente pour obtenir la réponse
- **Disponibilité** : La capacité à répondre même en cas de problèmes réseau

### Write Concern : Contrôler ce qu'on écrit

Le **Write Concern** définit le niveau d'accusé de réception requis pour considérer une écriture comme réussie. Il répond à : "Combien de nœuds doivent avoir persisté mes données avant que je puisse continuer ?"

Ce mécanisme est fondamental pour équilibrer :

- **Durabilité** : La probabilité que les données survivent à une panne
- **Performance d'écriture** : Le temps nécessaire pour confirmer l'opération
- **Disponibilité en écriture** : La capacité à écrire même si certains nœuds sont indisponibles

### L'interaction critique entre Read et Write Concern

L'aspect le plus subtil de la cohérence dans MongoDB est l'interaction entre ces deux mécanismes. Un Read Concern strict ne garantit rien si le Write Concern est laxiste, et vice-versa.

**Exemple d'interaction problématique** :

```
Configuration A :
- Write Concern: w=1 (confirmation après écriture sur le Primary uniquement)
- Read Concern: majority (lecture des données répliquées sur la majorité)

Problème : Une écriture peut être confirmée mais non visible en lecture si le Primary
tombe en panne avant réplication.
```

```
Configuration B :
- Write Concern: w=majority (confirmation après réplication sur la majorité)
- Read Concern: local (lecture depuis n'importe quel nœud, même non répliqué)

Problème : Une lecture peut retourner des données plus anciennes que des écritures
déjà confirmées.
```

## Compromis fondamentaux et décisions architecturales

### Le trilemme Cohérence-Performance-Disponibilité

Chaque choix de niveau de cohérence implique des compromis qu'il est essentiel de quantifier :

**Cas 1 : Application de trading financier**

Exigences :
- Cohérence absolue (aucune transaction ne doit être perdue)
- Isolation stricte (éviter les conditions de course)
- Audit complet

Configuration appropriée :
- Write Concern: `{ w: "majority", j: true, wtimeout: 5000 }`
- Read Concern: `"linearizable"` pour les lectures critiques, `"snapshot"` pour les transactions

Compromis acceptés :
- Latence d'écriture : 50-200ms selon la topologie
- Latence de lecture : 20-100ms supplémentaires
- Indisponibilité en cas de perte de majorité du Replica Set

**Cas 2 : Système de logs d'application**

Exigences :
- Volume élevé (millions d'écritures/seconde)
- Tolérance à la perte occasionnelle
- Latence minimale

Configuration appropriée :
- Write Concern: `{ w: 1, j: false }` ou même `{ w: 0 }` pour les logs non critiques
- Read Concern: `"local"` ou `"available"`

Compromis acceptés :
- Risque de perte de données en cas de crash (fenêtre de 100ms typiquement)
- Lectures potentiellement obsolètes
- Gain : latence d'écriture < 1ms, throughput maximal

**Cas 3 : Réseau social - Fil d'actualité**

Exigences :
- Scalabilité massive en lecture
- Tolérance à la latence éventuelle
- Haute disponibilité

Configuration appropriée :
- Write Concern: `{ w: "majority" }` pour les publications
- Read Concern: `"available"` pour le fil d'actualité (accepte la cohérence éventuelle)

Compromis acceptés :
- Un utilisateur peut ne pas voir immédiatement sa propre publication
- Des utilisateurs différents peuvent voir le fil dans un ordre légèrement différent
- Gain : latence de lecture très faible, distribution géographique optimale

### Cohérence causale : Un modèle hybride

MongoDB introduit également la notion de **cohérence causale** qui offre un modèle intermédiaire élégant. Elle garantit que si une opération B lit des données modifiées par une opération A, alors B verra également toutes les modifications antérieures à A.

**Exemple pratique** - Système de commentaires :

```
Session causale activée :

1. Alice publie un article (écriture A)
2. Bob commente l'article (écriture B, dépend de A)
3. Charlie lit les commentaires (lecture C)

Garantie : Si C voit le commentaire de Bob, elle verra forcément l'article d'Alice,
même si la lecture se fait depuis un Secondary avec un léger retard de réplication.
```

Sans cohérence causale, Charlie pourrait voir un commentaire orphelin référençant un article pas encore visible.

## Considérations pour les architectures modernes

### Microservices et cohérence distribuée

Dans une architecture microservices, chaque service peut avoir ses propres exigences de cohérence :

**Service d'authentification** :
- Write Concern: `majority + journal`
- Read Concern: `majority`
- Rationale : La sécurité prime sur la performance

**Service de recherche/catalogue** :
- Write Concern: `majority`
- Read Concern: `available` ou `local`
- Rationale : La fraîcheur immédiate n'est pas critique

**Service de panier d'achat** :
- Write Concern: `majority`
- Read Concern: `majority` ou `snapshot` en transaction
- Rationale : Équilibre entre cohérence et expérience utilisateur

### Impact sur les patterns d'architecture

Le choix des niveaux de cohérence influence directement les patterns d'architecture applicables :

**Event Sourcing avec MongoDB** :
- Nécessite `Write Concern: majority` pour garantir la durabilité des événements
- `Read Concern: majority` ou `snapshot` pour reconstruire l'état de manière cohérente

**CQRS (Command Query Responsibility Segregation)** :
- Command side : Write Concern strict (`majority`)
- Query side : Read Concern flexible (`available` ou `local`) depuis des Secondaires optimisés

**Cache-aside pattern** :
- L'invalidation du cache doit tenir compte de la cohérence éventuelle
- Risque de race condition si le cache est invalidé avant la réplication

## Métriques et observabilité

Pour prendre des décisions éclairées, il est crucial de mesurer l'impact réel des différents niveaux :

**Métriques clés à surveiller** :

1. **Replication Lag** :
   - Indique le retard entre Primary et Secondaries
   - Impact direct sur `Read Concern: majority`
   - Objectif typique : < 1 seconde en conditions normales

2. **Write Latency P99** :
   - Latence au 99ème percentile pour les écritures
   - Compare `w:1` vs `w:majority` sur votre topologie réelle
   - Exemple : 5ms vs 45ms sur un Replica Set intercontinental

3. **Read Latency par Read Concern** :
   - Mesure séparée pour `local`, `majority`, `linearizable`
   - Permet d'identifier les requêtes pénalisantes

4. **Rollback Events** :
   - Fréquence des rollbacks suite à des élections
   - Indique si `w:1` cause des pertes de données

**Dashboard de décision** :

```
Métriques actuelles (dernières 24h) :
- Replication Lag P99: 850ms
- Write Latency w:1: 12ms (P99)
- Write Latency w:majority: 95ms (P99)
- Read Latency local: 8ms (P99)
- Read Latency majority: 35ms (P99)
- Rollback events: 0

Recommandation : La latence de réplication élevée pénalise lourdement
les opérations avec Read/Write Concern majority. Envisager :
1. Optimisation réseau entre datacenters
2. Topologie avec plus de membres locaux
3. Réévaluation des exigences de cohérence stricte
```

## Évolution des exigences dans le temps

Un aspect souvent négligé : les exigences de cohérence évoluent avec la maturité du produit et la croissance de la base utilisateurs.

**Phase 1 : MVP (Minimum Viable Product)**
- Configuration : Cohérence forte partout (`w:majority`, `readConcern:majority`)
- Rationale : Simplicité, peu d'utilisateurs, debugging facilité

**Phase 2 : Croissance**
- Configuration : Différenciation par type d'opération
- Challenge : Identifier les opérations qui peuvent tolérer la cohérence éventuelle

**Phase 3 : Scale**
- Configuration : Stratégie sophistiquée avec sessions causales, Read Preference, Tagged Read
- Exemple : Lectures depuis Secondaries géographiquement proches, écritures avec `w:majority`

**Phase 4 : Global Scale**
- Configuration : Sharding avec Zone Awareness, compromis régionaux
- Exemple : Les utilisateurs européens tolèrent une cohérence éventuelle pour voir les contenus américains

## Antipatterns courants

### Antipattern 1 : "Majority partout par défaut"

**Problème** : Appliquer `w:majority` et `readConcern:majority` sans réflexion.

**Conséquence** : Performance dégradée inutilement, latence utilisateur élevée.

**Solution** : Analyse fine par endpoint/opération.

### Antipattern 2 : "Cohérence éventuelle sans gestion des conflicts"

**Problème** : Utiliser `w:1` et `local` reads sans mécanisme de réconciliation.

**Conséquence** : Conditions de course, données incohérentes visibles par les utilisateurs.

**Solution** : Versioning optimiste, timestamps, ou réévaluation des exigences.

### Antipattern 3 : "Ignorer le Replication Lag"

**Problème** : Ne pas monitorer le délai de réplication.

**Conséquence** : `readConcern:majority` devient très lent sans que l'équipe ne comprenne pourquoi.

**Solution** : Alertes sur Replication Lag, investigation réseau/matériel.

### Antipattern 4 : "Mélanger Write Concern faible et Read Concern fort"

**Problème** : `w:1` avec `readConcern:majority`.

**Conséquence** : Écritures confirmées mais invisible aux lectures strictes, confusion sur l'état du système.

**Solution** : Aligner Write et Read Concern selon les garanties souhaitées.

## Checklist de décision pour les niveaux de cohérence

Avant de choisir vos niveaux de Read et Write Concern, posez-vous ces questions :

**Questions sur les exigences métier** :

- [ ] Quelle est la criticité de cette opération pour le business ?
- [ ] Quelle est la tolérance à la perte de données (RPO) ?
- [ ] Quelle est la tolérance à l'incohérence temporaire ?
- [ ] Les utilisateurs peuvent-ils détecter des incohérences ?
- [ ] Y a-t-il des exigences réglementaires (SOX, RGPD, etc.) ?

**Questions sur l'architecture** :

- [ ] Quelle est la topologie du Replica Set (single DC, multi-DC, global) ?
- [ ] Quel est le Replication Lag typique observé ?
- [ ] Quelles sont les latences réseau entre les membres ?
- [ ] L'application utilise-t-elle des transactions multi-documents ?
- [ ] Y a-t-il des reads depuis les Secondaries ?

**Questions sur la performance** :

- [ ] Quel est le volume d'opérations par seconde attendu ?
- [ ] Quelle est la latence maximale acceptable (P99) ?
- [ ] L'application est-elle plus read-heavy ou write-heavy ?
- [ ] Y a-t-il des pics de charge prévisibles ?

**Questions sur la résilience** :

- [ ] Que se passe-t-il si un membre du Replica Set tombe ?
- [ ] Que se passe-t-il si la majorité des membres est indisponible ?
- [ ] L'application peut-elle tolérer une indisponibilité temporaire en écriture ?
- [ ] Y a-t-il un mécanisme de retry dans l'application ?

## Conclusion intermédiaire

Les niveaux de cohérence et d'isolation dans MongoDB ne sont pas de simples paramètres techniques, mais des leviers architecturaux majeurs qui déterminent le comportement de votre système sous charge, en cas de panne, et lors de la mise à l'échelle. Ils incarnent les compromis fondamentaux des systèmes distribués et nécessitent une compréhension approfondie tant des besoins métier que des contraintes techniques.

Dans les sections suivantes, nous explorerons en détail chaque niveau de Read Concern, Write Concern, et leurs interactions spécifiques, avec des exemples de configuration et des patterns d'utilisation concrets.

---

**Concepts clés à retenir** :

- Read Concern et Write Concern sont les deux leviers de cohérence dans MongoDB
- Chaque niveau représente un compromis entre cohérence, performance et disponibilité
- Les exigences varient selon le type d'opération et le contexte métier
- La cohérence causale offre un modèle hybride intéressant pour de nombreux cas d'usage
- Le monitoring est essentiel pour valider et ajuster vos choix
- Les antipatterns sont fréquents et coûteux : appliquer une réflexion systématique

**Prochaines sections** : Nous détaillerons les niveaux spécifiques de Read Concern, Write Concern, et leurs compromis performance/cohérence avec des exemples de configuration concrets.

⏭️ [Read Concern (local, available, majority, linearizable, snapshot)](/08-transactions/04.1-read-concern.md)
