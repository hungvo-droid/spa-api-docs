# Facebook Messenger API Documentation

This document describes the API endpoints for the Facebook Messenger integration service that handles webhook events, message processing, page management, and OAuth authentication through a Pub/Sub architecture.

## General Flow

1. **OAuth Authentication:** Complete Facebook OAuth flow to obtain user access tokens
2. **Page Management:** Get and manage Facebook pages with their access tokens
3. **Page Subscription:** Subscribe pages to Messenger events
4. **Webhook Setup:** Configure webhook URL for receiving Facebook events
5. **Message Processing:** Receive messages via webhook and publish to Pub/Sub
6. **Response Delivery:** Service receives responses from Pub/Sub and sends to Facebook users

This flow enables your application to authenticate users, connect to Facebook pages, process messages through AI agents, and maintain real-time conversations.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Environment Variables](#environment-variables)
- [OAuth Authentication](#oauth-authentication)
- [Page Management](#page-management)
- [Webhook Endpoints](#webhook-endpoints)
- [Message Management](#message-management)
- [Pub/Sub Integration](#pubsub-integration)
- [Admin Endpoints](#admin-endpoints)
- [Health Check](#health-check)
- [Error Handling](#error-handling)

## Architecture Overview

The service implements a Facebook Messenger integration with the following components:

- **OAuth Integration:** Complete Facebook OAuth flow for user authentication
- **Pub/Sub Integration:** Messages are published to Google Cloud Pub/Sub for processing
- **Message Debouncing:** Prevents duplicate processing and batches rapid messages (4-second window)
- **Session Management:** Maintains user sessions with visit counts and preferences
- **Page Management:** Database storage for Facebook page configurations
- **Webhook Forwarding:** Forwards messages to configured webhook URLs
- **AI Toggle:** Per-page control over AI processing

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `APP_URL` | Base URL for the agent API | Required |
| `FACEBOOK_PAGE_TOKEN` | Facebook page access token | Required |
| `FACEBOOK_VERIFY_TOKEN` | Webhook verification token | "digiworker" |
| `AGENT_APP_NAME` | Application name for sessions | "customer_support" |
| `PORT` | Server port | 8080 |
| `ADMIN_SECRET_KEY` | Admin authentication key | "admin123" |
| `WEBHOOK_URL` | Base webhook URL | Required |
| `FACEBOOK_APP_ID` | Facebook app ID for OAuth | Required |
| `FACEBOOK_APP_SECRET` | Facebook app secret for OAuth | Required |
| `FACEBOOK_REDIRECT_URI` | OAuth redirect URI | Auto-generated |
| `GOOGLE_CLOUD_PROJECT` | Google Cloud Project ID | Required |

## OAuth Authentication

### GET /api/facebook/oauth/url
Get Facebook OAuth URL for user login.

**Query Parameters:**
- `redirect_uri` - Optional redirect URI (defaults to config)
- `state` - Optional state parameter for CSRF protection
- `secret_key` - Admin secret key

**Response:**
```json
{
  "status": "success",
  "oauth_url": "https://www.facebook.com/v19.0/dialog/oauth?...",
  "state": "generated_state"
}
```

**OAuth Scopes:**
- `email` - User email access
- `pages_show_list` - List user's pages
- `pages_read_user_content` - Read page content
- `pages_read_engagement` - Read page engagement
- `pages_manage_metadata` - Manage page metadata
- `pages_messaging` - Send messages from pages

### GET /api/facebook/oauth/callback
Handle Facebook OAuth callback.

**Query Parameters:**
- `code` - Authorization code from Facebook
- `state` - State parameter for CSRF protection
- `error` - Error code if OAuth failed (optional)
- `error_reason` - Reason for the error (optional)
- `error_description` - Description of the error (optional)

**Response:**
```json
{
  "status": "success",
  "access_token": "user_access_token",
  "token_type": "bearer",
  "expires_in": 3600,
  "user": {
    "id": "user_id",
    "name": "User Name",
    "email": "user@example.com"
  }
}
```

## Page Management

### POST /api/facebook/pages
Get a list of Facebook pages that the user can manage.

**Request Body:**
```json
{
  "user_access_token": "user_access_token"
}
```

**Query Parameters:**
- `secret_key` - Admin secret key

**Response:**
```json
{
  "status": "success",
  "pages": [
    {
      "id": "page_id",
      "name": "Page Name",
      "category": "Business",
      "access_token": "page_access_token",
      "permissions": [...]
    }
  ]
}
```

### GET /api/facebook/pages/stored_adminid
Get all Facebook pages stored in the database for an admin.

**Query Parameters:**
- `admin_email` - Admin email address
- `secret_key` - Admin secret key
- `list_page` - If true, return only filtered fields

**Response:**
```json
{
  "status": "success",
  "pages": [
    {
      "page_id": "page_id",
      "page_name": "Page Name",
      "page_access_token": "page_access_token",
      "webhook_url": "https://webhook-url.com",
      "using_ai": true,
      "connected_at": "timestamp"
    }
  ]
}
```

### POST /api/facebook/subscribe
Subscribe the app to Messenger events for a Facebook Page.

**Request Body:**
```json
{
  "email": "admin_email",
  "page_id": "page_facebook_id",
  "user_access_token": "user_access_token",
  "fields": ["messages", "messaging_postbacks", "messaging_optins", "message_deliveries", "message_reads"],
  "callback_url": "https://optional-callback.com"
}
```

**Query Parameters:**
- `secret_key` - Admin secret key

**Response:**
```json
{
  "status": "success",
  "message": "Successfully subscribed page to Messenger events",
  "page_id": "page_facebook_id",
  "subscribed_fields": ["messages", "messaging_postbacks"]
}
```

### DELETE /api/facebook/unsubscribe
Unsubscribe the app from Messenger events for a Facebook Page.

**Query Parameters:**
- `page_id` - Facebook page ID
- `secret_key` - Admin secret key

**Response:**
```json
{
  "status": "success",
  "message": "Successfully unsubscribed page from Messenger events",
  "page_id": "page_facebook_id"
}
```

### PUT /api/facebook/pages/toggle_ai
Toggle AI usage for a Facebook page.

**Query Parameters:**
- `page_id` - Facebook page ID
- `using_ai` - Boolean to enable/disable AI
- `secret_key` - Admin secret key

**Response:**
```json
{
  "status": "success",
  "message": "AI setting updated successfully for page page_id",
  "page_id": "page_id",
  "using_ai": true
}
```

## Webhook Endpoints

### GET /webhook
Facebook webhook verification endpoint.

**Query Parameters:**
- `hub.mode` - Verification mode (subscribe)
- `hub.verify_token` - Verification token
- `hub.challenge` - Challenge string

**Response:** Returns the challenge string if verification is successful.

**Example:**
```bash
curl "https://your-domain.com/webhook?hub.mode=subscribe&hub.verify_token=digiworker&hub.challenge=1234567890"
```

### POST /webhook
Handles incoming webhook events from Facebook Messenger.

**Request Body:** Facebook webhook event data

**Response:** `{"status": "ok"}`

**Supported Events:**
- Text messages
- Postback events
- Message deliveries
- Message reads

**Features:**
- **Message Deduplication:** Prevents processing duplicate messages using unique keys
- **Message Debouncing:** Batches rapid messages from the same user (4-second window)
- **Page Validation:** Checks if page exists and is connected before processing
- **User Storage:** Stores user information in database
- **Webhook Forwarding:** Forwards messages to configured webhook URLs

## Message Management

### POST /api/facebook/send_message
Send a message to a Facebook user.

**Request Body:**
```json
{
  "recipient_id": "user_facebook_id",
  "message_text": "Hello from the bot!",
  "page_id": "page_facebook_id",
  "author": "admin_name",
  "role": "admin"
}
```

**Response:**
```json
{
  "status": "ok"
}
```

**Processing:**
1. Sends message to Facebook user via Graph API
2. Adds message to session for tracking
3. Forwards message to configured webhook URL

### POST /api/post_message
Generic endpoint to post message to any URL.

**Request Body:**
```json
{
  "url": "https://example.com/webhook",
  "message_data": {
    "role": "user",
    "message": "Hello"
  },
  "headers": {
    "Content-Type": "application/json"
  }
}
```

**Response:**
```json
{
  "status": "success",
  "status_code": 200,
  "data": "response_data"
}
```

## Webhook Configuration

### POST /api/facebook/pages/add_webhook
Add webhook URL to an existing Facebook page.

**Request Body:**
```json
{
  "page_id": "page_facebook_id",
  "webhook_url": "https://your-domain.com/webhook",
  "secret_key": "admin123"
}
```

**Response:**
```json
{
  "status": "success",
  "message": "Webhook URL added successfully to page page_id",
  "page_id": "page_facebook_id",
  "webhook_url": "https://your-domain.com/webhook"
}
```

### GET /api/facebook/pages/get_webhook
Get webhook URL for a specific page.

**Query Parameters:**
- `page_id` - Facebook page ID
- `secret_key` - Admin secret key

**Response:**
```json
{
  "status": "success",
  "page_id": "page_facebook_id",
  "webhook_url": "https://your-domain.com/webhook",
  "has_webhook": true
}
```

### DELETE /api/facebook/pages/delete_webhook
Remove webhook URL from a page.

**Query Parameters:**
- `page_id` - Facebook page ID
- `secret_key` - Admin secret key

**Response:**
```json
{
  "status": "success",
  "message": "Webhook URL removed successfully from page page_id",
  "page_id": "page_facebook_id"
}
```

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
2. Validates platform is "facebook"
3. Extracts user_id, message text, and page_id
4. Removes markdown formatting from response text
5. Sends message to Facebook user asynchronously
6. Forwards message to webhook URL

**Status Codes:**
- `204 No Content`: Successfully processed (always returns 204 to avoid Pub/Sub retries)

## Message Processing Flow

### 1. Message Reception
```
Facebook User → Facebook API → Webhook → Service
```

### 2. Message Publishing
```
Service → Pub/Sub Topic → Agent Processing
```

**Message Format:**
```json
{
  "app_name": "customer_support",
  "user_id": "facebook_user_id",
  "session_id": "facebook_{user_id}",
  "new_message": {
    "role": "user",
    "parts": [{"text": "user message"}]
  },
  "platform": "facebook",
  "streaming": false
}
```

### 3. Response Processing
```
Agent Processing → Pub/Sub Subscription → Service → Facebook API → User
```

**Response Format:**
```json
{
  "platform": "facebook",
  "user_id": "facebook_user_id",
  "new_message": {
    "role": "assistant",
    "parts": [{"text": "agent response"}]
  },
  "holder_id": "page_id"
}
```

## Admin Endpoints

### GET /admin
Admin interface for managing conversations.

**Query Parameters:**
- `secret_key` - Admin secret key

**Response:**
```json
{
  "active_users": ["user_id_1", "user_id_2"],
  "admin_controlled_users": ["user_id_3"]
}
```

### GET /api/get_manage_fb_info
Get managed Facebook page information for an admin.

**Query Parameters:**
- `email` - Admin email address

**Response:**
```json
{
  "status": "success",
  "users": [...],
  "pages": [...]
}
```

### POST /api/facebook/add_message_to_session
Add a message to the session for tracking.

**Request Body:**
```json
{
  "app_name": "customer_support",
  "user_id": "user_id",
  "session_id": "session_id",
  "role": "user|admin",
  "text": "message text",
  "author": "author_name"
}
```

**Response:**
```json
{
  "message": "Message added to session"
}
```

## Pub/Sub Management

### GET /pubsub/status
Check Pub/Sub topic and subscription status.

**Response:**
```json
{
  "status": "healthy",
  "monitoring_active": true,
  "topic": {
    "name": "topic_name",
    "exists": true
  },
  "subscription": {
    "name": "subscription_name",
    "status": "active"
  }
}
```

### POST /pubsub/start
Start Pub/Sub message monitoring.

**Response:**
```json
{
  "status": "success",
  "message": "Pub/Sub monitoring started",
  "subscription": "subscription_path"
}
```

### POST /pubsub/stop
Stop Pub/Sub message monitoring.

**Response:**
```json
{
  "status": "success",
  "message": "Pub/Sub monitoring stopped"
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
  "facebook_configured": true
}
```

**Status Codes:**
- `200 OK`: Service is healthy
- `500 Internal Server Error`: Service health check failed

## Error Handling

All endpoints return appropriate HTTP status codes:

- `200` - Success
- `204` - No Content (Pub/Sub responses)
- `400` - Bad Request (missing parameters, invalid data)
- `401` - Unauthorized (invalid access token)
- `403` - Forbidden (invalid admin key)
- `404` - Not Found (page not found)
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
- Uses unique message keys to prevent duplicate processing
- Maintains a cache of processed messages (max 10,000 entries)
- Automatically clears cache when full

### Session Management
- Unique session IDs for each Facebook user
- Tracks visit counts and message timestamps
- Maintains conversation context and preferences

### AI Toggle
- Per-page control over AI processing
- Messages from pages with AI disabled are stored but not processed
- Admin can toggle AI on/off via API

### Webhook Forwarding
- Forwards all messages to configured webhook URLs
- Includes message metadata and timestamps
- Supports custom headers and authentication

### Markdown Processing
- Removes markdown formatting from agent responses
- Converts to plain text for Facebook compatibility
- Handles code blocks, links, and formatting

## Database Operations

The service uses PostgreSQL to store:

- **Facebook Pages:** Page ID, access token, name, webhook URL, AI settings
- **User Sessions:** Conversation history and preferences
- **User Accounts:** Admin user information and page associations
- **Message Events:** Tracking of all message interactions

## Security Considerations

1. **Admin Authentication:** All sensitive operations require the admin secret key
2. **OAuth State Validation:** Prevents CSRF attacks in OAuth flow
3. **Token Management:** Securely stores and manages Facebook access tokens
4. **Input Validation:** Validates all incoming data and parameters
5. **Error Logging:** Comprehensive error logging without exposing sensitive data

## Rate Limiting

The service implements message queuing to handle high message volumes:

- Messages are queued per user
- Processing is delayed by 4 seconds to batch messages
- Duplicate messages are automatically filtered
- Processing tasks are cancelled and recreated for new messages

## Deployment

The service runs on the configured port (default: 8080) and requires:

1. Valid Facebook App configuration
2. PostgreSQL database access
3. Google Cloud authentication (for agent API)
4. Pub/Sub topic and subscription setup
5. Proper environment variable configuration

## Example Usage

### Complete Setup Flow

1. **Configure OAuth**:
```bash
curl "https://your-domain.com/api/facebook/oauth/url?secret_key=admin123"
```

2. **Get User Pages**:
```bash
curl -X POST "https://your-domain.com/api/facebook/pages?secret_key=admin123" \
  -H "Content-Type: application/json" \
  -d '{"user_access_token": "user_token"}'
```

3. **Subscribe to Page Events**:
```bash
curl -X POST "https://your-domain.com/api/facebook/subscribe?secret_key=admin123" \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@example.com", "page_id": "page_id", "user_access_token": "user_token"}'
```

4. **Add Webhook URL**:
```bash
curl -X POST "https://your-domain.com/api/facebook/pages/add_webhook" \
  -H "Content-Type: application/json" \
  -d '{"page_id": "page_id", "webhook_url": "https://your-domain.com/webhook", "secret_key": "admin123"}'
```

5. **Send Test Message**:
```bash
curl -X POST "https://your-domain.com/api/facebook/send_message" \
  -H "Content-Type: application/json" \
  -d '{"recipient_id": "user_id", "message": "Hello!", "page_id": "page_id", "author": "admin", "role": "admin"}'
```

6. **Toggle AI for Page**:
```bash
curl -X PUT "https://your-domain.com/api/facebook/pages/toggle_ai?page_id=page_id&using_ai=false&secret_key=admin123"
```

## Integration with Main Agent Service

This Facebook service integrates with the main agent service through Pub/Sub:

1. **Message Flow:** Facebook → Webhook → Pub/Sub → Agent → Pub/Sub → Facebook
2. **Session Management:** Each Facebook user gets a unique session ID
3. **OAuth Integration:** Complete Facebook OAuth flow for secure authentication
4. **Page Management:** Database storage and management of Facebook pages
5. **Webhook Forwarding:** Built-in webhook forwarding for external integrations

The service acts as a bridge between Facebook's Messenger platform and the main agent processing infrastructure, enabling scalable, secure Facebook Messenger integration with advanced features like AI toggling and webhook forwarding.
