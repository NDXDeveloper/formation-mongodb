🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.8 Driver Ruby

## Introduction

Le driver Ruby MongoDB est un driver officiel qui s'intègre élégamment avec l'écosystème Ruby. Il offre une API expressive et idiomatique qui suit les conventions Ruby. Cette section couvre l'utilisation du driver avec **Ruby 3.0+**, l'intégration avec **Rails** via **Mongoid**, et les meilleures pratiques pour des applications Ruby de production.

## Installation et configuration

### Installation

```bash
# Ruby version recommandée : 3.0+
ruby -v

# Driver MongoDB
gem install mongo

# Pour Rails avec Mongoid
gem install mongoid

# Ou via Gemfile
```

### Gemfile

```ruby
# Gemfile
source 'https://rubygems.org'

ruby '~> 3.2.0'

# Driver MongoDB
gem 'mongo', '~> 2.19'

# Pour Rails
gem 'rails', '~> 7.1'
gem 'mongoid', '~> 8.1'

# Utilitaires
gem 'dotenv', '~> 2.8'
gem 'logger', '~> 1.6'

group :development, :test do
  gem 'rspec', '~> 3.12'
  gem 'rubocop', '~> 1.50'
  gem 'factory_bot', '~> 6.2'
end
```

### Structure de projet

```
myapp/
├── Gemfile
├── Gemfile.lock
├── .env
├── Rakefile
├── config/
│   ├── database.rb
│   └── mongoid.yml
├── lib/
│   ├── models/
│   │   ├── user.rb
│   │   └── user_status.rb
│   ├── repositories/
│   │   ├── base_repository.rb
│   │   └── user_repository.rb
│   ├── services/
│   │   └── user_service.rb
│   └── database.rb
├── app/
│   └── controllers/
│       └── users_controller.rb
└── spec/
```

## Configuration

### .env

```env
MONGODB_URI=mongodb://localhost:27017
MONGODB_DATABASE=myapp
MONGODB_MAX_POOL_SIZE=50
MONGODB_MIN_POOL_SIZE=10
MONGODB_CONNECT_TIMEOUT=10
MONGODB_SOCKET_TIMEOUT=45
```

### Configuration de base

```ruby
# lib/database.rb
require 'mongo'
require 'dotenv/load'
require 'logger'

module MyApp
  class Database
    class << self
      attr_reader :client, :database

      def connect
        return if @client

        uri = ENV.fetch('MONGODB_URI', 'mongodb://localhost:27017')
        database_name = ENV.fetch('MONGODB_DATABASE', 'myapp')

        options = {
          max_pool_size: ENV.fetch('MONGODB_MAX_POOL_SIZE', 50).to_i,
          min_pool_size: ENV.fetch('MONGODB_MIN_POOL_SIZE', 10).to_i,
          connect_timeout: ENV.fetch('MONGODB_CONNECT_TIMEOUT', 10).to_i,
          socket_timeout: ENV.fetch('MONGODB_SOCKET_TIMEOUT', 45).to_i,
          server_selection_timeout: 5,
          retry_writes: true,
          retry_reads: true
        }

        @client = Mongo::Client.new(uri, options)
        @database = @client.database

        # Vérifier la connexion
        @database.command(ping: 1)
        logger.info '✅ Connected to MongoDB'

      rescue StandardError => e
        logger.error "❌ Failed to connect to MongoDB: #{e.message}"
        raise
      end

      def close
        @client&.close
        @client = nil
        @database = nil
        logger.info '🔌 Disconnected from MongoDB'
      end

      def collection(name)
        connect unless @database
        @database[name]
      end

      private

      def logger
        @logger ||= Logger.new($stdout)
      end
    end
  end
end
```

### Configuration avancée avec monitoring

