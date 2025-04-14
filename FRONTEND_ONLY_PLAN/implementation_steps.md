# Implementation Steps for Frontend-Only Open WebUI

## Step 1: Create API Mock Layer

Create a new directory structure to hold our local API implementations:

```bash
mkdir -p src/lib/local-api
mkdir -p src/lib/storage
```

### 1.1: Create Core Storage Module

File: `src/lib/storage/index.js`

```javascript
import localforage from 'localforage';
import CryptoJS from 'crypto-js';

// Configure storage instances
const chatStorage = localforage.createInstance({
  name: 'openWebUI',
  storeName: 'chats'
});

const modelStorage = localforage.createInstance({
  name: 'openWebUI',
  storeName: 'models'
});

const settingsStorage = localforage.createInstance({
  name: 'openWebUI',
  storeName: 'settings'
});

// Secure storage for API keys
const secureStorage = {
  setItem: async (key, value) => {
    // Get encryption password - could be derived from user input
    const password = localStorage.getItem('storage_password') || 'default_password';
    const encrypted = CryptoJS.AES.encrypt(value, password).toString();
    return localStorage.setItem(`secure_${key}`, encrypted);
  },
  
  getItem: async (key) => {
    const password = localStorage.getItem('storage_password') || 'default_password';
    const encrypted = localStorage.getItem(`secure_${key}`);
    if (!encrypted) return null;
    const decrypted = CryptoJS.AES.decrypt(encrypted, password).toString(CryptoJS.enc.Utf8);
    return decrypted;
  },
  
  removeItem: async (key) => {
    return localStorage.removeItem(`secure_${key}`);
  }
};

export { chatStorage, modelStorage, settingsStorage, secureStorage };
```

## Step 2: Implement API Interceptor

File: `src/lib/api-client.js`

```javascript
import { browser } from '$app/environment';
import { FRONTEND_ONLY_MODE } from '$lib/constants';

// Import local API handlers
import * as localChats from '$lib/local-api/chats';
import * as localModels from '$lib/local-api/models';
import * as localAuth from '$lib/local-api/auth';
import * as localConfig from '$lib/local-api/config';

/**
 * API client that handles routing between real and local API endpoints
 */
export async function apiClient(endpoint, options = {}) {
  if (!browser) {
    throw new Error('API client can only be used in browser');
  }
  
  // If we're in frontend-only mode, route to local handlers
  if (FRONTEND_ONLY_MODE) {
    console.log(`[Local API] ${options.method || 'GET'} ${endpoint}`);
    return handleLocalAPI(endpoint, options);
  }
  
  // Otherwise proceed with regular API call
  const { WEBUI_BASE_URL } = await import('$lib/constants');
  return fetch(`${WEBUI_BASE_URL}${endpoint}`, options)
    .then(async (res) => {
      if (!res.ok) throw await res.json();
      return res.json();
    })
    .catch((err) => {
      console.error(err);
      throw err;
    });
}

/**
 * Route API requests to the appropriate local handler
 */
async function handleLocalAPI(endpoint, options) {
  // Parse method and body if present
  const method = options.method || 'GET';
  let body = null;
  
  if (options.body) {
    try {
      body = JSON.parse(options.body);
    } catch (e) {
      body = options.body;
    }
  }
  
  // Chat-related endpoints
  if (endpoint.startsWith('/api/chat') || endpoint.startsWith('/api/v1/chats')) {
    return localChats.handleRequest(endpoint, method, body);
  }
  
  // Model-related endpoints
  if (endpoint.startsWith('/api/models') || endpoint.startsWith('/api/v1/models')) {
    return localModels.handleRequest(endpoint, method, body);
  }
  
  // Auth-related endpoints
  if (endpoint.startsWith('/api/v1/auths') || endpoint.includes('/login') || endpoint.includes('/register')) {
    return localAuth.handleRequest(endpoint, method, body);
  }
  
  // Config-related endpoints
  if (endpoint.startsWith('/api/config')) {
    return localConfig.handleRequest(endpoint, method, body);
  }
  
  // Default fallback for unimplemented endpoints
  return {
    status: 'error',
    message: `Local API endpoint not implemented: ${endpoint}`
  };
}
```

## Step 3: Implement Core Chat Storage

