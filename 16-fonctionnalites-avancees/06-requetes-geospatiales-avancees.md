🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.6 Requêtes géospatiales avancées

## Introduction

MongoDB offre des capacités géospatiales puissantes permettant de stocker, indexer et interroger des données basées sur la localisation. Ces fonctionnalités sont conformes aux standards **GeoJSON** et supportent des requêtes géométriques complexes sur des données 2D et sphériques (terre ronde).

Les requêtes géospatiales sont essentielles pour les applications modernes : livraison à la demande, recherche de proximité, géofencing, analyse de mobilité, ou tout système nécessitant des calculs basés sur la position géographique.

---

## Concepts fondamentaux

### Types de géométries GeoJSON

MongoDB supporte les types GeoJSON suivants :

```javascript
// 1. Point - Position unique
const point = {
  type: "Point",
  coordinates: [2.3522, 48.8566]  // [longitude, latitude]
};

// 2. LineString - Ligne composée de points
const lineString = {
  type: "LineString",
  coordinates: [
    [2.3522, 48.8566],  // Paris
    [2.2945, 48.8584],  // Arc de Triomphe
    [2.3376, 48.8606]   // Gare du Nord
  ]
};

// 3. Polygon - Zone fermée
const polygon = {
  type: "Polygon",
  coordinates: [[
    [2.2945, 48.8584],  // Coin 1
    [2.3522, 48.8566],  // Coin 2
    [2.3376, 48.8606],  // Coin 3
    [2.2800, 48.8650],  // Coin 4
    [2.2945, 48.8584]   // Retour au coin 1 (fermeture)
  ]]
};

// 4. MultiPoint - Plusieurs points
const multiPoint = {
  type: "MultiPoint",
  coordinates: [
    [2.3522, 48.8566],
    [2.2945, 48.8584],
    [2.3376, 48.8606]
  ]
};

// 5. MultiLineString - Plusieurs lignes
const multiLineString = {
  type: "MultiLineString",
  coordinates: [
    [[2.3522, 48.8566], [2.2945, 48.8584]],  // Ligne 1
    [[2.3376, 48.8606], [2.2800, 48.8650]]   // Ligne 2
  ]
};

// 6. MultiPolygon - Plusieurs polygones
const multiPolygon = {
  type: "MultiPolygon",
  coordinates: [
    [[[2.29, 48.85], [2.35, 48.85], [2.35, 48.86], [2.29, 48.86], [2.29, 48.85]]],
    [[[2.30, 48.87], [2.34, 48.87], [2.34, 48.88], [2.30, 48.88], [2.30, 48.87]]]
  ]
};

// 7. GeometryCollection - Collection de géométries
const geometryCollection = {
  type: "GeometryCollection",
  geometries: [
    { type: "Point", coordinates: [2.3522, 48.8566] },
    { type: "LineString", coordinates: [[2.29, 48.85], [2.35, 48.86]] }
  ]
};
```

### Index géospatiaux

```javascript
// Index 2dsphere - Pour données sphériques (recommandé)
// Supporte GeoJSON et calculs sur sphère
await db.collection('places').createIndex({ location: "2dsphere" });

// Index 2d - Pour données planes (legacy)
// Pour coordonnées plates, moins précis
await db.collection('legacy_places').createIndex({ location: "2d" });

// Index composé avec champs additionnels
await db.collection('restaurants').createIndex({
  location: "2dsphere",
  category: 1,
  rating: -1
});
```

---

## Opérateurs géospatiaux

### $near - Proximité

```javascript
// Trouver les 10 restaurants les plus proches
const userLocation = [2.3522, 48.8566];  // Paris centre

const nearestRestaurants = await db.collection('restaurants').find({
  location: {
    $near: {
      $geometry: {
        type: "Point",
        coordinates: userLocation
      },
      $maxDistance: 5000,  // 5 km en mètres
      $minDistance: 0
    }
  }
}).limit(10).toArray();

// Avec filtre additionnel
const nearestPizzas = await db.collection('restaurants').find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: userLocation },
      $maxDistance: 3000
    }
  },
  category: "Pizza"
}).toArray();
```

### $geoWithin - Zone

```javascript
// Trouver tous les points dans un polygon
const deliveryZone = {
  type: "Polygon",
  coordinates: [[
    [2.2945, 48.8584],
    [2.3522, 48.8566],
    [2.3376, 48.8606],
    [2.2800, 48.8650],
    [2.2945, 48.8584]
  ]]
};

const restaurantsInZone = await db.collection('restaurants').find({
  location: {
    $geoWithin: {
      $geometry: deliveryZone
    }
  }
}).toArray();

// Avec cercle
const restaurantsInCircle = await db.collection('restaurants').find({
  location: {
    $geoWithin: {
      $centerSphere: [
        [2.3522, 48.8566],  // Centre
        5 / 6378.1          // Rayon en radians (5 km / rayon terre)
      ]
    }
  }
}).toArray();
```

### $geoIntersects - Intersection

```javascript
// Trouver toutes les zones qui intersectent une ligne
const route = {
  type: "LineString",
  coordinates: [
    [2.3522, 48.8566],
    [2.2945, 48.8584]
  ]
};

const intersectingZones = await db.collection('delivery_zones').find({
  area: {
    $geoIntersects: {
      $geometry: route
    }
  }
}).toArray();
```

---

## Cas d'usage avancés

### Cas 1 : Application de livraison avec calcul de distance

