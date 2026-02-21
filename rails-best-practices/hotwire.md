# Hotwire Patterns

## Contents
- Turbo Streams for real-time updates
- Stimulus controller conventions
- Data attributes for configuration
- ActionCable presence tracking
- Import maps

## Turbo Streams for Real-Time Updates

Subscribe to model broadcasts in views:

```erb
<%# Subscribe to card updates %>
<%= turbo_stream_from @card %>

<%# Subscribe to multiple streams %>
<%= turbo_stream_from @card %>
<%= turbo_stream_from @card, :activity %>
<%= turbo_stream_from Current.user, :notifications %>
```

Broadcast changes from models:

```ruby
# app/models/message/broadcasts.rb
module Message::Broadcasts
  extend ActiveSupport::Concern

  included do
    after_create_commit :broadcast_create
    after_update_commit :broadcast_update
    after_destroy_commit :broadcast_remove
  end

  private
    def broadcast_create
      broadcast_append_to room, :messages, target: [room, :messages]
    end

    def broadcast_update
      broadcast_replace_to room, :messages
    end

    def broadcast_remove
      broadcast_remove_to room, :messages
    end
end
```

## Stimulus Controller Conventions

Use static declarations and private fields:

```javascript
// app/javascript/controllers/autosave_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static classes = ["clean", "dirty", "saving"]
  static values = { url: String, delay: { type: Number, default: 1000 } }
  static targets = ["form", "status"]

  #timer
  #dirty = false

  connect() {
    this.element.addEventListener("input", this.#markDirty)
  }

  disconnect() {
    clearTimeout(this.#timer)
  }

  schedule() {
    this.#dirty = true
    this.element.classList.add(this.dirtyClass)
    clearTimeout(this.#timer)
    this.#timer = setTimeout(() => this.save(), this.delayValue)
  }

  async save() {
    if (!this.#dirty) return

    this.element.classList.add(this.savingClass)

    try {
      const response = await fetch(this.urlValue, {
        method: "PATCH",
        body: new FormData(this.formTarget),
        headers: { "Accept": "text/vnd.turbo-stream.html" }
      })

      if (response.ok) {
        this.#dirty = false
        this.element.classList.remove(this.dirtyClass)
        this.element.classList.add(this.cleanClass)
      }
    } finally {
      this.element.classList.remove(this.savingClass)
    }
  }

  #markDirty = () => {
    this.#dirty = true
  }
}
```

## Data Attributes for Configuration

Pass configuration via data attributes:

```erb
<div data-controller="beacon lightbox"
     data-beacon-url-value="<%= card_reading_path(@card) %>"
     data-beacon-interval-value="30000"
     data-lightbox-open-class="lightbox--open">
  <%= render @card %>
</div>

<form data-controller="autosave"
      data-autosave-url-value="<%= card_path(@card) %>"
      data-autosave-delay-value="1000"
      data-action="input->autosave#schedule">
  <%= form_fields %>
</form>
```

## Stimulus Action Patterns

```erb
<%# Multiple actions on one element %>
<input type="text"
       data-action="input->search#filter keydown.enter->search#submit blur->search#clear">

<%# Window/document events %>
<div data-controller="shortcuts"
     data-action="keydown@window->shortcuts#handle">
</div>

<%# Debounced actions %>
<input data-controller="search"
       data-action="input->search#query:debounce(300)">
```

## ActionCable Presence Tracking

Track user presence with connection counts:

```ruby
# app/channels/presence_channel.rb
class PresenceChannel < ApplicationCable::Channel
  on_subscribe :present, unless: :subscription_rejected?
  on_unsubscribe :absent

  private
    def present
      membership.present  # Increments connection count
    end

    def absent
      membership.absent   # Decrements connection count
    end

    def membership
      @membership ||= room.memberships.find_by!(user: current_user)
    end

    def room
      Room.find(params[:room_id])
    end
end

# app/models/membership.rb
class Membership < ApplicationRecord
  def present
    increment!(:connections)
    broadcast_presence if connections == 1
  end

  def absent
    decrement!(:connections)
    broadcast_absence if connections.zero?
  end

  private
    def broadcast_presence
      broadcast_append_to room, :presences, target: [room, :presences]
    end

    def broadcast_absence
      broadcast_remove_to room, :presences
    end
end
```

## Import Maps Without Build Step

```ruby
# config/importmap.rb
pin "application"
pin "@hotwired/turbo-rails", to: "turbo.min.js"
pin "@hotwired/stimulus", to: "stimulus.min.js"
pin "@hotwired/stimulus-loading", to: "stimulus-loading.js"
pin_all_from "app/javascript/controllers", under: "controllers"

# External packages
pin "trix"
pin "@rails/actiontext", to: "actiontext.esm.js"

# Vendor packages
pin_all_from "vendor/javascript"
```

## Turbo Morphing (Rails 7.2+)

```erb
<%# Enable morphing for smoother updates %>
<%= turbo_refreshes_with method: :morph, scroll: :preserve %>

<%# In layout %>
<html>
<head>
  <%= turbo_refreshes_with method: :morph %>
</head>
</html>
```

## Custom Turbo Stream Actions

```ruby
# config/initializers/turbo_streams.rb
Turbo::Streams::TagBuilder.define_turbo_stream_action :redirect do |url|
  turbo_stream_action_tag :redirect, url: url
end

# Usage in controller
render turbo_stream: turbo_stream.redirect(root_path)
```

```javascript
// app/javascript/application.js
Turbo.StreamActions.redirect = function() {
  Turbo.visit(this.getAttribute("url"))
}
```
