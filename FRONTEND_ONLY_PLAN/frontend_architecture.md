# Frontend-Only Architecture for Open WebUI

This document describes the architecture for the frontend-only version of Open WebUI, where all functionality operates without a backend server.

## Architecture Overview

The frontend-only architecture uses a layered approach:

```
┌───────────────────────────────────────┐
│              UI Components            │
└───────────────┬───────────────────────┘
                │
┌───────────────▼───────────────────────┐
│            API Client Layer           │
│                                       │
│ ┌─────────────┐      ┌─────────────┐  │
│ │ Original API│      │  Local API  │  │
│ │  Endpoints  │─────▶│ Implementations│
│ └─────────────┘      └──────┬──────┘  │
└───────────────────────────┬───────────┘
                            │
┌───────────────────────────▼───────────┐
│           Storage Layer               │
│                                       │
│ ┌─────────────┐      ┌─────────────┐  │
│ │ Chat Storage│      │ Secure Key  │  │
│ │ (IndexedDB) │      │   Storage   │  │
│ └─────────────┘      └─────────────┘  │
└───────────────────────────┬───────────┘
                            │
┌───────────────────────────▼───────────┐
│          LLM Integration Layer        │
│                                       │
│ ┌─────────────┐      ┌─────────────┐  │
│ │OpenAI Client│      │ Anthropic   │  │
│ │             │      │   Client    │  │
│ └─────────────┘      └─────────────┘  │
└───────────────────────────────────────┘
```

## Key Components

### 1. API Client Layer

The API Client Layer serves as a proxy between the UI components and the data storage/LLM services. It uses a switchable architecture that can route requests either to the original backend API or to local implementations.

**Key Files:**
- `src/lib/api-client.js`: Main entry point for API requests
- `src/lib/local-api/*.js`: Local implementations of backend API endpoints

**Functionality:**
- Intercepts all API calls from the UI
- Routes calls to appropriate local handlers
- Maintains API compatibility with original backend
- Handles authentication locally

### 2. Storage Layer

The Storage Layer manages all data persistence in the browser, using IndexedDB for chats and a secure storage mechanism for API keys.

**Key Files:**
- `src/lib/storage/index.js`: Storage configuration and exports
- `src/lib/storage/chat-storage.js`: Chat data management
- `src/lib/storage/secure-storage.js`: Encrypted storage for sensitive data

**Functionality:**
- Stores and retrieves chat history
- Encrypts and securely stores API keys
- Manages user settings and preferences
- Provides import/export capabilities

### 3. LLM Integration Layer

The LLM Integration Layer handles direct communication with various language model providers, ensuring a consistent interface regardless of the underlying API differences.

**Key Files:**
- `src/lib/llm/index.js`: Common interface for LLM providers
- `src/lib/llm/openai.js`: OpenAI API integration
- `src/lib/llm/anthropic.js`: Anthropic API integration

**Functionality:**
- Creates provider-specific API clients
- Handles API key management
- Formats requests and responses
- Supports both streaming and non-streaming responses
- Converts between different API formats

## Data Flow

### Chat Flow

1. User sends a message
2. UI component calls the API Client
3. API Client routes to local chat implementation
4. Local chat handler:
   - Generates a unique message ID
   - Stores the message in IndexedDB
   - Prepares the response context
5. LLM client:
   - Retrieves API key from secure storage
   - Formats messages for the specific provider
   - Makes API call to the LLM service
   - Processes streaming or non-streaming response
6. Updates are sent back to the UI in real-time
7. Completed response is stored in IndexedDB

### Settings Flow

1. User updates settings
2. UI component calls the API Client
3. API Client routes to local settings implementation
4. Settings are stored in IndexedDB
5. UI is updated to reflect changes

### API Key Management Flow

1. User adds/updates API keys
2. UI component calls the secure storage directly
3. Keys are encrypted and stored in local storage
4. Confirmation is shown to the user

## Security Considerations

1. **API Key Security**
   - Keys are encrypted before storage
   - Encryption password can be user-provided
   - Keys are never sent to any server except the LLM provider

2. **Data Privacy**
   - All data stays in the browser
   - No tracking or analytics
   - Users can export/delete their data at any time

3. **Attack Surface Reduction**
   - No backend server to attack
   - No database to compromise
   - No user authentication system to breach

## Limitations

1. **No Multi-Device Sync**
   - Data is stored locally and not synced across devices
   - Users need to export/import data manually

2. **Browser Storage Limits**
   - IndexedDB has storage limits (typically 50-100MB)
   - Large chat histories may need pruning

3. **Limited Feature Set**
   - No knowledge base or document search
   - No collaborative features
   - No user management

4. **API Key Management**
   - Users must manage their own API keys
   - No rate limiting or usage tracking

## Extension Points

The architecture is designed to be extensible in several ways:

1. **Additional LLM Providers**
   - New providers can be added to the LLM Integration Layer
   - Common interface makes adding providers straightforward

2. **Enhanced Local Features**
   - Local search can be implemented using browser APIs
   - Simple knowledge base could be built with IndexedDB

3. **Progressive Web App Features**
   - Offline mode
   - Background syncing
   - Push notifications

4. **Integration with Cloud Storage**
   - Google Drive, Dropbox, or OneDrive for settings/chat backup
   - Encrypted backup options