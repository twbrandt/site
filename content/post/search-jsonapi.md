---
title: "Full-text search with JSONAPI Resources"
date: 2017-05-31T21:10:00-04:00
categories: ["ruby on rails", "software engineering"]
draft: false
---

PostgreSQL has a decent [full-text search facility](https://www.postgresql.org/docs/9.5/static/textsearch.html) built into it that is more than adequate for many use cases. This article shows how to use it along with [JSONAPI Resources](http://jsonapi-resources.com/) in a Rails app. I urge you to read the PostgreSQL documentation, especially chapters [12.1](https://www.postgresql.org/docs/9.5/static/textsearch-intro.html), [12.2](https://www.postgresql.org/docs/9.5/static/textsearch-tables.html), and [12.3](https://www.postgresql.org/docs/9.5/static/textsearch-controls.html), before continuing with this article so you have good understanding of the fundamentals of full-text search.

Let's say we want to search the `users` table on `first_name`, `last_name`, or both. We can use a [filter](http://jsonapi-resources.com/v0.9/guide/resources.html#Filters) in `UserResource` and apply a callable to it that actually performs the search.

# 1. Create failing test, empty method

First we need to set up a test. We want to test that it returns the records we expect when we search with a first name, a last name, and both first and last name.
{{< highlight ruby>}}
# spec/resources/user_resource_spec.rb
require 'rails_helper'

describe UserResource do
  describe "search filter" do
    let!(:u1)  { User.create(first_name: "Joe", last_name: "Brown") }
    let!(:u2)  { User.create(first_name: "Jane", last_name: "Brown") }
    let!(:u3)  { User.create(first_name: "Joe", last_name: "Jones") }

    let(:options) { {} }

    it "searches on first name" do
      records = described_class.find({ search: "joe" }, options).map(&:_model)
      expect(records).to contain_exactly(u1, u3)
    end

    it "searches on last name" do
      records = described_class.find({ search: "brown" }, options).map(&:_model)
      expect(records).to contain_exactly(u1, u2)
    end

    it "ANDs search terms" do
      records = described_class.find({ search: "joe jones"}, options).map(&:_model)
      expect(records).to contain_exactly(u3)
    end
  end
end
{{< /highlight >}}

We also need to add the filter to the resource. We can call the filter `:search`:
{{< highlight ruby>}}
class UserResource < JSONAPI::Resource
  attributes  :first_name, :last_name

  filter :search, apply: ->(records, value, _options) {
    [] # return an empty array until database is set up.
  }

end
{{< /highlight >}}

When we run the spec, it should fail (as opposed to blow up).

# 2. Set up database for full-text search

There are a number of ways to use full-text search in PostgreSQL, but the most efficient for our use case is to add a separate column to the `users` table to store the [tsvector](https://www.postgresql.org/docs/9.5/static/datatype-textsearch.html) representation of `first_name` and `last_name`, and add a [GIN index](https://www.postgresql.org/docs/9.5/static/textsearch-indexes.html) to speed up the search. In order to make sure the index reflects the current state of `first_name` and `last_name`, we have to add a trigger to the database which will automatically update the index whenever a new row is added to `users` or the value of `first_name` or `last_name` are updated on an existing row.

Note that `schema.rb` does not know about triggers, so we have to build a `structure.sql` schema file instead. To do this, modify `config.application.rb` with the following:

```
config.active_record.schema_format = :sql
```

Delete schema.rb:
```
rm db/schema.rb
```

We can now generate a migration and edit it to add the column, index, and trigger:
```
rails g migration add_tsvector_column_to_users
```

{{< highlight ruby>}}
class AddTsvectorColumnToUsers < ActiveRecord::Migration[5.1]
  def up
    execute <<-SQL
      ALTER TABLE users ADD COLUMN ts_names tsvector;
      UPDATE users SET ts_names =
        to_tsvector('english', coalesce(first_name,'') || ' ' || coalesce(last_name, ''));
      CREATE INDEX ts_names_idx ON users USING GIN (ts_names);
      CREATE TRIGGER ts_names_update BEFORE INSERT OR UPDATE
        ON users FOR EACH ROW EXECUTE PROCEDURE
        tsvector_update_trigger(ts_names, 'pg_catalog.english', first_name, last_name);
    SQL
  end

  def down
    execute <<-SQL
      DROP TRIGGER [ IF EXISTS ] ts_names_update ON users;
      DROP INDEX [ IF EXISTS ] ts_names_idx;
      ALTER TABLE users DROP COLUMN [ IF EXISTS ] ts_names;
    SQL
  end
end
{{< /highlight >}}

# 3. Add search to the filter

Modifying the filter to use search is straightforward:

{{< highlight ruby>}}
class UserResource < JSONAPI::Resource
  attributes  :first_name, :last_name

  filter :search, apply: ->(records, value, _options) {
    records.where("ts_names @@ plainto_tsquery(:query)", query: value)
  }

end
{{< /highlight >}}

When we run our spec again the tests should succeed.

The rails app can now correctly handle requests like this:
```
http://example.com/users?filter[search]=joe%40jones
```

You get pagination without any additional work:
```
http://example.com/users?filter[search]=jones&page%5Bnumber%5D=2&page%5Bsize%5D=10
```
