# Evolution API Production Stack

A production-ready deployment of **Evolution API**, a powerful WhatsApp API wrapper for building bots and automated messaging systems.

## What is Evolution API?

Evolution API is an open-source WhatsApp Business API that allows you to:
- Send and receive WhatsApp messages programmatically
- Create chatbots and automated workflows
- Manage multiple WhatsApp instances (accounts)
- Integrate WhatsApp with CRMs like Chatwoot
- Handle webhooks for real-time message events

**Use Cases:**
- Customer support automation
- Marketing campaigns
- Order notifications
- Appointment reminders
- Two-way conversations with customers

## Architecture

This stack runs a single Evolution API container that:
- Manages WhatsApp Web sessions (called "instances")
- Stores session data persistently in volumes
- Connects to PostgreSQL for metadata storage
- Uses Redis for caching and rate limiting
- Exposes REST API and WebSocket endpoints

**Important:** Each Evolution API instance represents one WhatsApp account. You can create multiple instances to manage multiple phone numbers.

## Prerequisites

### 1. Deploy Required Infrastructure

Evolution API requires PostgreSQL (optional but recommended) and Redis (optional). Deploy the `infra-db` stack:

```bash
cd stacks/infra-db
docker compose up -d
```

**Note:** Evolution API can work without databases (file-based storage), but PostgreSQL is recommended for production to persist instance metadata across restarts.

### 2. Understanding WhatsApp Rate Limits

WhatsApp enforces strict rate limits:
- **New accounts:** ~250 messages/24h (gradually increases)
- **Established accounts:** ~1,000-10,000 messages/24h
- **Ban risk:** Sending spam or violating ToS results in permanent ban

**Best Practices:**
- Don't send unsolicited messages
- Respect opt-out requests immediately
- Use verified Business accounts for higher limits
- Implement exponential backoff for retries

### 3. Generate API Key

Evolution API uses a master API key for authentication:

```bash
openssl rand -hex 32
```

Copy this to your `.env` as `AUTHENTICATION_API_KEY`.

## Setup Instructions

### Step 1: Configure Environment Variables

Copy the example file and configure your secrets:

```bash
cp .env.example .env
```

**Required Variables:**

| Variable | Example | Description |
|----------|---------|-------------|
| `EVOLUTION_DOMAIN` | `wa.yourdomain.com` | Domain where Evolution API will be accessible |
| `AUTHENTICATION_API_KEY` | `(64-char hex string)` | Master key for API authentication (generate with `openssl rand -hex 32`) |
| `DATABASE_CONNECTION_URI` | `postgresql://user:pass@postgres:5432/evolution` | PostgreSQL connection string (optional but recommended) |
| `HOST_DATA_PATH` | `/home/user/evolution_data` | Host folder for WhatsApp sessions and media |

**Optional but Recommended:**

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABASE_SAVE_DATA_INSTANCE` | `false` | Set to `true` to persist instances in PostgreSQL |
| `CACHE_REDIS_ENABLED` | `false` | Set to `true` to enable Redis caching |
| `CACHE_REDIS_URI` | - | Redis connection: `redis://:password@redis:6379` |
| `WEBHOOK_GLOBAL_ENABLED` | `false` | Enable global webhook for all instances |

### Step 2: Start the Stack

```bash
docker compose up -d
```

### Step 3: Verify API is Running

Check API health:

```bash
curl https://wa.yourdomain.com
```

Expected response:
```json
{
  "status": "ok",
  "version": "2.3.7"
}
```

## Creating Your First WhatsApp Instance

### Step 1: Create Instance via API

```bash
curl -X POST https://wa.yourdomain.com/instance/create \
  -H "apikey: YOUR_AUTHENTICATION_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "instanceName": "my-whatsapp",
    "qrcode": true
  }'
```

**Response:**
```json
{
  "instance": {
    "instanceName": "my-whatsapp",
    "status": "created"
  },
  "qrcode": {
    "base64": "data:image/png;base64,iVBORw0KGgo..."
  }
}
```

### Step 2: Connect WhatsApp

