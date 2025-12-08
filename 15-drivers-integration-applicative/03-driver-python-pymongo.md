🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.3 Driver Python (PyMongo)

## Introduction

PyMongo est le driver officiel MongoDB pour Python. Il offre une API pythonique, intuitive et puissante pour interagir avec MongoDB. Cette section couvre à la fois **PyMongo** (synchrone) et **Motor** (asynchrone), ainsi que les bonnes pratiques pour des applications Python de niveau production.

## Installation et configuration

### Installation de base

```bash
# PyMongo (synchrone)
pip install pymongo

# Pour le support TLS/SSL
pip install pymongo[srv]

# Motor (asynchrone avec asyncio)
pip install motor

# Pour le support de la compression
pip install pymongo[snappy,zstd]

# Versions recommandées
# PyMongo: 4.x
# Motor: 3.x
# Python minimum: 3.7+
```

### Requirements.txt pour production

```txt
# requirements.txt
pymongo==4.6.0
motor==3.3.2
python-dotenv==1.0.0
pydantic==2.5.0  # Pour la validation
dnspython==2.4.2  # Pour mongodb+srv://

# Optionnel mais recommandé
certifi==2023.11.17  # Certificats SSL
```

### Configuration avec variables d'environnement

```python
# config/database.py
import os
from typing import Optional
from pymongo import MongoClient
from pymongo.server_api import ServerApi
from dotenv import load_dotenv

load_dotenv()

class DatabaseConfig:
    """Configuration centralisée de la base de données"""

    def __init__(self):
        self.uri = os.getenv('MONGODB_URI', 'mongodb://localhost:27017/')
        self.database_name = os.getenv('MONGODB_DATABASE', 'myapp')

        # Configuration du pool de connexions
        self.max_pool_size = int(os.getenv('MONGODB_MAX_POOL_SIZE', '50'))
        self.min_pool_size = int(os.getenv('MONGODB_MIN_POOL_SIZE', '10'))

        # Timeouts
        self.server_selection_timeout_ms = int(
            os.getenv('MONGODB_SERVER_SELECTION_TIMEOUT', '5000')
        )
        self.socket_timeout_ms = int(
            os.getenv('MONGODB_SOCKET_TIMEOUT', '45000')
        )
        self.connect_timeout_ms = int(
            os.getenv('MONGODB_CONNECT_TIMEOUT', '10000')
        )

    def get_client_options(self) -> dict:
        """Retourne les options du client MongoDB"""
        return {
            'maxPoolSize': self.max_pool_size,
            'minPoolSize': self.min_pool_size,
            'serverSelectionTimeoutMS': self.server_selection_timeout_ms,
            'socketTimeoutMS': self.socket_timeout_ms,
            'connectTimeoutMS': self.connect_timeout_ms,
            'retryWrites': True,
            'retryReads': True,
            'server_api': ServerApi('1')  # Stable API
        }

# Instance singleton
db_config = DatabaseConfig()
```

```env
# .env
MONGODB_URI=mongodb://localhost:27017/
MONGODB_DATABASE=myapp
MONGODB_MAX_POOL_SIZE=50
MONGODB_MIN_POOL_SIZE=10
```

## PyMongo - Driver synchrone

### Connection Management (Pattern Singleton)

```python
# database/mongodb.py
from typing import Optional
from pymongo import MongoClient
from pymongo.database import Database
from pymongo.errors import ConnectionFailure, ServerSelectionTimeoutError
from config.database import db_config
import logging

logger = logging.getLogger(__name__)

class MongoDBConnection:
    """Gestionnaire de connexion MongoDB (Singleton)"""

    _instance: Optional['MongoDBConnection'] = None
    _client: Optional[MongoClient] = None
    _db: Optional[Database] = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def connect(self) -> Database:
        """Établit la connexion à MongoDB"""
        if self._client is None:
            try:
                self._client = MongoClient(
                    db_config.uri,
                    **db_config.get_client_options()
                )

                # Vérifier la connexion
                self._client.admin.command('ping')
                self._db = self._client[db_config.database_name]

                logger.info("✅ Connecté à MongoDB")

            except ConnectionFailure as e:
                logger.error(f"❌ Échec de connexion MongoDB: {e}")
                raise
            except ServerSelectionTimeoutError as e:
                logger.error(f"❌ Timeout de sélection du serveur: {e}")
                raise

        return self._db

    def get_database(self) -> Database:
        """Retourne la base de données"""
        if self._db is None:
            return self.connect()
        return self._db

    def get_client(self) -> MongoClient:
        """Retourne le client MongoDB"""
        if self._client is None:
            self.connect()
        return self._client

    def close(self):
        """Ferme la connexion"""
        if self._client:
            self._client.close()
            self._client = None
            self._db = None
            logger.info("🔌 Connexion MongoDB fermée")

# Instance globale
mongodb = MongoDBConnection()

# Helper pour obtenir la database
def get_db() -> Database:
    return mongodb.get_database()
```

### Modèles de données avec Pydantic

