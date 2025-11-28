# CLAUDE.md - Codebase Guide for AI Assistants

## Project Overview

This is a **Rails 8.1 e-commerce store application** that manages products with inventory tracking and customer notifications. The core functionality includes:

- Product catalog with rich text descriptions and image uploads
- User authentication system with session management
- Email notification system for product stock updates
- Subscriber management for out-of-stock product alerts

## Technology Stack

### Core Technologies
- **Ruby**: 3.4.7
- **Rails**: 8.1.1
- **Database**: SQLite3 (development, test, production)
- **Frontend**: Hotwire (Turbo + Stimulus), Importmap
- **Asset Pipeline**: Propshaft
- **Web Server**: Puma with Thruster (production)

### Key Gems & Features
- **bcrypt**: Password hashing for secure authentication
- **solid_cache/solid_queue/solid_cable**: Database-backed Rails infrastructure
- **image_processing**: Active Storage image transformations
- **kamal**: Docker-based deployment
- **bootsnap**: Boot time optimization

### Development Tools
- **RuboCop**: Rails Omakase style guide
- **Brakeman**: Security vulnerability scanner
- **Bundler Audit**: Gem security auditing
- **Capybara/Selenium**: System testing

## Database Schema

### Core Tables

**users**
- `email_address` (unique, normalized to lowercase)
- `password_digest` (bcrypt hashed)
- Has many sessions

**sessions**
- `user_id` (foreign key)
- `ip_address`, `user_agent` (tracking)
- Cookie-based authentication using signed permanent cookies

**products**
- `name` (required)
- `inventory_count` (default: 0, validated >= 0)
- Has one attached `featured_image` (Active Storage)
- Has rich text `description` (Action Text)
- Has many subscribers

**subscribers**
- `email`
- `product_id` (foreign key)
- Receives notifications when products are back in stock

### Supporting Tables
- Active Storage tables (blobs, attachments, variant_records)
- Action Text rich_texts table
- Solid Cache/Queue/Cable tables (separate SQLite databases in production)

## Architecture Patterns

### Authentication System
Located in `app/controllers/concerns/authentication.rb`:
- Cookie-based session management (`cookies.signed.permanent[:session_id]`)
- `Current.session` for request-scoped session access
- `require_authentication` before_action (can be skipped with `allow_unauthenticated_access`)
- Returns users to originally requested URL after authentication

### Model Concerns
**Product::Notifications** (`app/models/product/notifications.rb`):
- Automatically sends emails when `inventory_count` goes from 0 to positive
- Uses `after_update_commit` callback
- Delivers emails asynchronously via `deliver_later`

### Controllers
Follow RESTful conventions:
- Strong parameters using `params.expect()`
- Before actions for setting resources
- Allow unauthenticated access for public actions (index, show)
- Return `:unprocessable_entity` status for validation errors

### Mailers
- `PasswordsMailer`: Password reset emails
- `ProductMailer`: Stock notification emails
- Use `with()` syntax for passing parameters

## Directory Structure

```
app/
├── assets/
│   ├── images/          # Static images
│   └── stylesheets/     # CSS files
├── channels/            # Action Cable connections
├── controllers/
│   ├── concerns/        # Shared controller logic (Authentication)
│   └── *.rb            # RESTful controllers
├── helpers/             # View helpers
├── javascript/          # Stimulus controllers
├── jobs/                # Active Job classes
├── mailers/             # Action Mailer classes
├── models/
│   ├── concerns/        # Shared model logic
│   ├── product/         # Product-specific concerns (Notifications)
│   ├── current.rb       # Request-scoped current session
│   └── *.rb            # Active Record models
└── views/
    ├── layouts/         # Application layout templates
    ├── passwords/       # Password reset views
    ├── products/        # Product CRUD views
    ├── sessions/        # Login views
    └── *_mailer/        # Email templates (HTML + text)

bin/
├── setup              # Initial setup script
├── dev                # Start development server
├── ci                 # Run CI checks locally
├── brakeman           # Security scanner
├── bundler-audit      # Gem vulnerability check
├── rubocop            # Linter
└── kamal              # Deployment tool

config/
├── routes.rb          # Application routes
├── database.yml       # Database configuration
└── environments/      # Environment-specific configs

db/
├── migrate/           # Database migrations
├── schema.rb          # Current database schema (auto-generated)
├── seeds.rb           # Seed data
├── cache_schema.rb    # Solid Cache schema
├── queue_schema.rb    # Solid Queue schema
└── cable_schema.rb    # Solid Cable schema

test/
├── controllers/       # Controller tests
├── fixtures/          # Test data
├── helpers/           # Helper tests
├── integration/       # Integration tests
├── mailers/           # Mailer tests
├── models/            # Model tests
└── system/            # Capybara system tests
```

