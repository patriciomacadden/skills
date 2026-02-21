# Testing Patterns

## Contents
- Parallel test execution
- Custom test helpers
- Assertion blocks
- Current context isolation
- Fixture UUIDs

## Parallel Test Execution

Enable parallel testing for speed:

```ruby
# test/test_helper.rb
class ActiveSupport::TestCase
  parallelize(workers: :number_of_processors)
  fixtures :all
end
```

## Custom Test Helpers

Create reusable helpers for common operations:

```ruby
# test/test_helpers/session_test_helper.rb
module SessionTestHelper
  def sign_in(user)
    user = users(user) unless user.is_a?(User)
    post session_url, params: {
      email_address: user.email_address,
      password: "secret123456"
    }
    assert cookies[:session_token].present?
  end

  def sign_out
    delete session_url
    assert_not cookies[:session_token].present?
  end

  def signed_in_as(user)
    sign_in(user)
    yield
  ensure
    sign_out
  end
end

# test/test_helper.rb
class ActionDispatch::IntegrationTest
  include SessionTestHelper
end
```

## Account Context Helper

```ruby
# test/test_helpers/account_test_helper.rb
module AccountTestHelper
  def with_account(account)
    account = accounts(account) unless account.is_a?(Account)
    Current.account = account
    yield
  ensure
    Current.account = nil
  end

  def within_account(account, &block)
    with_account(account, &block)
  end
end

# Usage in tests
test "creates card in account context" do
  with_account(:acme) do
    assert_difference -> { Card.count } do
      Card.create!(title: "Test")
    end
  end
end
```

## Assertion Blocks

Use `assert_difference` and `assert_changes`:

```ruby
test "creating a message enqueues push notification" do
  assert_enqueued_jobs 1, only: [Room::PushMessageJob] do
    create_new_message_in rooms(:designers)
  end
end

test "create account with owner" do
  assert_changes -> { Account.count }, +1 do
    assert_changes -> { User.count }, +2 do
      Account.create_with_owner(account: {...}, owner: {...})
    end
  end
end

test "closing card updates timestamp" do
  card = cards(:open)

  assert_changes -> { card.reload.updated_at } do
    card.close
  end
end

test "does not create duplicate membership" do
  assert_no_difference -> { Membership.count } do
    @room.memberships.grant_to([@existing_member])
  end
end
```

## Broadcast Assertions

```ruby
test "message broadcasts to room" do
  assert_broadcasts rooms(:general), 1 do
    Message.create!(room: rooms(:general), body: "Hello")
  end
end

test "broadcasts correct content" do
  assert_broadcast_on rooms(:general) do
    Message.create!(room: rooms(:general), body: "Hello")
  end
end
```

## Current Context Isolation

Reset `Current` attributes in tests:

```ruby
# test/test_helper.rb
class ActiveSupport::TestCase
  setup do
    Current.reset
  end

  teardown do
    Current.reset
  end
end

# In specific tests
setup do
  Current.account = accounts(:acme)
  Current.user = users(:alice)
end

teardown do
  Current.reset
end

test "something without account context" do
  Current.without_account do
    # test code here
  end
end
```

## System Test Helpers

```ruby
# test/test_helpers/system_test_helper.rb
module SystemTestHelper
  def login_as(user)
    user = users(user) unless user.is_a?(User)
    visit new_session_path
    fill_in "Email", with: user.email_address
    fill_in "Password", with: "secret123456"
    click_button "Sign in"
    assert_text "Dashboard"
  end

  def within_turbo_frame(id, &block)
    within("turbo-frame##{id}", &block)
  end

  def wait_for_turbo
    assert_no_css ".turbo-progress-bar"
  end
end
```

## Fixture UUIDs with Deterministic Ordering

Generate deterministic UUIDs that maintain sort order:

```ruby
# test/support/fixture_uuid.rb
module FixtureUUID
  def generate_fixture_uuid(label)
    fixture_int = Zlib.crc32("fixtures/#{label}") % (2**30 - 1)
    base_time = Time.utc(2024, 1, 1, 0, 0, 0)
    timestamp = base_time + (fixture_int / 1000.0)
    uuid_v7_with_timestamp(timestamp, label)
  end

  private
    def uuid_v7_with_timestamp(timestamp, label)
      ms = (timestamp.to_f * 1000).to_i
      random_bytes = Digest::SHA256.digest(label)[0, 10]

      # UUID v7 format
      format(
        "%08x-%04x-7%03x-%04x-%012x",
        (ms >> 16) & 0xFFFFFFFF,
        ms & 0xFFFF,
        random_bytes[0..1].unpack1("n") & 0x0FFF,
        (random_bytes[2..3].unpack1("n") & 0x3FFF) | 0x8000,
        random_bytes[4..9].unpack1("Q>") & 0xFFFFFFFFFFFF
      )
    end
end
```

## Testing Callbacks

```ruby
test "broadcasts on create" do
  assert_broadcasts @room, 1 do
    Message.create!(room: @room, body: "Test")
  end
end

test "enqueues notification job" do
  assert_enqueued_with(job: NotifyMentionsJob) do
    Message.create!(room: @room, body: "Hey @alice")
  end
end

test "skips callback with suppress" do
  Event.suppress do
    assert_no_difference -> { Event.count } do
      User.create!(name: "Test")
    end
  end
end
```

## Testing Concerns

```ruby
# test/models/concerns/closeable_test.rb
class CloseableTest < ActiveSupport::TestCase
  setup do
    @card = cards(:open)
  end

  test "close creates closure" do
    assert_difference -> { Closure.count } do
      @card.close
    end
  end

  test "close is idempotent" do
    @card.close

    assert_no_difference -> { Closure.count } do
      @card.close
    end
  end

  test "closed? returns true when closed" do
    assert_not @card.closed?
    @card.close
    assert @card.closed?
  end

  test "open scope excludes closed cards" do
    @card.close
    assert_not_includes Card.open, @card
  end
end
```
