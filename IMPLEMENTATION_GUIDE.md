# Papercups Implementation Guide

This technical guide outlines the specific code changes needed to address the issues in our Papercups fork.

## 1. Disabling Subscription Features

### Remove Subscription UI Elements

**File**: `assets/src/components/billing/BillingOverview.tsx`
- Remove or modify the component to hide upgrade prompts

**File**: `assets/src/components/settings/AccountSettings.tsx`
- Remove subscription-related UI elements

### Set Default Subscription Plan to "Team"

**Database Update**:
```sql
-- Run in PostgreSQL
UPDATE accounts SET subscription_plan = 'team';
```

**Code Change**: `lib/chat_api/accounts/account.ex`
```elixir
# Change this line
field(:subscription_plan, :string, default: "starter")
# To
field(:subscription_plan, :string, default: "team")
```

## 2. Email Forwarding Solution

### Replace Domain References

**File**: `lib/chat_api/emails/email.ex`
- Replace all instances of "papercups.io" with your domain

**File**: `lib/chat_api_web/controllers/forwarding_address_controller.ex`
- Update email generation logic to use your domain

### Configure Email Processing

**File**: `config/prod.exs`
```elixir
config :chat_api, ChatApi.Mailers.Mailgun,
  domain: System.get_env("DOMAIN", "your-domain.com"),
  api_key: System.get_env("MAILGUN_API_KEY")
```

**Docker Configuration**: Update environment variables in `docker-compose.yml`
```yaml
environment:
  FROM_ADDRESS: "noreply@your-domain.com"
  DOMAIN: "your-domain.com"
```

## 3. Security Updates

### Update Dependencies

**Elixir Dependencies**: `mix.exs`
- Review and update dependency versions

**JavaScript Dependencies**: `assets/package.json`
- Update React and other frontend dependencies

### Secure Docker Configuration

**File**: `docker-compose.yml`
```yaml
services:
  papercups:
    # Add security-related settings
    security_opt:
      - no-new-privileges:true
    # Use non-root user
    user: "1000:1000"
    # Add health check
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

## 4. Implementation Steps

### Step 1: Disable Subscription System

```bash
# Access PostgreSQL in Docker
docker exec -it papercups_db_1 psql -U postgres

# In PostgreSQL
UPDATE accounts SET subscription_plan = 'team';
\q

# Verify in application
docker exec -it papercups_1 iex -S mix
> account = ChatApi.Accounts.get_account_by_company_name("your_company_name")
> IO.inspect(account.subscription_plan)
```

### Step 2: Update Email Domain

1. Edit the following files to replace "papercups.io" with your domain:
   - `.env` or `docker-compose.yml` environment variables
   - `lib/chat_api/emails/email.ex`
   - Any other files with hardcoded domain references

2. Configure your email service (AWS SES, Mailgun, etc.) to:
   - Send emails from your domain
   - Receive emails at your domain
   - Forward to the Papercups application

### Step 3: Update Dependencies

```bash
# Update Elixir dependencies
cd papercups
mix deps.update --all

# Update JavaScript dependencies
cd assets
npm update
```

### Step 4: Test Changes

1. Restart the application with your changes
2. Verify subscription prompts are gone
3. Test email forwarding with your domain
4. Check for any errors in the logs

## 5. Common Issues and Solutions

### Email Forwarding Not Working

1. Check DNS configuration for your domain
2. Verify email service (SES, Mailgun) is properly configured
3. Check Papercups logs for email processing errors

### Database Connection Issues

```bash
# Check PostgreSQL connection
docker exec -it papercups_1 psql -U postgres -h db -d postgres -c "SELECT 1;"
```

### Frontend Build Failures

```bash
# Rebuild frontend assets
docker exec -it papercups_1 bash -c "cd assets && npm install && npm run deploy"
```

## 6. Monitoring and Maintenance

### Log Monitoring

```bash
# View application logs
docker logs -f papercups_1
```

### Database Backups

```bash
# Create PostgreSQL backup
docker exec -it papercups_db_1 pg_dump -U postgres postgres > backup_$(date +%Y%m%d).sql
```

### Regular Updates

Schedule monthly dependency updates and security reviews to keep the application secure and stable.
