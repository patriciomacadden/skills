# Model Patterns

## Contents
- Concern-based organization
- State as separate records
- Domain-driven naming
- Default values with Current
- Delegated types
- Custom association methods
- Role-based concerns (DCI)
- Helper object delegation

## Concern-Based Organization

Place each concern in a subdirectory matching the model name:

```
app/models/
  card.rb
  card/
    assignable.rb
    closeable.rb
    postponable.rb
```

**Design principles:**
- **"Has trait" / "Acts as" semantics**: `include Closeable` means the model "is closeable"
- **Organization over reuse**: Concerns primarily organize code within a single model
- **Complement traditional OOP**: Use alongside POROs, inheritance, and composition

```ruby
# app/models/card/closeable.rb
module Card::Closeable
  extend ActiveSupport::Concern

  included do
    has_one :closure, dependent: :destroy
    scope :closed, -> { joins(:closure) }
    scope :open, -> { where.missing(:closure) }
  end

  def close(user: Current.user)
    unless closed?
      transaction do
        create_closure!(user: user)
        track_event(:closed, creator: user)
      end
    end
  end

  def closed?
    closure.present?
  end
end
```

## State as Separate Records

Store state changes as separate `has_one` records instead of boolean columns:

```ruby
# Instead of: closed: boolean
has_one :closure, dependent: :destroy
scope :closed, -> { joins(:closure) }
scope :open, -> { where.missing(:closure) }

# Instead of: postponed: boolean
has_one :not_now, dependent: :destroy
scope :postponed, -> { joins(:not_now) }
scope :active, -> { where.missing(:not_now) }
```

Benefits:
- Track who changed what and when
- Store additional metadata about the state change
- Query history of state transitions

## Domain-Driven Naming

Use expressive, domain-specific terminology:

```ruby
# Good - Domain-specific, expressive names
class Recording
  def incinerate
    transaction do
      decease
      erect_tombstone
    end
  end

  private
    def decease
      update!(deceased_at: Time.current)
    end

    def erect_tombstone
      create_tombstone!(title: title, deleted_by: Current.person)
    end
end

# Poor - Generic, implementation-focused names
class Recording
  def soft_delete
    transaction do
      mark_as_deleted
      create_deletion_record
    end
  end
end
```

**Techniques:**
- Use a dictionary to find precise vocabulary
- Prefer specific terms over generic ones (`decease` vs `remove`)
- Name methods after what they mean in the domain

## Default Values with Current Context

Use lambdas with `Current` for intelligent defaults:

```ruby
class Card < ApplicationRecord
  belongs_to :account, default: -> { board.account }
  belongs_to :creator, class_name: "User", default: -> { Current.user }
end

class Message < ApplicationRecord
  belongs_to :creator, class_name: "User", default: -> { Current.user }
end
```

## Delegated Types Over STI

For polymorphic content, prefer `delegated_type` (Rails 6.1+):

```ruby
# app/models/leaf.rb
class Leaf < ApplicationRecord
  delegated_type :leafable, types: %w[Page Section Picture], dependent: :destroy
end

# app/models/page.rb
class Page < ApplicationRecord
  include Leafable
  has_markdown :body
end
```

## Enums with Suffix

Use `suffix: true` to avoid naming conflicts:

```ruby
enum :theme, %w[black blue green magenta].index_by(&:itself), suffix: true, default: :blue
# Creates: theme_blue?, theme_blue!

enum :role, %i[member administrator bot]
# Creates: member?, administrator?, bot?
```

## Custom Association Methods

Extend associations with custom methods:

```ruby
has_many :memberships do
  def grant_to(users)
    room = proxy_association.owner
    Membership.insert_all(users.map { |u| { room_id: room.id, user_id: u.id } })
  end

  def revoke_from(users)
    where(user_id: users).delete_all
  end
end
```

## Role-Based Concerns (DCI-Inspired)

Organize code around roles objects play in specific contexts:

```ruby
# app/models/account/petitioner.rb
module Account::Petitioner
  extend ActiveSupport::Concern

  def petition_for(resource)
    petitions.create!(resource: resource, requested_at: Time.current)
  end

  def pending_petitions
    petitions.where(granted_at: nil, denied_at: nil)
  end
end

# app/models/account/examiner.rb
module Account::Examiner
  extend ActiveSupport::Concern

  def grant_petition(petition)
    petition.update!(granted_at: Time.current, examiner: self)
  end

  def deny_petition(petition, reason:)
    petition.update!(denied_at: Time.current, examiner: self, denial_reason: reason)
  end
end

# app/models/account.rb
class Account < ApplicationRecord
  include Petitioner  # Can request access
  include Examiner    # Can grant/deny access
end
```

## Helper Object Delegation

For complex operations, delegate to specialized helper classes:

```ruby
# app/models/recording.rb
class Recording < ApplicationRecord
  def incinerate
    Recording::Incineration.new(self).run
  end
end

# app/models/recording/incineration.rb
class Recording::Incineration
  def initialize(recording)
    @recording = recording
  end

  def run
    @recording.transaction do
      delete_associated_data
      notify_stakeholders
      schedule_storage_cleanup
      mark_as_incinerated
    end
  end

  private
    def delete_associated_data
      @recording.comments.destroy_all
      @recording.attachments.purge_later
    end

    def notify_stakeholders
      @recording.subscribers.each do |subscriber|
        IncinerationMailer.notify(subscriber, @recording).deliver_later
      end
    end

    def schedule_storage_cleanup
      StorageCleanupJob.perform_later(@recording.storage_key)
    end

    def mark_as_incinerated
      @recording.update!(incinerated_at: Time.current)
    end
end
```

## Intention-Revealing Method Names

Methods should express business intent:

```ruby
# Good
def postpone(user: Current.user)
def toggle_assignment(user)
def editable?(user: Current.user)

# Avoid
def set_postponed_flag(value)
def update_assignment_status(user, status)
```

## where.missing for Absent Associations

Rails 6.1+ provides cleaner queries:

```ruby
# Good
scope :unassigned, -> { where.missing(:assignments) }
scope :open, -> { where.missing(:closure) }

# Avoid
scope :unassigned, -> { left_joins(:assignments).where(assignments: { id: nil }) }
```
