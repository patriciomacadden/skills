# View Patterns

## Contents
- Partials for everything
- Named parameters in partials
- Stimulus data attributes in helpers
- Content-based caching
- Turbo Stream templates

## Partials for Everything

Break views into focused partials:

```erb
<%# app/views/cards/show.html.erb %>
<%= turbo_stream_from @card %>
<%= turbo_stream_from @card, :activity %>

<div data-controller="beacon lightbox">
  <%= render "cards/container", card: @card %>
  <%= render "cards/messages", card: @card %>
</div>
```

## Named Parameters in Partials

Always use named parameters for clarity:

```erb
<%# Good - Explicit parameter names %>
<%= render "cards/container", card: @card %>
<%= render "messages/message", message: @message, room: @room %>
<%= render "users/avatar", user: @user, size: :small %>

<%# Avoid - Implicit locals %>
<%= render "cards/container" %>
<%= render partial: "messages/message", object: @message %>
```

## Stimulus Data Attributes in Helpers

Encapsulate complex Stimulus configurations in helpers:

```ruby
# app/helpers/books_helper.rb
module BooksHelper
  def book_toc_tag(book, &)
    tag.ol class: "toc", tabindex: 0,
      data: {
        controller: "arrangement",
        action: arrangement_actions,
        arrangement_cursor_class: "arrangement-cursor",
        arrangement_selected_class: "arrangement-selected",
        arrangement_url_value: book_leaves_moves_url(book)
      }, &
  end

  private
    def arrangement_actions
      %w[
        keydown->arrangement#navigate
        dragstart->arrangement#dragStart
        dragend->arrangement#dragEnd
      ].join(" ")
    end
end

# Usage in view
<%= book_toc_tag(@book) do %>
  <% @book.leaves.each do |leaf| %>
    <%= render "leaves/leaf", leaf: leaf %>
  <% end %>
<% end %>
```

## Form Helpers with Stimulus

```ruby
# app/helpers/forms_helper.rb
module FormsHelper
  def autosave_form_tag(url:, &)
    form_with url: url, data: {
      controller: "autosave",
      autosave_url_value: url,
      autosave_delay_value: 1000,
      action: "input->autosave#schedule submit->autosave#save"
    }, &
  end

  def character_counter_field(form, attribute, max:)
    form.text_area attribute, data: {
      controller: "character-counter",
      character_counter_max_value: max,
      action: "input->character-counter#update"
    }
  end
end
```

## Content-Based Caching

Cache with meaningful dependencies:

```erb
<%# Simple model caching %>
<% cache message do %>
  <%= render "messages/content", message: message %>
<% end %>

<%# Multi-key caching %>
<% cache [@book, @book.editable?] do %>
  <%= render "books/content", book: @book %>
<% end %>

<%# Collection caching %>
<%= render partial: "cards/card", collection: @cards, cached: true %>

<%# Conditional caching %>
<% cache_if current_user.premium?, @dashboard do %>
  <%= render "dashboards/premium", dashboard: @dashboard %>
<% end %>
```

## Turbo Stream Templates

```erb
<%# app/views/cards/create.turbo_stream.erb %>
<%= turbo_stream.append :cards, @card %>
<%= turbo_stream.update :card_count, Card.count %>
<%= turbo_stream.remove :empty_state %>

<%# app/views/cards/update.turbo_stream.erb %>
<%= turbo_stream.replace @card %>

<%# app/views/cards/destroy.turbo_stream.erb %>
<%= turbo_stream.remove @card %>
<%= turbo_stream.update :card_count, Card.count %>
```

## Turbo Frame Patterns

```erb
<%# Lazy-loaded content %>
<%= turbo_frame_tag :comments, src: card_comments_path(@card), loading: :lazy do %>
  <p>Loading comments...</p>
<% end %>

<%# Inline editing %>
<%= turbo_frame_tag dom_id(@card, :edit) do %>
  <div class="card">
    <%= @card.title %>
    <%= link_to "Edit", edit_card_path(@card) %>
  </div>
<% end %>

<%# Modal content %>
<%= turbo_frame_tag :modal do %>
  <%# Content loaded here from modal links %>
<% end %>
```

## Jbuilder for JSON APIs

```ruby
# app/views/cards/show.json.jbuilder
json.extract! @card, :id, :title, :description, :created_at

json.creator do
  json.extract! @card.creator, :id, :name
end

json.tags @card.tags, :id, :name

json.url card_url(@card)

# app/views/cards/index.json.jbuilder
json.array! @cards do |card|
  json.extract! card, :id, :title
  json.url card_url(card)
end
```

## Dom ID Helpers

```erb
<%# Use dom_id for consistent element IDs %>
<div id="<%= dom_id(@card) %>">
  <%# renders as "card_123" %>
</div>

<div id="<%= dom_id(@card, :comments) %>">
  <%# renders as "comments_card_123" %>
</div>

<%# In Turbo Streams %>
<%= turbo_stream.replace dom_id(@card) do %>
  <%= render @card %>
<% end %>
```
