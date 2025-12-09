🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.5 Gaming et Leaderboards

## Introduction

Les applications de gaming présentent des défis uniques en termes de performance, concurrence, et expérience utilisateur :

- **Latence ultra-faible** : Leaderboards temps réel (< 50ms)
- **Concurrence élevée** : Milliers de mises à jour de scores simultanées
- **Consistance critique** : Pas de double comptage ou scores perdus
- **Scalabilité massive** : De milliers à millions de joueurs actifs
- **Données temporelles** : Sessions de jeu, historiques, statistiques
- **Matchmaking intelligent** : Appariement par niveau/compétence
- **Prévention de triche** : Validation et détection d'anomalies
- **Engagement** : Achievements, progression, récompenses
- **Social** : Amis, guildes, classements sociaux

MongoDB excelle dans ce contexte grâce à :
- **Opérations atomiques** : Incréments thread-safe pour scores
- **Index performants** : Tri rapide pour classements
- **Agrégations puissantes** : Statistiques et analytics temps réel
- **Schéma flexible** : Différents types de jeux et métriques
- **Sharding** : Distribution de charge pour millions de joueurs
- **TTL** : Nettoyage automatique de sessions expirées

## Architecture de référence

### Stack gaming moderne

```
┌─────────────────────────────────────────────────────┐
│              Game Clients                           │
│      Mobile • PC • Console • Web                    │
└────────────────────┬────────────────────────────────┘
                     │
                     │ WebSocket / UDP
                     │
        ┌────────────▼────────────┐
        │    API Gateway          │
        │   (Rate Limiting)       │
        └────────────┬────────────┘
                     │
     ┌───────────────┼──────────────┐
     │               │              │
┌────▼─────┐   ┌────▼────┐   ┌──────▼────────┐
│  Game    │   │  Match  │   │  Leaderboard  │
│  Server  │   │  Making │   │   Service     │
└────┬─────┘   └────┬────┘   └──────┬────────┘
     │              │               │
     │         ┌────▼────┐          │
     │         │  Redis  │←─────────┤
     │         │ (Cache) │          │
     │         └─────────┘          │
     │                              │
     └────────────┬─────────────────┘
                  │
     ┌────────────▼────────────┐
     │   MongoDB Cluster       │
     │  ┌──────────────────┐   │
     │  │  Shard 1         │   │
     │  │  Players (A-M)   │   │
     │  └──────────────────┘   │
     │  ┌──────────────────┐   │
     │  │  Shard 2         │   │
     │  │  Players (N-Z)   │   │
     │  └──────────────────┘   │
     │  ┌──────────────────┐   │
     │  │  Config Servers  │   │
     │  └──────────────────┘   │
     └─────────────────────────┘
                  │
     ┌────────────┼────────────┐
     │            │            │
┌────▼────┐  ┌────▼─────┐  ┌───▼──────┐
│Analytics│  │Anti-Cheat│  │ Rewards  │
│ Engine  │  │  System  │  │  System  │
└─────────┘  └──────────┘  └──────────┘
```

### Composants architecturaux

#### 1. Game Server
**Responsabilités :**
- Validation des actions de jeu
- Calcul des scores et récompenses
- État de session en temps réel
- Anti-triche côté serveur

#### 2. Leaderboard Service
**Caractéristiques :**
- Lecture ultra-rapide (< 50ms)
- Mise à jour atomique de scores
- Cache multi-niveaux
- Invalidation intelligente

#### 3. Matchmaking Service
**Fonctions :**
- Appariement par skill rating
- File d'attente avec timeout
- Balance des équipes
- Prévention de manipulation

#### 4. Cache Layer (Redis)
**Usage :**
- Top 100 de chaque leaderboard
- Sessions actives
- Matchmaking queues
- Rate limiting

## Modélisation des données

### 1. Profils de joueurs