```ruby
# lib/database_advanced.rb
require 'mongo'

module MyApp
  class DatabaseAdvanced
    class CommandLogger
      def initialize(logger)
        @logger = logger
      end

      def started(event)
        @logger.debug "MongoDB command started: #{event.command_name}"
      end

      def succeeded(event)
        duration_ms = event.duration * 1000
        @logger.debug "MongoDB command succeeded: #{event.command_name} (#{duration_ms.round(2)}ms)"
      end

      def failed(event)
        @logger.error "MongoDB command failed: #{event.command_name} - #{event.message}"
      end
    end

    attr_reader :client, :database

    def initialize(config = {}, logger: nil)
      @config = config
      @logger = logger || Logger.new($stdout)
      connect
    end

    private

    def connect
      uri = @config[:uri] || ENV.fetch('MONGODB_URI', 'mongodb://localhost:27017')
      database_name = @config[:database] || ENV.fetch('MONGODB_DATABASE', 'myapp')

      options = {
        max_pool_size: @config[:max_pool_size] || 50,
        min_pool_size: @config[:min_pool_size] || 10,
        connect_timeout: @config[:connect_timeout] || 10,
        socket_timeout: @config[:socket_timeout] || 45,
        server_selection_timeout: 5,
        retry_writes: true,
        retry_reads: true,
        monitoring: true
      }

      @client = Mongo::Client.new(uri, options)
      @database = @client.database

      # Ajouter monitoring
      subscribe_to_monitoring if @logger

      # Vérifier la connexion
      @database.command(ping: 1)
      @logger.info '✅ Connected to MongoDB'
    end

    def subscribe_to_monitoring
      subscriber = CommandLogger.new(@logger)

      Mongo::Monitoring::Global.subscribe(
        Mongo::Monitoring::COMMAND,
        subscriber
      )
    end
  end
end
```

## Modèles de données

### Classe UserStatus

```ruby
# lib/models/user_status.rb
module MyApp
  module Models
    class UserStatus
      ACTIVE = 'active'
      INACTIVE = 'inactive'
      SUSPENDED = 'suspended'
      DELETED = 'deleted'

      ALL = [ACTIVE, INACTIVE, SUSPENDED, DELETED].freeze

      def self.valid?(status)
        ALL.include?(status)
      end

      def self.label(status)
        case status
        when ACTIVE then 'Actif'
        when INACTIVE then 'Inactif'
        when SUSPENDED then 'Suspendu'
        when DELETED then 'Supprimé'
        else 'Inconnu'
        end
      end
    end
  end
end
```

### Modèle User

```ruby
# lib/models/user.rb
require 'time'
require_relative 'user_status'

module MyApp
  module Models
    class User
      attr_accessor :id, :name, :email, :age, :status, :roles, :preferences,
                    :created_at, :updated_at, :last_login, :login_count

      def initialize(attributes = {})
        @id = attributes[:_id] || attributes[:id]
        @name = attributes[:name] || ''
        @email = attributes[:email] || ''
        @age = attributes[:age]
        @status = attributes[:status] || UserStatus::ACTIVE
        @roles = attributes[:roles] || ['user']
        @preferences = attributes[:preferences]
        @created_at = attributes[:created_at] || Time.now.utc
        @updated_at = attributes[:updated_at] || Time.now.utc
        @last_login = attributes[:last_login]
        @login_count = attributes[:login_count] || 0
      end

      def self.from_document(doc)
        return nil if doc.nil?

        new(
          _id: doc['_id'],
          name: doc['name'],
          email: doc['email'],
          age: doc['age'],
          status: doc['status'],
          roles: doc['roles'],
          preferences: doc['preferences'],
          created_at: doc['created_at'],
          updated_at: doc['updated_at'],
          last_login: doc['last_login'],
          login_count: doc['login_count']
        )
      end

      def to_document
        doc = {
          name: @name,
          email: @email,
          status: @status,
          roles: @roles,
          created_at: @created_at,
          updated_at: @updated_at,
          login_count: @login_count
        }

        doc[:_id] = @id if @id
        doc[:age] = @age if @age
        doc[:preferences] = @preferences if @preferences
        doc[:last_login] = @last_login if @last_login

        doc
      end

      def to_public_hash
        {
          id: @id.to_s,
          name: @name,
          email: @email,
          status: @status,
          created_at: @created_at.iso8601
        }
      end

      def increment_login!
        @login_count += 1
        @last_login = Time.now.utc
        @updated_at = Time.now.utc
      end

      def active?
        @status == UserStatus::ACTIVE
      end

      def has_role?(role)
        @roles.include?(role)
      end
    end
  end
end
```