```javascript
class DeliveryService {
  constructor(db) {
    this.db = db;
    this.restaurants = null;
    this.drivers = null;
    this.orders = null;
  }

  async initialize() {
    this.restaurants = this.db.collection('restaurants');
    this.drivers = this.db.collection('drivers');
    this.orders = this.db.collection('orders');

    // Index géospatiaux
    await this.restaurants.createIndex({ location: "2dsphere" });
    await this.drivers.createIndex({ currentLocation: "2dsphere" });

    // Index composés
    await this.restaurants.createIndex({
      location: "2dsphere",
      category: 1,
      rating: -1,
      isOpen: 1
    });

    await this.drivers.createIndex({
      currentLocation: "2dsphere",
      status: 1,
      rating: -1
    });
  }

  async findNearbyRestaurants(userLocation, options = {}) {
    const {
      maxDistance = 5000,  // 5 km par défaut
      category = null,
      minRating = 0,
      limit = 20
    } = options;

    const query = {
      location: {
        $near: {
          $geometry: {
            type: "Point",
            coordinates: userLocation
          },
          $maxDistance: maxDistance
        }
      },
      isOpen: true
    };

    if (category) query.category = category;
    if (minRating > 0) query.rating = { $gte: minRating };

    const restaurants = await this.restaurants.find(query)
      .limit(limit)
      .toArray();

    // Enrichir avec distance calculée
    return restaurants.map(restaurant => ({
      ...restaurant,
      distance: this.calculateDistance(
        userLocation,
        restaurant.location.coordinates
      )
    }));
  }

  async findAvailableDrivers(restaurantLocation, maxDistance = 3000) {
    const drivers = await this.drivers.find({
      currentLocation: {
        $near: {
          $geometry: {
            type: "Point",
            coordinates: restaurantLocation
          },
          $maxDistance: maxDistance
        }
      },
      status: 'available'
    })
      .limit(10)
      .toArray();

    // Enrichir avec distance et temps estimé
    return drivers.map(driver => {
      const distance = this.calculateDistance(
        restaurantLocation,
        driver.currentLocation.coordinates
      );

      return {
        ...driver,
        distance,
        estimatedPickupTime: this.estimateTime(distance, 30)  // 30 km/h moyenne
      };
    }).sort((a, b) => a.distance - b.distance);
  }

  async assignOptimalDriver(orderId, restaurantLocation, deliveryLocation) {
    // Trouver les drivers disponibles proches du restaurant
    const availableDrivers = await this.findAvailableDrivers(restaurantLocation, 5000);

    if (availableDrivers.length === 0) {
      throw new Error('No drivers available');
    }

    // Calculer score pour chaque driver
    const scoredDrivers = availableDrivers.map(driver => {
      const distanceToRestaurant = driver.distance;

      const distanceRestaurantToDelivery = this.calculateDistance(
        restaurantLocation,
        deliveryLocation
      );

      const totalDistance = distanceToRestaurant + distanceRestaurantToDelivery;

      // Score basé sur distance, rating, et historique
      const score = (
        (1 - distanceToRestaurant / 5000) * 0.5 +  // 50% distance
        (driver.rating / 5) * 0.3 +                 // 30% rating
        (driver.completedOrders / 1000) * 0.2       // 20% expérience
      );

      return {
        ...driver,
        totalDistance,
        estimatedDeliveryTime: this.estimateTime(totalDistance, 25),
        score
      };
    }).sort((a, b) => b.score - a.score);

    // Assigner au meilleur driver
    const bestDriver = scoredDrivers[0];

    await this.drivers.updateOne(
      { _id: bestDriver._id },
      {
        $set: {
          status: 'assigned',
          currentOrderId: orderId
        }
      }
    );

    await this.orders.updateOne(
      { _id: orderId },
      {
        $set: {
          driverId: bestDriver._id,
          status: 'assigned',
          estimatedDeliveryTime: new Date(
            Date.now() + bestDriver.estimatedDeliveryTime * 60000
          )
        }
      }
    );

    return bestDriver;
  }

  async trackDelivery(orderId) {
    const order = await this.orders.findOne({ _id: orderId });
    if (!order) throw new Error('Order not found');

    const driver = await this.drivers.findOne({ _id: order.driverId });
    if (!driver) throw new Error('Driver not found');

    const restaurant = await this.restaurants.findOne({ _id: order.restaurantId });

    // Calculer distances actuelles
    const distanceToRestaurant = this.calculateDistance(
      driver.currentLocation.coordinates,
      restaurant.location.coordinates
    );

    const distanceToDelivery = this.calculateDistance(
      driver.currentLocation.coordinates,
      order.deliveryLocation.coordinates
    );

    return {
      orderId: order._id,
      status: order.status,
      driver: {
        name: driver.name,
        phone: driver.phone,
        location: driver.currentLocation,
        rating: driver.rating
      },
      distances: {
        toRestaurant: distanceToRestaurant,
        toDelivery: distanceToDelivery
      },
      estimatedTimeRemaining: order.status === 'picked_up'
        ? this.estimateTime(distanceToDelivery, 25)
        : this.estimateTime(distanceToRestaurant + distanceToDelivery, 25)
    };
  }

  async updateDriverLocation(driverId, newLocation) {
    await this.drivers.updateOne(
      { _id: driverId },
      {
        $set: {
          currentLocation: {
            type: "Point",
            coordinates: newLocation
          },
          lastLocationUpdate: new Date()
        }
      }
    );

    // Vérifier les ordres en cours
    const activeOrder = await this.orders.findOne({
      driverId,
      status: { $in: ['assigned', 'picked_up'] }
    });

    if (activeOrder) {
      // Recalculer ETAs
      await this.updateOrderETA(activeOrder._id);
    }
  }

  async updateOrderETA(orderId) {
    const order = await this.orders.findOne({ _id: orderId });
    const driver = await this.drivers.findOne({ _id: order.driverId });

    let distance;
    if (order.status === 'assigned') {
      const restaurant = await this.restaurants.findOne({ _id: order.restaurantId });
      const distToRestaurant = this.calculateDistance(
        driver.currentLocation.coordinates,
        restaurant.location.coordinates
      );
      const distToDelivery = this.calculateDistance(
        restaurant.location.coordinates,
        order.deliveryLocation.coordinates
      );
      distance = distToRestaurant + distToDelivery;
    } else {
      distance = this.calculateDistance(
        driver.currentLocation.coordinates,
        order.deliveryLocation.coordinates
      );
    }

    const eta = new Date(Date.now() + this.estimateTime(distance, 25) * 60000);

    await this.orders.updateOne(
      { _id: orderId },
      { $set: { estimatedDeliveryTime: eta } }
    );
  }

  calculateDistance(coord1, coord2) {
    // Formule Haversine pour distance entre deux points sur sphère
    const R = 6371e3; // Rayon de la Terre en mètres
    const φ1 = coord1[1] * Math.PI / 180;
    const φ2 = coord2[1] * Math.PI / 180;
    const Δφ = (coord2[1] - coord1[1]) * Math.PI / 180;
    const Δλ = (coord2[0] - coord1[0]) * Math.PI / 180;

    const a = Math.sin(Δφ / 2) * Math.sin(Δφ / 2) +
              Math.cos(φ1) * Math.cos(φ2) *
              Math.sin(Δλ / 2) * Math.sin(Δλ / 2);

    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

    return R * c; // Distance en mètres
  }

  estimateTime(distanceMeters, speedKmh) {
    // Retourne temps en minutes
    return (distanceMeters / 1000) / speedKmh * 60;
  }

  async getDeliveryStatistics(restaurantId) {
    return await this.orders.aggregate([
      {
        $match: {
          restaurantId,
          status: 'delivered',
          completedAt: {
            $gte: new Date(Date.now() - 30 * 86400000)  // 30 jours
          }
        }
      },
      {
        $project: {
          actualDeliveryTime: {
            $divide: [
              { $subtract: ['$completedAt', '$createdAt'] },
              60000  // Convertir en minutes
            ]
          },
          distance: {
            $function: {
              body: function(restaurantCoords, deliveryCoords) {
                // Haversine en JS
                const R = 6371e3;
                const φ1 = restaurantCoords[1] * Math.PI / 180;
                const φ2 = deliveryCoords[1] * Math.PI / 180;
                const Δφ = (deliveryCoords[1] - restaurantCoords[1]) * Math.PI / 180;
                const Δλ = (deliveryCoords[0] - restaurantCoords[0]) * Math.PI / 180;
                const a = Math.sin(Δφ/2) * Math.sin(Δφ/2) +
                         Math.cos(φ1) * Math.cos(φ2) *
                         Math.sin(Δλ/2) * Math.sin(Δλ/2);
                const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
                return R * c / 1000;
              },
              args: ['$restaurantLocation.coordinates', '$deliveryLocation.coordinates'],
              lang: 'js'
            }
          }
        }
      },
      {
        $group: {
          _id: null,
          avgDeliveryTime: { $avg: '$actualDeliveryTime' },
          avgDistance: { $avg: '$distance' },
          totalDeliveries: { $sum: 1 }
        }
      }
    ]).toArray();
  }
}

// Utilisation
const delivery = new DeliveryService(db);
await delivery.initialize();

// Chercher restaurants proches
const restaurants = await delivery.findNearbyRestaurants(
  [2.3522, 48.8566],  // Position utilisateur
  {
    maxDistance: 3000,
    category: 'Italian',
    minRating: 4.0,
    limit: 10
  }
);

// Assigner driver optimal
const driver = await delivery.assignOptimalDriver(
  orderId,
  [2.3522, 48.8566],  // Restaurant
  [2.3376, 48.8606]   // Livraison
);

// Tracker livraison
const tracking = await delivery.trackDelivery(orderId);
console.log('ETA:', tracking.estimatedTimeRemaining, 'minutes');

// Update position driver
await delivery.updateDriverLocation(driverId, [2.3450, 48.8590]);
```

