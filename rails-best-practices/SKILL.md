---
name: rails-best-practices
description: Applies Rails best practices. ALWAYS USE when working in a Rails application - this includes creating or modifying models, controllers, views, migrations, routes, tests, jobs, mailers, or any Rails-specific code. Also use when discussing Rails patterns, conventions, concerns, Hotwire, Turbo, Active Record, or Action Controller.
---

# Rails Best Practices

Best practices extracted from 37signals' ONCE applications.

## Core Philosophy

**Thin controllers, rich models.** No service objects. Business logic lives in domain models.

```ruby
# Good - Controller delegates to model
def update
  @card.update!(card_params)
  respond_to { |format| format.turbo_stream }
end

# Avoid - Logic in controller
def update
  if @card.valid? && current_user.can_edit?(@card)
    @card.save
    notify_subscribers(@card)
  end
end
```

**Composition over inheritance.** Use concerns (mixins) for capabilities:

```ruby
class Card < ApplicationRecord
  include Closeable, Assignable, Taggable, Broadcastable
end
```

**Database-backed infrastructure.** Prefer Solid Queue, Solid Cache, Solid Cable over Redis.

## Quick Reference

| Topic | Pattern |
|-------|---------|
| State changes | `has_one :closure` not `closed: boolean` |
| Missing assoc | `where.missing(:assignments)` |
| Defaults | `belongs_to :creator, default: -> { Current.user }` |
| Polymorphism | `delegated_type` over STI |
| Enums | `suffix: true` to avoid conflicts |
| Routes | Singular resources for state (`resource :closure`) |
| Params | `params.expect(card: [:title])` (Rails 8+) |

## Detailed References

- **Models**: See [models.md](models.md) for concerns, naming, delegation patterns
- **Controllers**: See [controllers.md](controllers.md) for auth, params, responses
- **Views**: See [views.md](views.md) for partials, caching, helpers
- **Hotwire**: See [hotwire.md](hotwire.md) for Turbo Streams, Stimulus, ActionCable
- **Testing**: See [testing.md](testing.md) for fixtures, assertions, helpers
- **Database**: See [database.md](database.md) for UUIDs, multi-tenancy, positioning
- **Security**: See [security.md](security.md) for auth, sanitization, webhooks

## Model Patterns (Summary)

### Concern Organization

Place concerns in subdirectories matching the model:

```
app/models/
  card.rb
  card/
    closeable.rb
    assignable.rb
```

### State as Records

```ruby
# Instead of: closed: boolean
has_one :closure, dependent: :destroy
scope :closed, -> { joins(:closure) }
scope :open, -> { where.missing(:closure) }

def close(user: Current.user)
  create_closure!(user: user) unless closed?
end

def closed?
  closure.present?
end
```

### Domain-Driven Naming

```ruby
# Good - Expressive domain language
def incinerate
  transaction do
    decease
    erect_tombstone
  end
end

# Avoid - Generic implementation terms
def soft_delete
  mark_as_deleted
  create_deletion_record
end
```

## Controller Patterns (Summary)

### DSL-Style Authentication

```ruby
module Authentication
  extend ActiveSupport::Concern

  included do
    before_action :require_authentication
  end

  class_methods do
    def allow_unauthenticated_access(**options)
      skip_before_action :require_authentication, **options
    end
  end
end
```

### Scoped Resource Finding

```ruby
def set_card
  @card = Current.user.accessible_cards.find_by!(number: params[:id])
end
```

### RESTful State Changes

```ruby
# config/routes.rb
resources :cards do
  scope module: :cards do
    resource :closure      # POST /cards/:id/closure
    resource :publish      # POST /cards/:id/publish
  end
end

# app/controllers/cards/closures_controller.rb
class Cards::ClosuresController < ApplicationController
  def create
    @card.close
    redirect_to @card
  end

  def destroy
    @card.reopen
    redirect_to @card
  end
end
```

## Callback Guidelines

**Use callbacks for:** Simple, orthogonal operations (broadcasts, notifications, timestamps)

```ruby
after_create_commit :broadcast_to_room
after_create_commit :notify_mentions
belongs_to :creator, default: -> { Current.person }
```

**Avoid callbacks for:** Complex orchestration with dependencies

```ruby
# Avoid - Hard to follow, fragile
after_create :reserve_inventory
after_create :charge_payment
after_create :send_confirmation

# Better - Explicit orchestration
def place
  transaction do
    reserve_inventory!
    charge_payment!
    send_confirmation
  end
end
```

## Background Jobs

Jobs coordinate only; logic lives in models:

```ruby
# Job - Thin wrapper
class Card::CleanJob < ApplicationJob
  def perform(card)
    card.clean_inaccessible_data
  end
end

# Model - Contains logic
def clean_later
  Card::CleanJob.perform_later(self)
end

def clean_now
  # actual cleaning logic
end
```

## CurrentAttributes

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :session, :user, :account
  attribute :request_id, :ip_address

  delegate :identity, to: :session, allow_nil: true
end

# Automatic audit tracking
class Comment < ApplicationRecord
  belongs_to :creator, default: -> { Current.person }
end
```

## Code Style

### Method Ordering

1. Class methods
2. Public instance methods (`initialize` first)
3. Protected methods (alphabetical order)
4. Private methods (alphabetical order)

### Bang Methods

Only use bang methods when a non-bang version exists: save/save!, create/create!

## Gem Stack

| Purpose | Gem |
|---------|-----|
| Jobs | `solid_queue` |
| Cache | `solid_cache` |
| WebSocket | `solid_cable` |
| Assets | `propshaft` |
| JS | `importmap-rails` |
| Frontend | `turbo-rails`, `stimulus-rails` |
| Style | `rubocop-rails-omakase`, `standard` |
