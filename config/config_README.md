# Configuration Guide

This directory contains configuration templates for the Matrix homeserver setup.

---

## Overview

The main configuration file is `homeserver.yaml`, which controls all Synapse settings including:
- Server identity and domain
- Database connection
- Encryption settings
- Bridge integrations (WhatsApp, etc.)
- Logging and media storage

---

## Initial Setup: Generating homeserver.yaml

### Step 1: Generate Default Configuration

When starting Synapse for the first time, you need to generate the default config:

```bash
docker compose run --rm synapse generate
```

This command:
- Creates `/data/homeserver.yaml` inside the container
- Generates security secrets (registration, macaroon, form secrets)
- Creates the signing key for federation
- **NOTE:** These secrets are automatically generated and MUST be kept safe

### Step 2: Copy Config to Your Local Machine

After generation, copy the generated config to your Mac for editing:

```bash
docker compose exec synapse cat /data/homeserver.yaml > ~/homeserver.yaml
```

This creates a local copy at `~/homeserver.yaml` that you can edit safely.

### Step 3: Edit Configuration Locally

Open the file with nano:

```bash
nano ~/homeserver.yaml
```

Or use your preferred editor (VS Code, etc.)

### Step 4: Key Changes to Make

See "Required Changes" section below for what to modify.

### Step 5: Copy Back to Container

Once you've made changes, copy the file back to the running container:

```bash
cat ~/homeserver.yaml | docker compose exec -T synapse bash -c 'cat > /data/homeserver.yaml'
```

The `-T` flag disables pseudo-TTY (required when piping).

### Step 6: Restart Synapse

For changes to take effect, restart Synapse:

```bash
docker compose restart synapse
sleep 10
docker compose logs synapse | tail -20
```

Check the logs to ensure there are no errors.

---

## Required Changes

### 1. Domain Name

Change `server_name` to your actual domain:

```yaml
# Before:
server_name: "tejastesting.it.com"

# After:
server_name: "your-actual-domain.com"
```


### 2. Database Password

Update PostgreSQL credentials to match your `docker-compose.yml`:

```yaml
database:
  name: psycopg2
  args:
    database: synapse
    user: synapse_tejas
    password: YOUR_SECURE_PASSWORD_HERE  # ← Change this
    host: db
    port: 5432
```

**Note:** Synapse needs to authenticate to PostgreSQL. Must match the `POSTGRES_PASSWORD` in docker-compose.yml.

### 3. Logging Path

Update the log config path to match your domain:

```yaml
# update your domain naime
log_config: "/data/your-domain.com.log.config"
```

---

## Optional Changes

### Encryption Settings

The default config enables encryption for all new rooms:

```yaml
encryption:
  enable_on_creation: true
  algorithms:
    - m.megolm.v1.aes-sha2
```

This is recommended for privacy. Don't change unless you have a specific reason.

### Application Services (Bridges)

If using bridges like WhatsApp, mautrix-whatsapp, etc., ensure they're registered:

```yaml
app_service_config_files:
  - /data/mautrix-whatsapp/registration.yaml
```

The bridge's registration file must exist at this path.

---