### DTOs

```ruby
# lib/models/create_user_dto.rb
module MyApp
  module Models
    class CreateUserDTO
      attr_reader :name, :email, :age, :password, :roles

      def initialize(name:, email:, password:, age: nil, roles: ['user'])
        @name = name
        @email = email
        @age = age
        @password = password
        @roles = roles
      end

      def self.from_hash(hash)
        new(
          name: hash[:name] || hash['name'],
          email: hash[:email] || hash['email'],
          age: hash[:age] || hash['age'],
          password: hash[:password] || hash['password'],
          roles: hash[:roles] || hash['roles'] || ['user']
        )
      end

      def valid?
        errors.empty?
      end

      def errors
        errs = {}

        errs[:name] = 'Name must be between 2 and 100 characters' unless name_valid?
        errs[:email] = 'Invalid email format' unless email_valid?
        errs[:age] = 'Age must be between 18 and 120' if age && !age_valid?
        errs[:password] = 'Password must be at least 8 characters' unless password_valid?

        errs
      end

      private

      def name_valid?
        @name && @name.length.between?(2, 100)
      end

      def email_valid?
        @email&.match?(/\A[\w+\-.]+@[a-z\d\-]+(\.[a-z\d\-]+)*\.[a-z]+\z/i)
      end

      def age_valid?
        @age.between?(18, 120)
      end

      def password_valid?
        @password && @password.length >= 8
      end
    end

    class UpdateUserDTO
      attr_reader :name, :email, :age

      def initialize(name: nil, email: nil, age: nil)
        @name = name
        @email = email
        @age = age
      end

      def self.from_hash(hash)
        new(
          name: hash[:name] || hash['name'],
          email: hash[:email] || hash['email'],
          age: hash[:age] || hash['age']
        )
      end

      def to_update_hash
        updates = {}
        updates[:name] = @name if @name
        updates[:email] = @email if @email
        updates[:age] = @age if @age
        updates
      end
    end
  end
end
```

## Repository Pattern

### Repository de base

```ruby
# lib/repositories/base_repository.rb
require 'mongo'

module MyApp
  module Repositories
    class BaseRepository
      attr_reader :collection

      def initialize(collection)
        @collection = collection
      end

      def find_by_id(id)
        id = BSON::ObjectId(id) if id.is_a?(String)
        @collection.find(_id: id).first
      end

      def find_one(filter)
        @collection.find(filter).first
      end

      def find_many(filter = {}, options = {})
        @collection.find(filter, options).to_a
      end

      def insert_one(document)
        result = @collection.insert_one(document)
        result.inserted_id
      end

      def update_one(filter, update)
        result = @collection.update_one(filter, update)
        result.modified_count.positive?
      end

      def update_by_id(id, update)
        id = BSON::ObjectId(id) if id.is_a?(String)
        update_one({ _id: id }, update)
      end

      def delete_one(filter)
        result = @collection.delete_one(filter)
        result.deleted_count.positive?
      end

      def delete_by_id(id)
        id = BSON::ObjectId(id) if id.is_a?(String)
        delete_one(_id: id)
      end

      def count(filter = {})
        @collection.count_documents(filter)
      end

      def exists?(filter)
        @collection.count_documents(filter, limit: 1).positive?
      end
    end
  end
end
```

### Repository utilisateur