### Cas 2 : Géofencing et zones de service

```javascript
class GeofencingService {
  constructor(db) {
    this.db = db;
    this.zones = null;
    this.devices = null;
    this.events = null;
  }

  async initialize() {
    this.zones = this.db.collection('geofence_zones');
    this.devices = this.db.collection('tracked_devices');
    this.events = this.db.collection('geofence_events');

    // Index
    await this.zones.createIndex({ area: "2dsphere" });
    await this.devices.createIndex({ currentLocation: "2dsphere" });
    await this.events.createIndex({ timestamp: -1, deviceId: 1 });
  }

  async createZone(zoneData) {
    const zone = {
      name: zoneData.name,
      type: zoneData.type,  // 'delivery', 'restricted', 'service', etc.
      area: zoneData.area,  // GeoJSON Polygon
      rules: zoneData.rules || {},
      metadata: zoneData.metadata || {},
      createdAt: new Date(),
      isActive: true
    };

    const result = await this.zones.insertOne(zone);
    return { ...zone, _id: result.insertedId };
  }

  async checkDeviceInZones(deviceId, location) {
    // Trouver toutes les zones contenant le device
    const zonesContaining = await this.zones.find({
      area: {
        $geoIntersects: {
          $geometry: {
            type: "Point",
            coordinates: location
          }
        }
      },
      isActive: true
    }).toArray();

    // Récupérer les zones précédentes du device
    const device = await this.devices.findOne({ _id: deviceId });
    const previousZones = device?.currentZones || [];

    // Détecter entrées et sorties
    const currentZoneIds = zonesContaining.map(z => z._id.toString());
    const previousZoneIds = previousZones.map(z => z.toString());

    const entered = zonesContaining.filter(
      z => !previousZoneIds.includes(z._id.toString())
    );

    const exited = previousZones.filter(
      zId => !currentZoneIds.includes(zId.toString())
    );

    // Mettre à jour le device
    await this.devices.updateOne(
      { _id: deviceId },
      {
        $set: {
          currentLocation: {
            type: "Point",
            coordinates: location
          },
          currentZones: zonesContaining.map(z => z._id),
          lastUpdate: new Date()
        }
      },
      { upsert: true }
    );

    // Créer événements
    const events = [];

    for (const zone of entered) {
      const event = {
        deviceId,
        zoneId: zone._id,
        zoneName: zone.name,
        eventType: 'enter',
        location: {
          type: "Point",
          coordinates: location
        },
        timestamp: new Date(),
        rules: zone.rules
      };

      await this.events.insertOne(event);
      events.push(event);

      // Exécuter actions basées sur règles
      await this.executeZoneRules(zone, deviceId, 'enter');
    }

    for (const zoneId of exited) {
      const zone = await this.zones.findOne({ _id: zoneId });
      if (zone) {
        const event = {
          deviceId,
          zoneId: zone._id,
          zoneName: zone.name,
          eventType: 'exit',
          location: {
            type: "Point",
            coordinates: location
          },
          timestamp: new Date()
        };

        await this.events.insertOne(event);
        events.push(event);

        await this.executeZoneRules(zone, deviceId, 'exit');
      }
    }

    return {
      currentZones: zonesContaining,
      entered,
      exited,
      events
    };
  }

  async executeZoneRules(zone, deviceId, eventType) {
    const rules = zone.rules[eventType] || [];

    for (const rule of rules) {
      switch (rule.action) {
        case 'notify':
          await this.sendNotification(deviceId, rule.message);
          break;

        case 'alert':
          await this.createAlert(deviceId, zone._id, rule.severity);
          break;

        case 'restrict':
          await this.restrictDevice(deviceId, rule.restrictions);
          break;

        case 'log':
          console.log(`Zone rule triggered: ${zone.name} - ${eventType}`);
          break;
      }
    }
  }

  async sendNotification(deviceId, message) {
    // Intégration avec système de notifications
    console.log(`Notification to ${deviceId}: ${message}`);
  }

  async createAlert(deviceId, zoneId, severity) {
    await this.db.collection('alerts').insertOne({
      deviceId,
      zoneId,
      severity,
      timestamp: new Date(),
      resolved: false
    });
  }

  async restrictDevice(deviceId, restrictions) {
    await this.devices.updateOne(
      { _id: deviceId },
      { $set: { restrictions, restrictedAt: new Date() } }
    );
  }

  async getDevicesInZone(zoneId) {
    const zone = await this.zones.findOne({ _id: zoneId });
    if (!zone) throw new Error('Zone not found');

    return await this.devices.find({
      currentLocation: {
        $geoWithin: {
          $geometry: zone.area
        }
      }
    }).toArray();
  }

  async getNearbyZones(location, maxDistance = 1000) {
    return await this.zones.find({
      area: {
        $near: {
          $geometry: {
            type: "Point",
            coordinates: location
          },
          $maxDistance: maxDistance
        }
      },
      isActive: true
    }).toArray();
  }

  async getZoneHistory(deviceId, startDate, endDate) {
    return await this.events.find({
      deviceId,
      timestamp: {
        $gte: startDate,
        $lte: endDate
      }
    })
      .sort({ timestamp: 1 })
      .toArray();
  }

  async analyzeDeviceMovement(deviceId, days = 7) {
    const since = new Date(Date.now() - days * 86400000);

    const movements = await this.events.aggregate([
      {
        $match: {
          deviceId,
          timestamp: { $gte: since }
        }
      },
      {
        $group: {
          _id: {
            zoneId: '$zoneId',
            zoneName: '$zoneName'
          },
          entries: {
            $sum: { $cond: [{ $eq: ['$eventType', 'enter'] }, 1, 0] }
          },
          exits: {
            $sum: { $cond: [{ $eq: ['$eventType', 'exit'] }, 1, 0] }
          },
          firstSeen: { $min: '$timestamp' },
          lastSeen: { $max: '$timestamp' }
        }
      },
      {
        $project: {
          zoneId: '$_id.zoneId',
          zoneName: '$_id.zoneName',
          entries: 1,
          exits: 1,
          firstSeen: 1,
          lastSeen: 1,
          _id: 0
        }
      },
      { $sort: { entries: -1 } }
    ]).toArray();

    return movements;
  }

  async createDeliveryZones(city) {
    // Exemple: créer zones de livraison pour une ville
    const zones = [
      {
        name: `${city} - Centre`,
        type: 'delivery',
        area: {
          type: "Polygon",
          coordinates: [[
            [2.3200, 48.8500],
            [2.3600, 48.8500],
            [2.3600, 48.8700],
            [2.3200, 48.8700],
            [2.3200, 48.8500]
          ]]
        },
        rules: {
          enter: [
            { action: 'notify', message: 'Entering delivery zone: Centre' }
          ]
        },
        metadata: {
          deliveryFee: 2.99,
          estimatedTime: 30
        }
      },
      {
        name: `${city} - Nord`,
        type: 'delivery',
        area: {
          type: "Polygon",
          coordinates: [[
            [2.3200, 48.8700],
            [2.3600, 48.8700],
            [2.3600, 48.8900],
            [2.3200, 48.8900],
            [2.3200, 48.8700]
          ]]
        },
        rules: {
          enter: [
            { action: 'notify', message: 'Entering delivery zone: Nord' }
          ]
        },
        metadata: {
          deliveryFee: 3.99,
          estimatedTime: 45
        }
      }
    ];

    for (const zone of zones) {
      await this.createZone(zone);
    }

    return zones.length;
  }
}

// Utilisation
const geofencing = new GeofencingService(db);
await geofencing.initialize();

// Créer une zone
const zone = await geofencing.createZone({
  name: 'Warehouse Area',
  type: 'restricted',
  area: {
    type: "Polygon",
    coordinates: [[
      [2.3200, 48.8500],
      [2.3400, 48.8500],
      [2.3400, 48.8600],
      [2.3200, 48.8600],
      [2.3200, 48.8500]
    ]]
  },
  rules: {
    enter: [
      { action: 'alert', severity: 'high' },
      { action: 'notify', message: 'Unauthorized entry' }
    ]
  }
});

// Vérifier position device
const check = await geofencing.checkDeviceInZones(
  'device-123',
  [2.3300, 48.8550]
);

console.log('Entered zones:', check.entered);
console.log('Exited zones:', check.exited);

// Analyser mouvement
const analysis = await geofencing.analyzeDeviceMovement('device-123', 7);
console.log('Most visited zones:', analysis);
```