```javascript
// Collection: players
{
  _id: ObjectId("..."),
  userId: ObjectId("..."),  // Référence vers users

  // Identité joueur
  profile: {
    username: "ProGamer123",
    displayName: "Pro Gamer",
    avatar: "https://cdn.example.com/avatars/progamer123.jpg",
    level: 42,
    xp: 125680,
    xpToNextLevel: 150000,

    // Prestige/Rank
    rank: "Diamond",
    rankTier: 3,
    rankPoints: 2847,

    // Badge et réalisations
    title: "Legend Slayer",
    badges: ["first_win", "streak_10", "tournament_winner"],

    // Personnalisation
    customization: {
      skin: "dragon_knight",
      emote: "victory_dance",
      banner: "golden_crown"
    }
  },

  // Statistiques globales
  stats: {
    totalGames: 1547,
    wins: 892,
    losses: 655,
    winRate: 0.577,  // 57.7%

    // Performance
    avgScore: 2847,
    highestScore: 9847,
    totalScore: 4402209,

    // Temps de jeu
    totalPlayTime: 2847000,  // secondes
    avgSessionDuration: 1840,

    // Combats
    kills: 15678,
    deaths: 8934,
    assists: 7823,
    kda: 2.63,  // (kills + assists) / deaths

    // Autres métriques spécifiques au jeu
    accuracy: 0.68,
    headshots: 3456,
    criticalHits: 5678
  },

  // Statistiques par mode de jeu
  statsByMode: {
    "ranked": {
      games: 847,
      wins: 492,
      losses: 355,
      winRate: 0.581,
      avgScore: 3124
    },
    "casual": {
      games: 500,
      wins: 280,
      losses: 220,
      winRate: 0.56,
      avgScore: 2456
    },
    "tournament": {
      games: 200,
      wins: 120,
      losses: 80,
      winRate: 0.60,
      avgScore: 3567
    }
  },

  // Classements
  rankings: {
    global: {
      rank: 1247,
      percentile: 95.2,  // Top 5%
      lastUpdated: ISODate("2024-12-09T14:30:00Z")
    },
    regional: {
      region: "EU_WEST",
      rank: 342,
      percentile: 98.1,
      lastUpdated: ISODate("2024-12-09T14:30:00Z")
    },
    friends: {
      rank: 3,
      totalFriends: 45,
      lastUpdated: ISODate("2024-12-09T14:30:00Z")
    }
  },

  // Progression
  progression: {
    seasonLevel: 42,
    seasonXP: 125680,

    // Battle Pass
    battlePass: {
      tier: 35,
      xp: 35000,
      isPremium: true,
      unlockedRewards: [1, 2, 3, 5, 10, 15, 20, 25, 30, 35]
    },

    // Achievements
    achievements: [
      {
        id: "first_win",
        name: "First Victory",
        unlockedAt: ISODate("2024-01-15T10:30:00Z"),
        progress: 1,
        maxProgress: 1
      },
      {
        id: "win_streak_10",
        name: "Unstoppable",
        unlockedAt: ISODate("2024-03-22T18:45:00Z"),
        progress: 10,
        maxProgress: 10
      },
      {
        id: "level_50",
        name: "Veteran",
        unlockedAt: null,  // En cours
        progress: 42,
        maxProgress: 50
      }
    ],

    // Quêtes actives
    activeQuests: [
      {
        id: "daily_win_3",
        name: "Win 3 Games",
        progress: 2,
        target: 3,
        reward: { xp: 500, coins: 100 },
        expiresAt: ISODate("2024-12-10T00:00:00Z")
      }
    ]
  },

  // Économie
  economy: {
    coins: 15680,
    gems: 234,  // Premium currency

    // Inventaire
    inventory: [
      {
        itemId: "legendary_sword",
        quantity: 1,
        acquiredAt: ISODate("2024-11-15T12:00:00Z")
      },
      {
        itemId: "health_potion",
        quantity: 15,
        acquiredAt: ISODate("2024-12-05T18:30:00Z")
      }
    ],

    // Historique transactions (limité)
    recentTransactions: [
      {
        type: "purchase",
        itemId: "legendary_sword",
        cost: 5000,
        currency: "coins",
        timestamp: ISODate("2024-11-15T12:00:00Z")
      }
    ]
  },

  // Social
  social: {
    friends: [ObjectId("friend_1"), ObjectId("friend_2")],
    friendRequests: [],

    guildId: ObjectId("guild_123"),
    guildRole: "member",
    guildJoinedAt: ISODate("2024-06-01T00:00:00Z"),

    // Bloqués
    blockedPlayers: []
  },

  // Préférences
  settings: {
    privacy: "friends",  // public, friends, private
    allowFriendRequests: true,
    showOnlineStatus: true,

    // Notifications
    notifications: {
      friendOnline: true,
      guildInvites: true,
      achievements: true
    }
  },

  // Anti-triche
  security: {
    suspicionScore: 0,  // 0-100, >80 = review
    reportsReceived: 0,
    lastReportedAt: null,

    // Historique de sanctions
    penalties: [],

    // Fingerprint device
    deviceIds: ["device_abc123", "device_def456"],
    lastIpAddress: "192.168.1.1",

    // Détection patterns anormaux
    anomalyFlags: []
  },

  // Métadonnées
  createdAt: ISODate("2024-01-15T10:00:00Z"),
  lastActiveAt: ISODate("2024-12-09T14:30:00Z"),
  lastMatchAt: ISODate("2024-12-09T14:00:00Z"),

  // Version du schéma
  schemaVersion: 2
}

// Index pour players
db.players.createIndex({ userId: 1 }, { unique: true });
db.players.createIndex({ "profile.username": 1 }, { unique: true });
db.players.createIndex({
  "profile.rank": 1,
  "profile.rankPoints": -1
});
db.players.createIndex({
  "stats.totalScore": -1,
  lastActiveAt: -1
});
db.players.createIndex({ "social.guildId": 1 });
db.players.createIndex({ lastActiveAt: -1 });
db.players.createIndex({ createdAt: 1 });
```

### 2. Leaderboards optimisés

#### Approche A : Embedded scores (pour top performers)

```javascript
// Collection: leaderboards
{
  _id: "global_alltime",
  type: "global",
  period: "alltime",  // alltime, season, monthly, weekly, daily

  // Métadonnées
  metadata: {
    name: "Global All-Time Leaderboard",
    description: "Top players of all time",
    gameMode: "ranked",
    metric: "totalScore",

    // Période (si applicable)
    startDate: null,
    endDate: null,

    // Stats
    totalPlayers: 1547892,
    lastUpdated: ISODate("2024-12-09T14:30:00Z")
  },

  // Top 1000 (dénormalisé pour performance)
  topPlayers: [
    {
      rank: 1,
      playerId: ObjectId("..."),
      username: "ProGamer123",
      avatar: "https://cdn.example.com/avatars/progamer123.jpg",

      score: 4402209,

      // Contexte additionnel
      level: 42,
      rank_badge: "Diamond",

      // Metadata
      lastGameAt: ISODate("2024-12-09T14:00:00Z"),
      gamesPlayed: 1547
    },
    {
      rank: 2,
      playerId: ObjectId("..."),
      username: "ElitePlayer456",
      avatar: "https://cdn.example.com/avatars/elite456.jpg",

      score: 4156780,

      level: 45,
      rank_badge: "Diamond",

      lastGameAt: ISODate("2024-12-09T13:45:00Z"),
      gamesPlayed: 1823
    }
    // ... jusqu'à 1000
  ],

  // Statistiques du leaderboard
  stats: {
    topScore: 4402209,
    averageTopScore: 2847560,
    medianScore: 2456789,
    cutoffTop100: 3567890,
    cutoffTop1000: 1234567
  }
}

// Index pour leaderboards
db.leaderboards.createIndex({ type: 1, period: 1 });
db.leaderboards.createIndex({
  "metadata.gameMode": 1,
  period: 1
});
```

#### Approche B : Scores individuels (pour tous les joueurs)