```ruby
# lib/repositories/user_repository.rb
require_relative 'base_repository'
require_relative '../models/user'
require_relative '../models/user_status'

module MyApp
  module Repositories
    class UserRepository < BaseRepository
      def initialize(collection)
        super(collection)
        ensure_indexes
      end

      def create(user)
        document = user.to_document
        id = insert_one(document)
        user.id = id
        user
      rescue Mongo::Error::OperationFailure => e
        raise 'Email already exists' if e.message.include?('E11000')
        raise
      end

      def find_by_email(email)
        doc = find_one(email: email)
        Models::User.from_document(doc)
      end

      def find_active_users(limit: 100, skip: 0)
        docs = find_many(
          { status: Models::UserStatus::ACTIVE },
          limit: limit,
          skip: skip,
          sort: { created_at: -1 }
        )

        docs.map { |doc| Models::User.from_document(doc) }
      end

      def update(id, updates)
        updates[:updated_at] = Time.now.utc
        update_by_id(id, { '$set': updates })
      end

      def increment_login_count(id)
        id = BSON::ObjectId(id) if id.is_a?(String)

        @collection.update_one(
          { _id: id },
          {
            '$inc': { login_count: 1 },
            '$set': {
              last_login: Time.now.utc,
              updated_at: Time.now.utc
            }
          }
        )
      end

      def search(search_term, limit: 20)
        regex = Regexp.new(search_term, Regexp::IGNORECASE)

        docs = find_many(
          {
            '$or': [
              { name: regex },
              { email: regex }
            ]
          },
          limit: limit
        )

        docs.map { |doc| Models::User.from_document(doc) }
      end

      def find_by_role(role)
        docs = find_many(roles: role)
        docs.map { |doc| Models::User.from_document(doc) }
      end

      def bulk_update_status(user_ids, status)
        object_ids = user_ids.map do |id|
          id.is_a?(String) ? BSON::ObjectId(id) : id
        end

        result = @collection.update_many(
          { _id: { '$in': object_ids } },
          {
            '$set': {
              status: status,
              updated_at: Time.now.utc
            }
          }
        )

        result.modified_count
      end

      def statistics
        total = count

        # Par status
        pipeline = [
          { '$group': { _id: '$status', count: { '$sum': 1 } } }
        ]

        by_status = {}
        @collection.aggregate(pipeline).each do |doc|
          by_status[doc['_id']] = doc['count']
        end

        # Inscriptions récentes (30 derniers jours)
        thirty_days_ago = Time.now.utc - (30 * 24 * 60 * 60)
        recent_signups = count(created_at: { '$gte': thirty_days_ago })

        {
          total: total,
          by_status: by_status,
          recent_signups: recent_signups
        }
      end

      private

      def ensure_indexes
        # Index unique sur email
        @collection.indexes.create_one(
          { email: 1 },
          unique: true
        )

        # Index sur status
        @collection.indexes.create_one(status: 1)

        # Index composé
        @collection.indexes.create_one(
          { status: 1, created_at: -1 }
        )

        puts 'Indexes created for users collection'
      end
    end
  end
end
```

## Service Layer

```ruby
# lib/services/user_service.rb
require_relative '../repositories/user_repository'
require_relative '../models/user'
require_relative '../models/user_status'
require 'logger'

module MyApp
  module Services
    class UserService
      def initialize(repository, logger: nil)
        @repository = repository
        @logger = logger || Logger.new($stdout)
      end

      def register_user(dto)
        # Valider
        raise ArgumentError, "Validation failed: #{dto.errors}" unless dto.valid?

        # Vérifier si l'email existe
        existing = @repository.find_by_email(dto.email)
        raise "Email #{dto.email} already registered" if existing

        # Créer l'utilisateur
        user = Models::User.new(
          name: dto.name,
          email: dto.email,
          age: dto.age,
          roles: dto.roles
        )

        # TODO: Hasher le mot de passe
        # user.password_hash = BCrypt::Password.create(dto.password)

        user = @repository.create(user)

        @logger.info "New user registered: #{user.email} (#{user.id})"

        user
      end

      def authenticate_user(email, password)
        user = @repository.find_by_email(email)
        raise 'Invalid credentials' unless user
        raise 'Account is not active' unless user.active?

        # TODO: Vérifier le mot de passe
        # raise 'Invalid credentials' unless BCrypt::Password.new(user.password_hash) == password

        # Incrémenter le compteur
        @repository.increment_login_count(user.id)

        @logger.info "User authenticated: #{user.email} (#{user.id})"

        user
      end

      def find_user_by_id(id)
        doc = @repository.find_by_id(id)
        raise 'User not found' unless doc

        Models::User.from_document(doc)
      end

      def update_user(id, dto)
        updates = dto.to_update_hash
        raise ArgumentError, 'No updates provided' if updates.empty?

        success = @repository.update(id, updates)
        raise 'Failed to update user' unless success

        find_user_by_id(id)
      end

      def search_users(query, limit: 20)
        @repository.search(query, limit: limit)
      end

      def dashboard_stats
        @repository.statistics
      end

      def suspend_user(id)
        success = @repository.update(id, { status: Models::UserStatus::SUSPENDED })
        @logger.info "User suspended: #{id}" if success
        success
      end
    end
  end
end
```

