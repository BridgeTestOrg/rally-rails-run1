# Rally

Rails 8.1 app (SQLite, importmap, Stimulus, Turbo, minitest).

## Requirements

Ruby 3.4.12 (see `.ruby-version`) and Bundler.

## Run

```bash
bundle install
bin/rails db:prepare
bin/rails server        # http://localhost:3000
```

## Test

```bash
bin/rails test
bin/rubocop
bin/brakeman
```