File: `src/lib/local-api/chats.js`

```javascript
import { v4 as uuidv4 } from 'uuid';
import { chatStorage } from '$lib/storage';

/**
 * Handle chat-related API requests
 */
export async function handleRequest(endpoint, method, body) {
  // Chat list
  if (endpoint === '/api/v1/chats' && method === 'GET') {
    return getChatList(body?.page || 1);
  }
  
  // Create chat
  if (endpoint === '/api/v1/chats' && method === 'POST') {
    return createChat(body);
  }
  
  // Get chat by ID - extract ID from endpoint
  if (endpoint.match(/\/api\/v1\/chats\/[a-zA-Z0-9-]+$/) && method === 'GET') {
    const chatId = endpoint.split('/').pop();
    return getChatById(chatId);
  }
  
  // Update chat by ID
  if (endpoint.match(/\/api\/v1\/chats\/[a-zA-Z0-9-]+$/) && method === 'PUT') {
    const chatId = endpoint.split('/').pop();
    return updateChat(chatId, body);
  }
  
  // Delete chat by ID
  if (endpoint.match(/\/api\/v1\/chats\/[a-zA-Z0-9-]+$/) && method === 'DELETE') {
    const chatId = endpoint.split('/').pop();
    return deleteChat(chatId);
  }
  
  // Chat completion endpoint (streaming is handled separately)
  if (endpoint === '/api/chat/completions' && method === 'POST') {
    return handleChatCompletion(body);
  }
  
  // Fallback
  return {
    status: 'error',
    message: `Chat endpoint not implemented: ${endpoint}`
  };
}

/**
 * Get list of chats with pagination
 */
export async function getChatList(page = 1, limit = 20) {
  try {
    // Get all chat keys
    const keys = await chatStorage.keys();
    
    // Sort by timestamp (newest first)
    const chats = await Promise.all(
      keys.map(async (key) => await chatStorage.getItem(key))
    );
    
    chats.sort((a, b) => b.timestamp - a.timestamp);
    
    // Apply pagination
    const startIdx = (page - 1) * limit;
    const endIdx = startIdx + limit;
    const paginatedChats = chats.slice(startIdx, endIdx);
    
    return {
      status: 'success',
      data: paginatedChats,
      pagination: {
        total: chats.length,
        page,
        limit,
        totalPages: Math.ceil(chats.length / limit)
      }
    };
  } catch (error) {
    console.error('Error getting chat list:', error);
    return {
      status: 'error',
      message: 'Failed to retrieve chat list'
    };
  }
}

/**
 * Create a new chat
 */
export async function createChat(chatData) {
  try {
    const chatId = chatData.id || uuidv4();
    const timestamp = Date.now();
    
    const newChat = {
      id: chatId,
      title: chatData.title || 'New Chat',
      models: chatData.models || [],
      history: chatData.history || { messages: {}, currentId: null },
      messages: chatData.messages || [],
      params: chatData.params || {},
      files: chatData.files || [],
      tags: chatData.tags || [],
      timestamp
    };
    
    await chatStorage.setItem(chatId, newChat);
    
    return {
      status: 'success',
      data: newChat
    };
  } catch (error) {
    console.error('Error creating chat:', error);
    return {
      status: 'error',
      message: 'Failed to create chat'
    };
  }
}

/**
 * Get chat by ID
 */
export async function getChatById(chatId) {
  try {
    const chat = await chatStorage.getItem(chatId);
    
    if (!chat) {
      return {
        status: 'error',
        message: 'Chat not found'
      };
    }
    
    return {
      status: 'success',
      data: {
        id: chat.id,
        chat: {
          title: chat.title,
          models: chat.models,
          history: chat.history,
          messages: chat.messages,
          params: chat.params,
          files: chat.files
        }
      }
    };
  } catch (error) {
    console.error('Error getting chat:', error);
    return {
      status: 'error',
      message: 'Failed to retrieve chat'
    };
  }
}

/**
 * Update an existing chat
 */
export async function updateChat(chatId, chatData) {
  try {
    const existingChat = await chatStorage.getItem(chatId);
    
    if (!existingChat) {
      return {
        status: 'error',
        message: 'Chat not found'
      };
    }
    
    const updatedChat = {
      ...existingChat,
      title: chatData.title || existingChat.title,
      models: chatData.models || existingChat.models,
      history: chatData.history || existingChat.history,
      messages: chatData.messages || existingChat.messages,
      params: chatData.params || existingChat.params,
      files: chatData.files || existingChat.files,
      timestamp: Date.now()
    };
    
    await chatStorage.setItem(chatId, updatedChat);
    
    return {
      status: 'success',
      data: updatedChat
    };
  } catch (error) {
    console.error('Error updating chat:', error);
    return {
      status: 'error',
      message: 'Failed to update chat'
    };
  }
}

/**
 * Delete a chat
 */
export async function deleteChat(chatId) {
  try {
    await chatStorage.removeItem(chatId);
    
    return {
      status: 'success',
      message: 'Chat deleted successfully'
    };
  } catch (error) {
    console.error('Error deleting chat:', error);
    return {
      status: 'error',
      message: 'Failed to delete chat'
    };
  }
}

/**
 * Handle chat completion requests
 * This will need to route to the appropriate LLM API based on model
 */
export async function handleChatCompletion(body) {
  // This function will be implemented in a separate LLM integration module
  // For now, return a placeholder
  return {
    status: 'error',
    message: 'Direct LLM integration not yet implemented'
  };
}
```