## Opérations avancées

### Pagination

```ruby
# lib/repositories/concerns/paginatable.rb
module MyApp
  module Repositories
    module Concerns
      module Paginatable
        def find_paginated(filter = {}, page: 1, page_size: 20)
          total = @collection.count_documents(filter)
          skip = (page - 1) * page_size

          documents = @collection.find(filter)
                                 .sort(created_at: -1)
                                 .skip(skip)
                                 .limit(page_size)
                                 .to_a

          total_pages = (total.to_f / page_size).ceil

          {
            data: documents,
            total: total,
            page: page,
            page_size: page_size,
            total_pages: total_pages
          }
        end
      end
    end
  end
end
```

### Bulk Operations

```ruby
# Mise à jour en masse
def bulk_update_prices(collection, updates)
  operations = updates.map do |update|
    {
      update_one: {
        filter: { _id: BSON::ObjectId(update[:product_id]) },
        update: {
          '$set': {
            'price.amount': update[:new_price],
            updated_at: Time.now.utc
          }
        }
      }
    }
  end

  result = collection.bulk_write(operations, ordered: false)
  result.modified_count
end
```

## Agrégations

```ruby
# lib/services/analytics_service.rb
module MyApp
  module Services
    class AnalyticsService
      def initialize(orders_collection, products_collection)
        @orders = orders_collection
        @products = products_collection
      end

      def sales_analytics(start_date, end_date)
        pipeline = [
          # Filtrer par période
          {
            '$match': {
              created_at: {
                '$gte': start_date,
                '$lte': end_date
              },
              status: 'completed'
            }
          },

          # Dérouler les items
          { '$unwind': '$items' },

          # Jointure avec products
          {
            '$lookup': {
              from: 'products',
              localField: 'items.product_id',
              foreignField: '_id',
              as: 'product_details'
            }
          },

          { '$unwind': '$product_details' },

          # Grouper par catégorie
          {
            '$group': {
              _id: '$product_details.category',
              total_sales: {
                '$sum': {
                  '$multiply': ['$items.quantity', '$items.price']
                }
              },
              total_items: { '$sum': '$items.quantity' },
              avg_order_value: { '$avg': '$total_amount' }
            }
          },

          # Trier
          { '$sort': { total_sales: -1 } }
        ]

        results = @orders.aggregate(pipeline).to_a

        {
          period: {
            start: start_date.iso8601,
            end: end_date.iso8601
          },
          by_category: results
        }
      end

      def product_dashboard
        pipeline = [
          {
            '$facet': {
              general_stats: [
                {
                  '$group': {
                    _id: nil,
                    total_products: { '$sum': 1 },
                    avg_price: { '$avg': '$price.amount' },
                    total_inventory: { '$sum': '$inventory.quantity' }
                  }
                }
              ],
              by_category: [
                {
                  '$group': {
                    _id: '$category',
                    count: { '$sum': 1 },
                    avg_price: { '$avg': '$price.amount' }
                  }
                },
                { '$sort': { count: -1 } }
              ],
              top_products: [
                { '$sort': { 'reviews.rating': -1 } },
                { '$limit': 10 },
                {
                  '$project': {
                    name: 1,
                    price: '$price.amount',
                    rating: '$reviews.rating'
                  }
                }
              ]
            }
          }
        ]

        @products.aggregate(pipeline).first
      end
    end
  end
end
```

## Transactions