### Cas 3 : Recherche de points d'intérêt avec agrégations géospatiales

```javascript
class POISearchService {
  constructor(db) {
    this.db = db;
    this.pois = null;
  }

  async initialize() {
    this.pois = this.db.collection('points_of_interest');

    // Index géospatial composé
    await this.pois.createIndex({
      location: "2dsphere",
      category: 1,
      rating: -1
    });

    await this.pois.createIndex({ tags: 1 });
  }

  async searchNearby(userLocation, options = {}) {
    const {
      maxDistance = 5000,
      categories = [],
      tags = [],
      minRating = 0,
      priceRange = null,
      limit = 50
    } = options;

    const pipeline = [
      {
        $geoNear: {
          near: {
            type: "Point",
            coordinates: userLocation
          },
          distanceField: "distance",
          maxDistance: maxDistance,
          spherical: true,
          query: this.buildQuery(categories, tags, minRating, priceRange)
        }
      },
      {
        $addFields: {
          distanceKm: { $round: [{ $divide: ['$distance', 1000] }, 2] }
        }
      },
      { $limit: limit }
    ];

    return await this.pois.aggregate(pipeline).toArray();
  }

  buildQuery(categories, tags, minRating, priceRange) {
    const query = {};

    if (categories.length > 0) {
      query.category = { $in: categories };
    }

    if (tags.length > 0) {
      query.tags = { $in: tags };
    }

    if (minRating > 0) {
      query.rating = { $gte: minRating };
    }

    if (priceRange) {
      query.priceLevel = { $gte: priceRange.min, $lte: priceRange.max };
    }

    return query;
  }

  async searchByRoute(routeCoordinates, buffer = 1000, options = {}) {
    // Chercher POIs le long d'une route avec buffer
    const routeLine = {
      type: "LineString",
      coordinates: routeCoordinates
    };

    // Créer buffer autour de la route
    const query = {
      location: {
        $geoWithin: {
          $geometry: this.createBuffer(routeLine, buffer)
        }
      }
    };

    if (options.categories) {
      query.category = { $in: options.categories };
    }

    return await this.pois.find(query).toArray();
  }

  createBuffer(geometry, bufferMeters) {
    // Simplification - dans production utiliser turf.js
    // Retourne un polygon approximatif autour de la géométrie
    // Pour l'exemple, on retourne juste la géométrie
    return geometry;
  }

  async clusterPOIs(center, radius, clusterRadius = 500) {
    // Grouper POIs proches géographiquement
    return await this.pois.aggregate([
      {
        $geoNear: {
          near: { type: "Point", coordinates: center },
          distanceField: "distance",
          maxDistance: radius,
          spherical: true
        }
      },
      {
        $addFields: {
          // Grouper par grille
          gridX: {
            $floor: {
              $divide: [
                { $arrayElemAt: ['$location.coordinates', 0] },
                clusterRadius / 111320  // Degrés approximatifs
              ]
            }
          },
          gridY: {
            $floor: {
              $divide: [
                { $arrayElemAt: ['$location.coordinates', 1] },
                clusterRadius / 111320
              ]
            }
          }
        }
      },
      {
        $group: {
          _id: { gridX: '$gridX', gridY: '$gridY' },
          count: { $sum: 1 },
          pois: { $push: '$$ROOT' },
          centerLng: { $avg: { $arrayElemAt: ['$location.coordinates', 0] } },
          centerLat: { $avg: { $arrayElemAt: ['$location.coordinates', 1] } },
          categories: { $addToSet: '$category' },
          avgRating: { $avg: '$rating' }
        }
      },
      {
        $project: {
          _id: 0,
          cluster: {
            center: {
              type: "Point",
              coordinates: ['$centerLng', '$centerLat']
            },
            count: '$count',
            categories: '$categories',
            avgRating: '$avgRating'
          },
          pois: {
            $cond: {
              if: { $gt: ['$count', 10] },
              then: { $slice: ['$pois', 10] },  // Limiter si cluster large
              else: '$pois'
            }
          }
        }
      }
    ]).toArray();
  }

  async findOptimalMeetingPoint(locations, options = {}) {
    // Trouver POI optimal pour rencontre entre plusieurs personnes
    const {
      category = 'restaurant',
      minRating = 4.0
    } = options;

    // Calculer centre géométrique
    const center = this.calculateCentroid(locations);

    // Calculer rayon maximum (distance du point le plus éloigné)
    const maxDistance = Math.max(...locations.map(loc =>
      this.calculateDistance(center, loc)
    ));

    // Chercher POIs dans cette zone
    const pois = await this.pois.find({
      location: {
        $geoWithin: {
          $centerSphere: [
            center,
            maxDistance / 6378100  // Convertir en radians
          ]
        }
      },
      category,
      rating: { $gte: minRating }
    }).toArray();

    // Scorer chaque POI basé sur distance totale pour tous
    const scored = pois.map(poi => {
      const totalDistance = locations.reduce((sum, loc) =>
        sum + this.calculateDistance(poi.location.coordinates, loc),
        0
      );

      const avgDistance = totalDistance / locations.length;

      // Score combinant distance moyenne et rating
      const score = (
        (1 - avgDistance / maxDistance) * 0.6 +
        (poi.rating / 5) * 0.4
      );

      return {
        ...poi,
        totalDistance,
        avgDistance,
        score,
        distancesFromEach: locations.map(loc => ({
          distance: this.calculateDistance(poi.location.coordinates, loc),
          time: this.estimateTime(
            this.calculateDistance(poi.location.coordinates, loc),
            30
          )
        }))
      };
    }).sort((a, b) => b.score - a.score);

    return scored;
  }

  calculateCentroid(locations) {
    const sum = locations.reduce((acc, loc) => ({
      lng: acc.lng + loc[0],
      lat: acc.lat + loc[1]
    }), { lng: 0, lat: 0 });

    return [
      sum.lng / locations.length,
      sum.lat / locations.length
    ];
  }

  calculateDistance(coord1, coord2) {
    const R = 6371e3;
    const φ1 = coord1[1] * Math.PI / 180;
    const φ2 = coord2[1] * Math.PI / 180;
    const Δφ = (coord2[1] - coord1[1]) * Math.PI / 180;
    const Δλ = (coord2[0] - coord1[0]) * Math.PI / 180;

    const a = Math.sin(Δφ / 2) * Math.sin(Δφ / 2) +
              Math.cos(φ1) * Math.cos(φ2) *
              Math.sin(Δλ / 2) * Math.sin(Δλ / 2);

    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

    return R * c;
  }

  estimateTime(distanceMeters, speedKmh) {
    return (distanceMeters / 1000) / speedKmh * 60;
  }

  async getHeatmapData(bounds, gridSize = 0.01) {
    // Générer données pour heatmap
    const { minLng, minLat, maxLng, maxLat } = bounds;

    return await this.pois.aggregate([
      {
        $match: {
          location: {
            $geoWithin: {
              $box: [
                [minLng, minLat],
                [maxLng, maxLat]
              ]
            }
          }
        }
      },
      {
        $addFields: {
          gridX: {
            $floor: {
              $divide: [
                { $subtract: [{ $arrayElemAt: ['$location.coordinates', 0] }, minLng] },
                gridSize
              ]
            }
          },
          gridY: {
            $floor: {
              $divide: [
                { $subtract: [{ $arrayElemAt: ['$location.coordinates', 1] }, minLat] },
                gridSize
              ]
            }
          }
        }
      },
      {
        $group: {
          _id: { x: '$gridX', y: '$gridY' },
          count: { $sum: 1 },
          avgRating: { $avg: '$rating' },
          categories: { $addToSet: '$category' }
        }
      },
      {
        $project: {
          _id: 0,
          x: '$_id.x',
          y: '$_id.y',
          lng: { $add: [minLng, { $multiply: ['$_id.x', gridSize] }] },
          lat: { $add: [minLat, { $multiply: ['$_id.y', gridSize] }] },
          intensity: '$count',
          avgRating: 1,
          categories: 1
        }
      }
    ]).toArray();
  }

  async analyzeDensity(category, city) {
    // Analyser densité de POIs par zone
    return await this.pois.aggregate([
      {
        $match: {
          category,
          'address.city': city
        }
      },
      {
        $group: {
          _id: {
            district: '$address.district',
            neighborhood: '$address.neighborhood'
          },
          count: { $sum: 1 },
          avgRating: { $avg: '$rating' },
          locations: {
            $push: '$location'
          }
        }
      },
      {
        $addFields: {
          density: {
            $divide: ['$count', 1]  // Simplification
          }
        }
      },
      { $sort: { count: -1 } }
    ]).toArray();
  }
}

// Utilisation
const poiSearch = new POISearchService(db);
await poiSearch.initialize();

// Recherche proximity standard
const nearby = await poiSearch.searchNearby(
  [2.3522, 48.8566],
  {
    maxDistance: 2000,
    categories: ['restaurant', 'cafe'],
    minRating: 4.0,
    tags: ['wifi', 'outdoor-seating']
  }
);

// POIs le long d'une route
const route = [
  [2.3522, 48.8566],
  [2.3376, 48.8606],
  [2.3200, 48.8650]
];

const alongRoute = await poiSearch.searchByRoute(route, 500, {
  categories: ['gas_station', 'restaurant']
});

// Clustering
const clusters = await poiSearch.clusterPOIs(
  [2.3522, 48.8566],
  5000,
  500
);

// Point de rencontre optimal
const meetingPoint = await poiSearch.findOptimalMeetingPoint(
  [
    [2.3522, 48.8566],  // Personne 1
    [2.2945, 48.8584],  // Personne 2
    [2.3376, 48.8606]   // Personne 3
  ],
  {
    category: 'restaurant',
    minRating: 4.5
  }
);

console.log('Best meeting point:', meetingPoint[0]);

// Heatmap
const heatmap = await poiSearch.getHeatmapData({
  minLng: 2.30,
  minLat: 48.85,
  maxLng: 2.40,
  maxLat: 48.88
}, 0.005);
```