```javascript
// Collection: player_scores
{
  _id: ObjectId("..."),

  // Identifiants
  playerId: ObjectId("..."),
  leaderboardId: "global_alltime",

  // Score et contexte
  score: 4402209,
  rank: null,  // Calculé à la demande

  // Dénormalisation pour performance
  playerInfo: {
    username: "ProGamer123",
    avatar: "https://cdn.example.com/avatars/progamer123.jpg",
    level: 42,
    rankBadge: "Diamond"
  },

  // Métadonnées
  gamesPlayed: 1547,
  lastGameScore: 2847,
  lastGameAt: ISODate("2024-12-09T14:00:00Z"),

  // Dates
  firstScoreAt: ISODate("2024-01-15T10:30:00Z"),
  updatedAt: ISODate("2024-12-09T14:00:00Z")
}

// Index CRITIQUES pour performance
db.player_scores.createIndex({
  leaderboardId: 1,
  score: -1
});

db.player_scores.createIndex({
  leaderboardId: 1,
  playerId: 1
}, {
  unique: true
});

db.player_scores.createIndex({
  playerId: 1,
  leaderboardId: 1
});
```

### 3. Sessions de jeu

```javascript
// Collection: game_sessions
{
  _id: ObjectId("..."),
  sessionId: "session_abc123def456",

  // Type de session
  gameMode: "ranked",  // ranked, casual, tournament, practice
  matchType: "solo",   // solo, duo, squad, team

  // Joueurs
  players: [
    {
      playerId: ObjectId("..."),
      username: "ProGamer123",
      team: "blue",

      // Performance
      score: 2847,
      kills: 12,
      deaths: 3,
      assists: 8,

      // Résultat
      result: "win",
      rankPointsChange: +25,
      xpGained: 450,

      // Récompenses
      rewards: {
        coins: 150,
        items: ["victory_crate"]
      }
    },
    {
      playerId: ObjectId("..."),
      username: "ElitePlayer456",
      team: "red",

      score: 2134,
      kills: 8,
      deaths: 5,
      assists: 6,

      result: "loss",
      rankPointsChange: -18,
      xpGained: 200,

      rewards: {
        coins: 75
      }
    }
  ],

  // Résultat du match
  outcome: {
    winnerTeam: "blue",
    duration: 1847,  // secondes
    mapId: "desert_storm",

    // Statistiques globales
    totalKills: 42,
    totalDamage: 15678,
    objectivesCompleted: 5
  },

  // Timestamps
  startedAt: ISODate("2024-12-09T13:30:00Z"),
  endedAt: ISODate("2024-12-09T14:00:00Z"),

  // Validation et anti-triche
  validation: {
    validated: true,
    suspiciousActivity: false,
    checksPerformed: [
      "score_validation",
      "performance_consistency",
      "input_pattern_analysis"
    ],
    validatedAt: ISODate("2024-12-09T14:00:05Z")
  },

  // Métadonnées
  serverRegion: "EU_WEST",
  serverInstance: "eu-west-1-server-042",
  gameVersion: "2.5.1"
}

// Index pour sessions
db.game_sessions.createIndex({ "players.playerId": 1, startedAt: -1 });
db.game_sessions.createIndex({ gameMode: 1, startedAt: -1 });
db.game_sessions.createIndex({ startedAt: -1 });
db.game_sessions.createIndex({ sessionId: 1 }, { unique: true });

// TTL pour cleanup automatique (garder 90 jours)
db.game_sessions.createIndex(
  { endedAt: 1 },
  { expireAfterSeconds: 7776000 }  // 90 jours
);
```

### 4. Matchmaking

```javascript
// Collection: matchmaking_queue
{
  _id: ObjectId("..."),

  // Joueur
  playerId: ObjectId("..."),
  username: "ProGamer123",

  // Paramètres du match recherché
  gameMode: "ranked",
  matchType: "solo",
  region: "EU_WEST",

  // Skill rating pour matchmaking
  skillRating: 2847,
  skillRatingRange: {
    min: 2697,  // ±150 initial
    max: 2997
  },

  // Expansion progressive de la recherche
  searchExpansions: 0,
  maxExpansionSteps: 5,
  expansionRate: 100,  // +100 à chaque expansion

  // État
  status: "searching",  // searching, matched, cancelled, expired

  // Timestamps
  queuedAt: ISODate("2024-12-09T14:30:00Z"),
  expiresAt: ISODate("2024-12-09T14:35:00Z"),  // 5 minutes

  // Métadonnées
  priority: 0,  // Peut être augmenté pour long wait time
  waitTime: 0,  // Mis à jour périodiquement

  // Préférences
  preferences: {
    allowCrossPlatform: true,
    voiceChatEnabled: true
  }
}

// Index pour matchmaking
db.matchmaking_queue.createIndex({
  gameMode: 1,
  matchType: 1,
  region: 1,
  status: 1,
  skillRating: 1
});

db.matchmaking_queue.createIndex({ playerId: 1, status: 1 });

// TTL pour cleanup automatique
db.matchmaking_queue.createIndex(
  { expiresAt: 1 },
  { expireAfterSeconds: 0 }
);
```

## Services critiques

### 1. Service de Leaderboard haute performance

