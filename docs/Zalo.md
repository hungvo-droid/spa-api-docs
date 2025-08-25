# Zalo OA API Documentation

This document describes the API endpoints for the Zalo Official Account (OA) integration service that handles webhook events, message processing, and OA management through a Pub/Sub architecture.

## General Flow

1. **OAuth Authentication:** Complete Zalo OAuth flow to obtain access and refresh tokens
2. **Webhook Setup:** Configure webhook URL for receiving Zalo events
3. **Message Reception:** Receive messages via webhook and publish to Pub/Sub
4. **Message Processing:** Agent processes messages and publishes responses back to Pub/Sub
5. **Response Delivery:** Service receives responses from Pub/Sub and sends to Zalo users

This flow enables your application to handle Zalo OA interactions, process messages through AI agents, and maintain real-time conversations with automatic token refresh.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Environment Variables](#environment-variables)
- [OAuth Authentication](#oauth-authentication)
- [Webhook Endpoints](#webhook-endpoints)
- [Pub/Sub Integration](#pubsub-integration)
- [Health Check](#health-check)
- [Error Handling](#error-handling)

## Architecture Overview

The service implements a Zalo OA integration with the following components:

- **OAuth Integration:** Complete PKCE OAuth flow for Zalo OA authentication
- **Pub/Sub Integration:** Messages are published to Google Cloud Pub/Sub for processing
- **Message Debouncing:** Prevents duplicate processing and batches rapid messages (4-second window)
- **Session Management:** Maintains user sessions with visit counts and preferences
- **Token Management:** Automatic refresh of expired access tokens
- **Inactivity Monitoring:** Automatic cancellation of appointments after 15 minutes of inactivity

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `APP_URL` | Base URL for the agent API | Required |
| `ZALO_APP_ID` | Zalo application ID | Required |
| `ZALO_APP_SECRET` | Zalo application secret | Required |
| `ZALO_ACCESS_TOKEN` | Initial Zalo access token | "None" |
| `ZALO_REFRESH_TOKEN` | Initial Zalo refresh token | "None" |
| `AGENT_APP_NAME` | Application name for sessions | "customer_support" |
| `PORT` | Server port | 8000 |
| `INACTIVITY_TIMEOUT` | Inactivity timeout in seconds | 900 (15 minutes) |
| `CHECK_INACTIVE_USERS_ENABLED` | Enable inactivity checking | "true" |
| `GOOGLE_CLOUD_PROJECT` | Google Cloud Project ID | Required |
| `WEBHOOK_URL` | Webhook URL for OAuth callback | Auto-generated |

## OAuth Authentication

### GET /auth
Initiate Zalo OAuth authentication flow.

**Description:** Redirects user to Zalo OAuth authorization page with PKCE challenge.

**Response:** HTTP 302 redirect to Zalo OAuth URL

**OAuth Flow:**
1. Generates PKCE code verifier and challenge
2. Creates state parameter for CSRF protection
3. Redirects to Zalo OAuth endpoint with required parameters

**Example:**
```bash
curl "https://your-domain.com/auth"
```

### GET /webhook (OAuth Callback)
Handle OAuth callback from Zalo.

**Query Parameters:**
- `code` - Authorization code from Zalo
- `state` - State parameter for CSRF protection
- `error` - Error code if OAuth failed (optional)

**Response:**
- `200 OK`: "Authentication successful! You can close this window."
- `400 Bad Request`: Authentication error details

**Processing:**
1. Validates state parameter
2. Exchanges authorization code for access and refresh tokens
3. Stores tokens in application state
4. Updates global token variables

## Webhook Endpoints

### GET /webhook (Verification)
Zalo webhook verification endpoint.

**Headers:**
- `verifier` - Verification token from Zalo

**Response:** Returns the verifier token for webhook verification

**Example:**
```bash
curl "https://your-domain.com/webhook" \
  -H "verifier: your_verification_token"
```

### POST /webhook
Handles incoming webhook events from Zalo OA.

**Request Body:** Zalo webhook event data

**Response:** `"OK"`

**Supported Events:**
- `user_send_text` - Text messages from users
- `user_send_image` - Image messages from users
- `user_send_file` - File messages from users
- `oa_send_text` - Text messages sent by OA

**Features:**
- **User Tag Filtering:** Checks user tags to determine if message should be handled by AI or admin
- **Message Debouncing:** Batches rapid messages from the same user (4-second window)
- **Session Management:** Maintains user sessions with visit counts and preferences
- **Inactivity Tracking:** Monitors user activity for appointment management

**Event Processing:**
1. **Text Messages:** Processed through AI agent or admin based on user tags
2. **Media Messages:** Sends fallback message requesting text format
3. **OA Messages:** Updates session with admin message

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
2. Validates platform is "zalo"
3. Extracts user_id and message text
4. Removes markdown formatting from response text
5. Sends message to Zalo user asynchronously

**Status Codes:**
- `204 No Content`: Successfully processed (always returns 204 to avoid Pub/Sub retries)

## Message Processing Flow

### 1. Message Reception
```
Zalo User → Zalo OA API → Webhook → Service
```

### 2. Message Publishing
```
Service → Pub/Sub Topic → Agent Processing
```

**Message Format:**
```json
{
  "app_name": "customer_support",
  "user_id": "zalo_user_id",
  "session_id": "zalo_{user_id}",
  "new_message": {
    "role": "user",
    "parts": [{"text": "user message"}]
  },
  "platform": "zalo",
  "streaming": false
}
```

### 3. Response Processing
```
Agent Processing → Pub/Sub Subscription → Service → Zalo OA API → User
```

**Response Format:**
```json
{
  "platform": "zalo",
  "user_id": "zalo_user_id",
  "new_message": {
    "role": "assistant",
    "parts": [{"text": "agent response"}]
  }
}
```

## Health Check

### GET /_ah/health
Health check endpoint to verify service status.

**Response:**
```json
{
  "status": "healthy|unhealthy|degraded",
  "details": {
    "zalo": {
      "oa_id": "string",
      "oa_name": "string"
    }
  }
}
```

**Status Codes:**
- `200 OK`: Service is healthy or degraded
- `500 Internal Server Error`: Service is unhealthy

**Health Checks:**
1. **Token Validation:** Verifies Zalo access token is valid
2. **OA Profile:** Fetches OA profile information
3. **Token Refresh:** Attempts token refresh if needed

## Error Handling

All endpoints return appropriate HTTP status codes:

- `200` - Success
- `204` - No Content (Pub/Sub responses)
- `400` - Bad Request (OAuth errors, invalid parameters)
- `404` - Not Found (static files)
- `500` - Internal Server Error

Error responses include detailed error messages:

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

### User Tag Filtering
- Checks user tags via Zalo API
- Messages from users with "NonAI" tag are handled by admin
- Other messages are processed through AI agent

### Session Management
- Unique session IDs for each Zalo user
- Tracks visit counts and message timestamps
- Maintains conversation context and preferences

### Inactivity Monitoring
- Monitors user activity for appointment-related conversations
- Automatically cancels appointments after 15 minutes of inactivity
- Sends cancellation messages and updates session state

### Markdown Processing
- Removes markdown formatting from agent responses
- Converts to plain text for Zalo compatibility
- Handles code blocks, links, and formatting

### Token Management
- Automatic refresh of expired access tokens
- Retry logic for failed API calls
- Graceful handling of authentication errors

## Database Operations

The service integrates with the main agent service for session management:

- **Session Creation:** Creates or updates user sessions via agent API
- **Event Logging:** Records conversation events and admin messages
- **State Management:** Maintains session state with user preferences

## Security Considerations

1. **OAuth Security:** Implements PKCE flow for secure OAuth authentication
2. **State Validation:** Prevents CSRF attacks in OAuth flow
3. **Token Management:** Securely stores and refreshes Zalo access tokens
4. **Input Validation:** Validates all incoming webhook data
5. **Error Logging:** Comprehensive error logging without exposing sensitive data

## Rate Limiting

The service implements message queuing to handle high message volumes:

- Messages are queued per user
- Processing is delayed by 4 seconds to batch messages
- Duplicate messages are automatically filtered
- Processing tasks are cancelled and recreated for new messages

## Deployment

The service runs on the configured port (default: 8000) and requires:

1. Valid Zalo OA application configuration
2. Google Cloud Project with Pub/Sub setup
3. Agent API access for session management
4. Proper environment variable configuration

## Example Usage

### 1. OAuth Authentication
```bash
# Start OAuth flow
curl "https://your-domain.com/auth"

# Complete OAuth callback (browser redirect)
# https://your-domain.com/webhook?code=authorization_code&state=state_param
```

### 2. Webhook Verification
```bash
curl "https://your-domain.com/webhook" \
  -H "verifier: your_verification_token"
```

### 3. Test Health Check
```bash
curl "https://your-domain.com/_ah/health"
```

### 4. Send Test Message (via Zalo OA API)
```bash
curl -X POST "https://openapi.zalo.me/v3.0/oa/message/cs" \
  -H "Content-Type: application/json" \
  -H "access_token: your_access_token" \
  -d '{
    "recipient": {"user_id": "user_id"},
    "message": {"text": "Hello from OA!"}
  }'
```

## Integration with Main Agent Service

This Zalo service integrates with the main agent service through Pub/Sub:

1. **Message Flow:** Zalo → Webhook → Pub/Sub → Agent → Pub/Sub → Zalo
2. **Session Management:** Each Zalo user gets a unique session ID
3. **OAuth Integration:** Complete PKCE OAuth flow for secure authentication
4. **Token Management:** Automatic refresh and validation of access tokens
5. **Inactivity Monitoring:** Built-in appointment management with automatic cancellation

The service acts as a bridge between Zalo's OA platform and the main agent processing infrastructure, enabling scalable, secure Zalo OA integration with advanced features like inactivity monitoring and user tag filtering.