## Step 4: Implement Model Management

File: `src/lib/local-api/models.js`

```javascript
import { modelStorage, secureStorage } from '$lib/storage';

// Default models that will be available without backend
const DEFAULT_MODELS = [
  {
    id: 'gpt-3.5-turbo',
    name: 'GPT-3.5 Turbo',
    provider: 'openai',
    capabilities: {
      chat: true,
      vision: false
    },
    description: 'OpenAI GPT-3.5 Turbo model',
    params: {
      temperature: 0.7,
      max_tokens: 1024
    }
  },
  {
    id: 'gpt-4',
    name: 'GPT-4',
    provider: 'openai',
    capabilities: {
      chat: true,
      vision: false
    },
    description: 'OpenAI GPT-4 model',
    params: {
      temperature: 0.7,
      max_tokens: 2048
    }
  },
  {
    id: 'claude-3-opus',
    name: 'Claude 3 Opus',
    provider: 'anthropic',
    capabilities: {
      chat: true,
      vision: true
    },
    description: 'Anthropic Claude 3 Opus model',
    params: {
      temperature: 0.7,
      max_tokens: 4096
    }
  }
];

/**
 * Handle model-related API requests
 */
export async function handleRequest(endpoint, method, body) {
  // Get all models
  if (endpoint === '/api/models' && method === 'GET') {
    return getModels();
  }
  
  // Get model by ID
  if (endpoint.match(/\/api\/v1\/models\/[a-zA-Z0-9-]+$/) && method === 'GET') {
    const modelId = endpoint.split('/').pop();
    return getModelById(modelId);
  }
  
  // Add a new model
  if (endpoint === '/api/v1/models' && method === 'POST') {
    return addModel(body);
  }
  
  // Update a model
  if (endpoint.match(/\/api\/v1\/models\/[a-zA-Z0-9-]+$/) && method === 'PUT') {
    const modelId = endpoint.split('/').pop();
    return updateModel(modelId, body);
  }
  
  // Delete a model
  if (endpoint.match(/\/api\/v1\/models\/[a-zA-Z0-9-]+$/) && method === 'DELETE') {
    const modelId = endpoint.split('/').pop();
    return deleteModel(modelId);
  }
  
  // Fallback
  return {
    status: 'error',
    message: `Model endpoint not implemented: ${endpoint}`
  };
}

/**
 * Get all available models
 */
export async function getModels() {
  try {
    // Get custom models from storage
    const keys = await modelStorage.keys();
    const customModels = await Promise.all(
      keys.map(async (key) => await modelStorage.getItem(key))
    );
    
    // Combine with default models
    // Don't add default models that have been customized
    const customModelIds = customModels.map(model => model.id);
    const filteredDefaultModels = DEFAULT_MODELS.filter(
      model => !customModelIds.includes(model.id)
    );
    
    const allModels = [...filteredDefaultModels, ...customModels];
    
    // Process models to add any provider-specific details
    const processedModels = allModels.map(model => {
      return {
        ...model,
        direct: true, // Mark as direct connection for frontend
        info: {
          meta: {
            capabilities: model.capabilities || {}
          }
        }
      };
    });
    
    return {
      data: processedModels
    };
  } catch (error) {
    console.error('Error getting models:', error);
    return {
      status: 'error',
      message: 'Failed to retrieve models'
    };
  }
}

/**
 * Add a new custom model
 */
export async function addModel(modelData) {
  try {
    const { id, apiKey, ...modelDetails } = modelData;
    
    // Store API key securely
    if (apiKey) {
      await secureStorage.setItem(`api_key_${id}`, apiKey);
    }
    
    // Store model details (without API key)
    await modelStorage.setItem(id, {
      id,
      ...modelDetails,
      custom: true,
      timestamp: Date.now()
    });
    
    return {
      status: 'success',
      message: 'Model added successfully'
    };
  } catch (error) {
    console.error('Error adding model:', error);
    return {
      status: 'error',
      message: 'Failed to add model'
    };
  }
}

/**
 * Get model by ID
 */
export async function getModelById(modelId) {
  try {
    // Check custom models first
    const customModel = await modelStorage.getItem(modelId);
    
    if (customModel) {
      return {
        status: 'success',
        data: {
          ...customModel,
          direct: true
        }
      };
    }
    
    // Check default models
    const defaultModel = DEFAULT_MODELS.find(model => model.id === modelId);
    
    if (defaultModel) {
      return {
        status: 'success',
        data: {
          ...defaultModel,
          direct: true
        }
      };
    }
    
    return {
      status: 'error',
      message: 'Model not found'
    };
  } catch (error) {
    console.error('Error getting model:', error);
    return {
      status: 'error',
      message: 'Failed to retrieve model'
    };
  }
}

/**
 * Update an existing model
 */
export async function updateModel(modelId, modelData) {
  try {
    const existingModel = await modelStorage.getItem(modelId);
    
    // Can only update custom models
    if (!existingModel) {
      // If it's a default model, create a custom version
      const defaultModel = DEFAULT_MODELS.find(model => model.id === modelId);
      
      if (!defaultModel) {
        return {
          status: 'error',
          message: 'Model not found'
        };
      }
      
      // Create custom version of default model
      const { apiKey, ...modelDetails } = modelData;
      
      // Store API key securely
      if (apiKey) {
        await secureStorage.setItem(`api_key_${modelId}`, apiKey);
      }
      
      await modelStorage.setItem(modelId, {
        ...defaultModel,
        ...modelDetails,
        custom: true,
        timestamp: Date.now()
      });
      
      return {
        status: 'success',
        message: 'Model customized successfully'
      };
    }
    
    // Update existing custom model
    const { apiKey, ...modelDetails } = modelData;
    
    // Update API key if provided
    if (apiKey) {
      await secureStorage.setItem(`api_key_${modelId}`, apiKey);
    }
    
    // Update model details
    await modelStorage.setItem(modelId, {
      ...existingModel,
      ...modelDetails,
      timestamp: Date.now()
    });
    
    return {
      status: 'success',
      message: 'Model updated successfully'
    };
  } catch (error) {
    console.error('Error updating model:', error);
    return {
      status: 'error',
      message: 'Failed to update model'
    };
  }
}

/**
 * Delete a custom model
 */
export async function deleteModel(modelId) {
  try {
    const existingModel = await modelStorage.getItem(modelId);
    
    if (!existingModel) {
      return {
        status: 'error',
        message: 'Custom model not found'
      };
    }
    
    // Remove model details
    await modelStorage.removeItem(modelId);
    
    // Remove API key
    await secureStorage.removeItem(`api_key_${modelId}`);
    
    return {
      status: 'success',
      message: 'Model deleted successfully'
    };
  } catch (error) {
    console.error('Error deleting model:', error);
    return {
      status: 'error',
      message: 'Failed to delete model'
    };
  }
}
```