---

## Agrégations géospatiales avancées

### $geoNear dans aggregation pipeline

```javascript
// Agrégation avec distance calculée
const results = await db.collection('stores').aggregate([
  {
    $geoNear: {
      near: { type: "Point", coordinates: [2.3522, 48.8566] },
      distanceField: "distance",
      maxDistance: 10000,
      spherical: true,
      query: { category: "supermarket" }
    }
  },
  {
    $addFields: {
      distanceKm: { $round: [{ $divide: ['$distance', 1000] }, 2] }
    }
  },
  {
    $group: {
      _id: '$chain',
      storeCount: { $sum: 1 },
      avgDistance: { $avg: '$distance' },
      nearestStore: { $min: '$distance' },
      stores: {
        $push: {
          name: '$name',
          distance: '$distanceKm'
        }
      }
    }
  },
  { $sort: { avgDistance: 1 } }
]).toArray();
```

### Calculs géométriques complexes

```javascript
async function analyzeServiceCoverage(db, servicePoints, city Boundary) {
  // Analyser couverture d'une zone par des points de service
  return await db.collection('service_analysis').aggregate([
    {
      $geoNear: {
        near: cityBoundary.centroid,
        distanceField: "distanceFromCenter",
        maxDistance: 50000,
        spherical: true
      }
    },
    {
      $addFields: {
        isInCity: {
          $let: {
            vars: {
              pointInPolygon: {
                $function: {
                  body: function(point, polygon) {
                    // Algorithm point-in-polygon
                    // Simplification
                    return true;
                  },
                  args: ['$location', cityBoundary],
                  lang: 'js'
                }
              }
            },
            in: '$$pointInPolygon'
          }
        }
      }
    },
    {
      $match: { isInCity: true }
    },
    {
      $facet: {
        coverage: [
          {
            $bucket: {
              groupBy: '$distanceFromCenter',
              boundaries: [0, 5000, 10000, 15000, 20000, 50000],
              default: 'Other',
              output: {
                count: { $sum: 1 },
                avgRating: { $avg: '$rating' }
              }
            }
          }
        ],
        byType: [
          {
            $group: {
              _id: '$serviceType',
              count: { $sum: 1 },
              avgDistance: { $avg: '$distanceFromCenter' }
            }
          }
        ]
      }
    }
  ]).toArray();
}
```

