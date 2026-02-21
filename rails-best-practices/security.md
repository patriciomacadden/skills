# Security Patterns

## Contents
- CurrentAttributes pattern
- Secure session cookies
- Signed IDs for tokens
- Role-based authorization
- HTML sanitization
- SSRF prevention
- Webhook security

## CurrentAttributes Pattern

Use `ActiveSupport::CurrentAttributes` for request context:

```ruby
# app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  attribute :session, :user, :identity, :account
  attribute :request_id, :user_agent, :ip_address

  def session=(value)
    super(value)
    self.identity = session&.identity
  end

  def identity=(value)
    super(value)
    self.user = identity&.users&.find_by(account: account)
  end
end

# app/controllers/concerns/set_current_request.rb
module SetCurrentRequest
  extend ActiveSupport::Concern

  included do
    before_action :set_current_request_details
  end

  private
    def set_current_request_details
      Current.request_id = request.request_id
      Current.user_agent = request.user_agent
      Current.ip_address = request.remote_ip
    end
end
```

**Automatic audit tracking with association defaults:**

```ruby
class Comment < ApplicationRecord
  belongs_to :creator, class_name: "Person", default: -> { Current.person }
  belongs_to :updater, class_name: "Person", default: -> { Current.person }
end

class Event < ApplicationRecord
  belongs_to :creator, class_name: "Person", default: -> { Current.person }

  after_create :record_creation_metadata

  private
    def record_creation_metadata
      update_columns(
        created_from_ip: Current.ip_address,
        created_with_user_agent: Current.user_agent
      )
    end
end
```

## Secure Session Cookies

Use signed, HTTP-only cookies with SameSite:

```ruby
# app/controllers/concerns/authentication.rb
module Authentication
  private
    def set_current_session(session)
      Current.session = session
      cookies.signed.permanent[:session_token] = {
        value: session.token,
        httponly: true,
        same_site: :lax
      }
    end

    def clear_current_session
      Current.session = nil
      cookies.delete(:session_token)
    end

    def restore_authentication
      if token = cookies.signed[:session_token]
        Current.session = Session.find_by(token: token)
      end
    end
end

# app/models/session.rb
class Session < ApplicationRecord
  has_secure_token

  belongs_to :identity

  def self.authenticate(email:, password:)
    identity = Identity.authenticate_by(email: email, password: password)
    identity&.sessions&.create
  end
end
```

## Signed IDs for Secure Tokens

Use Rails signed IDs for time-limited tokens:

```ruby
# app/models/user/transferable.rb
module User::Transferable
  extend ActiveSupport::Concern

  TRANSFER_LINK_EXPIRY_DURATION = 24.hours

  def transfer_id
    signed_id(purpose: :transfer, expires_in: TRANSFER_LINK_EXPIRY_DURATION)
  end

  class_methods do
    def find_by_transfer_id(id)
      find_signed(id, purpose: :transfer)
    end
  end
end

# app/models/user/recoverable.rb
module User::Recoverable
  PASSWORD_RESET_EXPIRY = 4.hours

  def password_reset_token
    signed_id(purpose: :password_reset, expires_in: PASSWORD_RESET_EXPIRY)
  end

  class_methods do
    def find_by_password_reset_token(token)
      find_signed(token, purpose: :password_reset)
    end
  end
end

# Usage
user = User.find_by_transfer_id(params[:token])
# Returns nil if token is invalid, expired, or tampered
```

## Role-Based Authorization

Keep authorization simple with enums and predicates:

```ruby
# app/models/user/role.rb
module User::Role
  extend ActiveSupport::Concern

  included do
    enum :role, %i[member administrator bot], default: :member
  end

  def admin?
    role.in?(%w[owner admin administrator])
  end

  def can_administer?(record = nil)
    administrator? || self == record&.creator
  end

  def can_edit?(resource)
    admin? || resource.creator == self
  end

  def can_delete?(resource)
    admin? || resource.creator == self
  end
end

# app/controllers/concerns/authorization.rb
module Authorization
  extend ActiveSupport::Concern

  class NotAuthorizedError < StandardError; end

  included do
    rescue_from NotAuthorizedError, with: :render_forbidden
  end

  private
    def authorize(action, resource)
      unless Current.user.send("can_#{action}?", resource)
        raise NotAuthorizedError
      end
    end

    def render_forbidden
      render plain: "Forbidden", status: :forbidden
    end
end
```

