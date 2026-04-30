# Self-Hosted Matrix Homeserver with WhatsApp Bridge V1

A production-ready, self-hosted Matrix homeserver built with Synapse, PostgreSQL, and a fully functional WhatsApp bridge.


## Current Deployment Status
 
**Environment:** Local development on macOS  
**Client:** Element Desktop (http://localhost:8008)  
**Internet Access:** Not yet publicly accessible (there are CGNAT limitations for my router due to which I am not able to access port443/80)  
**Testing Status:** Fully functional locally with WhatsApp bridge working
 
### What's Working
 
- Synapse homeserver running on PostgreSQL
- WhatsApp bridge fully authenticated and syncing bidirectionally
- End-to-end encryption (Megolm) enabled by default for all chats
- Element Desktop client connected and messaging
- Docker Compose orchestration
- Caddy reverse proxy configuration

## Overview (some learnings for me)

Matrix is an open protocol for real-time communication. The primary problem that needs solving is the fact that chat platforms are essentially silos and users are forced to use their client, their rules and terms. On a simple level, matrix leverages a homeserver(brain/memory/coordinater), bridge (connection made from homeserver to various chat platforms), client app (a user interface made to ease retrival, send replies, etc). 
This implementation as of V1 provides:

- **Decentralized messaging** — No dependence on centralized platforms
- **Protocol bridging** — Seamless WhatsApp integration via mautrix-whatsapp
- **End-to-end encryption** — Megolm encryption enabled by default
- **Production database** — PostgreSQL for scalability and reliability 
Note: Sqlite is also available for testing purposes
- **Container orchestration** — Docker Compose for reproducible deployment

## Quick Start

### Prerequisites

- Docker & Docker Compose
- Domain name (e.g., tejastesting.it.com) - I bought one for testing!
- Ports 80, 443 accessible (or ngrok for CGNAT) - In case, want to use Element Web. However, Element Desktop (local) was used for V1

### Installation

```bash
# clone
git clone https://github.com/yourusername/matrix-homeserver.git
cd matrix-homeserver

# configure your domain
cp .env.example .env
nano .env  # Update SYNAPSE_SERVER_NAME to your domain and postgres details 

# generate Synapse configuration (first time only)
# This auto-generates homeserver.yaml with default settings - which require configuration 
# For detailed configuration, see [config/README.md](config/README.md)
docker compose run --rm synapse generate

docker compose up -d

docker compose exec synapse register_new_matrix_user \
  -c /data/homeserver.yaml http://localhost:8008

docker compose ps #ensure all the containers are running
```

### First Login

1. Download [Element Desktop](https://element.io/download)
2. Click "Sign In"
3. Click "Other homeserver"
4. Enter: `http://localhost:8008` (or your domain)
5. Sign in with admin credentials


### Components

**Caddy (Reverse Proxy)**
- Terminates HTTPS/TLS connections
- Handles automatic certificate management
- Serves Matrix `.well-known` files
- Only component requiring root privilege

**Synapse (Homeserver)**
- Implements Matrix protocol
- Stores users, messages, and rooms by connecting with postgresql
- Manages encryption keys
- Runs as unprivileged user

**PostgreSQL**
- Persistent data storage
- Supports horizontal scaling
- Encrypted at rest ready
- Backup/restore capable

**mautrix-whatsapp (Bridge - puppeting)**
- Puppeting mode (logs in as user)
- Bidirectional message sync
- Media support (images, videos)
- Automatic room creation


## Usage (inside the Element)

### WhatsApp Bridge

Connect your WhatsApp account:

```bash
# 1. Open Element and find the WhatsApp bot
# Search for: @whatsappbot:<domain_name>

# 2. Start a direct message with the bot

# 3. Send the login command - this outputs a qr to scan
!wa login qr 

# 4. A QR code appears
# Scan it with WhatsApp → Settings → Linked Devices

# 5. Your WhatsApp chats appear as Matrix rooms
```

### Sending Messages

- Messages sent in Matrix appear in WhatsApp
- WhatsApp messages appear in Matrix rooms
- All bidirectional, real-time
- Supports text, images, videos, documents

### Creating Rooms

Matrix rooms are created automatically:
- When you add a contact to WhatsApp
- When someone messages you
- When you join a group chat

## Management

### View Logs

```bash
# Synapse logs
docker compose logs synapse | tail -50

# WhatsApp bridge logs
docker compose logs mautrix-whatsapp | tail -50

# Caddy logs
docker compose logs caddy | tail -50

# All services
docker compose logs -f
```

### Database Management

```bash
# Create a backup
docker compose exec db pg_dump -U synapse_tejas synapse > backup.sql

# Restore from backup
docker compose exec db psql -U synapse_tejas synapse < backup.sql

# Access PostgreSQL directly
docker compose exec db psql -U synapse_tejas -d synapse
```

### User Management

```bash
# Create a new user
docker compose exec synapse register_new_matrix_user \
  -c /data/homeserver.yaml http://localhost:8008

# Deactivate a user
docker compose exec synapse deactivate_user @username:tejastesting.it.com

# Change user password
docker compose exec synapse hash_password -c /data/homeserver.yaml
```

## Configuration

### Environment Variables

Create a `.env` file (from `.env.example`):

```env
POSTGRES_USER=synapse_tejas
POSTGRES_PASSWORD=your_secure_password
POSTGRES_DB=synapse

SYNAPSE_SERVER_NAME=tejastesting.it.com
SYNAPSE_NO_TLS=true
```

### Caddyfile

Edit `Caddyfile` to customize:
- Domain name
- Reverse proxy rules
- TLS settings
- Compression

### homeserver.yaml

Key settings (auto-generated):
- `server_name` — Your domain
- `database` — PostgreSQL connection
- `app_service_config_files` — Bridge registration
- `encryption` — Encryption settings

## Network Considerations

### Port Forwarding

This setup requires ports 80 and 443 accessible from the internet:

```
Internet → Your ISP Router → Your Home Router → Your Machine
            Port 80/443     Port 80/443        :80/:443
```

### CGNAT (Carrier-Grade NAT) - problem i faced

If you have a private IP and can't open ports, tried the follwoing:

**Option 1: ngrok tunnel**
```bash
ngrok http 443
```

**Option 2: Cloudflare Tunnel**
```bash
# Setup a free Cloudflare Tunnel
cloudflare-tunnel run
```

**Option 3: Contact ISP**
- Request IPv6 address
- Request public IP (not CGNAT)

## Development & Extension

### Adding More Bridges

The architecture supports multiple bridges:

```yaml
# Add to app_service_config_files in homeserver.yaml
app_service_config_files:
  - /data/mautrix-whatsapp/registration.yaml
  - /data/mautrix-telegram/registration.yaml  # Add Telegram
  - /data/mautrix-discord/registration.yaml   # Add Discord
```

### Custom Configuration

All services are configurable:
- `homeserver.yaml` — Synapse settings
- `Caddyfile` — Reverse proxy rules
- `config.yaml` — Bridge configuration
- `docker-compose.yml` — Service definitions


## What I Learned

Through the notion doc, matrix docs, building this project I learned the following:

### Matrix architecture
- Synapse - matrix homeserver
- Bridge (puppeting) - mautrix-whatsapp
- Client App - Element

### Infrastructure
- Reverse proxy configuration (Caddy)
- Network architecture and port forwarding
- TLS/HTTPS encryption

### Security
- End-to-end encryption (Megolm)
- TLS/HTTPS implementation
- Secure password storage

### Some unexpected problem solving (kind of)
- CGNAT/NAT traversal
- DNS resolution and dynamic updates
- Container networking
- Configuration management
- Troubleshooting distributed systems

## Resources

- [Matrix.org](https://matrix.org) — Protocol specification
- [Synapse Docs](https://matrix-org.github.io/synapse/) — Server documentation
- [mautrix-whatsapp](https://github.com/mautrix/whatsapp) — Bridge repository
- [Caddy Docs](https://caddyserver.com/docs/) — Reverse proxy documentation
- [Element](https://element.io) — Reference client

**Last Updated:** May 2026  