```ruby
# lib/services/transaction_service.rb
module MyApp
  module Services
    class TransactionService
      def initialize(client)
        @client = client
      end

      def transfer_funds(from_account_id, to_account_id, amount)
        session = @client.start_session

        session.with_transaction do
          accounts = @client[:myapp][:accounts]

          # Débiter
          debit_result = accounts.update_one(
            {
              _id: from_account_id,
              balance: { '$gte': amount }
            },
            {
              '$inc': { balance: -amount },
              '$push': {
                transactions: {
                  type: 'debit',
                  amount: amount,
                  timestamp: Time.now.utc
                }
              }
            },
            session: session
          )

          raise 'Insufficient funds or account not found' if debit_result.modified_count.zero?

          # Créditer
          accounts.update_one(
            { _id: to_account_id },
            {
              '$inc': { balance: amount },
              '$push': {
                transactions: {
                  type: 'credit',
                  amount: amount,
                  timestamp: Time.now.utc
                }
              }
            },
            session: session
          )

          puts '✅ Transaction completed successfully'
        end
      rescue StandardError => e
        puts "❌ Transaction failed: #{e.message}"
        raise
      ensure
        session.end_session
      end

      def create_order_with_inventory_update(order_data, max_retries: 3)
        (1..max_retries).each do |attempt|
          session = @client.start_session

          begin
            order_id = nil

            session.with_transaction do
              db = @client[:myapp]
              orders = db[:orders]
              products = db[:products]
              users = db[:users]

              # 1. Réserver l'inventaire
              order_data[:items].each do |item|
                result = products.update_one(
                  {
                    _id: item[:product_id],
                    'inventory.quantity': { '$gte': item[:quantity] }
                  },
                  {
                    '$inc': { 'inventory.quantity': -item[:quantity] },
                    '$set': { updated_at: Time.now.utc }
                  },
                  session: session
                )

                raise "Product #{item[:product_id]} out of stock" if result.modified_count.zero?
              end

              # 2. Créer la commande
              order = {
                user_id: order_data[:user_id],
                items: order_data[:items],
                status: 'pending',
                total_amount: order_data[:total_amount],
                created_at: Time.now.utc,
                updated_at: Time.now.utc
              }

              result = orders.insert_one(order, session: session)
              order_id = result.inserted_id

              # 3. Mettre à jour les stats utilisateur
              users.update_one(
                { _id: order_data[:user_id] },
                {
                  '$inc': { 'stats.total_orders': 1 },
                  '$set': { 'stats.last_order_date': Time.now.utc }
                },
                session: session
              )
            end

            puts "✅ Order created: #{order_id}"
            return order_id

          rescue Mongo::Error::OperationFailure => e
            if e.message.include?('TransientTransactionError') && attempt < max_retries
              puts "Transaction failed (attempt #{attempt}), retrying..."
              sleep(0.1 * attempt)
              next
            end

            raise
          ensure
            session.end_session
          end
        end

        raise "Transaction failed after #{max_retries} attempts"
      end
    end
  end
end
```

## Intégration Rails avec Mongoid

### Configuration Mongoid

```yaml
# config/mongoid.yml
development:
  clients:
    default:
      database: myapp_development
      hosts:
        - localhost:27017
      options:
        max_pool_size: 50
        min_pool_size: 10
        connect_timeout: 10
        socket_timeout: 45
        server_selection_timeout: 5
        retry_writes: true
        retry_reads: true

test:
  clients:
    default:
      database: myapp_test
      hosts:
        - localhost:27017
      options:
        max_pool_size: 5

production:
  clients:
    default:
      uri: <%= ENV['MONGODB_URI'] %>
      options:
        max_pool_size: 100
        min_pool_size: 20
```

### Modèle Mongoid

```ruby
# app/models/user.rb
class User
  include Mongoid::Document
  include Mongoid::Timestamps

  # Champs
  field :name, type: String
  field :email, type: String
  field :age, type: Integer
  field :status, type: String, default: 'active'
  field :roles, type: Array, default: ['user']
  field :preferences, type: Hash
  field :last_login, type: Time
  field :login_count, type: Integer, default: 0

  # Index
  index({ email: 1 }, { unique: true })
  index({ status: 1 })
  index({ status: 1, created_at: -1 })

  # Validations
  validates :name, presence: true, length: { minimum: 2, maximum: 100 }
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }, uniqueness: true
  validates :age, numericality: { greater_than_or_equal_to: 18, less_than_or_equal_to: 120 }, allow_nil: true
  validates :status, inclusion: { in: %w[active inactive suspended deleted] }

  # Relations
  has_many :orders, dependent: :destroy

  # Scopes
  scope :active, -> { where(status: 'active') }
  scope :recent, -> { order(created_at: :desc) }
  scope :by_role, ->(role) { where(roles: role) }

  # Callbacks
  before_save :downcase_email
  after_create :send_welcome_email

  # Méthodes
  def active?
    status == 'active'
  end

  def has_role?(role)
    roles.include?(role)
  end

  def increment_login!
    inc(login_count: 1)
    update_attributes(last_login: Time.now.utc)
  end

  def to_public_hash
    {
      id: id.to_s,
      name: name,
      email: email,
      status: status,
      created_at: created_at.iso8601
    }
  end

  private

  def downcase_email
    self.email = email.downcase if email.present?
  end

  def send_welcome_email
    # UserMailer.welcome_email(self).deliver_later
  end
end
```