## HTML Sanitization

Define allowed tags for rich content:

```ruby
# app/models/html_scrubber.rb
class HtmlScrubber < Rails::Html::PermitScrubber
  def initialize
    super
    self.tags = Rails::Html::WhiteListSanitizer.allowed_tags + %w[
      audio video source table thead tbody tr th td details summary
    ]
    self.attributes = Rails::Html::WhiteListSanitizer.allowed_attributes + %w[
      controls src type colspan rowspan
    ]
  end
end

# app/helpers/sanitize_helper.rb
module SanitizeHelper
  def sanitize_rich_text(html)
    sanitize(html, scrubber: HtmlScrubber.new)
  end
end

# config/initializers/action_text.rb
Rails.application.config.after_initialize do
  ActionText::ContentHelper.sanitizer = Rails::Html::Sanitizer.safe_list_sanitizer.new
end
```

## SSRF Prevention

Guard against server-side request forgery:

```ruby
# app/models/concerns/url_validator.rb
module UrlValidator
  extend ActiveSupport::Concern

  private
    def private_network_request?(uri)
      ip = IPAddr.new(Resolv.getaddress(uri.host))
      ip.private? || ip.loopback? || ip.link_local?
    rescue Resolv::ResolvError
      true  # Block unresolvable hosts
    end

    def safe_url?(url)
      uri = URI.parse(url)
      return false unless uri.scheme.in?(%w[http https])
      return false if private_network_request?(uri)
      true
    rescue URI::InvalidURIError
      false
    end
end

# Usage
class WebhookDelivery
  include UrlValidator

  def deliver(url, payload)
    raise ArgumentError, "Unsafe URL" unless safe_url?(url)
    # proceed with delivery
  end
end
```

## Webhook Security

Sign webhook payloads and verify URLs:

```ruby
# app/models/webhook.rb
class Webhook < ApplicationRecord
  SLACK_WEBHOOK_URL_REGEX = %r{//hooks\.slack\.com/services/T[^/]+/B[^/]+/[^/]+\Z}i
  PERMITTED_URL_PATTERN = %r{\Ahttps://}i

  has_secure_token :signing_secret

  validates :url, format: { with: PERMITTED_URL_PATTERN }
  validate :url_not_private_network

  def sign_payload(payload)
    OpenSSL::HMAC.hexdigest("SHA256", signing_secret, payload.to_json)
  end

  def deliver(payload)
    signature = sign_payload(payload)

    HTTP.headers(
      "Content-Type" => "application/json",
      "X-Webhook-Signature" => signature
    ).post(url, json: payload)
  end

  private
    def url_not_private_network
      uri = URI.parse(url)
      ip = IPAddr.new(Resolv.getaddress(uri.host))

      if ip.private? || ip.loopback?
        errors.add(:url, "cannot point to private network")
      end
    rescue URI::InvalidURIError, Resolv::ResolvError
      errors.add(:url, "is invalid")
    end
end
```

## Parameter Filtering

```ruby
# config/initializers/filter_parameter_logging.rb
Rails.application.config.filter_parameters += [
  :password, :password_confirmation,
  :token, :secret, :api_key,
  :credit_card, :card_number, :cvv,
  :ssn, :social_security
]
```

## Content Security Policy

```ruby
# config/initializers/content_security_policy.rb
Rails.application.configure do
  config.content_security_policy do |policy|
    policy.default_src :self
    policy.font_src    :self, :data
    policy.img_src     :self, :data, :blob
    policy.object_src  :none
    policy.script_src  :self
    policy.style_src   :self, :unsafe_inline
    policy.connect_src :self

    if Rails.env.development?
      policy.script_src *policy.script_src, :unsafe_eval
      policy.connect_src *policy.connect_src, "ws://localhost:*"
    end
  end

  config.content_security_policy_nonce_generator = ->(request) {
    request.session.id.to_s
  }
  config.content_security_policy_nonce_directives = %w[script-src]
end
```
