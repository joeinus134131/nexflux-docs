# NexFlux Frontend API Requirements

Dokumentasi lengkap API endpoints yang dibutuhkan oleh frontend NexFlux untuk integrasi dengan backend.

**Base URL:** `VITE_API_URL` (default: `http://localhost:8080/api/v1`)

**Authentication:** Semua endpoint (kecuali Auth) memerlukan header:
```
Authorization: Bearer <token>
```

---

## 📋 Table of Contents

1. [Authentication](#1-authentication)
2. [Users](#2-users)
3. [Projects](#3-projects)
4. [Circuits](#4-circuits)
5. [Components](#5-components)
6. [Achievements & Gamification](#6-achievements--gamification)
7. [Remote Lab](#7-remote-lab)
8. [Notifications](#8-notifications)
9. [Documentation](#9-documentation)
10. [Challenges](#10-challenges)
11. [File Upload](#11-file-upload)
12. [Leaderboard](#12-leaderboard)

---

## 1. Authentication

### POST `/auth/register`
Register new user.

**Request Body:**
```json
{
  "name": "string",
  "email": "string",
  "password": "string"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "string",
      "name": "string",
      "email": "string"
    },
    "token": "string"
  }
}
```

### POST `/auth/login`
User login.

**Request Body:**
```json
{
  "email": "string",
  "password": "string"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "string",
      "name": "string",
      "email": "string",
      "avatar_url": "string"
    },
    "token": "string"
  }
}
```

### POST `/auth/logout`
Logout user.

---

## 2. Users

### GET `/users/me`
Get current user profile. **CRITICAL - Used across many components.**

**Response:**
```json
{
  "data": {
    "id": "string",
    "name": "string",
    "username": "string",
    "email": "string",
    "avatar_url": "string",
    "level": 1,
    "total_xp": 200,
    "current_xp": 200,
    "target_xp": 500,
    "streak_days": 5,
    "projects_count": 3,
    "challenges_completed": 2,
    "is_pro": false,
    "subscription": {
      "plan": "free | premium"
    },
    "created_at": "2024-12-26T00:00:00Z",
    "updated_at": "2024-12-26T00:00:00Z"
  }
}
```

### PUT `/users/me`
Update current user profile.

**Request Body (form-data supported for avatar):**
```json
{
  "name": "string",
  "username": "string",
  "email": "string",
  "bio": "string",
  "avatar_url": "string",
  "address": "string",
  "phone": "string"
}
```

### GET `/users/me/stats`
Get detailed user statistics.

**Response:**
```json
{
  "data": {
    "level": 1,
    "total_xp": 200,
    "current_xp": 200,
    "target_xp": 500,
    "streak_days": 5,
    "projects_count": 3,
    "challenges_completed": 2,
    "achievements": {
      "unlocked": 5,
      "total": 20
    }
  }
}
```

### GET `/streak`
Get user streak data.

**Response:**
```json
{
  "data": {
    "current_streak": 5,
    "longest_streak": 10,
    "last_activity_date": "2024-12-26"
  }
}
```

---

## 3. Projects

### GET `/projects`
List user projects.

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | number | Max results |
| `page` | number | Page number |
| `is_public` | boolean | Filter public projects |
| `search` | string | Search by name |

**Response:**
```json
{
  "data": [
    {
      "id": "string",
      "name": "string",
      "description": "string",
      "thumbnail_url": "string",
      "difficulty": "Beginner | Intermediate | Advanced",
      "progress": 75,
      "xp_reward": 100,
      "is_public": false,
      "is_favorite": false,
      "components_count": 10,
      "collaborators_count": 2,
      "created_at": "2024-12-26T00:00:00Z",
      "updated_at": "2024-12-26T00:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 50
  }
}
```

### POST `/projects`
Create new project.

**Request Body:**
```json
{
  "name": "string",
  "description": "string",
  "difficulty": "Beginner | Intermediate | Advanced",
  "is_public": false,
  "hardware_platform": "string",
  "tags": ["string"]
}
```

### GET `/projects/:id`
Get project details.

### PUT `/projects/:id`
Update project.

### DELETE `/projects/:id`
Delete project.

### POST `/projects/:id/duplicate`
Duplicate a project.

**Response:**
```json
{
  "data": {
    "id": "new-project-id",
    "name": "Project Name (Copy)"
  }
}
```

### PUT `/projects/:id/favorite`
Toggle project favorite status.

### GET `/projects/:id/collaborators`
Get project collaborators.

**Response:**
```json
{
  "data": [
    {
      "id": "string",
      "user_id": "string",
      "name": "string",
      "email": "string",
      "avatar_url": "string",
      "role": "owner | editor | viewer"
    }
  ]
}
```

### POST `/projects/:id/collaborators`
Add collaborator to project.

**Request Body:**
```json
{
  "email": "string",
  "role": "editor | viewer"
}
```

### DELETE `/projects/:id/collaborators/:userId`
Remove collaborator.

### PATCH `/projects/:id/collaborators/:userId`
Update collaborator role.

**Request Body:**
```json
{
  "role": "editor | viewer"
}
```

---

## 4. Circuits

### GET `/circuits`
List saved circuits.

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `project_id` | string | Filter by project |
| `status` | string | Filter by status (draft, running, completed, paused, error) |
| `search` | string | Search by name |

**Response:**
```json
{
  "data": [
    {
      "id": "string",
      "name": "string",
      "description": "string",
      "type": "string",
      "status": "draft | running | completed | paused | error",
      "thumbnail_url": "string",
      "components_count": 10,
      "wires_count": 15,
      "run_count": 5,
      "total_runtime_ms": 1500,
      "last_run_at": "2024-12-26T00:00:00Z",
      "project_id": "string",
      "project_name": "string",
      "schema_data": {
        "components": [],
        "wires": [],
        "groundNodeId": "string",
        "powerSourceIds": []
      },
      "created_at": "2024-12-26T00:00:00Z",
      "updated_at": "2024-12-26T00:00:00Z"
    }
  ]
}
```

### POST `/circuits`
Create new circuit.

**Request Body:**
```json
{
  "name": "string",
  "description": "string",
  "project_id": "string (optional)",
  "schema_data": {
    "components": [
      {
        "id": "string",
        "type": "power_source | ground | resistor | led | capacitor | switch | button | ...",
        "name": "string",
        "position": { "x": 100, "y": 200 },
        "rotation": 0,
        "properties": {
          "voltage": 5,
          "resistance": 1000
        }
      }
    ],
    "wires": [
      {
        "id": "string",
        "startComponentId": "string",
        "startPinId": "string",
        "endComponentId": "string",
        "endPinId": "string",
        "resistance": 0.01
      }
    ],
    "groundNodeId": "string",
    "powerSourceIds": ["string"]
  }
}
```

### GET `/circuits/:id`
Get circuit details.

### PUT `/circuits/:id`
Update circuit.

### DELETE `/circuits/:id`
Delete circuit.

### GET `/circuits/stats`
Get circuit statistics.

**Response:**
```json
{
  "data": {
    "total_circuits": 10,
    "running_now": 1,
    "completed": 5,
    "paused": 2,
    "error": 0,
    "total_runtime_hours": 48.5,
    "success_rate": 94.2,
    "circuits_this_week": 3
  }
}
```

---

## 5. Components

### GET `/components`
List electronic components.

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `category` | string | Filter by category |
| `search` | string | Search by name |
| `page` | number | Page number |
| `limit` | number | Results per page |

**Response:**
```json
{
  "data": [
    {
      "id": "string",
      "name": "string",
      "description": "string",
      "category": "string",
      "image_url": "string",
      "datasheet_url": "string",
      "package_type": "string",
      "is_favorite": false,
      "specifications": {
        "voltage_rating": "5V",
        "power_rating": "0.25W"
      }
    }
  ]
}
```

### GET `/components/categories`
Get component categories.

**Response:**
```json
{
  "data": [
    {
      "id": "string",
      "name": "Resistors",
      "slug": "resistors",
      "icon": "string",
      "count": 150
    }
  ]
}
```

### POST `/components/:id/favorite`
Toggle component favorite.

---

## 6. Achievements & Gamification

### GET `/achievements`
Get all available achievements.

**Response:**
```json
{
  "data": [
    {
      "id": "string",
      "name": "First Circuit",
      "description": "Create your first circuit",
      "icon": "trophy | medal | crown | star | fire | bolt | rocket | gem",
      "rarity": "common | rare | epic | legendary",
      "requirement_type": "xp | level | streak | projects | challenges",
      "requirement_value": 100,
      "xp_reward": 50,
      "is_active": true
    }
  ]
}
```

### GET `/achievements/user`
Get user's unlocked achievements.

**Response:**
```json
{
  "data": [
    {
      "id": "string",
      "achievement_id": "string",
      "unlocked_at": "2024-12-26T00:00:00Z"
    }
  ]
}
```

---

## 7. Remote Lab

### GET `/labs`
List available labs.

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | string | Filter by status (available, busy, maintenance) |
| `difficulty` | string | Filter by difficulty |
| `platform` | string | Filter by hardware platform |

**Response:**
```json
{
  "data": [
    {
      "id": "string",
      "name": "Arduino Uno Lab",
      "description": "string",
      "hardware_platform": "arduino_uno",
      "difficulty": "Beginner",
      "status": "available | busy | maintenance",
      "thumbnail_url": "string",
      "current_queue": 3,
      "avg_session_minutes": 30,
      "requires_code_upload": true
    }
  ]
}
```

### GET `/labs/:id`
Get lab details.

### POST `/labs/:id/queue`
Join lab queue.

### POST `/labs/:id/session/start`
Start lab session.

**Response:**
```json
{
  "data": {
    "session_id": "string",
    "expires_at": "2024-12-26T00:30:00Z",
    "livekit_token": "string",
    "livekit_url": "wss://...",
    "room_name": "string"
  }
}
```

### POST `/labs/:id/session/end`
End lab session.

### POST `/labs/:id/code/submit`
Submit code to lab.

**Request Body:**
```json
{
  "code": "void setup() { ... }",
  "language": "cpp | python"
}
```

**Response:**
```json
{
  "data": {
    "compilation_status": "success | error",
    "output": "string",
    "errors": [],
    "warnings": []
  }
}
```

### POST `/labs/:id/actuators/control`
Control lab actuators.

**Request Body:**
```json
{
  "actuator_id": "string",
  "action": "on | off | set",
  "value": 255
}
```

---

## 8. Notifications

### GET `/notifications`
Get user notifications.

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `user_id` | string | User ID (optional) |

**Response:**
```json
{
  "data": [
    {
      "id": "string",
      "type": "achievement | xp | project | system | social | challenge",
      "title": "string",
      "message": "string",
      "is_read": false,
      "created_at": "2024-12-26T00:00:00Z"
    }
  ]
}
```

### PUT `/notifications/:id/read`
Mark notification as read.

### PUT `/notifications/read-all`
Mark all notifications as read.

### DELETE `/notifications/:id`
Delete notification.

---

## 9. Documentation

### GET `/docs/categories/:slug`
Get documentation category.

**Response:**
```json
{
  "data": {
    "id": "string",
    "name": "Getting Started",
    "slug": "getting-started",
    "description": "string",
    "articles": [
      {
        "id": "string",
        "title": "Quick Start Guide",
        "slug": "quick-start",
        "excerpt": "string",
        "read_time_minutes": 5
      }
    ]
  }
}
```

### GET `/docs/articles/:slug`
Get documentation article.

**Response:**
```json
{
  "data": {
    "id": "string",
    "title": "string",
    "slug": "string",
    "content": "string (markdown)",
    "category": {
      "id": "string",
      "name": "string",
      "slug": "string"
    },
    "read_time_minutes": 5,
    "view_count": 1234
  }
}
```

### POST `/docs/articles/:slug/view`
Track article view.

---

## 10. Challenges

### GET `/challenges`
Get all challenges.

**Response:**
```json
{
  "data": {
    "challenges": [
      {
        "id": "string",
        "title": "string",
        "description": "string",
        "type": "daily | weekly | special | milestone",
        "difficulty": "Easy | Medium | Hard",
        "xp_reward": 100,
        "status": "locked | available | in_progress | completed",
        "progress": 50,
        "deadline": "2024-12-26T23:59:59Z"
      }
    ],
    "stats": {
      "current_streak": 5,
      "challenges_completed": 10,
      "xp_this_week": 500
    }
  }
}
```

### GET `/challenges/daily`
Get daily challenge.

### GET `/challenges/progress`
Get challenge progress.

### POST `/challenges/:id/start`
Start a challenge.

---

## 11. File Upload

### POST `/upload`
Upload file.

**Request Body (multipart/form-data):**
| Field | Type | Description |
|-------|------|-------------|
| `file` | File | File to upload |
| `type` | string | Type: avatar, project, component |

**Response:**
```json
{
  "data": {
    "url": "https://storage.nexflux.io/uploads/...",
    "filename": "string",
    "size": 1024,
    "mime_type": "image/png"
  }
}
```

### DELETE `/upload`
Delete uploaded file.

**Request Body:**
```json
{
  "url": "https://storage.nexflux.io/uploads/..."
}
```

---

## 12. Leaderboard

### GET `/leaderboard`
Get leaderboard.

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | weekly, monthly, all_time |
| `limit` | number | Max entries (default 10) |

**Response:**
```json
{
  "data": {
    "type": "weekly",
    "period": {
      "start": "2024-12-23T00:00:00Z",
      "end": "2024-12-29T23:59:59Z"
    },
    "entries": [
      {
        "rank": 1,
        "user": {
          "id": "string",
          "name": "string",
          "avatar_url": "string"
        },
        "xp": 5000,
        "is_current_user": false
      }
    ],
    "current_user_rank": 15
  }
}
```

---

## 📌 Response Format Standard

All API responses should follow this format:

### Success Response
```json
{
  "success": true,
  "data": { ... },
  "message": "Optional success message"
}
```

### Error Response
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable error message"
  }
}
```

### Pagination Response
```json
{
  "success": true,
  "data": [ ... ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 100,
    "total_pages": 10
  }
}
```

---

## 🔐 HTTP Status Codes

| Code | Description |
|------|-------------|
| 200 | Success |
| 201 | Created |
| 400 | Bad Request |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not Found |
| 422 | Validation Error |
| 500 | Internal Server Error |

---

## 📊 Priority Levels

### Critical (Must Have)
- ✅ `GET /users/me` - Used everywhere
- ✅ `GET /streak` - Gamification
- ✅ `POST /auth/login`, `POST /auth/register`
- ✅ `GET /projects`, `POST /projects`, `PUT /projects/:id`, `DELETE /projects/:id`
- ✅ `GET /circuits`, `POST /circuits`, `PUT /circuits/:id`, `DELETE /circuits/:id`

### High Priority
- ⭐ `GET /achievements`, `GET /achievements/user`
- ⭐ `GET /leaderboard`
- ⭐ `GET /notifications`
- ⭐ `GET /challenges`

### Medium Priority
- 📋 `GET /labs`, `POST /labs/:id/session/start`
- 📋 `GET /components`, `GET /components/categories`
- 📋 `GET /docs/categories/:slug`, `GET /docs/articles/:slug`

### Low Priority
- 📝 File upload endpoints
- 📝 Notification management endpoints

---

*Last Updated: 2024-12-27*
*Frontend Version: NexFlux v1.0*