```python
# models/user.py
from datetime import datetime
from typing import Optional, List
from enum import Enum
from pydantic import BaseModel, EmailStr, Field, ConfigDict
from bson import ObjectId

class PyObjectId(ObjectId):
    """Type personnalisé pour ObjectId avec Pydantic"""

    @classmethod
    def __get_validators__(cls):
        yield cls.validate

    @classmethod
    def validate(cls, v, info):
        if not ObjectId.is_valid(v):
            raise ValueError("Invalid ObjectId")
        return ObjectId(v)

    @classmethod
    def __get_pydantic_core_schema__(cls, source_type, handler):
        from pydantic_core import core_schema
        return core_schema.union_schema([
            core_schema.is_instance_schema(ObjectId),
            core_schema.no_info_plain_validator_function(cls.validate),
        ])

class UserStatus(str, Enum):
    ACTIVE = "active"
    INACTIVE = "inactive"
    SUSPENDED = "suspended"

class UserPreferences(BaseModel):
    """Préférences utilisateur"""
    language: str = "en"
    timezone: str = "UTC"
    notifications: bool = True

class UserBase(BaseModel):
    """Schéma de base pour un utilisateur"""
    name: str = Field(..., min_length=2, max_length=100)
    email: EmailStr
    age: Optional[int] = Field(None, ge=18, le=120)
    status: UserStatus = UserStatus.ACTIVE
    roles: List[str] = Field(default_factory=lambda: ["user"])
    preferences: Optional[UserPreferences] = None

class UserCreate(UserBase):
    """Schéma pour la création d'utilisateur"""
    password: str = Field(..., min_length=8)

class UserUpdate(BaseModel):
    """Schéma pour la mise à jour d'utilisateur"""
    name: Optional[str] = Field(None, min_length=2, max_length=100)
    email: Optional[EmailStr] = None
    age: Optional[int] = Field(None, ge=18, le=120)
    status: Optional[UserStatus] = None
    preferences: Optional[UserPreferences] = None

class UserInDB(UserBase):
    """Schéma pour utilisateur en base de données"""
    id: PyObjectId = Field(default_factory=PyObjectId, alias="_id")
    created_at: datetime
    updated_at: datetime
    last_login: Optional[datetime] = None
    login_count: int = 0

    model_config = ConfigDict(
        populate_by_name=True,
        arbitrary_types_allowed=True,
        json_encoders={ObjectId: str}
    )

class UserPublic(BaseModel):
    """Schéma public (sans données sensibles)"""
    id: str = Field(alias="_id")
    name: str
    email: str
    status: UserStatus
    created_at: datetime

    model_config = ConfigDict(populate_by_name=True)
```

### Repository Pattern

```python
# repositories/base_repository.py
from typing import TypeVar, Generic, List, Optional, Dict, Any
from pymongo.collection import Collection
from pymongo.results import InsertOneResult, UpdateResult, DeleteResult
from bson import ObjectId
from pydantic import BaseModel

T = TypeVar('T', bound=BaseModel)

class BaseRepository(Generic[T]):
    """Repository de base générique"""

    def __init__(self, collection: Collection):
        self.collection = collection

    def find_by_id(self, id: str | ObjectId) -> Optional[Dict[str, Any]]:
        """Recherche par ID"""
        object_id = ObjectId(id) if isinstance(id, str) else id
        return self.collection.find_one({"_id": object_id})

    def find_one(self, filter: Dict[str, Any]) -> Optional[Dict[str, Any]]:
        """Recherche un document"""
        return self.collection.find_one(filter)

    def find_many(
        self,
        filter: Dict[str, Any] = None,
        skip: int = 0,
        limit: int = 100,
        sort: List[tuple] = None
    ) -> List[Dict[str, Any]]:
        """Recherche plusieurs documents"""
        filter = filter or {}
        cursor = self.collection.find(filter)

        if sort:
            cursor = cursor.sort(sort)

        cursor = cursor.skip(skip).limit(limit)
        return list(cursor)

    def create(self, document: Dict[str, Any]) -> InsertOneResult:
        """Crée un document"""
        return self.collection.insert_one(document)

    def update(
        self,
        filter: Dict[str, Any],
        update: Dict[str, Any]
    ) -> UpdateResult:
        """Met à jour un document"""
        return self.collection.update_one(filter, update)

    def update_by_id(
        self,
        id: str | ObjectId,
        update: Dict[str, Any]
    ) -> UpdateResult:
        """Met à jour par ID"""
        object_id = ObjectId(id) if isinstance(id, str) else id
        return self.collection.update_one({"_id": object_id}, update)

    def delete(self, filter: Dict[str, Any]) -> DeleteResult:
        """Supprime un document"""
        return self.collection.delete_one(filter)

    def delete_by_id(self, id: str | ObjectId) -> DeleteResult:
        """Supprime par ID"""
        object_id = ObjectId(id) if isinstance(id, str) else id
        return self.collection.delete_one({"_id": object_id})

    def count(self, filter: Dict[str, Any] = None) -> int:
        """Compte les documents"""
        filter = filter or {}
        return self.collection.count_documents(filter)

    def exists(self, filter: Dict[str, Any]) -> bool:
        """Vérifie l'existence d'un document"""
        return self.collection.count_documents(filter, limit=1) > 0
```