---

## Performance et optimisations

### Best practices pour index géospatiaux

```javascript
// ✅ Index 2dsphere sur champ GeoJSON
await collection.createIndex({ location: "2dsphere" });

// ✅ Index composé pour requêtes filtrées
await collection.createIndex({
  location: "2dsphere",
  category: 1,
  isActive: 1
});

// ✅ Ordre: geo first pour $geoNear
await collection.createIndex({ location: "2dsphere", rating: -1 });

// ⚠️ Limite: 1 index 2dsphere par collection (mais peut être composé)

// ✅ Index partiel pour sous-ensemble
await collection.createIndex(
  { location: "2dsphere" },
  {
    partialFilterExpression: {
      category: { $in: ['restaurant', 'cafe'] },
      isActive: true
    }
  }
);
```

### Optimisation des requêtes

```javascript
// ❌ LENT: Scan puis distance
const slow = await collection.find({ category: 'restaurant' })
  .toArray()
  .then(docs => docs.filter(d =>
    calculateDistance(userLoc, d.location.coordinates) < 5000
  ));

// ✅ RAPIDE: Index geo + filtre
const fast = await collection.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: userLoc },
      $maxDistance: 5000
    }
  },
  category: 'restaurant'
}).toArray();

// ✅ OPTIMAL: $geoNear en aggregation (support query)
const optimal = await collection.aggregate([
  {
    $geoNear: {
      near: { type: "Point", coordinates: userLoc },
      distanceField: "distance",
      maxDistance: 5000,
      spherical: true,
      query: { category: 'restaurant', isOpen: true }
    }
  }
]).toArray();
```

