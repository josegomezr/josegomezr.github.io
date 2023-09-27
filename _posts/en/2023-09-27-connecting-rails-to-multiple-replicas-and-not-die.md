---
title: Connecting to multiple DB Replicas in Rails
layout: post
tags: [blog, ruby-on-rails, database, replicas, postgres]
categories: [blog, programming, rails]
lang: en
---

**IT TOOK ME THREE MONTHS TO FIGURE THIS SHIT OUT.**

How to connect Ruby on Rails to multiple databases and not die of frustration.

<small>Why I have to ruby-debugger deep into rails internals to find this out???</small>

<!--more-->

# A new rabbit hole

I started this journey on June 14th, wondering: _How could I connect to a
readonly replica? This definitely could help me out dealing with high traffic_


So I went directly to [the official guide for Ruby on Rails for "Multiple Databases with Active Record"][rails-ar-multiple-db] as _you should_. Right?

# "Multiple Databases with Active Record" Steps

Went through all the steps:

## 1. Prepare `config/database.yml`

Start by preparting your `config/database.yml` for the multiple connections

```yaml
production:
  primary:
    database: my_primary_database
    username: root
    password: <%= ENV['ROOT_PASSWORD'] %>
    adapter: postgresql
  replica:
    database: my_primary_database
    username: root_readonly
    password: <%= ENV['ROOT_READONLY_PASSWORD'] %>
    adapter: postgresql
    replica: true
    database_tasks: false # this is important, the replica doesn't need db:migrate and friends.
```

## 2. Configure your ApplicationRecord

```ruby
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true

  connects_to database: { writing: :primary, reading: :replica }
end
```

## 3. Run the Role Switching Generator

```sh
bin/rails g active_record:multi_db
```

and uncomment:

```ruby
Rails.application.configure do
  config.active_record.database_selector = { delay: 2.seconds }
  config.active_record.database_resolver = ActiveRecord::Middleware::DatabaseSelector::Resolver
  config.active_record.database_resolver_context = ActiveRecord::Middleware::DatabaseSelector::Resolver::Session
end
```

## 4. Run your app and... it should work...

_Spoiler alert: It didn't._


```
[2023-09-27T11:39:47.316+02:00] FATAL: 
ActiveRecord::ConnectionNotEstablished - No connection pool for 'ActiveRecord::Base' found for the 'reading' role.:
```

# So why did it break?

As the error says: `ConnectionNotEstablished`, now why didn't? we did everything
as the manual suggested... except no manual page at all tells you that rails will
establish a database connection just when `ActiveRecord` is loaded.

These sneaky detail is on the railtie for [ActiveRecord in Rails 6][rails-ar-railtie-6] & [Rails 7][rails-ar-railtie-7]. (They're pretty much the same code, I'll stick to Rails 7 from here on)

```ruby
# [...]
initializer "active_record.initialize_database" do
  ActiveSupport.on_load(:active_record) do
    if ActiveRecord::Base.legacy_connection_handling
      self.connection_handlers = { writing_role => ActiveRecord::Base.default_connection_handler }
    end
    self.configurations = Rails.application.config.database_configuration
    establish_connection # Here, a connection to ActiveRecord::Base.default_role is established
  end
end
# [...]
```

The default `ActiveRecord::Base.default_role` is always the `writing` role. [See it yourself in `ActiveRecord::Core`][rails-ar-core-7].


The first connection established by rails automatically **will always be the writing** connection (aka: connection to the primary database).

# How to fix it?

Just warm up the connection using the same technique as rails. In the very same
`config/initializers/multi_db.rb` created by rails, in the end:


```ruby
ActiveSupport.on_load(:active_record) do
  # When ActiveRecord loads, establish connections to all roles.
  #
  # This way the first request reaching the server will be ready to be served.
  %i[reading writing].each do |role|
    connected_to(role: role) { establish_connection }
  end
end
```

It's probably not the prettiest, but now all connections are initialized and you
will be able to use them on server boot.

<small>Took 3 month, 3 months... why there's no documentation on when Active Record does things? <br>Why do I have to put debuggers deep into railties? why?? `-.-`</small>

[rails-ar-multiple-db]: https://guides.rubyonrails.org/active_record_multiple_databases.html
[rails-ar-railtie-6]: https://github.com/rails/rails/blob/6-1-stable/activerecord/lib/active_record/railtie.rb#L216-L224
[rails-ar-railtie-7]: https://github.com/rails/rails/blob/7-1-stable/activerecord/lib/active_record/railtie.rb#L300C4-L306C8
[rails-ar-core-7]: https://github.com/rails/rails/blob/7-1-stable/activerecord/lib/active_record/core.rb#L222