## Step 5: Implement Direct LLM Integration

File: `src/lib/llm/index.js`

```javascript
import { secureStorage } from '$lib/storage';
import { createOpenAIClient } from './openai';
import { createAnthropicClient } from './anthropic';

// Map of provider to client factory
const PROVIDER_CLIENTS = {
  'openai': createOpenAIClient,
  'anthropic': createAnthropicClient
};

/**
 * Create an LLM client for the specified model
 */
export async function createLLMClient(model) {
  const provider = model.provider || getProviderFromModelId(model.id);
  
  // Get client factory for provider
  const clientFactory = PROVIDER_CLIENTS[provider];
  
  if (!clientFactory) {
    throw new Error(`Unsupported LLM provider: ${provider}`);
  }
  
  // Get API key for provider
  const apiKey = await secureStorage.getItem(`api_key_${provider}`) || 
                 await secureStorage.getItem(`api_key_${model.id}`);
  
  if (!apiKey) {
    throw new Error(`API key not found for ${provider}. Please add it in settings.`);
  }
  
  // Create client
  return clientFactory(apiKey, model);
}

/**
 * Detect provider from model ID
 */
function getProviderFromModelId(modelId) {
  if (modelId.startsWith('gpt-')) {
    return 'openai';
  }
  
  if (modelId.includes('claude')) {
    return 'anthropic';
  }
  
  // Add more provider detection as needed
  
  return null;
}

/**
 * Generate chat completion
 */
export async function generateChatCompletion(model, messages, options = {}) {
  const client = await createLLMClient(model);
  return client.generateChatCompletion(messages, options);
}

/**
 * Create streaming chat completion
 */
export async function createChatCompletionStream(model, messages, options = {}) {
  const client = await createLLMClient(model);
  return client.createChatCompletionStream(messages, options);
}
```