```javascript
class LeaderboardService {
  constructor(db, redis) {
    this.db = db;
    this.redis = redis;
    this.CACHE_TTL = 60;  // 1 minute
  }

  async getLeaderboard(leaderboardId, page = 1, limit = 50) {
    const cacheKey = `leaderboard:${leaderboardId}:page:${page}`;

    // Essayer cache Redis
    let cached = await this.redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }

    // Récupérer depuis MongoDB
    const skip = (page - 1) * limit;

    const results = await this.db.collection('player_scores')
      .find({ leaderboardId })
      .sort({ score: -1 })
      .skip(skip)
      .limit(limit)
      .toArray();

    // Calculer ranks
    const startRank = skip + 1;
    const leaderboard = results.map((entry, index) => ({
      rank: startRank + index,
      playerId: entry.playerId,
      username: entry.playerInfo.username,
      avatar: entry.playerInfo.avatar,
      score: entry.score,
      gamesPlayed: entry.gamesPlayed
    }));

    // Cacher résultat
    await this.redis.setex(
      cacheKey,
      this.CACHE_TTL,
      JSON.stringify(leaderboard)
    );

    return leaderboard;
  }

  async getPlayerRank(playerId, leaderboardId) {
    const cacheKey = `rank:${leaderboardId}:${playerId}`;

    // Essayer cache
    let rank = await this.redis.get(cacheKey);
    if (rank) {
      return parseInt(rank);
    }

    // Compter combien de joueurs ont un score supérieur
    const playerScore = await this.db.collection('player_scores')
      .findOne({ playerId: ObjectId(playerId), leaderboardId });

    if (!playerScore) {
      return null;
    }

    const betterPlayersCount = await this.db.collection('player_scores')
      .countDocuments({
        leaderboardId,
        score: { $gt: playerScore.score }
      });

    rank = betterPlayersCount + 1;

    // Cacher pour 5 minutes
    await this.redis.setex(cacheKey, 300, rank.toString());

    return rank;
  }

  async updateScore(playerId, leaderboardId, newScore, gameSession) {
    const session = this.db.client.startSession();

    try {
      await session.withTransaction(async () => {
        // Mettre à jour ou insérer score
        const result = await this.db.collection('player_scores').findOneAndUpdate(
          { playerId: ObjectId(playerId), leaderboardId },
          {
            $max: { score: newScore },  // Ne prend que si supérieur
            $inc: { gamesPlayed: 1 },
            $set: {
              lastGameScore: gameSession.score,
              lastGameAt: new Date(),
              updatedAt: new Date()
            }
          },
          {
            upsert: true,
            returnDocument: 'after',
            session
          }
        );

        // Invalider caches
        await this.invalidatePlayerCaches(playerId, leaderboardId);

        // Si dans top 1000, mettre à jour leaderboard dénormalisé
        const rank = await this.getPlayerRank(playerId, leaderboardId);

        if (rank <= 1000) {
          await this.updateTopLeaderboard(
            leaderboardId,
            playerId,
            result.value,
            session
          );
        }
      });

      return { success: true };
    } finally {
      await session.endSession();
    }
  }

  async invalidatePlayerCaches(playerId, leaderboardId) {
    const patterns = [
      `rank:${leaderboardId}:${playerId}`,
      `leaderboard:${leaderboardId}:*`,  // Invalider toutes les pages
      `player:${playerId}:leaderboards`
    ];

    for (const pattern of patterns) {
      const keys = await this.redis.keys(pattern);
      if (keys.length > 0) {
        await this.redis.del(...keys);
      }
    }
  }

  async updateTopLeaderboard(leaderboardId, playerId, scoreDoc, session) {
    // Récupérer top 1000
    const top1000 = await this.db.collection('player_scores')
      .find({ leaderboardId })
      .sort({ score: -1 })
      .limit(1000)
      .toArray();

    // Formater pour dénormalisation
    const topPlayers = top1000.map((entry, index) => ({
      rank: index + 1,
      playerId: entry.playerId,
      username: entry.playerInfo.username,
      avatar: entry.playerInfo.avatar,
      score: entry.score,
      level: entry.playerInfo.level,
      rankBadge: entry.playerInfo.rankBadge,
      lastGameAt: entry.lastGameAt,
      gamesPlayed: entry.gamesPlayed
    }));

    // Mettre à jour leaderboard dénormalisé
    await this.db.collection('leaderboards').updateOne(
      { _id: leaderboardId },
      {
        $set: {
          topPlayers,
          'metadata.lastUpdated': new Date(),
          'stats.topScore': top1000[0]?.score || 0,
          'stats.cutoffTop100': top1000[99]?.score || 0,
          'stats.cutoffTop1000': top1000[999]?.score || 0
        }
      },
      { session }
    );
  }

  async getSurroundingPlayers(playerId, leaderboardId, range = 5) {
    // Récupérer score du joueur
    const playerScore = await this.db.collection('player_scores')
      .findOne({ playerId: ObjectId(playerId), leaderboardId });

    if (!playerScore) {
      return null;
    }

    // Récupérer joueurs au-dessus
    const playersAbove = await this.db.collection('player_scores')
      .find({
        leaderboardId,
        score: { $gt: playerScore.score }
      })
      .sort({ score: 1 })
      .limit(range)
      .toArray();

    // Récupérer joueurs en-dessous
    const playersBelow = await this.db.collection('player_scores')
      .find({
        leaderboardId,
        score: { $lt: playerScore.score }
      })
      .sort({ score: -1 })
      .limit(range)
      .toArray();

    // Combiner et calculer ranks
    const surrounding = [
      ...playersAbove.reverse(),
      playerScore,
      ...playersBelow
    ];

    const playerRank = await this.getPlayerRank(playerId, leaderboardId);

    return surrounding.map((entry, index) => ({
      rank: playerRank - playersAbove.length + index,
      playerId: entry.playerId,
      username: entry.playerInfo.username,
      avatar: entry.playerInfo.avatar,
      score: entry.score,
      isCurrentPlayer: entry.playerId.equals(playerScore.playerId)
    }));
  }
}
```

### 2. Service de Matchmaking