### Caching de résultats géospatiaux

```javascript
class GeoQueryCache {
  constructor(redisClient, ttl = 300) {
    this.redis = redisClient;
    this.ttl = ttl;
  }

  generateKey(location, radius, filters = {}) {
    // Arrondir coordonnées pour cache hits
    const roundedLoc = [
      Math.round(location[0] * 100) / 100,
      Math.round(location[1] * 100) / 100
    ];

    return `geo:${roundedLoc.join(',')}:${radius}:${JSON.stringify(filters)}`;
  }

  async get(location, radius, filters) {
    const key = this.generateKey(location, radius, filters);
    const cached = await this.redis.get(key);

    if (cached) {
      return { cached: true, data: JSON.parse(cached) };
    }

    return { cached: false };
  }

  async set(location, radius, filters, data) {
    const key = this.generateKey(location, radius, filters);
    await this.redis.setex(key, this.ttl, JSON.stringify(data));
  }

  async findNearby(collection, location, radius, filters = {}) {
    // Check cache
    const cached = await this.get(location, radius, filters);
    if (cached.cached) {
      return cached.data;
    }

    // Query database
    const results = await collection.find({
      location: {
        $near: {
          $geometry: { type: "Point", coordinates: location },
          $maxDistance: radius
        }
      },
      ...filters
    }).toArray();

    // Cache results
    await this.set(location, radius, filters, results);

    return results;
  }
}
```

