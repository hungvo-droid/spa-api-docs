# Telegram Bot API Documentation

This document describes the API endpoints for the Telegram bot integration service that handles webhook events, message processing, and bot management through a Pub/Sub architecture.

## General Flow

1. **Bot Registration:** Register a Telegram bot with its token and webhook configuration
2. **Webhook Setup:** Configure webhook URL for each bot with a secret token
3. **Message Reception:** Receive messages via webhook and publish to Pub/Sub
4. **Message Processing:** Agent processes messages and publishes responses back to Pub/Sub
5. **Response Delivery:** Service receives responses from Pub/Sub and sends to Telegram users

This flow enables your application to handle multiple Telegram bots, process messages through AI agents, and maintain real-time conversations.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Environment Variables](#environment-variables)
- [Bot Management](#bot-management)
- [Webhook Endpoints](#webhook-endpoints)
- [Pub/Sub Integration](#pubsub-integration)
- [Health Check](#health-check)
- [Error Handling](#error-handling)

## Architecture Overview

The service implements a multi-bot architecture with the following components:

- **Multi-bot Support:** Single webhook endpoint handles multiple bots identified by secret tokens
- **Pub/Sub Integration:** Messages are published to Google Cloud Pub/Sub for processing
- **Message Debouncing:** Prevents duplicate processing and batches rapid messages
- **Database Storage:** Bot configurations and user sessions stored in PostgreSQL
- **Asynchronous Processing:** Non-blocking message handling and response delivery

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `AGENT_APP_NAME` | Application name for sessions | "customer_support" |
| `PORT` | Server port | 8080 |
| `GOOGLE_CLOUD_PROJECT` | Google Cloud Project ID | Required |
| `PUBSUB_TOPIC` | Pub/Sub topic for message publishing | "agent-listener" |
| `PUBSUB_SUBSCRIPTION` | Pub/Sub subscription for responses | "agent-push" |

## Bot Management

### POST /telegram/create_bot
Create a new Telegram bot configuration.

**Request Body:**
```json
{
  "bot_id": "string",
  "bot_username": "string",
  "bot_name": "string",
  "bot_token": "string",
  "webhook_secret": "string",
  "webhook_url": "string",
  "managed_by_id": "string"
}
```

**Response:**
```json
{
  "status": "success"
}
```

**Status Codes:**
- `200 OK`: Successfully created bot
- `500 Internal Server Error`: Database error

### POST /telegram/delete_bot
Delete a Telegram bot configuration.

**Request Body:**
```json
{
  "bot_token": "string"
}
```

**Response:**
```json
{
  "status": "success"
}
```

**Status Codes:**
- `200 OK`: Successfully deleted bot
- `500 Internal Server Error`: Database error

### POST /telegram/update_bot
Update an existing Telegram bot configuration.

**Request Body:**
```json
{
  "bot_token": "string",
  "bot_username": "string",
  "bot_name": "string",
  "webhook_secret": "string",
  "webhook_url": "string",
  "managed_by_id": "string"
}
```

**Response:**
```json
{
  "status": "success"
}
```

**Status Codes:**
- `200 OK`: Successfully updated bot
- `500 Internal Server Error`: Database error

### GET /telegram/get_bot
Get bot configuration by token.

**Query Parameters:**
- `bot_token` (string, required): The bot's access token

**Response:**
```json
{
  "status": "success",
  "bot": {
    "bot_id": "string",
    "bot_username": "string",
    "bot_name": "string",
    "bot_token": "string",
    "webhook_secret": "string",
    "webhook_url": "string",
    "managed_by_id": "string"
  }
}
```

**Status Codes:**
- `200 OK`: Successfully retrieved bot
- `404 Not Found`: Bot not found

## Webhook Endpoints

### GET /telegram/webhook
Telegram webhook verification endpoint.

**Response:** Returns "OK" for webhook verification.

**Example:**
```bash
curl "https://your-domain.com/telegram/webhook"
```

### POST /telegram/webhook
Handles incoming webhook events from Telegram.

**Headers:**
- `X-Telegram-Bot-Api-Secret-Token` (required): Secret token to identify the bot

**Request Body:** Telegram webhook event data

**Response:**
```json
{
  "status": "ok"
}
```

**Supported Events:**
- Text messages
- Edited messages
- Callback queries

**Features:**
- **Bot Identification:** Uses `X-Telegram-Bot-Api-Secret-Token` header to identify which bot sent the message
- **Message Deduplication:** Prevents processing duplicate messages using `update_id`
- **Message Debouncing:** Batches rapid messages from the same user (4-second window)
- **Error Handling:** Sends friendly error messages on processing failures

## Pub/Sub Integration

### POST /post_message_pubsub
Receives Pub/Sub push deliveries from the agent processing system.

**Request Body:** `PubSubPushEnvelope`
```json
{
  "message": {
    "data": "string (base64 encoded)",
    "message_id": "string",
    "publish_time": "string"
  },
  "subscription": "string"
}
```

**Response:** No content (HTTP 204)

**Processing Logic:**
1. Decodes base64 message data
2. Validates platform is "telegram"
3. Extracts chat_id, message text, and bot_token
4. Removes markdown formatting from response text
5. Sends message to Telegram user asynchronously

**Status Codes:**
- `204 No Content`: Successfully processed (always returns 204 to avoid Pub/Sub retries)

## Message Processing Flow

### 1. Message Reception
```
Telegram User → Telegram API → Webhook → Service
```

### 2. Message Publishing
```
Service → Pub/Sub Topic → Agent Processing
```

**Message Format:**
```json
{
  "app_name": "customer_support",
  "user_id": "telegram_chat_id",
  "session_id": "telegram_{chat_id}",
  "new_message": {
    "role": "user",
    "parts": [{"text": "user message"}]
  },
  "platform": "telegram",
  "holder_id": "telegram_bot",
  "streaming": false,
  "telegram": {
    "bot_token": "bot_access_token"
  },
  "created_at": "2024-01-01T12:00:00"
}
```

### 3. Response Processing
```
Agent Processing → Pub/Sub Subscription → Service → Telegram API → User
```

**Response Format:**
```json
{
  "platform": "telegram",
  "user_id": "telegram_chat_id",
  "new_message": {
    "role": "assistant",
    "parts": [{"text": "agent response"}]
  },
  "telegram": {
    "bot_token": "bot_access_token"
  }
}
```

## Health Check

### GET /health
Health check endpoint to verify service status.

**Response:**
```json
{
  "status": "healthy",
  "app_name": "customer_support",
  "pubsub_topic": "agent-listener",
  "pubsub_subscription": "agent-push"
}
```

**Status Codes:**
- `200 OK`: Service is healthy
- `500 Internal Server Error`: Service health check failed

## Error Handling

All endpoints return appropriate HTTP status codes:

- `200` - Success
- `204` - No Content (Pub/Sub responses)
- `403` - Forbidden (invalid bot secret)
- `404` - Not Found (bot not found)
- `500` - Internal Server Error

Error responses include a `detail` field with the error message:

```json
{
  "detail": "Error description"
}
```

## Message Features

### Debouncing
- Messages from the same user are queued for 4 seconds
- Rapid messages are combined into a single request
- Prevents spam and improves processing efficiency

### Deduplication
- Uses Telegram's `update_id` to prevent duplicate processing
- Maintains a cache of processed updates (max 20,000 entries)
- Automatically clears cache when full

### Markdown Processing
- Removes markdown formatting from agent responses
- Converts to plain text for Telegram compatibility
- Handles code blocks, links, and formatting

### Error Recovery
- Sends friendly error messages to users on processing failures
- Implements retry logic for Pub/Sub publishing
- Graceful handling of network failures

## Database Schema

The service requires the following database functions:

```python
# Required functions
add_telegram_bot(bot_id, bot_username, bot_name, bot_token, webhook_secret, webhook_url, managed_by_id)
delete_telegram_bot(bot_token)
update_telegram_bot(bot_token, bot_username, bot_name, webhook_secret, webhook_url, managed_by_id)
get_telegram_bot_by_secret(secret_token) -> dict | None

# Optional functions
get_telegram_bot_by_token(bot_token) -> dict | None
```

## Security Considerations

1. **Bot Secret Validation:** All webhook requests must include the correct `X-Telegram-Bot-Api-Secret-Token`
2. **Token Management:** Bot tokens are stored securely in the database
3. **Message Validation:** All incoming messages are validated before processing
4. **Error Logging:** Comprehensive error logging without exposing sensitive data
5. **Rate Limiting:** Built-in debouncing prevents message spam

## Deployment

The service runs on the configured port (default: 8080) and requires:

1. Valid Google Cloud Project configuration
2. Pub/Sub topic and subscription setup
3. PostgreSQL database with required functions
4. Proper environment variable configuration

## Example Usage

### 1. Create a Bot
```bash
curl -X POST "https://your-domain.com/telegram/create_bot" \
  -H "Content-Type: application/json" \
  -d '{
    "bot_id": "123456789",
    "bot_username": "my_bot",
    "bot_name": "My Telegram Bot",
    "bot_token": "1234567890:ABCdefGHIjklMNOpqrsTUVwxyz",
    "webhook_secret": "my_secret_token",
    "webhook_url": "https://your-domain.com/telegram/webhook",
    "managed_by_id": "admin_user"
  }'
```

### 2. Set Webhook (via Telegram Bot API)
```bash
curl -X POST "https://api.telegram.org/bot{bot_token}/setWebhook" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-domain.com/telegram/webhook",
    "secret_token": "my_secret_token"
  }'
```

### 3. Test Health Check
```bash
curl "https://your-domain.com/health"
```

### 4. Get Bot Configuration
```bash
curl "https://your-domain.com/telegram/get_bot?bot_token=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz"
```

## Integration with Main Agent Service

This Telegram service integrates with the main agent service through Pub/Sub:

1. **Message Flow:** Telegram → Webhook → Pub/Sub → Agent → Pub/Sub → Telegram
2. **Session Management:** Each Telegram chat gets a unique session ID
3. **Multi-bot Support:** Single service handles multiple bots simultaneously
4. **Asynchronous Processing:** Non-blocking message handling for better performance

The service acts as a bridge between Telegram's webhook system and the main agent processing infrastructure, enabling scalable, multi-bot Telegram integration.
