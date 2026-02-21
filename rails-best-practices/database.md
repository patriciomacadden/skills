# Database Patterns

## Contents
- UUID primary keys
- Multi-tenancy via account ID
- Composite unique constraints
- UTF8MB4 for full Unicode
- Float-based positioning

## UUID Primary Keys

Configure generators to use UUIDs by default:

```ruby
# config/application.rb
config.generators do |g|
  g.orm :active_record, primary_key_type: :uuid
end

# db/migrate/xxx_enable_uuid.rb
class EnableUuid < ActiveRecord::Migration[7.1]
  def change
    enable_extension "pgcrypto"  # PostgreSQL
  end
end
```

## Multi-Tenancy via Account ID

Include `account_id` on all tenant-scoped tables:

```ruby
# db/migrate/xxx_create_cards.rb
create_table "cards", id: :uuid do |t|
  t.uuid "account_id", null: false
  t.uuid "board_id", null: false
  t.uuid "creator_id", null: false
  t.string "title", null: false
  t.integer "number", null: false

  t.index ["account_id", "number"], unique: true
  t.index ["board_id"]
  t.index ["creator_id"]

  t.timestamps
end

# app/models/card.rb
class Card < ApplicationRecord
  belongs_to :account, default: -> { board.account }

  # Scoped queries through account
  scope :for_account, ->(account) { where(account: account) }
end

# app/models/application_record.rb
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true

  # Default scope for multi-tenant models
  def self.scoped_to_account
    default_scope { where(account_id: Current.account&.id) if Current.account }
  end
end
```

## Composite Unique Constraints

Use composite indexes for per-tenant uniqueness:

```ruby
# Unique card number per account
t.index ["account_id", "number"], unique: true

# Unique membership per room/user pair
t.index ["room_id", "user_id"], unique: true

# Unique slug per account
t.index ["account_id", "slug"], unique: true

# In model validations (for better error messages)
validates :number, uniqueness: { scope: :account_id }
validates :slug, uniqueness: { scope: :account_id }
```

## UTF8MB4 for Full Unicode Support

Always use `utf8mb4` charset for MySQL:

```ruby
# db/migrate/xxx_create_accounts.rb
create_table "accounts", id: :uuid, charset: "utf8mb4", collation: "utf8mb4_0900_ai_ci" do |t|
  t.string "name", null: false
  t.timestamps
end

# config/database.yml
default: &default
  adapter: trilogy
  encoding: utf8mb4
  collation: utf8mb4_0900_ai_ci
```

## Float-Based Positioning

Use floats instead of integers for position to allow insertions:

```ruby
# db/migrate/xxx_add_position_to_cards.rb
add_column :cards, :position_score, :float, null: false, default: 0.0
add_index :cards, [:board_id, :position_score]

# app/models/card/positionable.rb
module Card::Positionable
  extend ActiveSupport::Concern

  included do
    scope :positioned, -> { order(:position_score) }
  end

  def move_before(other)
    target = other.position_score
    previous = siblings.where("position_score < ?", target).maximum(:position_score) || 0
    update!(position_score: (previous + target) / 2.0)
  end

  def move_after(other)
    target = other.position_score
    following = siblings.where("position_score > ?", target).minimum(:position_score)
    new_position = following ? (target + following) / 2.0 : target + 1.0
    update!(position_score: new_position)
  end

  def move_to_top
    first_position = siblings.minimum(:position_score) || 1.0
    update!(position_score: first_position - 1.0)
  end

  def move_to_bottom
    last_position = siblings.maximum(:position_score) || 0.0
    update!(position_score: last_position + 1.0)
  end

  private
    def siblings
      board.cards.where.not(id: id)
    end
end
```

## State Tables Pattern

Create separate tables for state tracking:

```ruby
# db/migrate/xxx_create_closures.rb
create_table :closures, id: :uuid do |t|
  t.uuid :card_id, null: false, index: { unique: true }
  t.uuid :user_id, null: false
  t.timestamps

  t.foreign_key :cards, on_delete: :cascade
  t.foreign_key :users
end

# db/migrate/xxx_create_not_nows.rb
create_table :not_nows, id: :uuid do |t|
  t.uuid :card_id, null: false, index: { unique: true }
  t.uuid :user_id, null: false
  t.datetime :until
  t.timestamps

  t.foreign_key :cards, on_delete: :cascade
end
```

## Counter Caches

```ruby
# db/migrate/xxx_add_counters.rb
add_column :boards, :cards_count, :integer, default: 0, null: false
add_column :rooms, :messages_count, :integer, default: 0, null: false

# app/models/card.rb
belongs_to :board, counter_cache: true

# app/models/message.rb
belongs_to :room, counter_cache: true
```

## Efficient Batch Operations

```ruby
# Use insert_all for bulk inserts (no callbacks)
Membership.insert_all(
  users.map { |u| { room_id: room.id, user_id: u.id, created_at: Time.current } }
)

# Use upsert_all for insert-or-update
Tag.upsert_all(
  tags.map { |t| { name: t, account_id: account.id } },
  unique_by: [:account_id, :name]
)

# Use delete_all for bulk deletes (no callbacks)
Membership.where(room_id: room.id, user_id: users).delete_all

# Use update_all for bulk updates
Card.where(board_id: old_board.id).update_all(board_id: new_board.id)
```

## JSON Columns

```ruby
# db/migrate/xxx_add_settings.rb
add_column :users, :settings, :jsonb, default: {}, null: false
add_index :users, :settings, using: :gin

# app/models/user.rb
class User < ApplicationRecord
  store_accessor :settings, :theme, :notifications_enabled, :timezone

  # With types
  attribute :notifications_enabled, :boolean, default: true
end
```