### Controller Rails

```ruby
# app/controllers/users_controller.rb
class UsersController < ApplicationController
  before_action :set_user, only: [:show, :update, :destroy]

  # GET /users
  def index
    @users = User.active
                 .recent
                 .page(params[:page])
                 .per(params[:per_page] || 20)

    render json: @users
  end

  # GET /users/:id
  def show
    render json: @user.to_public_hash
  end

  # POST /users
  def create
    @user = User.new(user_params)

    if @user.save
      render json: @user.to_public_hash, status: :created
    else
      render json: { errors: @user.errors.full_messages }, status: :unprocessable_entity
    end
  end

  # PATCH/PUT /users/:id
  def update
    if @user.update(user_params)
      render json: @user.to_public_hash
    else
      render json: { errors: @user.errors.full_messages }, status: :unprocessable_entity
    end
  end

  # DELETE /users/:id
  def destroy
    @user.destroy
    head :no_content
  end

  # GET /users/search
  def search
    query = params[:q]

    @users = User.any_of(
      { name: /#{query}/i },
      { email: /#{query}/i }
    ).limit(20)

    render json: @users
  end

  # GET /users/stats
  def stats
    stats = {
      total: User.count,
      by_status: User.collection.aggregate([
        { '$group': { _id: '$status', count: { '$sum': 1 } } }
      ]).to_a,
      recent_signups: User.where(:created_at.gte => 30.days.ago).count
    }

    render json: stats
  end

  private

  def set_user
    @user = User.find(params[:id])
  rescue Mongoid::Errors::DocumentNotFound
    render json: { error: 'User not found' }, status: :not_found
  end

  def user_params
    params.require(:user).permit(:name, :email, :age, roles: [])
  end
end
```

### Routes Rails

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :users do
    collection do
      get :search
      get :stats
    end
  end
end
```

## Intégration Sinatra

```ruby
# app.rb
require 'sinatra'
require 'sinatra/json'
require_relative 'lib/database'
require_relative 'lib/services/user_service'
require_relative 'lib/repositories/user_repository'
require_relative 'lib/models/create_user_dto'
require_relative 'lib/models/update_user_dto'

# Configuration
configure do
  MyApp::Database.connect

  set :user_repository, MyApp::Repositories::UserRepository.new(
    MyApp::Database.collection(:users)
  )

  set :user_service, MyApp::Services::UserService.new(
    settings.user_repository
  )
end

# Routes
get '/users' do
  users = settings.user_repository.find_active_users(
    limit: params[:limit]&.to_i || 20,
    skip: params[:skip]&.to_i || 0
  )

  json users.map(&:to_public_hash)
end

get '/users/:id' do
  user = settings.user_service.find_user_by_id(params[:id])
  json user.to_public_hash
rescue StandardError => e
  status 404
  json error: e.message
end

post '/users' do
  request.body.rewind
  data = JSON.parse(request.body.read, symbolize_names: true)

  dto = MyApp::Models::CreateUserDTO.from_hash(data)
  user = settings.user_service.register_user(dto)

  status 201
  json user.to_public_hash
rescue ArgumentError => e
  status 400
  json error: e.message
rescue StandardError => e
  status 422
  json error: e.message
end

put '/users/:id' do
  request.body.rewind
  data = JSON.parse(request.body.read, symbolize_names: true)

  dto = MyApp::Models::UpdateUserDTO.from_hash(data)
  user = settings.user_service.update_user(params[:id], dto)

  json user.to_public_hash
rescue StandardError => e
  status 400
  json error: e.message
end