---

## Bonnes pratiques de production

### ✅ DO (À faire)

```javascript
// 1. Toujours utiliser GeoJSON format correct
const location = {
  type: "Point",
  coordinates: [longitude, latitude]  // ✅ [lng, lat]
};

// 2. Créer index 2dsphere
await collection.createIndex({ location: "2dsphere" });

// 3. Utiliser $geoNear en aggregation pour meilleure performance
const results = await collection.aggregate([
  {
    $geoNear: {
      near: { type: "Point", coordinates: userLoc },
      distanceField: "distance",
      spherical: true,
      query: filters
    }
  }
]).toArray();

// 4. Valider coordonnées
function validateCoordinates(lng, lat) {
  return lng >= -180 && lng <= 180 && lat >= -90 && lat <= 90;
}

// 5. Gérer erreurs de distance
try {
  const distance = calculateDistance(coord1, coord2);
  if (isNaN(distance)) {
    throw new Error('Invalid distance calculation');
  }
} catch (error) {
  console.error('Distance error:', error);
}
```

### ❌ DON'T (À éviter)

```javascript
// 1. Ne pas inverser lng/lat
// ❌ INCORRECT
const wrong = {
  type: "Point",
  coordinates: [latitude, longitude]  // Inversé!
};

// 2. Ne pas utiliser coordinates plates
// ❌ Legacy format
const legacy = {
  loc: [lng, lat]  // Pas GeoJSON
};

// 3. Ne pas oublier spherical: true
// ❌ Calculs incorrects
const incorrect = await collection.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: userLoc }
      // Manque spherical: true
    }
  }
});

// 4. Ne pas faire calculs côté application
// ❌ LENT
const allDocs = await collection.find().toArray();
const filtered = allDocs.filter(d =>
  calculateDistance(userLoc, d.location.coordinates) < 5000
);

// 5. Ne pas oublier limites de distance
// ❌ Peut retourner trop de résultats
const unlimited = await collection.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: userLoc }
      // Manque $maxDistance
    }
  }
});
```

---

## Outils et bibliothèques recommandés

### Turf.js - Géométrie avancée

```javascript
const turf = require('@turf/turf');

// Créer buffer autour d'un point
const point = turf.point([2.3522, 48.8566]);
const buffered = turf.buffer(point, 5, { units: 'kilometers' });

// Calculer surface
const polygon = turf.polygon([[
  [2.32, 48.85],
  [2.36, 48.85],
  [2.36, 48.87],
  [2.32, 48.87],
  [2.32, 48.85]
]]);
const area = turf.area(polygon);  // En mètres carrés

// Point dans polygon
const pt = turf.point([2.34, 48.86]);
const poly = polygon;
const isInside = turf.booleanPointInPolygon(pt, poly);

// Distance
const from = turf.point([2.3522, 48.8566]);
const to = turf.point([2.2945, 48.8584]);
const distance = turf.distance(from, to, { units: 'kilometers' });
```

---

## Conclusion

Les requêtes géospatiales de MongoDB offrent :
- ✅ **Support GeoJSON complet** (Point, LineString, Polygon, etc.)
- ✅ **Index 2dsphere optimisés** pour calculs sphériques
- ✅ **Opérateurs riches** ($near, $geoWithin, $geoIntersects)
- ✅ **Agrégations géospatiales** ($geoNear dans pipeline)
- ✅ **Performance excellente** avec index appropriés

**Points clés à retenir :**
1. Toujours utiliser format GeoJSON standard
2. Ordre: [longitude, latitude] (pas l'inverse!)
3. Index 2dsphere pour données terre ronde
4. $geoNear en aggregation pour meilleur contrôle
5. Valider coordonnées et gérer erreurs
6. Utiliser Turf.js pour opérations complexes

**Cas d'usage idéaux :**
- Applications de livraison/transport
- Recherche de proximité (magasins, restaurants)
- Géofencing et alertes de zone
- Analyse de mobilité et tracking
- Optimisation de routes
- Points d'intérêt et recommandations

---


⏭️ [Recherche Full-Text avancée](/16-fonctionnalites-avancees/07-recherche-full-text-avancee.md)