```javascript
class MatchmakingService {
  constructor(db, redis) {
    this.db = db;
    this.redis = redis;
    this.MATCH_SIZE = 10;  // 10 joueurs par match
    this.SEARCH_INTERVAL = 2000;  // Chercher match toutes les 2s
  }

  async joinQueue(playerId, options) {
    const player = await this.db.collection('players')
      .findOne({ _id: ObjectId(playerId) });

    const queueEntry = {
      playerId: ObjectId(playerId),
      username: player.profile.username,
      gameMode: options.gameMode,
      matchType: options.matchType,
      region: options.region,

      // Skill rating pour matchmaking
      skillRating: player.profile.rankPoints,
      skillRatingRange: {
        min: player.profile.rankPoints - 150,
        max: player.profile.rankPoints + 150
      },

      searchExpansions: 0,
      maxExpansionSteps: 5,
      expansionRate: 100,

      status: "searching",

      queuedAt: new Date(),
      expiresAt: new Date(Date.now() + 300000),  // 5 minutes

      priority: 0,
      waitTime: 0,

      preferences: options.preferences || {}
    };

    await this.db.collection('matchmaking_queue').insertOne(queueEntry);

    // Démarrer recherche de match
    this.startMatchSearch(playerId, options.gameMode);

    return { success: true, queueId: queueEntry._id };
  }

  async startMatchSearch(playerId, gameMode) {
    const searchInterval = setInterval(async () => {
      const queueEntry = await this.db.collection('matchmaking_queue')
        .findOne({
          playerId: ObjectId(playerId),
          status: "searching"
        });

      if (!queueEntry) {
        clearInterval(searchInterval);
        return;
      }

      // Chercher match
      const match = await this.findMatch(queueEntry);

      if (match) {
        clearInterval(searchInterval);
        await this.createMatch(match);
      } else {
        // Élargir la recherche
        await this.expandSearch(queueEntry);
      }
    }, this.SEARCH_INTERVAL);
  }

  async findMatch(queueEntry) {
    // Chercher joueurs compatibles
    const compatiblePlayers = await this.db.collection('matchmaking_queue')
      .find({
        _id: { $ne: queueEntry._id },
        gameMode: queueEntry.gameMode,
        matchType: queueEntry.matchType,
        region: queueEntry.region,
        status: "searching",

        // Range de skill compatible
        skillRating: {
          $gte: queueEntry.skillRatingRange.min,
          $lte: queueEntry.skillRatingRange.max
        }
      })
      .sort({ queuedAt: 1, priority: -1 })
      .limit(this.MATCH_SIZE - 1)
      .toArray();

    // Vérifier si assez de joueurs
    if (compatiblePlayers.length < this.MATCH_SIZE - 1) {
      return null;
    }

    // Créer groupe de joueurs
    const matchPlayers = [queueEntry, ...compatiblePlayers];

    // Vérifier que tous sont encore en recherche (race condition)
    const allSearching = await this.db.collection('matchmaking_queue')
      .countDocuments({
        _id: { $in: matchPlayers.map(p => p._id) },
        status: "searching"
      });

    if (allSearching !== this.MATCH_SIZE) {
      return null;
    }

    return matchPlayers;
  }

  async createMatch(matchPlayers) {
    const session = this.db.client.startSession();

    try {
      await session.withTransaction(async () => {
        // Marquer tous les joueurs comme matched
        const playerIds = matchPlayers.map(p => p._id);

        await this.db.collection('matchmaking_queue').updateMany(
          { _id: { $in: playerIds } },
          { $set: { status: "matched" } },
          { session }
        );

        // Créer session de match
        const matchSession = {
          sessionId: `match_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
          gameMode: matchPlayers[0].gameMode,
          matchType: matchPlayers[0].matchType,

          players: matchPlayers.map(p => ({
            playerId: p.playerId,
            username: p.username,
            skillRating: p.skillRating
          })),

          // Balance des équipes (si team-based)
          teams: this.balanceTeams(matchPlayers),

          status: "starting",

          matchDetails: {
            serverRegion: matchPlayers[0].region,
            avgSkillRating: matchPlayers.reduce((sum, p) =>
              sum + p.skillRating, 0) / matchPlayers.length,

            matchQuality: this.calculateMatchQuality(matchPlayers)
          },

          createdAt: new Date()
        };

        await this.db.collection('match_sessions').insertOne(
          matchSession,
          { session }
        );

        // Notifier tous les joueurs
        for (const player of matchPlayers) {
          await this.notifyPlayerMatchFound(
            player.playerId,
            matchSession
          );
        }
      });
    } finally {
      await session.endSession();
    }
  }

  balanceTeams(players) {
    // Algorithme simple: alterner par skill rating
    const sorted = [...players].sort((a, b) =>
      b.skillRating - a.skillRating
    );

    const teamA = [];
    const teamB = [];

    sorted.forEach((player, index) => {
      if (index % 2 === 0) {
        teamA.push(player);
      } else {
        teamB.push(player);
      }
    });

    return {
      blue: teamA.map(p => ({
        playerId: p.playerId,
        username: p.username,
        skillRating: p.skillRating
      })),
      red: teamB.map(p => ({
        playerId: p.playerId,
        username: p.username,
        skillRating: p.skillRating
      }))
    };
  }

  calculateMatchQuality(players) {
    // Qualité basée sur variance de skill
    const ratings = players.map(p => p.skillRating);
    const avg = ratings.reduce((a, b) => a + b) / ratings.length;

    const variance = ratings.reduce((sum, r) =>
      sum + Math.pow(r - avg, 2), 0) / ratings.length;

    const stdDev = Math.sqrt(variance);

    // Qualité inversement proportionnelle à stdDev
    // 100 = parfait, 0 = très disparate
    return Math.max(0, 100 - stdDev / 10);
  }

  async expandSearch(queueEntry) {
    const newExpansion = queueEntry.searchExpansions + 1;

    if (newExpansion > queueEntry.maxExpansionSteps) {
      // Abandon après max expansions
      await this.db.collection('matchmaking_queue').updateOne(
        { _id: queueEntry._id },
        {
          $set: {
            status: "expired",
            expiredAt: new Date()
          }
        }
      );
      return;
    }

    // Élargir range de skill
    const expansion = queueEntry.expansionRate * newExpansion;

    await this.db.collection('matchmaking_queue').updateOne(
      { _id: queueEntry._id },
      {
        $set: {
          'skillRatingRange.min': queueEntry.skillRating - 150 - expansion,
          'skillRatingRange.max': queueEntry.skillRating + 150 + expansion,
          searchExpansions: newExpansion
        },
        $inc: {
          waitTime: this.SEARCH_INTERVAL,
          priority: 1  // Augmenter priorité
        }
      }
    );
  }

  async leaveQueue(playerId) {
    await this.db.collection('matchmaking_queue').updateOne(
      { playerId: ObjectId(playerId), status: "searching" },
      {
        $set: {
          status: "cancelled",
          cancelledAt: new Date()
        }
      }
    );

    return { success: true };
  }
}
```

### 3. Service de progression

```javascript
class ProgressionService {
  async awardXP(playerId, xpAmount, source) {
    const session = this.db.client.startSession();

    try {
      await session.withTransaction(async () => {
        const player = await this.db.collection('players')
          .findOne({ _id: ObjectId(playerId) }, { session });

        const currentXP = player.profile.xp;
        const currentLevel = player.profile.level;
        const xpToNextLevel = player.profile.xpToNextLevel;

        let newXP = currentXP + xpAmount;
        let newLevel = currentLevel;
        const levelsGained = [];

        // Vérifier level up
        while (newXP >= xpToNextLevel) {
          newXP -= xpToNextLevel;
          newLevel++;
          levelsGained.push(newLevel);

          // Calculer XP pour prochain niveau
          const nextLevelXP = this.calculateXPForLevel(newLevel + 1);
        }

        // Mettre à jour joueur
        await this.db.collection('players').updateOne(
          { _id: ObjectId(playerId) },
          {
            $set: {
              'profile.xp': newXP,
              'profile.level': newLevel,
              'profile.xpToNextLevel': this.calculateXPForLevel(newLevel + 1)
            }
          },
          { session }
        );

        // Récompenses pour level ups
        for (const level of levelsGained) {
          await this.grantLevelRewards(playerId, level, session);
        }

        // Logger progression
        await this.db.collection('progression_log').insertOne({
          playerId: ObjectId(playerId),
          type: 'xp_gained',
          amount: xpAmount,
          source,
          resultingLevel: newLevel,
          levelsGained: levelsGained.length,
          timestamp: new Date()
        }, { session });
      });
    } finally {
      await session.endSession();
    }
  }