## Routes

```ruby
root "products#index"                          # Homepage: product listing

resource :session                              # Login/logout
resources :passwords, param: :token            # Password reset

resources :products do                         # Product CRUD
  resources :subscribers, only: [:create]      # Subscribe to product
end

resource :unsubscribe, only: [:show]           # Unsubscribe from notifications

get "up" => "rails/health#show"                # Health check endpoint
```

## Development Workflow

### Initial Setup
```bash
bin/setup                    # Install dependencies, prepare database, start server
bin/setup --reset            # Reset database during setup
bin/setup --skip-server      # Setup without starting server
```

### Running the Application
```bash
bin/dev                      # Start development server
bin/rails server             # Alternative: start server only
bin/rails console            # Interactive Rails console
```

### Database Operations
```bash
bin/rails db:migrate         # Run pending migrations
bin/rails db:rollback        # Rollback last migration
bin/rails db:reset           # Drop, create, migrate, seed
bin/rails db:seed            # Load seed data
bin/rails db:prepare         # Create database if needed, run migrations
```

### Testing
```bash
bin/rails test               # Run all tests
bin/rails test:system        # Run system tests only
bin/rails test test/models/product_test.rb     # Run specific test file
bin/ci                       # Run all CI checks locally
```

### Code Quality
```bash
bin/rubocop                  # Lint Ruby code
bin/rubocop -a               # Auto-fix violations
bin/brakeman                 # Security vulnerability scan
bin/bundler-audit            # Check for vulnerable gems
bin/importmap audit          # Check JavaScript dependencies
```

### Deployment
```bash
bin/kamal deploy             # Deploy with Kamal
bin/kamal setup              # Initial server setup
bin/kamal rollback           # Rollback deployment
```

## Testing Strategy

### Test Structure
- **Unit Tests**: `test/models/` - Test model logic and validations
- **Controller Tests**: `test/controllers/` - Test HTTP responses and redirects
- **System Tests**: `test/system/` - Browser-based integration tests with Capybara
- **Mailer Tests**: `test/mailers/` - Test email generation
- **Fixtures**: `test/fixtures/` - Test data in YAML format

### Testing Patterns
```ruby
# Model test example (test/models/product_test.rb)
test "sends email notifications when back in stock" do
  product = products(:tshirt)
  product.update(inventory_count: 0)

  assert_emails 2 do
    product.update(inventory_count: 99)
  end
end
```

### Test Helpers
- `include ActionMailer::TestHelper` for email assertions
- Fixtures referenced by symbol: `products(:tshirt)`
- `assert_emails`, `assert_no_emails` for mailer testing

## Code Conventions

### Ruby Style Guide
- Follows **Rails Omakase** style via `rubocop-rails-omakase`
- Configuration in `.rubocop.yml`
- Run `bin/rubocop` to check compliance

### Naming Conventions
- Models: Singular, PascalCase (`Product`, `User`)
- Controllers: Plural, PascalCase with `Controller` suffix (`ProductsController`)
- Tables: Plural, snake_case (`products`, `users`)
- Foreign keys: Singular with `_id` suffix (`user_id`, `product_id`)
- Concerns: Descriptive modules (`Authentication`, `Product::Notifications`)

### File Organization
- Model concerns in `app/models/concerns/` or `app/models/model_name/`
- Controller concerns in `app/controllers/concerns/`
- Namespace-specific concerns in subdirectories (e.g., `app/models/product/notifications.rb`)

