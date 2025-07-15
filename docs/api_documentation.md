# SPA Customer Service API Documentation

This document provides details for all API endpoints in the SPA Customer Service application.

## Application Endpoints

### List Applications
**Endpoint:** `GET /list-apps`  
**Description:** List all available applications  
**Input Schema:** No parameters required  
**Output Schema:** Array of strings
```json
[
  "string"
]
```
**Status Codes:**
- `200 OK`: Successfully retrieved applications list

### List Sessions
**Endpoint:** `GET /apps/{app_name}/users/{user_id}/sessions`  
**Description:** List all sessions for a specific user and application  
**Input Schema:** Path parameters
- `app_name` (string, required): Name of the application
- `user_id` (string, required): ID of the user

**Output Schema:** Array of session objects
```json
[session_object]
```
**Status Codes:**
- `200 OK`: Successfully retrieved sessions list
- `403 Forbidden`: Not authorized to access these sessions
- `404 Not Found`: User or application not found

### Create Session
**Endpoint:** `POST /apps/{app_name}/users/{user_id}/sessions`  
**Description:** Create a new session with an auto-generated ID  
**Input Schema:** 
- **Path parameters:**
  - `app_name` (string, required): Name of the application
  - `user_id` (string, required): ID of the user

- **Request Body:**
```json
session_object
```

**Output Schema:** Same as the Get Session response

**Status Codes:**
- `200 OK`: Successfully created session
- `400 Bad Request`: Invalid request parameters
- `403 Forbidden`: Not authorized to create a session

### Get Session with session_id
**Endpoint:** `GET /apps/{app_name}/users/{user_id}/sessions/{session_id}`  
**Description:** Retrieve a specific session by ID  
**Input Schema:** Path parameters
- `app_name` (string, required): Name of the application
- `user_id` (string, required): ID of the user
- `session_id` (string, required): ID of the session to retrieve

**Output Schema:** Session object
```json
session_object
```
**Status Codes:**
- `200 OK`: Successfully retrieved session
- `404 Not Found`: Session not found
- `403 Forbidden`: Not authorized to access this session

**The data we use:**
To get the text of the n-th event in the chat:
`text = session["events"][n]["content"]["parts"][0]["text"]`

### Create Session With ID
**Endpoint:** `POST /apps/{app_name}/users/{user_id}/sessions/{session_id}`  
**Description:** Create a new session with a specified ID  
**Input Schema:** 
- **Path parameters:**
  - `app_name` (string, required): Name of the application
  - `user_id` (string, required): ID of the user
  - `session_id` (string, required): Desired ID for the new session

- **Request Body:**
**Output Schema:** Session object
```json
session_object
```

**Output Schema:** Same as the Get Session response

**Status Codes:**
- `200 OK`: Successfully created session
- `400 Bad Request`: Invalid request parameters
- `403 Forbidden`: Not authorized to create a session
- `409 Conflict`: Session ID already exists

### Delete Session
**Endpoint:** `DELETE /apps/{app_name}/users/{user_id}/sessions/{session_id}`  
**Description:** Delete a specific session by ID  
**Input Schema:** Path parameters
- `app_name` (string, required): Name of the application
- `user_id` (string, required): ID of the user
- `session_id` (string, required): ID of the session to delete

**Output Schema:**
```json
"string"
```

**Status Codes:**
- `200 OK`: Successfully deleted session
- `404 Not Found`: Session not found
- `403 Forbidden`: Not authorized to delete this session

## Authentication Endpoints

### Register User
**Endpoint:** `POST /register`  
**Description:** Register a new user in the system  
**Input Schema:** `UserCreate`
```json
{
  "email": "string (valid email format)",
  "full_name": "string (min length: 3)",
  "password": "string (min length: 8)"
}
```
**Output Schema:** `UserResponse`
```json
{
  "user_id": "string (UUID format)",
  "email": "string (valid email format)",
  "full_name": "string"
}
```
**Status Codes:**
- `200 OK`: Successfully registered
- `400 Bad Request`: Email already registered
- `500 Internal Server Error`: Database error

### Login User
**Endpoint:** `POST /login`  
**Description:** Authenticate a user and receive a JWT token  
**Input Schema:** `UserLogin`
```json
{
  "email": "string (valid email format)",
  "password": "string"
}
```
**Output Schema:** `Token`
```json
{
  "access_token": "string (JWT format)",
  "token_type": "string (always 'bearer')"
}
```
**Status Codes:**
- `200 OK`: Successfully authenticated
- `401 Unauthorized`: Incorrect email or password