  calculateXPForLevel(level) {
    // Formule exponentielle
    return Math.floor(1000 * Math.pow(level, 1.5));
  }

  async grantLevelRewards(playerId, level, session) {
    // Définir récompenses par niveau
    const rewards = this.getLevelRewards(level);

    if (rewards.coins) {
      await this.db.collection('players').updateOne(
        { _id: ObjectId(playerId) },
        { $inc: { 'economy.coins': rewards.coins } },
        { session }
      );
    }

    if (rewards.items) {
      await this.db.collection('players').updateOne(
        { _id: ObjectId(playerId) },
        {
          $push: {
            'economy.inventory': {
              $each: rewards.items.map(itemId => ({
                itemId,
                quantity: 1,
                acquiredAt: new Date()
              }))
            }
          }
        },
        { session }
      );
    }

    // Logger récompenses
    await this.db.collection('rewards_log').insertOne({
      playerId: ObjectId(playerId),
      type: 'level_reward',
      level,
      rewards,
      timestamp: new Date()
    }, { session });
  }

  getLevelRewards(level) {
    const rewards = {
      coins: level * 100,
      items: []
    };

    // Récompenses spéciales à certains niveaux
    if (level % 5 === 0) {
      rewards.items.push('rare_crate');
    }

    if (level % 10 === 0) {
      rewards.items.push('legendary_crate');
    }

    return rewards;
  }

  async checkAchievements(playerId, event) {
    // Récupérer achievements du joueur
    const player = await this.db.collection('players')
      .findOne({ _id: ObjectId(playerId) });

    // Vérifier chaque achievement
    const unlockedAchievements = [];

    for (const achievement of player.progression.achievements) {
      if (achievement.unlockedAt) continue;  // Déjà débloqué

      // Vérifier si conditions remplies
      const progress = await this.calculateAchievementProgress(
        playerId,
        achievement.id,
        event
      );

      if (progress >= achievement.maxProgress) {
        // Débloquer achievement
        await this.unlockAchievement(playerId, achievement.id);
        unlockedAchievements.push(achievement);
      } else {
        // Mettre à jour progression
        await this.db.collection('players').updateOne(
          {
            _id: ObjectId(playerId),
            'progression.achievements.id': achievement.id
          },
          {
            $set: {
              'progression.achievements.$.progress': progress
            }
          }
        );
      }
    }

    return unlockedAchievements;
  }