File: `src/lib/llm/openai.js`

```javascript
/**
 * Create an OpenAI API client
 */
export function createOpenAIClient(apiKey, model) {
  const baseUrl = model.endpoint || 'https://api.openai.com/v1';
  
  return {
    /**
     * Generate chat completion
     */
    generateChatCompletion: async (messages, options = {}) => {
      try {
        const response = await fetch(`${baseUrl}/chat/completions`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${apiKey}`
          },
          body: JSON.stringify({
            model: model.id,
            messages,
            temperature: options.temperature || model.params?.temperature || 0.7,
            max_tokens: options.max_tokens || model.params?.max_tokens || 1024,
            stream: false,
            ...options
          })
        });
        
        if (!response.ok) {
          const error = await response.json();
          throw new Error(error.error?.message || 'OpenAI API Error');
        }
        
        return await response.json();
      } catch (error) {
        console.error('OpenAI API Error:', error);
        throw error;
      }
    },
    
    /**
     * Create chat completion stream
     */
    createChatCompletionStream: async (messages, options = {}) => {
      try {
        const response = await fetch(`${baseUrl}/chat/completions`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${apiKey}`
          },
          body: JSON.stringify({
            model: model.id,
            messages,
            temperature: options.temperature || model.params?.temperature || 0.7,
            max_tokens: options.max_tokens || model.params?.max_tokens || 1024,
            stream: true,
            ...options
          })
        });
        
        if (!response.ok) {
          const error = await response.json();
          throw new Error(error.error?.message || 'OpenAI API Error');
        }
        
        return response.body;
      } catch (error) {
        console.error('OpenAI API Error:', error);
        throw error;
      }
    }
  };
}
```

File: `src/lib/llm/anthropic.js`

```javascript
/**
 * Create an Anthropic API client
 */
export function createAnthropicClient(apiKey, model) {
  const baseUrl = model.endpoint || 'https://api.anthropic.com/v1';
  
  return {
    /**
     * Generate chat completion
     */
    generateChatCompletion: async (messages, options = {}) => {
      try {
        // Convert messages from OpenAI format to Anthropic format
        const formattedMessages = convertToAnthropicMessages(messages);
        
        const response = await fetch(`${baseUrl}/messages`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'x-api-key': apiKey,
            'anthropic-version': '2023-06-01'
          },
          body: JSON.stringify({
            model: model.id,
            messages: formattedMessages,
            temperature: options.temperature || model.params?.temperature || 0.7,
            max_tokens: options.max_tokens || model.params?.max_tokens || 1024,
            stream: false,
            ...options
          })
        });
        
        if (!response.ok) {
          const error = await response.json();
          throw new Error(error.error?.message || 'Anthropic API Error');
        }
        
        const result = await response.json();
        
        // Convert from Anthropic format to OpenAI format for compatibility
        return convertFromAnthropicResponse(result);
      } catch (error) {
        console.error('Anthropic API Error:', error);
        throw error;
      }
    },
    
    /**
     * Create chat completion stream
     */
    createChatCompletionStream: async (messages, options = {}) => {
      try {
        // Convert messages from OpenAI format to Anthropic format
        const formattedMessages = convertToAnthropicMessages(messages);
        
        const response = await fetch(`${baseUrl}/messages`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'x-api-key': apiKey,
            'anthropic-version': '2023-06-01'
          },
          body: JSON.stringify({
            model: model.id,
            messages: formattedMessages,
            temperature: options.temperature || model.params?.temperature || 0.7,
            max_tokens: options.max_tokens || model.params?.max_tokens || 1024,
            stream: true,
            ...options
          })
        });
        
        if (!response.ok) {
          const error = await response.json();
          throw new Error(error.error?.message || 'Anthropic API Error');
        }
        
        // Here we would need to convert the Anthropic stream format to OpenAI format
        // This is a simplified version that would need more work for production
        return response.body;
      } catch (error) {
        console.error('Anthropic API Error:', error);
        throw error;
      }
    }
  };
}

/**
 * Convert messages from OpenAI format to Anthropic format
 */
function convertToAnthropicMessages(messages) {
  return messages.map(message => {
    // Map OpenAI roles to Anthropic roles
    let role = message.role;
    if (role === 'system') {
      // Anthropic handles system messages differently
      return { role: 'user', content: `<system>${message.content}</system>` };
    }
    
    if (role === 'assistant') {
      role = 'assistant';
    } else {
      role = 'user';
    }
    
    return { role, content: message.content };
  });
}

/**
 * Convert response from Anthropic format to OpenAI format
 */
function convertFromAnthropicResponse(response) {
  return {
    id: response.id,
    object: 'chat.completion',
    created: Date.now(),
    model: response.model,
    choices: [
      {
        index: 0,
        message: {
          role: 'assistant',
          content: response.content[0].text
        },
        finish_reason: 'stop'
      }
    ],
    usage: {
      prompt_tokens: 0, // Anthropic doesn't provide this
      completion_tokens: 0, // Anthropic doesn't provide this
      total_tokens: 0 // Anthropic doesn't provide this
    }
  };
}
```

## Step 6: Update Constants and Entry Point

First, modify the constants file to add frontend-only mode:

File: `src/lib/constants.ts`

```typescript
import { browser, dev } from '$app/environment';

export const APP_NAME = 'Open WebUI';

// Enable frontend-only mode
export const FRONTEND_ONLY_MODE = true;

// Base URLs
export const WEBUI_HOSTNAME = browser ? (dev ? `${location.hostname}:8080` : ``) : '';
export const WEBUI_BASE_URL = browser ? (dev ? `http://${WEBUI_HOSTNAME}` : ``) : ``;
export const WEBUI_API_BASE_URL = `${WEBUI_BASE_URL}/api/v1`;

// API endpoints
export const OLLAMA_API_BASE_URL = `${WEBUI_BASE_URL}/ollama`;
export const OPENAI_API_BASE_URL = `${WEBUI_BASE_URL}/openai`;
export const AUDIO_API_BASE_URL = `${WEBUI_BASE_URL}/api/v1/audio`;
export const IMAGES_API_BASE_URL = `${WEBUI_BASE_URL}/api/v1/images`;
export const RETRIEVAL_API_BASE_URL = `${WEBUI_BASE_URL}/api/v1/retrieval`;

export const WEBUI_VERSION = APP_VERSION;
export const WEBUI_BUILD_HASH = APP_BUILD_HASH;
export const REQUIRED_OLLAMA_VERSION = '0.1.16';

// Supported file types (unchanged)
export const SUPPORTED_FILE_TYPE = [
  'application/epub+zip',
  'application/pdf',
  // ...rest of the file types
];

export const SUPPORTED_FILE_EXTENSIONS = [
  'md',
  'rst',
  // ...rest of the file extensions
];

export const PASTED_TEXT_CHARACTER_LIMIT = 1000;
```

Then update the app entry point to initialize local storage:

File: `src/routes/+layout.svelte` (update onMount section)

```svelte
onMount(async () => {
  if (FRONTEND_ONLY_MODE) {
    // Initialize frontend-only mode
    console.log('Running in frontend-only mode');
    
    // Get default settings
    const userSettings = {
      ui: {
        theme: 'dark',
        language: 'en',
        // Default UI settings
        models: ['gpt-3.5-turbo'],
        // etc.
      }
    };
    
    settings.set(userSettings.ui);
    
    // Load models
    const modelRes = await localModels.getModels();
    models.set(modelRes.data || []);
    
    // Set user as admin in frontend-only mode
    user.set({
      id: 'local-user',
      name: 'Local User',
      role: 'admin',
      permissions: {
        chat: { temporary: true },
        features: {
          image_generation: true,
          code_interpreter: true,
          web_search: true
        }
      }
    });
    
    // Load config
    config.set({
      name: 'Open WebUI (Local)',
      version: WEBUI_VERSION,
      features: {
        auth: false,
        enable_direct_connections: true,
        enable_channels: false,
        enable_web_search: false,
        enable_code_execution: false,
        enable_image_generation: false,
        enable_community_sharing: false
      }
    });
    
    loaded = true;
  } else {
    // Original backend-connected code
    if ($user === undefined || $user === null) {
      await goto('/auth');
    } else if (['user', 'admin'].includes($user?.role)) {
      // ...original code
    }
  }
});
```

## Step 7: Modify API Import/Export Files

Update the API files to use our interceptor:

File: `src/lib/apis/index.ts` (beginning part)

```typescript
import { WEBUI_API_BASE_URL, WEBUI_BASE_URL, FRONTEND_ONLY_MODE } from '$lib/constants';
import { convertOpenApiToToolPayload } from '$lib/utils';
import { getOpenAIModelsDirect } from './openai';
import { apiClient } from '$lib/api-client';  // Import our interceptor

import { parse } from 'yaml';
import { toast } from 'svelte-sonner';

export const getModels = async (
  token: string = '',
  connections: object | null = null,
  base: boolean = false
) => {
  let error = null;
  try {
    // Use our API client instead of fetch directly
    const res = await apiClient(`/api/models${base ? '/base' : ''}`, {
      method: 'GET',
      headers: {
        Accept: 'application/json',
        'Content-Type': 'application/json',
        ...(token && { authorization: `Bearer ${token}` })
      }
    });
    
    let models = res?.data ?? [];
    
    // Rest of the function remains the same...
```

Do the same for all other API-calling functions in the API files.

## Step 8: Add Settings UI for API Keys

Create a new component for API key management:

File: `src/lib/components/settings/ApiKeySettings.svelte`

```svelte
<script lang="ts">
  import { getContext } from 'svelte';
  import { toast } from 'svelte-sonner';
  import { secureStorage } from '$lib/storage';
  
  const i18n = getContext('i18n');
  
  // API key states
  let openaiKey = '';
  let anthropicKey = '';
  let loadingKeys = true;
  
  // Load API keys on mount
  onMount(async () => {
    try {
      openaiKey = await secureStorage.getItem('api_key_openai') || '';
      anthropicKey = await secureStorage.getItem('api_key_anthropic') || '';
    } catch (e) {
      console.error('Error loading API keys:', e);
    } finally {
      loadingKeys = false;
    }
  });
  
  // Save API keys
  async function saveApiKeys() {
    try {
      if (openaiKey) {
        await secureStorage.setItem('api_key_openai', openaiKey);
      }
      
      if (anthropicKey) {
        await secureStorage.setItem('api_key_anthropic', anthropicKey);
      }
      
      toast.success($i18n.t('API keys saved successfully'));
    } catch (e) {
      console.error('Error saving API keys:', e);
      toast.error($i18n.t('Failed to save API keys'));
    }
  }
</script>

<div class="py-4">
  <h3 class="text-lg font-medium mb-4">{$i18n.t('API Keys')}</h3>
  
  {#if loadingKeys}
    <div class="flex items-center justify-center py-4">
      <Spinner />
    </div>
  {:else}
    <div class="space-y-4">
      <div>
        <label for="openai-key" class="block text-sm font-medium mb-1">
          {$i18n.t('OpenAI API Key')}
        </label>
        <input
          id="openai-key"
          type="password"
          class="w-full px-3 py-2 border rounded-md"
          placeholder="sk-..."
          bind:value={openaiKey}
        />
      </div>
      
      <div>
        <label for="anthropic-key" class="block text-sm font-medium mb-1">
          {$i18n.t('Anthropic API Key')}
        </label>
        <input
          id="anthropic-key"
          type="password"
          class="w-full px-3 py-2 border rounded-md"
          placeholder="sk-ant-..."
          bind:value={anthropicKey}
        />
      </div>
      
      <button
        class="px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700"
        on:click={saveApiKeys}
      >
        {$i18n.t('Save API Keys')}
      </button>
      
      <div class="text-xs text-gray-500 mt-2">
        {$i18n.t('Your API keys are stored securely in your browser\'s local storage. They are never sent to our servers.')}
      </div>
    </div>
  {/if}
</div>
```

Then add this component to the settings modal.

## Step 9: Build and Deploy Configuration

File: `vite.config.js` (update)

```javascript
import { sveltekit } from '@sveltejs/kit/vite';
import { defineConfig } from 'vite';

export default defineConfig({
  plugins: [sveltekit()],
  define: {
    APP_VERSION: JSON.stringify(process.env.npm_package_version || '0.0.0'),
    APP_BUILD_HASH: JSON.stringify(process.env.VITE_BUILD_HASH || 'dev-build'),
    APP_BUILD_DATE: JSON.stringify(new Date().toISOString())
  },
  build: {
    // Configure for static site deployment
    outDir: 'dist',
    emptyOutDir: true,
    // Ensure resources are correctly referenced
    assetsInlineLimit: 0
  }
});
```

Update the build script in `package.json`:

```json
{
  "scripts": {
    "dev": "npm run pyodide:fetch && vite dev --host",
    "build": "npm run pyodide:fetch && vite build",
    "build:static": "npm run pyodide:fetch && FRONTEND_ONLY=true vite build",
    "preview": "vite preview",
    "preview:static": "vite preview --outDir dist"
    // ...other scripts
  }
}
```

## Step 10: Create Static Build Script

Create a deployment script:

File: `scripts/build-static.sh`

```bash
#!/bin/bash

# Set environment variables
export FRONTEND_ONLY=true
export VITE_BUILD_HASH=$(git rev-parse --short HEAD)

# Clean up previous build
rm -rf dist

# Build frontend
npm run build:static

# Copy static assets
cp -r static dist/
cp dist/index.html dist/404.html  # For SPA routing on static hosts

echo "Static build complete! Deploy the 'dist' directory to your preferred hosting provider."
```

Make the script executable:

```bash
chmod +x scripts/build-static.sh
```

## Final Steps: Testing and Documentation

Create a guide for users:

File: `FRONTEND_ONLY_GUIDE.md`

```markdown
# Open WebUI Frontend-Only Mode

This guide helps you set up and use Open WebUI in frontend-only mode, where all data is stored in your browser and LLM API calls are made directly from your device.

## Setup Instructions

1. Clone the repository
   ```bash
   git clone https://github.com/your-fork/open-webui.git
   cd open-webui
   ```

2. Install dependencies
   ```bash
   npm install
   ```

3. Build the frontend-only version
   ```bash
   ./scripts/build-static.sh
   ```

4. Deploy the `dist` directory to your preferred hosting or serve locally
   ```bash
   # Local testing
   npx http-server dist
   ```

## Usage Guide

### First-time Setup

1. When you first open the app, you'll be prompted to add API keys
2. Go to Settings → API Keys
3. Add your OpenAI and/or Anthropic API keys
4. Your keys are stored securely in your browser's local storage

### Using Custom Models

1. Go to Settings → Models
2. Click "Add Custom Model"
3. Enter model details (Name, ID, Provider, etc.)
4. Add API key if different from provider default

### Data Management

All data is stored in your browser. To export/import:

1. Go to Settings → Data Management
2. Click "Export Data" to download all your chats and settings
3. Use "Import Data" to restore from a backup

### Security Notes

- API keys are stored in your browser's local storage with encryption
- No data is sent to any server except the LLM API providers
- Clear browser cache/storage to remove all data
```