### Run with Server-Sent Events
**Endpoint:** `POST /run_sse`  
**Description:** Standard-compatible endpoint that matches the ADK API format, supporting both streaming and non-streaming responses. Get the response from Agent LLM 
**Input Schema:** `RunRequest`
```json
{
  "app_name": "string",
  "user_id": "string",
  "session_id": "string",
  "new_message": {
    "role": "string (usually 'user')",
    "parts": [
      {
        "text": "string"
      }
    ]
  },
  "streaming": "boolean (default: false)"
}
```
**Output (Streaming):** Server-Sent Events stream (Content-Type: text/event-stream)
```
data: {"output": "string", "is_final": false}

data: {"output": "string", "is_final": true}

data: [DONE]
```

**Output Schema (Non-Streaming):**
```json
{
  "output": "string"
}
```
**Status Codes:**
- `200 OK`: Successfully processed
- `400 Bad Request`: Invalid message format
- `500 Internal Server Error`: Processing error

## Authentication

Most endpoints require authentication using a JWT token. Include the token in the Authorization header:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

The token is obtained from the `/login` endpoint and expires after 24 hours (JWT_EXPIRATION_MINUTES = 1440). 

## Data Models

### Session Object

The Session object represents a conversation session between a user and the customer support agent. It contains the following fields:

```json
{
  "id": "string",                 // Unique identifier for the session
  "appName": "string",            // Name of the application
  "userId": "string",             // ID of the user who owns the session
  "state": {                      // Session state data
    "additionalProp1": {}         // Custom state properties
  },
  "events": [                     // Array of events in the session
    {
      "content": {                // Content of the event
        "parts": [                // Parts that make up the content
          {
            "videoMetadata": {    // Metadata for video content
              "fps": 0,
              "endOffset": "string",
              "startOffset": "string"
            },
            "thought": true,      // Whether this part is a thought (internal reasoning)
            "inlineData": {       // Inline data like images
              "displayName": "string",
              "data": "string",
              "mimeType": "string"
            },
            "fileData": {         // File attachment data
              "displayName": "string",
              "fileUri": "string",
              "mimeType": "string"
            },
            "thoughtSignature": "string",
            "codeExecutionResult": {  // Results of code execution
              "outcome": "OUTCOME_UNSPECIFIED",
              "output": "string"
            },
            "executableCode": {    // Code that can be executed
              "code": "string",
              "language": "LANGUAGE_UNSPECIFIED"
            },
            "functionCall": {      // Function call details
              "id": "string",
              "args": {
                "additionalProp1": {}
              },
              "name": "string"
            },
            "functionResponse": {   // Response from function calls
              "willContinue": true,
              "scheduling": "SCHEDULING_UNSPECIFIED",
              "id": "string",
              "name": "string",
              "response": {
                "additionalProp1": {}
              }
            },
            "text": "string"        // Plain text content
          }
        ],
        "role": "string"           // Role of the content creator (user/assistant)
      },
      "groundingMetadata": {       // Metadata for grounding information
        "groundingChunks": [       // Chunks of information used for grounding
          {
            "retrievedContext": {
              "ragChunk": {
                "pageSpan": {
                  "firstPage": 0,
                  "lastPage": 0
                },
                "text": "string"
              },
              "text": "string",
              "title": "string",
              "uri": "string"
            },
            "web": {
              "domain": "string",
              "title": "string",
              "uri": "string"
            }
          }
        ],
        "groundingSupports": [     // Support information for grounding
          {
            "confidenceScores": [
              0
            ],
            "groundingChunkIndices": [
              0
            ],
            "segment": {
              "endIndex": 0,
              "partIndex": 0,
              "startIndex": 0,
              "text": "string"
            }
          }
        ],
        "retrievalMetadata": {
          "googleSearchDynamicRetrievalScore": 0
        },
        "retrievalQueries": [
          "string"
        ],
        "searchEntryPoint": {
          "renderedContent": "string",
          "sdkBlob": "string"
        },
        "webSearchQueries": [
          "string"
        ]
      },
      "partial": true,            // Whether this is a partial event
      "turnComplete": true,       // Whether this turn is complete
      "errorCode": "string",      // Error code if applicable
      "errorMessage": "string",   // Error message if applicable
      "interrupted": true,        // Whether this event was interrupted
      "customMetadata": {         // Custom metadata
        "additionalProp1": {}
      },
      "usageMetadata": {          // Usage statistics
        "cacheTokensDetails": [
          {
            "modality": "MODALITY_UNSPECIFIED",
            "tokenCount": 0
          }
        ],
        "cachedContentTokenCount": 0,
        "candidatesTokenCount": 0,
        "candidatesTokensDetails": [
          {
            "modality": "MODALITY_UNSPECIFIED",
            "tokenCount": 0
          }
        ],
        "promptTokenCount": 0,
        "promptTokensDetails": [
          {
            "modality": "MODALITY_UNSPECIFIED",
            "tokenCount": 0
          }
        ],
        "thoughtsTokenCount": 0,
        "toolUsePromptTokenCount": 0,
        "toolUsePromptTokensDetails": [
          {
            "modality": "MODALITY_UNSPECIFIED",
            "tokenCount": 0
          }
        ],
        "totalTokenCount": 0,
        "trafficType": "TRAFFIC_TYPE_UNSPECIFIED"
      },
      "invocationId": "",         // ID of the invocation
      "author": "string",         // Author of the event
      "actions": {                // Actions associated with the event
        "skipSummarization": true,
        "stateDelta": {
          "additionalProp1": {}
        },
        "artifactDelta": {
          "additionalProp1": 0,
          "additionalProp2": 0,
          "additionalProp3": 0
        },
        "transferToAgent": "string",
        "escalate": true,
        "requestedAuthConfigs": {
          "additionalProp1": {
            "authScheme": {
              "type": "apiKey",
              "description": "string",
              "in": "query",
              "name": "string",
              "additionalProp1": {}
            },
            "rawAuthCredential": {
              "authType": "apiKey",
              "resourceRef": "string",
              "apiKey": "string",
              "http": {
                "scheme": "string",
                "credentials": {
                  "username": "string",
                  "password": "string",
                  "token": "string",
                  "additionalProp1": {}
                },
                "additionalProp1": {}
              },
              "serviceAccount": {
                "serviceAccountCredential": {
                  "type": "",
                  "projectId": "string",
                  "privateKeyId": "string",
                  "privateKey": "string",
                  "clientEmail": "string",
                  "clientId": "string",
                  "authUri": "string",
                  "tokenUri": "string",
                  "authProviderX509CertUrl": "string",
                  "clientX509CertUrl": "string",
                  "universeDomain": "string",
                  "additionalProp1": {}
                },
                "scopes": [
                  "string"
                ],
                "useDefaultCredential": false,
                "additionalProp1": {}
              },
              "oauth2": {
                "clientId": "string",
                "clientSecret": "string",
                "authUri": "string",
                "state": "string",
                "redirectUri": "string",
                "authResponseUri": "string",
                "authCode": "string",
                "accessToken": "string",
                "refreshToken": "string",
                "additionalProp1": {}
              },
              "additionalProp1": {}
            },
            "exchangedAuthCredential": {
              "authType": "apiKey",
              "resourceRef": "string",
              "apiKey": "string",
              "http": {
                "scheme": "string",
                "credentials": {
                  "username": "string",
                  "password": "string",
                  "token": "string",
                  "additionalProp1": {}
                },
                "additionalProp1": {}
              },
              "serviceAccount": {
                "serviceAccountCredential": {
                  "type": "",
                  "projectId": "string",
                  "privateKeyId": "string",
                  "privateKey": "string",
                  "clientEmail": "string",
                  "clientId": "string",
                  "authUri": "string",
                  "tokenUri": "string",
                  "authProviderX509CertUrl": "string",
                  "clientX509CertUrl": "string",
                  "universeDomain": "string",
                  "additionalProp1": {}
                },
                "scopes": [
                  "string"
                ],
                "useDefaultCredential": false,
                "additionalProp1": {}
              },
              "oauth2": {
                "clientId": "string",
                "clientSecret": "string",
                "authUri": "string",
                "state": "string",
                "redirectUri": "string",
                "authResponseUri": "string",
                "authCode": "string",
                "accessToken": "string",
                "refreshToken": "string",
                "additionalProp1": {}
              },
              "additionalProp1": {}
            },
            "additionalProp1": {}
          }
        }
      },
      "longRunningToolIds": [     // IDs of long-running tools
        "string"
      ],
      "branch": "string",         // Branch information
      "id": "",                   // Event ID
      "timestamp": 0              // Event timestamp
    }
  ],
  "lastUpdateTime": 0             // Last update timestamp
}
``` 