```python
# repositories/user_repository.py
from datetime import datetime
from typing import List, Optional, Dict, Any
from pymongo.collection import Collection
from pymongo.errors import DuplicateKeyError
from bson import ObjectId
from .base_repository import BaseRepository
from models.user import UserCreate, UserInDB, UserUpdate, UserStatus

class UserRepository(BaseRepository):
    """Repository pour les utilisateurs"""

    def __init__(self, collection: Collection):
        super().__init__(collection)
        self._ensure_indexes()

    def _ensure_indexes(self):
        """Crée les index nécessaires"""
        # Index unique sur email
        self.collection.create_index("email", unique=True)

        # Index sur status pour les recherches
        self.collection.create_index("status")

        # Index composé pour recherche et tri
        self.collection.create_index([
            ("status", 1),
            ("created_at", -1)
        ])

        # Index texte pour recherche full-text
        self.collection.create_index([
            ("name", "text"),
            ("email", "text")
        ])

    def create_user(self, user_data: UserCreate) -> UserInDB:
        """Crée un nouvel utilisateur"""
        now = datetime.utcnow()

        user_dict = user_data.model_dump(exclude={'password'})
        user_dict.update({
            'created_at': now,
            'updated_at': now,
            'login_count': 0
        })

        try:
            result = self.create(user_dict)
            user_dict['_id'] = result.inserted_id
            return UserInDB(**user_dict)
        except DuplicateKeyError:
            raise ValueError(f"Email {user_data.email} already exists")

    def find_by_email(self, email: str) -> Optional[UserInDB]:
        """Recherche par email"""
        user_dict = self.find_one({"email": email})
        return UserInDB(**user_dict) if user_dict else None

    def find_active_users(
        self,
        limit: int = 100,
        skip: int = 0
    ) -> List[UserInDB]:
        """Récupère les utilisateurs actifs"""
        users = self.find_many(
            filter={"status": UserStatus.ACTIVE.value},
            skip=skip,
            limit=limit,
            sort=[("created_at", -1)]
        )
        return [UserInDB(**user) for user in users]

    def update_profile(
        self,
        user_id: str | ObjectId,
        updates: UserUpdate
    ) -> bool:
        """Met à jour le profil utilisateur"""
        update_data = updates.model_dump(exclude_unset=True)

        if not update_data:
            return False

        update_data['updated_at'] = datetime.utcnow()

        result = self.update_by_id(
            user_id,
            {"$set": update_data}
        )

        return result.modified_count > 0

    def increment_login_count(self, user_id: str | ObjectId):
        """Incrémente le compteur de connexions"""
        self.update_by_id(
            user_id,
            {
                "$inc": {"login_count": 1},
                "$set": {
                    "last_login": datetime.utcnow(),
                    "updated_at": datetime.utcnow()
                }
            }
        )

    def search_users(
        self,
        search_term: str,
        limit: int = 20
    ) -> List[UserInDB]:
        """Recherche full-text d'utilisateurs"""
        users = self.find_many(
            filter={"$text": {"$search": search_term}},
            limit=limit,
            sort=[("score", {"$meta": "textScore"})]
        )
        return [UserInDB(**user) for user in users]

    def find_by_role(self, role: str) -> List[UserInDB]:
        """Recherche par rôle"""
        users = self.find_many({"roles": role})
        return [UserInDB(**user) for user in users]

    def bulk_update_status(
        self,
        user_ids: List[ObjectId],
        status: UserStatus
    ) -> int:
        """Met à jour le statut de plusieurs utilisateurs"""
        result = self.collection.update_many(
            {"_id": {"$in": user_ids}},
            {
                "$set": {
                    "status": status.value,
                    "updated_at": datetime.utcnow()
                }
            }
        )
        return result.modified_count

    def get_statistics(self) -> Dict[str, Any]:
        """Récupère les statistiques utilisateurs"""
        pipeline = [
            {
                "$facet": {
                    "total": [{"$count": "count"}],
                    "by_status": [
                        {
                            "$group": {
                                "_id": "$status",
                                "count": {"$sum": 1}
                            }
                        }
                    ],
                    "recent_signups": [
                        {
                            "$match": {
                                "created_at": {
                                    "$gte": datetime.utcnow().replace(
                                        day=1, hour=0, minute=0, second=0
                                    )
                                }
                            }
                        },
                        {"$count": "count"}
                    ],
                    "top_active": [
                        {"$sort": {"login_count": -1}},
                        {"$limit": 10},
                        {
                            "$project": {
                                "name": 1,
                                "email": 1,
                                "login_count": 1
                            }
                        }
                    ]
                }
            }
        ]

        result = list(self.collection.aggregate(pipeline))[0]

        return {
            "total": result["total"][0]["count"] if result["total"] else 0,
            "by_status": {
                item["_id"]: item["count"]
                for item in result["by_status"]
            },
            "recent_signups": (
                result["recent_signups"][0]["count"]
                if result["recent_signups"] else 0
            ),
            "top_active_users": result["top_active"]
        }
```

### Service Layer

