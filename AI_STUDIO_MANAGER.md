# AI Studio Manager - Admin Panel

## Overview

The AI Studio Manager is an admin panel feature for managing AI Studio configurations, including prompt templates, AI model configurations, and usage monitoring.

## Features

### 1. Prompt Template Management
- **CRUD Operations**: Create, Read, Update, Delete prompt templates
- **Categories**: circuit_design, code_gen, debug, ideas, general
- **Variables Support**: Define template variables (e.g., `{{board_type}}`, `{{user_input}}`)
- **Active/Inactive Toggle**: Enable or disable templates

### 2. Model Configuration
- **Multiple Providers**: OpenAI, Anthropic, Google, Azure, Local
- **Model Settings**: 
  - Model ID (e.g., gpt-4, claude-3)
  - Max Tokens
  - Temperature
  - Priority (higher = preferred)
- **Default Model**: Set one model as default
- **API Configuration**: Custom endpoints and API key environment variable names

### 3. Usage Monitoring
- **Overview Stats**:
  - Total Requests
  - Tokens Used
  - Code Generations
  - Circuit Designs
  - Unique Users
  - Sessions & Messages
- **Per-User Statistics**: View each user's AI usage
- **Time Period Filter**: 7, 30, or 90 days

## API Endpoints

### Prompt Templates
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/admin/ai-studio/templates` | List all templates |
| GET | `/api/v1/admin/ai-studio/templates/:id` | Get single template |
| POST | `/api/v1/admin/ai-studio/templates` | Create template |
| PUT | `/api/v1/admin/ai-studio/templates/:id` | Update template |
| DELETE | `/api/v1/admin/ai-studio/templates/:id` | Delete template |

### Model Configurations
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/admin/ai-studio/models` | List all models |
| GET | `/api/v1/admin/ai-studio/models/active` | Get active models |
| GET | `/api/v1/admin/ai-studio/models/:id` | Get single model |
| POST | `/api/v1/admin/ai-studio/models` | Create model |
| PUT | `/api/v1/admin/ai-studio/models/:id` | Update model |
| DELETE | `/api/v1/admin/ai-studio/models/:id` | Delete model |

### Usage Monitoring
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/admin/ai-studio/usage/overview` | Get usage overview |
| GET | `/api/v1/admin/ai-studio/usage/users` | Get per-user stats |
| GET | `/api/v1/admin/ai-studio/usage/users/:userId` | Get user detail |

## Database Models

### AIPromptTemplate
```go
type AIPromptTemplate struct {
    ID          uuid.UUID
    Name        string
    Description string
    Category    string
    Prompt      string
    Variables   datatypes.JSON
    IsActive    bool
    CreatedAt   time.Time
    UpdatedAt   time.Time
}
```

### AIModelConfig
```go
type AIModelConfig struct {
    ID           uuid.UUID
    Name         string
    Provider     string      // openai, anthropic, google, local, azure
    ModelID      string      // gpt-4, claude-3, etc.
    Description  string
    APIEndpoint  *string
    APIKeyEnvVar string
    MaxTokens    int
    Temperature  float64
    Settings     datatypes.JSON
    IsActive     bool
    IsDefault    bool
    Priority     int
    CreatedAt    time.Time
    UpdatedAt    time.Time
}
```

## Access

- **URL**: `/admin/ai-studio`
- **Required Role**: Admin only
- **Menu Location**: Administrator → AI Studio Manager

## Files Created/Modified

### Backend
- `api/handlers/admin.ai_studio.handler.go` - Handler for admin AI Studio endpoints
- `dto/admin.ai_studio.dto.go` - DTOs for requests/responses
- `models/ai_studio.model.go` - Added `AIModelConfig` model
- `api/routes/routes.go` - Added admin AI Studio routes
- `database/db.go` - Added `AIModelConfig` to migrations

### Frontend
- `src/pages/Admin/AIStudio/index.tsx` - Admin page component
- `src/App.tsx` - Added route for `/admin/ai-studio`
- `src/layout/AppSidebar.tsx` - Added menu item

## Usage Example

1. Navigate to Admin Panel → AI Studio Manager
2. **Templates Tab**: Add/edit prompt templates for different use cases
3. **Models Tab**: Configure AI models with API keys and settings
4. **Usage Tab**: Monitor AI usage across users