  async unlockAchievement(playerId, achievementId) {
    const session = this.db.client.startSession();

    try {
      await session.withTransaction(async () => {
        // Marquer comme débloqué
        await this.db.collection('players').updateOne(
          {
            _id: ObjectId(playerId),
            'progression.achievements.id': achievementId
          },
          {
            $set: {
              'progression.achievements.$.unlockedAt': new Date(),
              'progression.achievements.$.progress': achievement.maxProgress
            }
          },
          { session }
        );

        // Accorder récompenses
        const achievementDef = await this.getAchievementDefinition(achievementId);
        if (achievementDef.rewards) {
          await this.grantRewards(playerId, achievementDef.rewards, session);
        }

        // Logger
        await this.db.collection('achievements_log').insertOne({
          playerId: ObjectId(playerId),
          achievementId,
          unlockedAt: new Date()
        }, { session });
      });
    } finally {
      await session.endSession();
    }
  }
}
```

## Anti-triche et validation

### 1. Système de détection

```javascript
class AntiCheatService {
  async validateGameSession(sessionData) {
    const suspicionFlags = [];
    let suspicionScore = 0;

    // 1. Validation de score (impossible scores)
    const scoreValidation = this.validateScore(sessionData);
    if (!scoreValidation.valid) {
      suspicionFlags.push('impossible_score');
      suspicionScore += 50;
    }

    // 2. Validation de performance (inhuman performance)
    const perfValidation = this.validatePerformance(sessionData);
    if (!perfValidation.valid) {
      suspicionFlags.push('inhuman_performance');
      suspicionScore += 40;
    }

    // 3. Validation de patterns (bot detection)
    const patternValidation = await this.validatePatterns(sessionData);
    if (!patternValidation.valid) {
      suspicionFlags.push('suspicious_patterns');
      suspicionScore += 30;
    }

    // 4. Validation de device (multi-accounting)
    const deviceValidation = await this.validateDevice(sessionData);
    if (!deviceValidation.valid) {
      suspicionFlags.push('suspicious_device');
      suspicionScore += 20;
    }

    const isValid = suspicionScore < 80;

    // Logger si suspicion
    if (suspicionScore > 0) {
      await this.logSuspiciousActivity({
        sessionId: sessionData.sessionId,
        playerId: sessionData.playerId,
        suspicionScore,
        flags: suspicionFlags,
        timestamp: new Date()
      });
    }

    // Bannir si score trop élevé
    if (suspicionScore >= 100) {
      await this.banPlayer(sessionData.playerId, 'automated_detection', suspicionFlags);
    }

    return {
      valid: isValid,
      suspicionScore,
      flags: suspicionFlags
    };
  }

  validateScore(sessionData) {
    // Vérifier limites physiques du jeu
    const maxPossibleScore = this.calculateMaxPossibleScore(
      sessionData.duration,
      sessionData.gameMode
    );

    if (sessionData.score > maxPossibleScore * 1.1) {  // 10% marge
      return {
        valid: false,
        reason: `Score ${sessionData.score} exceeds maximum possible ${maxPossibleScore}`
      };
    }

    return { valid: true };
  }

  validatePerformance(sessionData) {
    const player = sessionData.players[0];

    // Vérifier ratios impossibles
    if (player.kills > 0 && player.deaths === 0 && player.kills > 50) {
      return {
        valid: false,
        reason: 'Impossible K/D ratio'
      };
    }

    // Vérifier accuracy anormale
    if (player.accuracy > 0.95 && player.kills > 20) {
      return {
        valid: false,
        reason: 'Suspiciously high accuracy'
      };
    }

    return { valid: true };
  }

  async validatePatterns(sessionData) {
    // Récupérer historique du joueur
    const recentSessions = await this.db.collection('game_sessions')
      .find({
        'players.playerId': sessionData.playerId,
        endedAt: { $gte: new Date(Date.now() - 86400000) }  // 24h
      })
      .sort({ endedAt: -1 })
      .limit(10)
      .toArray();

    if (recentSessions.length < 3) {
      return { valid: true };
    }

    // Vérifier patterns anormaux
    const scores = recentSessions.map(s =>
      s.players.find(p => p.playerId.equals(sessionData.playerId)).score
    );

    const avgScore = scores.reduce((a, b) => a + b) / scores.length;
    const stdDev = Math.sqrt(
      scores.reduce((sum, score) => sum + Math.pow(score - avgScore, 2), 0)
      / scores.length
    );

    // Si nouveau score >> moyenne + 4 écarts-types = suspicion
    if (sessionData.score > avgScore + 4 * stdDev) {
      return {
        valid: false,
        reason: 'Sudden performance spike'
      };
    }

    return { valid: true };
  }

  async validateDevice(sessionData) {
    // Vérifier si device déjà associé à autre compte
    const otherAccounts = await this.db.collection('players')
      .find({
        _id: { $ne: sessionData.playerId },
        'security.deviceIds': sessionData.deviceId
      })
      .toArray();

    if (otherAccounts.length > 0) {
      return {
        valid: false,
        reason: 'Device used by multiple accounts'
      };
    }

    return { valid: true };
  }

  async logSuspiciousActivity(data) {
    await this.db.collection('suspicious_activity').insertOne(data);

    // Incrémenter suspicion score du joueur
    await this.db.collection('players').updateOne(
      { _id: data.playerId },
      { $inc: { 'security.suspicionScore': data.suspicionScore / 10 } }
    );
  }

  async banPlayer(playerId, reason, evidence) {
    await this.db.collection('players').updateOne(
      { _id: ObjectId(playerId) },
      {
        $set: {
          'status.state': 'banned',
          'status.bannedAt': new Date(),
          'status.banReason': reason
        },
        $push: {
          'security.penalties': {
            type: 'ban',
            reason,
            evidence,
            appliedAt: new Date(),
            duration: 'permanent'
          }
        }
      }
    );

    console.log(`Player ${playerId} banned for ${reason}`);
  }
}
```

## Performance et scalabilité

### 1. Stratégie de cache Redis

```javascript
// Configuration cache pour gaming
const CacheStrategy = {
  // Top 100 de chaque leaderboard (hot data)
  leaderboard_top: {
    ttl: 60,  // 1 minute
    pattern: 'leaderboard:{id}:top100',

    async get(redis, leaderboardId) {
      const key = `leaderboard:${leaderboardId}:top100`;
      const cached = await redis.get(key);
      return cached ? JSON.parse(cached) : null;
    },

    async set(redis, leaderboardId, data) {
      const key = `leaderboard:${leaderboardId}:top100`;
      await redis.setex(key, 60, JSON.stringify(data));
    }
  },

  // Rank d'un joueur spécifique
  player_rank: {
    ttl: 300,  // 5 minutes
    pattern: 'rank:{leaderboardId}:{playerId}',

    async get(redis, leaderboardId, playerId) {
      const key = `rank:${leaderboardId}:${playerId}`;
      const rank = await redis.get(key);
      return rank ? parseInt(rank) : null;
    },

    async set(redis, leaderboardId, playerId, rank) {
      const key = `rank:${leaderboardId}:${playerId}`;
      await redis.setex(key, 300, rank.toString());
    }
  },

  // Session de joueur active
  active_session: {
    ttl: 3600,  // 1 heure
    pattern: 'session:{playerId}',

    async get(redis, playerId) {
      const key = `session:${playerId}`;
      const session = await redis.get(key);
      return session ? JSON.parse(session) : null;
    },

    async set(redis, playerId, sessionData) {
      const key = `session:${playerId}`;
      await redis.setex(key, 3600, JSON.stringify(sessionData));
    }
  }
};
```

### 2. Sharding pour millions de joueurs

```javascript
// Stratégie de sharding pour grande échelle