```python
# services/user_service.py
from typing import List, Optional
from datetime import datetime
from bson import ObjectId
from repositories.user_repository import UserRepository
from models.user import (
    UserCreate, UserInDB, UserUpdate,
    UserPublic, UserStatus
)
import logging

logger = logging.getLogger(__name__)

class UserService:
    """Service métier pour les utilisateurs"""

    def __init__(self, user_repository: UserRepository):
        self.repository = user_repository

    def register_user(self, user_data: UserCreate) -> UserPublic:
        """Enregistre un nouvel utilisateur"""
        # Vérifier si l'email existe
        existing_user = self.repository.find_by_email(user_data.email)
        if existing_user:
            raise ValueError(f"Email {user_data.email} already registered")

        # Créer l'utilisateur
        user = self.repository.create_user(user_data)

        # TODO: Envoyer email de bienvenue
        logger.info(f"New user registered: {user.email}")

        return UserPublic(
            _id=str(user.id),
            name=user.name,
            email=user.email,
            status=user.status,
            created_at=user.created_at
        )

    def authenticate_user(self, email: str, password: str) -> UserInDB:
        """Authentifie un utilisateur"""
        user = self.repository.find_by_email(email)

        if not user:
            raise ValueError("Invalid credentials")

        if user.status != UserStatus.ACTIVE:
            raise ValueError("Account is not active")

        # TODO: Vérifier le mot de passe hashé
        # verify_password(password, user.password_hash)

        # Incrémenter le compteur de connexions
        self.repository.increment_login_count(user.id)

        logger.info(f"User logged in: {user.email}")
        return user

    def get_user_by_id(self, user_id: str) -> Optional[UserInDB]:
        """Récupère un utilisateur par ID"""
        user_dict = self.repository.find_by_id(user_id)
        return UserInDB(**user_dict) if user_dict else None

    def update_user(
        self,
        user_id: str,
        updates: UserUpdate
    ) -> UserPublic:
        """Met à jour un utilisateur"""
        success = self.repository.update_profile(user_id, updates)

        if not success:
            raise ValueError("Failed to update user")

        user = self.get_user_by_id(user_id)
        if not user:
            raise ValueError("User not found")

        return UserPublic(
            _id=str(user.id),
            name=user.name,
            email=user.email,
            status=user.status,
            created_at=user.created_at
        )

    def search_users(self, query: str) -> List[UserPublic]:
        """Recherche d'utilisateurs"""
        users = self.repository.search_users(query)
        return [
            UserPublic(
                _id=str(user.id),
                name=user.name,
                email=user.email,
                status=user.status,
                created_at=user.created_at
            )
            for user in users
        ]

    def get_dashboard_stats(self) -> dict:
        """Récupère les statistiques du dashboard"""
        return self.repository.get_statistics()

    def suspend_user(self, user_id: str) -> bool:
        """Suspend un utilisateur"""
        updates = UserUpdate(status=UserStatus.SUSPENDED)
        return self.repository.update_profile(user_id, updates)
```

### Opérations CRUD avancées

```python
# Recherche avec pagination et filtres
from typing import List, Dict, Any, Optional
from dataclasses import dataclass

@dataclass
class PaginationParams:
    page: int = 1
    page_size: int = 20

    @property
    def skip(self) -> int:
        return (self.page - 1) * self.page_size

@dataclass
class PaginatedResult:
    data: List[Dict[str, Any]]
    total: int
    page: int
    page_size: int
    total_pages: int

def search_products(
    filters: Dict[str, Any],
    pagination: PaginationParams,
    sort_by: str = "created_at",
    sort_order: int = -1
) -> PaginatedResult:
    """Recherche paginée de produits"""
    collection = get_db()['products']

    # Construction du filtre
    query = {}

    if 'category' in filters:
        query['category'] = filters['category']

    if 'min_price' in filters or 'max_price' in filters:
        query['price.amount'] = {}
        if 'min_price' in filters:
            query['price.amount']['$gte'] = filters['min_price']
        if 'max_price' in filters:
            query['price.amount']['$lte'] = filters['max_price']

    if 'tags' in filters:
        query['tags'] = {'$all': filters['tags']}

    if 'search' in filters:
        query['$or'] = [
            {'name': {'$regex': filters['search'], '$options': 'i'}},
            {'description': {'$regex': filters['search'], '$options': 'i'}}
        ]

    # Compter le total
    total = collection.count_documents(query)

    # Récupérer les données
    cursor = collection.find(query)
    cursor = cursor.sort(sort_by, sort_order)
    cursor = cursor.skip(pagination.skip).limit(pagination.page_size)

    data = list(cursor)

    return PaginatedResult(
        data=data,
        total=total,
        page=pagination.page,
        page_size=pagination.page_size,
        total_pages=(total + pagination.page_size - 1) // pagination.page_size
    )
```

```python
# Mise à jour avec opérateurs multiples
def update_product_inventory(
    product_id: ObjectId,
    quantity_change: int,
    operation: str = "add"
) -> bool:
    """Met à jour l'inventaire d'un produit"""
    collection = get_db()['products']

    update = {
        "$inc": {
            "inventory.quantity": quantity_change if operation == "add" else -quantity_change
        },
        "$set": {
            "updated_at": datetime.utcnow()
        }
    }

    # Empêcher les quantités négatives
    query = {"_id": product_id}
    if operation == "subtract":
        query["inventory.quantity"] = {"$gte": quantity_change}

    result = collection.update_one(query, update)
    return result.modified_count > 0
```

```python
# Bulk operations pour performance
from pymongo import UpdateOne, InsertOne, DeleteOne

def bulk_update_prices(price_updates: List[Dict[str, Any]]) -> Dict[str, int]:
    """Mise à jour en masse des prix"""
    collection = get_db()['products']

    operations = []
    for update in price_updates:
        operations.append(
            UpdateOne(
                {"_id": ObjectId(update['product_id'])},
                {
                    "$set": {
                        "price.amount": update['new_price'],
                        "updated_at": datetime.utcnow()
                    }
                }
            )
        )

    result = collection.bulk_write(operations, ordered=False)

    return {
        "matched": result.matched_count,
        "modified": result.modified_count,
        "errors": len(result.bulk_api_result.get('writeErrors', []))
    }
```

