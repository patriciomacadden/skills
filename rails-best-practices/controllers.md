# Controller Patterns

## Contents
- Concern-based organization
- DSL-style class methods
- Parameter handling
- Multi-format responses
- Scoped resource finding
- Rate limiting
- Singular resources for state changes

## Concern-Based Organization

Group cross-cutting concerns into controller modules:

```ruby
class ApplicationController < ActionController::Base
  include Authentication
  include Authorization
  include SetCurrentRequest
  include VersionHeaders
end
```

## DSL-Style Class Methods

Create declarative APIs for common patterns:

```ruby
module Authentication
  extend ActiveSupport::Concern

  included do
    before_action :require_authentication
    helper_method :signed_in?
  end

  class_methods do
    def allow_unauthenticated_access(**options)
      skip_before_action :require_authentication, **options
      before_action :restore_authentication, **options
    end

    def require_unauthenticated_access(**options)
      allow_unauthenticated_access **options
      before_action :redirect_signed_in_user_to_root, **options
    end
  end

  private
    def require_authentication
      restore_authentication
      request_authentication unless authenticated?
    end

    def restore_authentication
      Current.session = Session.find_by_token(cookies.signed[:session_token])
    end

    def authenticated?
      Current.session.present?
    end

    def signed_in?
      authenticated?
    end
end

# Usage in controller
class SessionsController < ApplicationController
  allow_unauthenticated_access only: %i[new create]
end
```

## Modern Parameter Handling

Use `params.expect` (Rails 8.0+) for explicit parameter requirements:

```ruby
def card_params
  params.expect(card: [:title, :description, :image, :created_at])
end

# For nested attributes
def order_params
  params.expect(order: [:customer_id, line_items: [[:product_id, :quantity]]])
end
```

## Multi-Format Responses

Support multiple response formats in single actions:

```ruby
def create
  @card = @board.cards.create!(card_params)

  respond_to do |format|
    format.html { redirect_to card_path(@card) }
    format.turbo_stream
    format.json { head :created, location: card_path(@card) }
  end
end

def update
  @card.update!(card_params)

  respond_to do |format|
    format.turbo_stream { render }
    format.html { head :no_content }
    format.json { render :show }
  end
end

def destroy
  @card.destroy

  respond_to do |format|
    format.html { redirect_to cards_path }
    format.turbo_stream
    format.json { head :no_content }
  end
end
```

## Scoped Resource Finding

Find resources through the current user's accessible scope:

```ruby
def set_card
  @card = Current.user.accessible_cards.find_by!(number: params[:id])
end

def set_book
  @book = Book.accessible_or_published(user: Current.user).find(params[:book_id])
end

def set_room
  @room = Current.account.rooms.find(params[:room_id])
end
```

## Rate Limiting (Rails 7.2+)

Use built-in rate limiting for protection:

```ruby
class SessionsController < ApplicationController
  rate_limit to: 10, within: 3.minutes, only: :create,
             with: -> { render_rejection :too_many_requests }
end

class PasswordResetsController < ApplicationController
  rate_limit to: 5, within: 1.hour, only: :create
end
```

## Singular Resources for State Changes

Model state changes as singular nested resources:

```ruby
# config/routes.rb
resources :cards do
  scope module: :cards do
    resource :closure      # POST /cards/:id/closure (close)
    resource :publish      # POST /cards/:id/publish
    resource :goldness     # POST/DELETE /cards/:id/goldness
  end
end

# app/controllers/cards/closures_controller.rb
class Cards::ClosuresController < ApplicationController
  before_action :set_card

  def create
    @card.close
    redirect_to @card
  end

  def destroy
    @card.reopen
    redirect_to @card
  end

  private
    def set_card
      @card = Current.user.accessible_cards.find(params[:card_id])
    end
end

# app/controllers/cards/publishes_controller.rb
class Cards::PublishesController < ApplicationController
  before_action :set_card

  def create
    @card.publish
    redirect_to @card
  end

  def destroy
    @card.unpublish
    redirect_to @card
  end
end
```

## Version Headers

Add version headers to all responses:

```ruby
# app/controllers/concerns/version_headers.rb
module VersionHeaders
  extend ActiveSupport::Concern

  included do
    before_action :set_version_headers
  end

  private
    def set_version_headers
      response.headers["X-Version"] = Rails.application.config.app_version
      response.headers["X-Rev"] = Rails.application.config.git_revision
    end
end
```

## ETag Versioning

Invalidate browser cache on app version changes:

```ruby
class ApplicationController < ActionController::Base
  etag { "v1" }
  stale_when_importmap_changes
end
```

## HTTP Caching with fresh_when

Use ETags for conditional GET requests:

```ruby
def index
  @messages = find_paged_messages
  if @messages.any?
    fresh_when @messages
  else
    head :no_content
  end
end

def show
  @card = Current.user.accessible_cards.find(params[:id])
  fresh_when @card
end
```