// Shard key: userId (hashed)
// Justification:
// - Distribution uniforme des joueurs
// - Évite hotspots (pas de celebrity players problem)
// - Queries par joueur sont optimales (single shard)

sh.enableSharding("gaming");

sh.shardCollection(
  "gaming.players",
  { userId: "hashed" }
);

// Pour leaderboards: shard par leaderboardId + score range
sh.shardCollection(
  "gaming.player_scores",
  { leaderboardId: 1, score: 1 }
);

// Zone sharding pour isolation régionale
sh.addShardToZone("shard01", "NA");
sh.addShardToZone("shard02", "EU");
sh.addShardToZone("shard03", "ASIA");

// Tag ranges pour sessions par région
sh.updateZoneKeyRange(
  "gaming.game_sessions",
  { serverRegion: "NA", startedAt: MinKey },
  { serverRegion: "NA", startedAt: MaxKey },
  "NA"
);
```

### 3. Métriques de performance gaming

```javascript
const gamingMetrics = {
  // Leaderboard
  'leaderboard.read_latency.p95': {
    description: 'Latency for leaderboard reads',
    target: 50,  // ms
    alert_threshold: 100
  },

  'leaderboard.update_latency.p95': {
    description: 'Latency for score updates',
    target: 100,  // ms
    alert_threshold: 200
  },

  'leaderboard.cache_hit_rate': {
    description: 'Cache hit rate for leaderboards',
    target: 0.95,  // 95%
    alert_threshold: 0.80
  },

  // Matchmaking
  'matchmaking.avg_wait_time': {
    description: 'Average matchmaking wait time',
    target: 30000,  // 30 seconds
    alert_threshold: 60000
  },

  'matchmaking.match_quality.avg': {
    description: 'Average match quality score',
    target: 85,  // 0-100
    alert_threshold: 70
  },

  // Gaming
  'game.sessions_per_second': {
    description: 'New game sessions per second',
    monitor: true
  },

  'game.concurrent_players': {
    description: 'Current concurrent players',
    monitor: true
  },

  // Anti-cheat
  'anticheat.detection_rate': {
    description: 'Percentage of sessions flagged',
    monitor: true,
    alert_threshold: 0.05  // > 5% = problem
  }
};
```

## Checklist de déploiement

### ✅ Architecture

- [ ] MongoDB Replica Set ou Sharded Cluster
- [ ] Redis pour cache haute performance
- [ ] Rate limiting configuré sur API
- [ ] WebSocket pour temps réel
- [ ] CDN pour assets statiques

### ✅ Leaderboards

- [ ] Index optimisés (leaderboardId + score)
- [ ] Cache Redis pour top 100
- [ ] Invalidation intelligente
- [ ] Leaderboards multiples (global, régional, amis)
- [ ] API pagination performante

### ✅ Matchmaking

- [ ] File d'attente avec expiration
- [ ] Expansion progressive de recherche
- [ ] Balance des équipes
- [ ] Quality metrics tracking
- [ ] Notification temps réel

### ✅ Progression

- [ ] XP et leveling configurés
- [ ] Achievements définis
- [ ] Rewards system
- [ ] Battle Pass (optionnel)
- [ ] Économie in-game

### ✅ Anti-triche

- [ ] Validation côté serveur
- [ ] Détection d'anomalies
- [ ] Device fingerprinting
- [ ] Reports système
- [ ] Ban workflow

### ✅ Performance

- [ ] Opérations atomiques pour scores
- [ ] Batch updates où possible
- [ ] Index couvrants
- [ ] Monitoring latence
- [ ] Load testing effectué

### ✅ Scalabilité

- [ ] Sharding configuré si > 1M joueurs
- [ ] Connection pooling optimisé
- [ ] Read replicas si nécessaire
- [ ] Auto-scaling infrastructure
- [ ] CDN global

## Conclusion

MongoDB est particulièrement adapté au gaming grâce à :

**✅ Forces démontrées :**
- Opérations atomiques pour scores thread-safe
- Index performants pour classements rapides
- Schéma flexible pour différents types de jeux
- Agrégations pour statistiques temps réel
- TTL pour cleanup automatique de sessions
- Sharding transparent pour scalabilité massive
- Change Streams pour événements temps réel

**⚠️ Considérations importantes :**
- Cache Redis essentiel pour top leaderboards
- Validation côté serveur obligatoire (anti-triche)
- Monitoring de latence critique
- Tests de charge pour pics d'activité
- Design anti-manipulation des classements

**🎯 Patterns essentiels gaming :**
1. **Atomic Updates** pour scores ($max, $inc)
2. **Dénormalisation** pour top leaderboards
3. **Cache multi-niveaux** pour performance
4. **Sharding par userId** pour distribution
5. **TTL Index** pour cleanup automatique
6. **Change Streams** pour temps réel

Cette architecture supporte des jeux de quelques milliers à plusieurs millions de joueurs actifs, avec des classements mis à jour en temps réel et un matchmaking intelligent.

---

**Références :**
- "Multiplayer Game Programming" - Joshua Glazer
- Riot Games Engineering Blog
- Supercell Tech Blog
- "Game Programming Patterns" - Robert Nystrom
- AWS GameTech Blog

⏭️ [Analyse en temps réel](/20-cas-usage-architectures/06-analyse-temps-reel.md)