### Agrégations complexes

```python
# Agrégation avec pipeline complexe
def get_sales_analytics(start_date: datetime, end_date: datetime) -> Dict[str, Any]:
    """Analyse des ventes avec agrégation"""
    collection = get_db()['orders']

    pipeline = [
        # Filtrer par période
        {
            "$match": {
                "created_at": {"$gte": start_date, "$lte": end_date},
                "status": "completed"
            }
        },

        # Dérouler les items
        {"$unwind": "$items"},

        # Jointure avec products
        {
            "$lookup": {
                "from": "products",
                "localField": "items.product_id",
                "foreignField": "_id",
                "as": "product_details"
            }
        },

        {"$unwind": "$product_details"},

        # Grouper par catégorie
        {
            "$group": {
                "_id": "$product_details.category",
                "total_sales": {
                    "$sum": {
                        "$multiply": ["$items.quantity", "$items.price"]
                    }
                },
                "total_items": {"$sum": "$items.quantity"},
                "unique_products": {"$addToSet": "$items.product_id"},
                "avg_order_value": {"$avg": "$total_amount"}
            }
        },

        # Ajouter le nombre de produits uniques
        {
            "$addFields": {
                "unique_products_count": {"$size": "$unique_products"}
            }
        },

        # Projeter les champs finaux
        {
            "$project": {
                "category": "$_id",
                "total_sales": 1,
                "total_items": 1,
                "unique_products_count": 1,
                "avg_order_value": {"$round": ["$avg_order_value", 2]},
                "_id": 0
            }
        },

        # Trier par ventes
        {"$sort": {"total_sales": -1}}
    ]

    results = list(collection.aggregate(pipeline))

    # Calculer les totaux
    total_sales = sum(r['total_sales'] for r in results)
    total_items = sum(r['total_items'] for r in results)

    return {
        "period": {
            "start": start_date.isoformat(),
            "end": end_date.isoformat()
        },
        "summary": {
            "total_sales": total_sales,
            "total_items": total_items,
            "categories_count": len(results)
        },
        "by_category": results
    }
```

```python
# Agrégation avec facettes
def get_product_dashboard() -> Dict[str, Any]:
    """Dashboard produits avec facettes"""
    collection = get_db()['products']

    pipeline = [
        {
            "$facet": {
                # Statistiques générales
                "general_stats": [
                    {
                        "$group": {
                            "_id": None,
                            "total_products": {"$sum": 1},
                            "avg_price": {"$avg": "$price.amount"},
                            "total_inventory": {"$sum": "$inventory.quantity"},
                            "out_of_stock": {
                                "$sum": {
                                    "$cond": [
                                        {"$eq": ["$inventory.quantity", 0]},
                                        1,
                                        0
                                    ]
                                }
                            }
                        }
                    }
                ],

                # Par catégorie
                "by_category": [
                    {
                        "$group": {
                            "_id": "$category",
                            "count": {"$sum": 1},
                            "avg_price": {"$avg": "$price.amount"}
                        }
                    },
                    {"$sort": {"count": -1}}
                ],

                # Distribution des prix
                "price_distribution": [
                    {
                        "$bucket": {
                            "groupBy": "$price.amount",
                            "boundaries": [0, 10, 50, 100, 500, 1000],
                            "default": "1000+",
                            "output": {
                                "count": {"$sum": 1}
                            }
                        }
                    }
                ],

                # Top 10 produits
                "top_products": [
                    {"$sort": {"reviews.rating": -1}},
                    {"$limit": 10},
                    {
                        "$project": {
                            "name": 1,
                            "price": "$price.amount",
                            "rating": "$reviews.rating"
                        }
                    }
                ]
            }
        }
    ]

    result = list(collection.aggregate(pipeline))[0]
    return result
```

## Motor - Driver asynchrone

### Configuration asynchrone

```python
# database/async_mongodb.py
from typing import Optional
from motor.motor_asyncio import AsyncIOMotorClient, AsyncIOMotorDatabase
from config.database import db_config
import logging

logger = logging.getLogger(__name__)

class AsyncMongoDBConnection:
    """Gestionnaire de connexion MongoDB asynchrone (Singleton)"""

    _instance: Optional['AsyncMongoDBConnection'] = None
    _client: Optional[AsyncIOMotorClient] = None
    _db: Optional[AsyncIOMotorDatabase] = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    async def connect(self) -> AsyncIOMotorDatabase:
        """Établit la connexion asynchrone"""
        if self._client is None:
            self._client = AsyncIOMotorClient(
                db_config.uri,
                **db_config.get_client_options()
            )

            # Vérifier la connexion
            await self._client.admin.command('ping')
            self._db = self._client[db_config.database_name]

            logger.info("✅ Connecté à MongoDB (async)")

        return self._db

    def get_database(self) -> AsyncIOMotorDatabase:
        """Retourne la base de données"""
        if self._db is None:
            raise RuntimeError("Database not connected. Call connect() first.")
        return self._db

    async def close(self):
        """Ferme la connexion"""
        if self._client:
            self._client.close()
            self._client = None
            self._db = None
            logger.info("🔌 Connexion MongoDB fermée (async)")

# Instance globale
async_mongodb = AsyncMongoDBConnection()

# Helper
async def get_async_db() -> AsyncIOMotorDatabase:
    return await async_mongodb.connect()
```