### Strong Parameters
Use Rails 8's `params.expect()` syntax:
```ruby
def product_params
  params.expect(product: [:name, :description, :featured_image, :inventory_count])
end
```

## Key Features & Implementation

### 1. Authentication
**Location**: `app/controllers/concerns/authentication.rb`

**Pattern**:
```ruby
class MyController < ApplicationController
  allow_unauthenticated_access only: [:index, :show]

  # All other actions require authentication
end
```

**Session Management**:
- Sessions stored in database (`sessions` table)
- Identified by signed permanent cookie
- Accessed via `Current.session`
- Tracks IP address and user agent

### 2. Product Notifications
**Location**: `app/models/product/notifications.rb`

**How it works**:
1. User subscribes to out-of-stock product
2. When `inventory_count` changes from 0 to positive
3. `after_update_commit` callback triggers
4. Email sent to all subscribers via `ProductMailer.in_stock.deliver_later`
5. Background job processes emails asynchronously

### 3. Rich Content
- **Images**: Active Storage with `has_one_attached :featured_image`
- **Text**: Action Text with `has_rich_text :description`
- Image processing available for variants/thumbnails

### 4. Email System
**Templates**: HTML + text versions
- `app/views/passwords_mailer/reset.{html,text}.erb`
- `app/views/product_mailer/in_stock.{html,text}.erb`

**Delivery**: Background jobs via Solid Queue

## CI/CD Pipeline

### GitHub Actions (`.github/workflows/ci.yml`)

**Jobs**:
1. **scan_ruby**: Brakeman security scan, Bundler Audit
2. **scan_js**: Importmap audit for JavaScript vulnerabilities
3. **lint**: RuboCop style checking with caching
4. **test**: Unit and integration tests
5. **system-test**: Browser-based Capybara tests, uploads screenshots on failure

**Triggers**:
- Pull requests
- Pushes to `master` branch

### Environment Variables
- `RAILS_ENV`: Set to `test` for CI
- `RAILS_MASTER_KEY`: Optional, commented out in CI config
- `REDIS_URL`: Optional, currently using SQLite-backed alternatives

## Deployment

### Docker
**Dockerfile**: Multi-stage build optimized for production
- Base image: Ruby 3.4.7 slim
- Uses jemalloc for memory optimization
- Non-root user (rails:1000)
- Precompiles assets and bootsnap cache
- Exposes port 80
- Default command: `bin/thrust bin/rails server`

### Kamal
- Configuration in `.kamal/` directory
- Hooks available in `.kamal/hooks/` (samples provided)
- Secrets in `.kamal/secrets`
- Deploy with `bin/kamal deploy`

### Production Databases
SQLite databases stored in `storage/` directory:
- `production.sqlite3` (main)
- `production_cache.sqlite3` (Solid Cache)
- `production_queue.sqlite3` (Solid Queue)
- `production_cable.sqlite3` (Solid Cable)

**Important**: Mount `storage/` as persistent volume in production

## Common AI Assistant Tasks

### Adding a New Model
1. Generate migration: `bin/rails generate model ModelName field:type`
2. Review and edit migration in `db/migrate/`
3. Run migration: `bin/rails db:migrate`
4. Add validations and associations to model
5. Create fixtures in `test/fixtures/`
6. Write model tests in `test/models/`
7. Run tests: `bin/rails test test/models/model_name_test.rb`

### Adding a New Controller
1. Generate: `bin/rails generate controller ControllerName action1 action2`
2. Add authentication if needed: `allow_unauthenticated_access only: [...]`
3. Implement actions with strong parameters
4. Add routes to `config/routes.rb`
5. Create views in `app/views/controller_name/`
6. Write controller tests in `test/controllers/`
7. Run tests: `bin/rails test test/controllers/controller_name_test.rb`

### Adding a Model Concern
1. Create file in `app/models/concerns/` or `app/models/model_name/`
2. Define module with `extend ActiveSupport::Concern`
3. Use `included do ... end` for class-level code
4. Include in model: `include ConcernName`
5. Write tests covering the concern's functionality