1. Copy the QR code base64 string
2. Decode and display it (or use Evolution API's built-in QR viewer)
3. Open WhatsApp on your phone
4. Go to **Settings** > **Linked Devices** > **Link a Device**
5. Scan the QR code

**Alternative (Built-in QR Viewer):**

Navigate to: `https://wa.yourdomain.com/instance/qr/my-whatsapp?apikey=YOUR_API_KEY`

Scan directly from browser.

### Step 3: Verify Connection

```bash
curl https://wa.yourdomain.com/instance/connectionState/my-whatsapp \
  -H "apikey: YOUR_AUTHENTICATION_API_KEY"
```

**Response when connected:**
```json
{
  "instance": {
    "instanceName": "my-whatsapp",
    "state": "open"
  }
}
```

## Sending Messages

### Send Text Message

```bash
curl -X POST https://wa.yourdomain.com/message/sendText/my-whatsapp \
  -H "apikey: YOUR_AUTHENTICATION_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "5511999999999",
    "text": "Hello from Evolution API!"
  }'
```

**Important:** Phone numbers must include country code without `+` symbol (e.g., `5511999999999` for Brazil).

### Send Media (Image/Video/Document)

```bash
curl -X POST https://wa.yourdomain.com/message/sendMedia/my-whatsapp \
  -H "apikey: YOUR_AUTHENTICATION_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "5511999999999",
    "mediatype": "image",
    "media": "https://example.com/image.jpg",
    "caption": "Check this out!"
  }'
```

### Send Media with Base64

```bash
curl -X POST https://wa.yourdomain.com/message/sendMedia/my-whatsapp \
  -H "apikey: YOUR_AUTHENTICATION_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "5511999999999",
    "mediatype": "image",
    "media": "data:image/png;base64,iVBORw0KGgo...",
    "fileName": "photo.png"
  }'
```

## Webhooks (Receiving Messages)

### Configure Webhook per Instance

```bash
curl -X POST https://wa.yourdomain.com/webhook/set/my-whatsapp \
  -H "apikey: YOUR_AUTHENTICATION_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-backend.com/webhook",
    "webhook_by_events": false,
    "events": [
      "MESSAGES_UPSERT",
      "MESSAGES_UPDATE",
      "CONNECTION_UPDATE"
    ]
  }'
```

### Webhook Event Types

| Event | Description |
|-------|-------------|
| `MESSAGES_UPSERT` | New message received |
| `MESSAGES_UPDATE` | Message status changed (sent, delivered, read) |
| `CONNECTION_UPDATE` | WhatsApp connection state changed |
| `CALL` | Incoming call detected |
| `GROUP_PARTICIPANTS_UPDATE` | Group member added/removed |

### Webhook Payload Example

```json
{
  "event": "messages.upsert",
  "instance": "my-whatsapp",
  "data": {
    "key": {
      "remoteJid": "5511999999999@s.whatsapp.net",
      "fromMe": false,
      "id": "3EB0ABC123DEF456"
    },
    "message": {
      "conversation": "Hello, I need support"
    },
    "messageTimestamp": 1703001234
  }
}
```

## Integration with Chatwoot

Connect Evolution API to Chatwoot for unified inbox management.

### Step 1: Enable Chatwoot in Evolution API

In `.env`, set:
```bash
CHATWOOT_ENABLED=true
```

### Step 2: Create API Inbox in Chatwoot

1. Go to Chatwoot > **Settings** > **Inboxes** > **Add Inbox**
2. Select **API**
3. Copy the **Inbox Identifier** and **API Access Token**

### Step 3: Create Evolution Instance with Chatwoot Integration

```bash
curl -X POST https://wa.yourdomain.com/instance/create \
  -H "apikey: YOUR_AUTHENTICATION_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "instanceName": "my-whatsapp-chatwoot",
    "integration": "CHATWOOT",
    "chatwootAccountId": "1",
    "chatwootToken": "CHATWOOT_API_ACCESS_TOKEN",
    "chatwootUrl": "https://chat.yourdomain.com",
    "chatwootInboxId": "CHATWOOT_INBOX_ID",
    "chatwootSignMsg": true
  }'
```

Now all WhatsApp messages will appear in your Chatwoot inbox!

## Managing Multiple Instances

### List All Instances

```bash
curl https://wa.yourdomain.com/instance/fetchInstances \
  -H "apikey: YOUR_AUTHENTICATION_API_KEY"
```

### Delete Instance

```bash
curl -X DELETE https://wa.yourdomain.com/instance/delete/my-whatsapp \
  -H "apikey: YOUR_AUTHENTICATION_API_KEY"
```

### Logout Instance (Unlink WhatsApp)

```bash
curl -X DELETE https://wa.yourdomain.com/instance/logout/my-whatsapp \
  -H "apikey: YOUR_AUTHENTICATION_API_KEY"
```

## Common Issues

### 1. QR Code Expired / Not Connecting

**Problem:** QR code expires after 60 seconds or connection fails.

**Solution:**
```bash
# Restart instance to generate new QR
curl -X PUT https://wa.yourdomain.com/instance/restart/my-whatsapp \
  -H "apikey: YOUR_AUTHENTICATION_API_KEY"

# Get new QR code
curl https://wa.yourdomain.com/instance/qr/my-whatsapp \
  -H "apikey: YOUR_AUTHENTICATION_API_KEY"
```

**Common Causes:**
- Network issues between container and WhatsApp servers
- Firewall blocking WebSocket connections
- DNS resolution problems (configured DNS: 8.8.8.8, 8.8.4.4)

### 2. "Instance not found" Error

**Problem:** API returns 404 when accessing instance.

**Solution:**
```bash
# Verify instance exists
curl https://wa.yourdomain.com/instance/fetchInstances \
  -H "apikey: YOUR_AUTHENTICATION_API_KEY"

# If not found, create it again
```

**Cause:** Instance data not persisted. Enable database storage:
```bash
DATABASE_SAVE_DATA_INSTANCE=true
```

### 3. Messages Not Sending / Rate Limited

**Problem:** Messages fail with 429 error or don't deliver.

**Solution:**

**Check WhatsApp connection:**
```bash
curl https://wa.yourdomain.com/instance/connectionState/my-whatsapp \
  -H "apikey: YOUR_AUTHENTICATION_API_KEY"
```

**Implement rate limiting in your code:**
```javascript
// Send max 20 messages per minute
const delay = ms => new Promise(resolve => setTimeout(resolve, ms));

for (const message of messages) {
  await sendMessage(message);
  await delay(3000); // Wait 3 seconds between messages
}
```

**Common Causes:**
- Exceeding WhatsApp rate limits (~250/day for new accounts)
- Account flagged for spam
- Network issues

### 4. Webhooks Not Receiving Events

**Problem:** Webhook endpoint not receiving POST requests.

**Solution:**

**Test webhook manually:**
```bash
curl -X POST https://your-backend.com/webhook \
  -H "Content-Type: application/json" \
  -d '{"test": "data"}'
```

**Verify webhook configuration:**
```bash
curl https://wa.yourdomain.com/webhook/find/my-whatsapp \
  -H "apikey: YOUR_AUTHENTICATION_API_KEY"
```

**Common Causes:**
- Webhook URL not publicly accessible (use ngrok for testing)
- Firewall blocking outbound requests from Evolution API
- Webhook endpoint returning errors (must return 200 OK)

### 5. "Session Logged Out" Unexpectedly

**Problem:** WhatsApp session disconnects randomly.

**Solution:**

**Enable session persistence:**
```bash
DATABASE_SAVE_DATA_INSTANCE=true
```

**Check logs for errors:**
```bash
docker compose logs evolution_api --tail 100 | grep -i error
```

**Common Causes:**
- WhatsApp detected suspicious activity
- Account used on multiple devices simultaneously
- Network instability
- Container restarted without persistent storage

### 6. High Memory Usage

**Problem:** Evolution API consuming excessive RAM.

**Solution:**

Limit container memory:
```yaml
services:
  evolution_api:
    mem_limit: 1g
    memswap_limit: 1g
```

Reduce concurrent instances:
```bash
# Each instance uses ~100-200MB RAM
# For 1GB limit, run max 4-5 instances
```

### 7. Media Files Not Downloading

**Problem:** Images/videos not accessible after sending.

**Solution:**

Verify volume permissions:
```bash
sudo chown -R 1000:1000 ${HOST_DATA_PATH}/instances
```

Check disk space:
```bash
df -h ${HOST_DATA_PATH}
```

## Maintenance

### Updating Evolution API

```bash
# Pull latest image
docker compose pull

# Restart with new version
docker compose up -d
```

**Important:** Always check [release notes](https://github.com/EvolutionAPI/evolution-api/releases) for breaking changes.

### Backing Up Instances

**Session Data Backup:**
```bash
tar -czf evolution_instances_$(date +%Y%m%d).tar.gz ${HOST_DATA_PATH}/instances
```

**Database Backup (if using PostgreSQL):**
```bash
docker exec infra-db-postgres-1 pg_dump -U postgres evolution > evolution_db_$(date +%Y%m%d).sql
```

### Viewing Logs

```bash
# Real-time logs
docker compose logs -f evolution_api

# Last 100 lines
docker compose logs --tail 100 evolution_api

# Filter by keyword
docker compose logs evolution_api | grep -i "error"
```

## Performance Optimization

### For Single Instance (1 WhatsApp account)

Default configuration is sufficient.

### For Multiple Instances (5-10 accounts)

Enable Redis caching:
```bash
CACHE_REDIS_ENABLED=true
CACHE_REDIS_URI=redis://:password@redis:6379
```

Increase container resources:
```yaml
services:
  evolution_api:
    mem_limit: 2g
    cpus: '1.0'
```

### For High Volume (10+ accounts, 1000+ msgs/day)

Consider:
- Dedicated PostgreSQL instance
- Redis for session caching
- Load balancer for multiple Evolution API containers
- S3 for media storage instead of local volumes

## Security Best Practices

1. **Protect your API key** - Never commit to Git or expose publicly
2. **Use HTTPS only** - Enforced by Traefik integration
3. **Whitelist IPs** - Restrict API access to known IPs if possible
4. **Rotate API keys regularly** - Update `AUTHENTICATION_API_KEY` every 90 days
5. **Monitor webhook endpoints** - Validate incoming webhook signatures
6. **Don't share QR codes** - QR codes provide full account access
7. **Enable database encryption** - For sensitive metadata

## Compliance & WhatsApp Terms

**Important:** Using WhatsApp Business API unofficially may violate WhatsApp Terms of Service. This implementation is NOT officially supported by Meta/WhatsApp.

**Risks:**
- Account permanent ban
- Phone number blocked from WhatsApp
- Legal action from Meta

**Use at your own risk.** For official WhatsApp Business API, use [Meta's official platform](https://business.whatsapp.com).

## Resources

- [Evolution API Documentation](https://doc.evolution-api.com)
- [GitHub Repository](https://github.com/EvolutionAPI/evolution-api)
- [API Swagger](https://wa.yourdomain.com/docs) (when deployed)
- [Discord Community](https://evolution-api.com/discord)