### Repository asynchrone

```python
# repositories/async_user_repository.py
from typing import List, Optional, Dict, Any
from datetime import datetime
from motor.motor_asyncio import AsyncIOMotorCollection
from bson import ObjectId
from models.user import UserCreate, UserInDB, UserUpdate, UserStatus

class AsyncUserRepository:
    """Repository asynchrone pour les utilisateurs"""

    def __init__(self, collection: AsyncIOMotorCollection):
        self.collection = collection

    async def ensure_indexes(self):
        """Crée les index nécessaires"""
        await self.collection.create_index("email", unique=True)
        await self.collection.create_index("status")
        await self.collection.create_index([
            ("status", 1),
            ("created_at", -1)
        ])

    async def create_user(self, user_data: UserCreate) -> UserInDB:
        """Crée un utilisateur"""
        now = datetime.utcnow()

        user_dict = user_data.model_dump(exclude={'password'})
        user_dict.update({
            'created_at': now,
            'updated_at': now,
            'login_count': 0
        })

        result = await self.collection.insert_one(user_dict)
        user_dict['_id'] = result.inserted_id

        return UserInDB(**user_dict)

    async def find_by_id(
        self,
        user_id: str | ObjectId
    ) -> Optional[UserInDB]:
        """Recherche par ID"""
        object_id = ObjectId(user_id) if isinstance(user_id, str) else user_id
        user_dict = await self.collection.find_one({"_id": object_id})
        return UserInDB(**user_dict) if user_dict else None

    async def find_by_email(self, email: str) -> Optional[UserInDB]:
        """Recherche par email"""
        user_dict = await self.collection.find_one({"email": email})
        return UserInDB(**user_dict) if user_dict else None

    async def find_active_users(
        self,
        limit: int = 100,
        skip: int = 0
    ) -> List[UserInDB]:
        """Récupère les utilisateurs actifs"""
        cursor = self.collection.find(
            {"status": UserStatus.ACTIVE.value}
        ).sort("created_at", -1).skip(skip).limit(limit)

        users = await cursor.to_list(length=limit)
        return [UserInDB(**user) for user in users]

    async def update_profile(
        self,
        user_id: str | ObjectId,
        updates: UserUpdate
    ) -> bool:
        """Met à jour le profil"""
        update_data = updates.model_dump(exclude_unset=True)

        if not update_data:
            return False

        update_data['updated_at'] = datetime.utcnow()

        object_id = ObjectId(user_id) if isinstance(user_id, str) else user_id
        result = await self.collection.update_one(
            {"_id": object_id},
            {"$set": update_data}
        )

        return result.modified_count > 0

    async def search_users(
        self,
        search_term: str,
        limit: int = 20
    ) -> List[UserInDB]:
        """Recherche full-text"""
        cursor = self.collection.find(
            {"$text": {"$search": search_term}}
        ).limit(limit)

        users = await cursor.to_list(length=limit)
        return [UserInDB(**user) for user in users]
```

### Service asynchrone

```python
# services/async_user_service.py
from typing import List, Optional
from repositories.async_user_repository import AsyncUserRepository
from models.user import UserCreate, UserInDB, UserUpdate, UserPublic
import logging

logger = logging.getLogger(__name__)

class AsyncUserService:
    """Service métier asynchrone"""

    def __init__(self, repository: AsyncUserRepository):
        self.repository = repository

    async def register_user(self, user_data: UserCreate) -> UserPublic:
        """Enregistre un utilisateur"""
        existing = await self.repository.find_by_email(user_data.email)
        if existing:
            raise ValueError(f"Email {user_data.email} already exists")

        user = await self.repository.create_user(user_data)

        logger.info(f"New user registered: {user.email}")

        return UserPublic(
            _id=str(user.id),
            name=user.name,
            email=user.email,
            status=user.status,
            created_at=user.created_at
        )

    async def get_user_by_id(self, user_id: str) -> Optional[UserInDB]:
        """Récupère un utilisateur"""
        return await self.repository.find_by_id(user_id)

    async def search_users(self, query: str) -> List[UserPublic]:
        """Recherche d'utilisateurs"""
        users = await self.repository.search_users(query)
        return [
            UserPublic(
                _id=str(user.id),
                name=user.name,
                email=user.email,
                status=user.status,
                created_at=user.created_at
            )
            for user in users
        ]
```

### Application FastAPI avec Motor

```python
# main.py
from fastapi import FastAPI, HTTPException, Depends
from contextlib import asynccontextmanager
from database.async_mongodb import async_mongodb, get_async_db
from services.async_user_service import AsyncUserService
from repositories.async_user_repository import AsyncUserRepository
from models.user import UserCreate, UserPublic
from motor.motor_asyncio import AsyncIOMotorDatabase

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Gestion du cycle de vie de l'application"""
    # Startup
    await async_mongodb.connect()
    yield
    # Shutdown
    await async_mongodb.close()

app = FastAPI(lifespan=lifespan)

async def get_user_service(
    db: AsyncIOMotorDatabase = Depends(get_async_db)
) -> AsyncUserService:
    """Dependency injection pour le service utilisateur"""
    repository = AsyncUserRepository(db['users'])
    return AsyncUserService(repository)

@app.post("/users", response_model=UserPublic)
async def create_user(
    user_data: UserCreate,
    service: AsyncUserService = Depends(get_user_service)
):
    """Crée un utilisateur"""
    try:
        user = await service.register_user(user_data)
        return user
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))

@app.get("/users/{user_id}", response_model=UserPublic)
async def get_user(
    user_id: str,
    service: AsyncUserService = Depends(get_user_service)
):
    """Récupère un utilisateur"""
    user = await service.get_user_by_id(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")

    return UserPublic(
        _id=str(user.id),
        name=user.name,
        email=user.email,
        status=user.status,
        created_at=user.created_at
    )

@app.get("/users/search/{query}", response_model=List[UserPublic])
async def search_users(
    query: str,
    service: AsyncUserService = Depends(get_user_service)
):
    """Recherche d'utilisateurs"""
    return await service.search_users(query)
```