### Modifying Database Schema
1. Generate migration: `bin/rails generate migration DescriptiveName`
2. Edit migration file with `change`, `up`/`down`, or individual methods
3. Run: `bin/rails db:migrate`
4. Update `db/schema.rb` will auto-update
5. Update model validations/associations if needed
6. Update tests and fixtures
7. Run full test suite: `bin/rails test`

### Security Considerations
- **Never commit secrets**: Use Rails credentials or environment variables
- **Always use strong parameters**: Whitelist permitted attributes
- **Validate user input**: Add validations to models
- **Use authenticated actions**: Default to requiring authentication
- **Scan regularly**: Run `bin/brakeman` and `bin/bundler-audit`
- **CSRF protection**: Enabled by default in Rails
- **SQL injection**: Use parameterized queries (Active Record does this)

## Git Workflow

### Branches
- `master`: Main production branch
- Feature branches: Use descriptive names
- AI assistant branches: Prefixed with `claude/`

### Commits
- Write clear, descriptive commit messages
- Use present tense: "Add feature" not "Added feature"
- Reference issue numbers if applicable
- Keep commits focused and atomic

### Before Committing
1. Run tests: `bin/rails test`
2. Run linter: `bin/rubocop`
3. Run security scans: `bin/brakeman` and `bin/bundler-audit`
4. Review changes: `git diff`
5. Ensure no sensitive data is included

## Troubleshooting

### Common Issues

**Database locked errors**:
- SQLite doesn't handle high concurrency well
- Consider PostgreSQL for production if experiencing issues
- Check for long-running transactions

**Asset not found in production**:
- Run `bin/rails assets:precompile`
- Check `public/assets/` directory
- Verify `RAILS_ENV=production`

**Authentication not working**:
- Check `Current.session` is being set
- Verify cookie settings (httponly, same_site)
- Ensure session record exists in database

**Emails not sending**:
- Check Solid Queue is running: `bin/jobs`
- Verify mailer configuration in `config/environments/`
- Check logs for delivery errors

**Tests failing**:
- Run `bin/rails db:test:prepare` to ensure test database is current
- Check fixtures are valid
- Verify test environment configuration

## Performance Considerations

### Database
- SQLite is suitable for low-to-medium traffic
- Solid Cache/Queue/Cable use separate databases to avoid contention
- Add indexes for frequently queried columns
- Use `includes()` to avoid N+1 queries

### Caching
- Solid Cache provides database-backed caching
- Use fragment caching in views for expensive renders
- HTTP caching headers set by Thruster in production

### Background Jobs
- Use `deliver_later` for emails
- Process long-running tasks in background jobs
- Monitor Solid Queue for job failures

### Assets
- Importmap for JavaScript (no build step needed)
- Propshaft for asset pipeline (simpler than Sprockets)
- Thruster provides compression and caching in production

## Additional Resources

### Rails Guides
- [Rails Guides](https://guides.rubyonrails.org/): Official documentation
- [Rails API](https://api.rubyonrails.org/): API reference
- [Hotwire](https://hotwired.dev/): Turbo and Stimulus documentation

### Project-Specific
- Check `bin/` directory for available commands
- Review tests for usage examples
- Examine fixtures for data structure examples
- Read migrations for schema evolution history

## AI Assistant Best Practices

1. **Read before modifying**: Always read existing files before making changes
2. **Follow existing patterns**: Match the style and structure of current code
3. **Test your changes**: Run relevant tests after modifications
4. **Update tests**: Add/modify tests when changing functionality
5. **Use conventions**: Follow Rails and Ruby conventions
6. **Check security**: Run security scanners before committing
7. **Validate thoroughly**: Ensure models have appropriate validations
8. **Document complex logic**: Add comments for non-obvious code
9. **Keep it simple**: Don't over-engineer solutions
10. **Run CI locally**: Use `bin/ci` to catch issues before pushing

---

**Last Updated**: 2025-11-28
**Rails Version**: 8.1.1
**Ruby Version**: 3.4.7