delete '/users/:id' do
  success = settings.user_repository.delete_by_id(params[:id])

  if success
    status 204
  else
    status 404
    json error: 'User not found'
  end
end

get '/users/search' do
  query = params[:q]
  users = settings.user_repository.search(query)

  json users.map(&:to_public_hash)
end

get '/stats' do
  stats = settings.user_repository.statistics
  json stats
end

# Fermeture propre
at_exit do
  MyApp::Database.close
end
```

## Gestion d'erreurs

```ruby
# lib/errors/mongodb_error.rb
module MyApp
  module Errors
    class MongoDBError < StandardError
      attr_reader :original_error, :status_code

      def initialize(message, original_error: nil, status_code: 500)
        super(message)
        @original_error = original_error
        @status_code = status_code
      end

      def self.from_mongo_error(error)
        case error
        when Mongo::Error::OperationFailure
          if error.message.include?('E11000')
            new('Duplicate key error', original_error: error, status_code: 409)
          else
            new('Operation failed', original_error: error, status_code: 400)
          end
        when Mongo::Error::NoServerAvailable
          new('Database unavailable', original_error: error, status_code: 503)
        when Mongo::Error::SocketTimeoutError
          new('Database timeout', original_error: error, status_code: 504)
        else
          new('Database error', original_error: error, status_code: 500)
        end
      end
    end
  end
end
```

## Bonnes pratiques

### ✅ DO - À faire

```ruby
# 1. Utiliser des symboles pour les clés
collection.find(email: 'user@example.com')

# 2. Utiliser des blocks pour cleanup automatique
MyApp::Database.collection(:users).find(filter) do |cursor|
  cursor.each { |doc| process(doc) }
end

# 3. Utiliser BSON::ObjectId pour les IDs
id = BSON::ObjectId(id_string)

# 4. Gérer les erreurs spécifiques
begin
  collection.insert_one(document)
rescue Mongo::Error::OperationFailure => e
  raise 'Duplicate email' if e.message.include?('E11000')
end

# 5. Utiliser des scopes dans Mongoid
scope :active, -> { where(status: 'active') }

# 6. Logger les opérations importantes
logger.info "User created: #{user.id}"

# 7. Utiliser Time.now.utc pour les timestamps
created_at: Time.now.utc

# 8. Valider avant d'insérer
raise ArgumentError unless dto.valid?

# 9. Fermer les sessions de transaction
ensure
  session.end_session
end

# 10. Utiliser Mongoid callbacks
before_save :normalize_email
after_create :send_notification
```

### ❌ DON'T - À éviter

```ruby
# ❌ Ne pas créer un nouveau client par requête
# ❌ Ne pas utiliser des strings pour les clés (préférer symboles)
# ❌ Ignorer les erreurs
# ❌ Ne pas valider les ObjectId
# ❌ Utiliser Time.now au lieu de Time.now.utc
# ❌ Ne pas fermer les sessions/curseurs
# ❌ Faire des requêtes N+1
# ❌ Ne pas utiliser d'index
# ❌ Exposer des données sensibles
```

## Comparaison des approches

| Approche | Avantages | Inconvénients |
|----------|-----------|---------------|
| **Driver natif** | Contrôle total, Flexible | Plus verbeux |
| **Mongoid** | Familier (comme ActiveRecord), Validations | Overhead, Moins performant |
| **Sinatra + Driver** | Léger, Simple | Moins de fonctionnalités |

## Conclusion

Le driver Ruby MongoDB offre une excellente expérience idiomatique :

1. **API expressive** : Syntaxe Ruby élégante
2. **Mongoid** : ODM complet pour Rails
3. **Blocks** : Cleanup automatique des ressources
4. **Symboles** : Keys idiomatiques
5. **Callbacks** : Lifecycle hooks
6. **Scopes** : Requêtes réutilisables

**Points clés** :
- Utiliser symboles pour les clés
- Blocks pour cleanup
- Time.now.utc pour timestamps
- BSON::ObjectId pour IDs
- Mongoid pour Rails
- Gestion d'erreurs robuste
- Validation des données
- Index appropriés

---


⏭️ [Connection String et options](/15-drivers-integration-applicative/09-connection-string-options.md)