## Transactions avec PyMongo

```python
# Transactions synchrones
from pymongo.errors import OperationFailure

def transfer_funds(
    from_account_id: ObjectId,
    to_account_id: ObjectId,
    amount: float
) -> bool:
    """Transfert de fonds avec transaction"""
    client = mongodb.get_client()

    with client.start_session() as session:
        with session.start_transaction():
            try:
                accounts = get_db()['accounts']

                # Débiter
                debit_result = accounts.update_one(
                    {
                        "_id": from_account_id,
                        "balance": {"$gte": amount}
                    },
                    {
                        "$inc": {"balance": -amount},
                        "$push": {
                            "transactions": {
                                "type": "debit",
                                "amount": amount,
                                "timestamp": datetime.utcnow()
                            }
                        }
                    },
                    session=session
                )

                if debit_result.modified_count == 0:
                    raise ValueError("Insufficient funds")

                # Créditer
                accounts.update_one(
                    {"_id": to_account_id},
                    {
                        "$inc": {"balance": amount},
                        "$push": {
                            "transactions": {
                                "type": "credit",
                                "amount": amount,
                                "timestamp": datetime.utcnow()
                            }
                        }
                    },
                    session=session
                )

                # Commit implicite à la fin du context manager
                return True

            except Exception as e:
                # Rollback automatique en cas d'erreur
                logger.error(f"Transaction failed: {e}")
                raise
```

```python
# Transactions asynchrones avec Motor
async def async_transfer_funds(
    from_account_id: ObjectId,
    to_account_id: ObjectId,
    amount: float
) -> bool:
    """Transfert asynchrone avec transaction"""
    client = async_mongodb._client

    async with await client.start_session() as session:
        async with session.start_transaction():
            try:
                db = await get_async_db()
                accounts = db['accounts']

                # Débiter
                debit_result = await accounts.update_one(
                    {
                        "_id": from_account_id,
                        "balance": {"$gte": amount}
                    },
                    {
                        "$inc": {"balance": -amount},
                        "$push": {
                            "transactions": {
                                "type": "debit",
                                "amount": amount,
                                "timestamp": datetime.utcnow()
                            }
                        }
                    },
                    session=session
                )

                if debit_result.modified_count == 0:
                    raise ValueError("Insufficient funds")

                # Créditer
                await accounts.update_one(
                    {"_id": to_account_id},
                    {
                        "$inc": {"balance": amount},
                        "$push": {
                            "transactions": {
                                "type": "credit",
                                "amount": amount,
                                "timestamp": datetime.utcnow()
                            }
                        }
                    },
                    session=session
                )

                return True

            except Exception as e:
                logger.error(f"Transaction failed: {e}")
                raise
```

## Change Streams

```python
# Change Streams synchrone
def watch_user_changes():
    """Écoute les changements sur users"""
    collection = get_db()['users']

    with collection.watch(
        [{"$match": {"operationType": {"$in": ["insert", "update"]}}}],
        full_document='updateLookup'
    ) as stream:
        for change in stream:
            print(f"Change detected: {change['operationType']}")

            if change['operationType'] == 'insert':
                user = change['fullDocument']
                print(f"New user: {user['email']}")
                # Envoyer email de bienvenue

            elif change['operationType'] == 'update':
                user = change['fullDocument']
                print(f"Updated user: {user['email']}")
                # Invalider cache, etc.
```

```python
# Change Streams asynchrone
import asyncio

async def async_watch_user_changes():
    """Écoute asynchrone des changements"""
    db = await get_async_db()
    collection = db['users']

    async with collection.watch(
        [{"$match": {"operationType": {"$in": ["insert", "update"]}}}],
        full_document='updateLookup'
    ) as stream:
        async for change in stream:
            print(f"Change detected: {change['operationType']}")

            if change['operationType'] == 'insert':
                user = change['fullDocument']
                await send_welcome_email(user['email'])

            elif change['operationType'] == 'update':
                user = change['fullDocument']
                await invalidate_cache(user['_id'])

# Exécution
# asyncio.run(async_watch_user_changes())
```

## Gestion d'erreurs

