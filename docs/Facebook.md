# Facebook Messenger API Documentation

This document describes the API endpoints for the Facebook Messenger integration service that handles webhook events, message processing, and page management.


## General Flow

1. **Obtain Login URL:** Retrieve the Facebook login URL to initiate the OAuth process.
2. **User Login:** The user logs in with Facebook and receives a `user_access_token`.
3. **Get List Page:** Get the list page for selection of app subscription (including id and access token for page)
4. **Subscribe App to Page:** Use the user access token to subscribe your app to the desired Facebook page.
5. **Register Webhook:** Set up the webhook to receive events and data from Facebook.

- You can manually send the message by calling /api/facebook/send_message 

This flow enables your application to authenticate users, connect to Facebook pages, and receive real-time updates via webhooks.

## Table of Contents

- [Authentication](#authentication)
- [Environment Variables](#environment-variables)
- [Webhook Endpoints](#webhook-endpoints)
- [Message Management](#message-management)
- [Page Management](#page-management)
- [OAuth Integration](#oauth-integration)
- [Admin Endpoints](#admin-endpoints)
- [Health Check](#health-check)



## Authentication

Most endpoints require an admin secret key for authentication. Set this via the `ADMIN_SECRET_KEY` environment variable.

**Header/Query Parameter:** `secret_key`

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

## Message Management

### POST /api/facebook/send_message
Send a message to a Facebook user.

**Request Body:**
```json
{
  "recipient_id": "user_facebook_id",
  "message_text": "Hello from the bot!",
  "page_id": "page_facebook_id"
}
```

**Response:**
```json
{
  "status": "ok"
}
```

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

## Page Management

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

### GET /api/facebook/pages/stored
Get all Facebook pages stored in the database.

**Query Parameters:**
- `secret_key` - Admin secret key

**Response:**
```json
{
  "status": "success",
  "pages": [
    {
      "page_id": "page_id",
      "page_name": "Page Name",
      "page_access_token": "page_access_token",
      "webhook_url": "https://webhook-url.com"
    }
  ]
}
```

### POST /api/facebook/subscribe
Subscribe the app to Messenger events for a Facebook Page.

**Request Body:**
```json
{
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

## OAuth Integration

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

## Error Handling

All endpoints return appropriate HTTP status codes:

- `200` - Success
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

## Message Processing

The service processes incoming messages with the following features:

1. **Message Deduplication**: Prevents processing duplicate messages
2. **Message Queuing**: Collects messages for 8 seconds before processing
3. **Session Management**: Maintains user sessions with visit counts and preferences
4. **AI Integration**: Processes messages through the configured AI agent
5. **Error Handling**: Sends fallback messages on errors
6. **Webhook Forwarding**: Forwards messages to configured webhook URLs

## Database Operations

The service uses PostgreSQL to store:

- Facebook page information (ID, access token, name, webhook URL)
- User sessions and conversation history
- Message processing state

## Security Considerations

1. **Admin Authentication**: All sensitive operations require the admin secret key
2. **OAuth State Validation**: Prevents CSRF attacks in OAuth flow
3. **Token Management**: Securely stores and manages Facebook access tokens
4. **Input Validation**: Validates all incoming data and parameters
5. **Error Logging**: Comprehensive error logging without exposing sensitive data

## Rate Limiting

The service implements message queuing to handle high message volumes:

- Messages are queued per user
- Processing is delayed by 8 seconds to batch messages
- Duplicate messages are automatically filtered
- Processing tasks are cancelled and recreated for new messages

## Deployment

The service runs on the configured port (default: 8080) and requires:

1. Valid Facebook App configuration
2. PostgreSQL database access
3. Google Cloud authentication (for agent API)
4. Proper environment variable configuration

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
  -d '{"page_id": "page_id", "user_access_token": "user_token"}'
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
  -d '{"recipient_id": "user_id", "message_text": "Hello!", "page_id": "page_id"}'
```