```python
# Gestion d'erreurs PyMongo
from pymongo.errors import (
    ConnectionFailure,
    ServerSelectionTimeoutError,
    OperationFailure,
    DuplicateKeyError,
    BulkWriteError
)
from bson.errors import InvalidId

class MongoDBErrorHandler:
    """Gestionnaire centralisé d'erreurs"""

    @staticmethod
    def handle_error(error: Exception) -> None:
        """Gère les erreurs MongoDB"""
        if isinstance(error, DuplicateKeyError):
            # Clé dupliquée
            field = list(error.details['keyPattern'].keys())[0]
            raise ValueError(f"Duplicate value for field: {field}")

        elif isinstance(error, InvalidId):
            raise ValueError("Invalid ObjectId format")

        elif isinstance(error, ConnectionFailure):
            logger.error(f"Connection failure: {error}")
            raise RuntimeError("Database connection failed")

        elif isinstance(error, ServerSelectionTimeoutError):
            logger.error(f"Server selection timeout: {error}")
            raise RuntimeError("Database server not available")

        elif isinstance(error, OperationFailure):
            if error.code == 50:  # MaxTimeMSExpired
                raise RuntimeError("Query timeout exceeded")
            elif error.code == 13:  # Unauthorized
                raise RuntimeError("Database authentication failed")

        elif isinstance(error, BulkWriteError):
            logger.error(f"Bulk write error: {error.details}")
            raise

        else:
            logger.error(f"Unexpected error: {error}")
            raise

# Utilisation avec decorator
from functools import wraps

def handle_mongo_errors(func):
    """Decorator pour gérer les erreurs MongoDB"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except Exception as e:
            MongoDBErrorHandler.handle_error(e)
    return wrapper

# Pour async
def async_handle_mongo_errors(func):
    """Decorator async pour gérer les erreurs"""
    @wraps(func)
    async def wrapper(*args, **kwargs):
        try:
            return await func(*args, **kwargs)
        except Exception as e:
            MongoDBErrorHandler.handle_error(e)
    return wrapper
```

## Performance et optimisation

```python
# Batch processing pour grands volumes
def process_large_dataset_batch():
    """Traitement par batch"""
    collection = get_db()['users']
    batch_size = 1000

    cursor = collection.find({}).batch_size(batch_size)

    processed = 0
    for user in cursor:
        # Traiter chaque document
        process_user(user)
        processed += 1

        if processed % 10000 == 0:
            logger.info(f"Processed {processed} users")

    logger.info(f"Total processed: {processed}")

# Version asynchrone
async def async_process_large_dataset():
    """Traitement asynchrone par batch"""
    db = await get_async_db()
    collection = db['users']

    cursor = collection.find({}).batch_size(1000)

    processed = 0
    async for user in cursor:
        await async_process_user(user)
        processed += 1

        if processed % 10000 == 0:
            logger.info(f"Processed {processed} users")
```

```python
# Projection pour économiser la bande passante
def get_user_summaries():
    """Récupère uniquement les champs nécessaires"""
    collection = get_db()['users']

    # ❌ MAUVAIS - Charge tous les champs
    # users = list(collection.find({}))

    # ✅ BON - Projection spécifique
    users = list(collection.find(
        {},
        {"name": 1, "email": 1, "status": 1, "_id": 0}
    ))

    return users
```

## Bonnes pratiques

### ✅ DO - À faire

```python
# 1. Utiliser un singleton pour la connexion
mongodb = MongoDBConnection()

# 2. Utiliser Pydantic pour la validation
class User(BaseModel):
    name: str
    email: EmailStr

# 3. Gérer les index au démarrage
repository._ensure_indexes()

# 4. Utiliser le repository pattern
user_repo = UserRepository(collection)

# 5. Typer les retours de fonction
def get_user(user_id: str) -> Optional[UserInDB]:
    pass

# 6. Fermer proprement les connexions
try:
    # opérations
    pass
finally:
    mongodb.close()

# 7. Utiliser des projections
collection.find({}, {"password": 0})

# 8. Gérer les erreurs spécifiques
except DuplicateKeyError:
    # Handle duplicate

# 9. Logger les opérations importantes
logger.info(f"User created: {user.email}")

# 10. Utiliser Motor pour async
async def get_user_async():
    return await collection.find_one(...)
```

### ❌ DON'T - À éviter

```python
# ❌ Créer une nouvelle connexion par requête
# ❌ Ne pas valider les données
# ❌ Ignorer les erreurs
# ❌ Charger tous les documents avec .toArray()
# ❌ Ne pas utiliser d'index
# ❌ Faire des requêtes N+1
# ❌ Mélanger sync et async incorrectement
# ❌ Ne pas fermer les curseurs/connexions
```

## Comparaison PyMongo vs Motor

| Aspect | PyMongo | Motor |
|--------|---------|-------|
| **Modèle** | Synchrone | Asynchrone |
| **Performance** | Bon | Excellent (I/O bound) |
| **Frameworks** | Flask, Django | FastAPI, Tornado |
| **Complexité** | Simple | Moyenne |
| **Cas d'usage** | APIs simples, scripts | APIs haute performance |
| **Concurrence** | Threading | asyncio |

## Conclusion

PyMongo et Motor offrent des API puissantes et pythoniques pour MongoDB :

1. **PyMongo** : Parfait pour applications synchrones, scripts, Django
2. **Motor** : Idéal pour FastAPI, applications asynchrones haute performance
3. **Pydantic** : Validation et typage robuste des données
4. **Repository Pattern** : Séparation des responsabilités
5. **Gestion d'erreurs** : Robustesse et diagnostics

**Points clés** :
- Singleton pour les connexions
- Validation avec Pydantic
- Index appropriés
- Gestion d'erreurs spécifique
- Projection pour performance
- Choix sync/async selon contexte

---


⏭️ [Driver Java](/15-drivers-integration-applicative/04-driver-java.md)
