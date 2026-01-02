# User Settings API Documentation

## Overview

Dokumen ini menjelaskan API endpoints dan struktur data yang diperlukan untuk fitur User Settings di NexFlux. Settings memungkinkan user untuk mengkustomisasi preferensi notifikasi, tampilan, dan privasi mereka.

---

## Database Schema

### Tabel: `user_settings`

```sql
CREATE TABLE user_settings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    
    -- Notification Settings
    notification_email BOOLEAN DEFAULT TRUE,
    notification_push BOOLEAN DEFAULT TRUE,
    notification_marketing BOOLEAN DEFAULT FALSE,
    notification_updates BOOLEAN DEFAULT TRUE,
    notification_project_invites BOOLEAN DEFAULT TRUE,
    notification_comments BOOLEAN DEFAULT TRUE,
    notification_mentions BOOLEAN DEFAULT TRUE,
    
    -- Appearance Settings
    appearance_theme VARCHAR(20) DEFAULT 'system' CHECK (appearance_theme IN ('light', 'dark', 'system')),
    appearance_language VARCHAR(5) DEFAULT 'en' CHECK (appearance_language IN ('en', 'id', 'jp')),
    appearance_compact_mode BOOLEAN DEFAULT FALSE,
    appearance_animations BOOLEAN DEFAULT TRUE,
    
    -- Privacy Settings
    privacy_show_profile BOOLEAN DEFAULT TRUE,
    privacy_show_activity BOOLEAN DEFAULT TRUE,
    privacy_show_projects BOOLEAN DEFAULT TRUE,
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Index for faster lookups
CREATE INDEX idx_user_settings_user_id ON user_settings(user_id);

-- Trigger for auto-updating updated_at
CREATE OR REPLACE FUNCTION update_user_settings_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_user_settings_updated_at
    BEFORE UPDATE ON user_settings
    FOR EACH ROW
    EXECUTE FUNCTION update_user_settings_updated_at();
```

### Alternative: JSON Column Approach

Jika lebih fleksibel, bisa menggunakan JSON column:

```sql
CREATE TABLE user_settings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    settings JSONB NOT NULL DEFAULT '{
        "notifications": {
            "email": true,
            "push": true,
            "marketing": false,
            "updates": true,
            "project_invites": true,
            "comments": true,
            "mentions": true
        },
        "appearance": {
            "theme": "system",
            "language": "en",
            "compact_mode": false,
            "animations": true
        },
        "privacy": {
            "show_profile": true,
            "show_activity": true,
            "show_projects": true
        }
    }'::jsonb,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

---

## API Endpoints

### Base URL
```
/api/v1/users/settings
```

---

### 1. Get User Settings

Mengambil settings user yang sedang login.

**Endpoint:** `GET /api/v1/users/settings`

**Headers:**
```
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

**Response Success (200):**
```json
{
    "success": true,
    "message": "Settings retrieved successfully",
    "data": {
        "notifications": {
            "email": true,
            "push": true,
            "marketing": false,
            "updates": true,
            "project_invites": true,
            "comments": true,
            "mentions": true
        },
        "appearance": {
            "theme": "dark",
            "language": "en",
            "compact_mode": false,
            "animations": true
        },
        "privacy": {
            "show_profile": true,
            "show_activity": true,
            "show_projects": true
        }
    }
}
```

**Response Error (401 - Unauthorized):**
```json
{
    "success": false,
    "message": "Unauthorized",
    "error": "Invalid or expired token"
}
```

**Response - No Settings Found (200):**
Jika user belum memiliki settings, kembalikan default values:
```json
{
    "success": true,
    "message": "Settings retrieved successfully (defaults)",
    "data": {
        "notifications": {
            "email": true,
            "push": true,
            "marketing": false,
            "updates": true,
            "project_invites": true,
            "comments": true,
            "mentions": true
        },
        "appearance": {
            "theme": "system",
            "language": "en",
            "compact_mode": false,
            "animations": true
        },
        "privacy": {
            "show_profile": true,
            "show_activity": true,
            "show_projects": true
        }
    }
}
```

---

### 2. Update User Settings

Mengupdate settings user. Mendukung partial update.

**Endpoint:** `PUT /api/v1/users/settings`

**Headers:**
```
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

**Request Body (Full Update):**
```json
{
    "notifications": {
        "email": true,
        "push": false,
        "marketing": false,
        "updates": true,
        "project_invites": true,
        "comments": true,
        "mentions": false
    },
    "appearance": {
        "theme": "dark",
        "language": "id",
        "compact_mode": false,
        "animations": true
    },
    "privacy": {
        "show_profile": true,
        "show_activity": false,
        "show_projects": true
    }
}
```

**Request Body (Partial Update):**
Frontend mungkin hanya mengirim bagian yang berubah:
```json
{
    "notifications": {
        "email": true,
        "push": true,
        "marketing": false,
        "updates": true,
        "project_invites": true,
        "comments": true,
        "mentions": true
    }
}
```

**Response Success (200):**
```json
{
    "success": true,
    "message": "Settings updated successfully",
    "data": {
        "notifications": {
            "email": true,
            "push": false,
            "marketing": false,
            "updates": true,
            "project_invites": true,
            "comments": true,
            "mentions": false
        },
        "appearance": {
            "theme": "dark",
            "language": "id",
            "compact_mode": false,
            "animations": true
        },
        "privacy": {
            "show_profile": true,
            "show_activity": false,
            "show_projects": true
        }
    }
}
```

**Response Error (400 - Validation Error):**
```json
{
    "success": false,
    "message": "Validation failed",
    "errors": [
        {
            "field": "appearance.theme",
            "message": "Invalid theme value. Must be 'light', 'dark', or 'system'"
        }
    ]
}
```

---

## Data Types & Validation

### Settings Object Structure

```typescript
interface UserSettings {
    notifications: NotificationSettings;
    appearance: AppearanceSettings;
    privacy: PrivacySettings;
}

interface NotificationSettings {
    email: boolean;           // Email notification untuk updates & alerts
    push: boolean;            // Browser push notifications
    marketing: boolean;       // Marketing & promotional emails
    updates: boolean;         // Project update notifications
    project_invites: boolean; // Notifikasi undangan project
    comments: boolean;        // Notifikasi komentar di project
    mentions: boolean;        // Notifikasi saat di-mention
}

interface AppearanceSettings {
    theme: 'light' | 'dark' | 'system';  // Tema aplikasi
    language: 'en' | 'id' | 'jp';        // Bahasa UI
    compact_mode: boolean;                // Mode tampilan compact
    animations: boolean;                  // Enable/disable animasi UI
}

interface PrivacySettings {
    show_profile: boolean;    // Apakah profil bisa dilihat public
    show_activity: boolean;   // Tampilkan recent activity di profil
    show_projects: boolean;   // Tampilkan public projects di profil
}
```

### Validation Rules

| Field | Type | Required | Allowed Values | Default |
|-------|------|----------|----------------|---------|
| `notifications.email` | boolean | No | true/false | true |
| `notifications.push` | boolean | No | true/false | true |
| `notifications.marketing` | boolean | No | true/false | false |
| `notifications.updates` | boolean | No | true/false | true |
| `notifications.project_invites` | boolean | No | true/false | true |
| `notifications.comments` | boolean | No | true/false | true |
| `notifications.mentions` | boolean | No | true/false | true |
| `appearance.theme` | string | No | light, dark, system | system |
| `appearance.language` | string | No | en, id, jp | en |
| `appearance.compact_mode` | boolean | No | true/false | false |
| `appearance.animations` | boolean | No | true/false | true |
| `privacy.show_profile` | boolean | No | true/false | true |
| `privacy.show_activity` | boolean | No | true/false | true |
| `privacy.show_projects` | boolean | No | true/false | true |

---

## Go/Fiber Implementation Example

### DTO (Data Transfer Objects)

```go
// dto/user_settings.go

package dto

// UserSettingsResponse represents the full settings response
type UserSettingsResponse struct {
    Notifications NotificationSettings `json:"notifications"`
    Appearance    AppearanceSettings   `json:"appearance"`
    Privacy       PrivacySettings      `json:"privacy"`
}

type NotificationSettings struct {
    Email          bool `json:"email"`
    Push           bool `json:"push"`
    Marketing      bool `json:"marketing"`
    Updates        bool `json:"updates"`
    ProjectInvites bool `json:"project_invites"`
    Comments       bool `json:"comments"`
    Mentions       bool `json:"mentions"`
}

type AppearanceSettings struct {
    Theme       string `json:"theme" validate:"omitempty,oneof=light dark system"`
    Language    string `json:"language" validate:"omitempty,oneof=en id jp"`
    CompactMode bool   `json:"compact_mode"`
    Animations  bool   `json:"animations"`
}

type PrivacySettings struct {
    ShowProfile  bool `json:"show_profile"`
    ShowActivity bool `json:"show_activity"`
    ShowProjects bool `json:"show_projects"`
}

// UpdateUserSettingsRequest for updating settings
type UpdateUserSettingsRequest struct {
    Notifications *NotificationSettings `json:"notifications,omitempty"`
    Appearance    *AppearanceSettings   `json:"appearance,omitempty"`
    Privacy       *PrivacySettings      `json:"privacy,omitempty"`
}
```

### Model

```go
// models/user_settings.go

package models

import (
    "time"
    "github.com/google/uuid"
)

type UserSettings struct {
    ID        uuid.UUID `gorm:"type:uuid;primary_key;default:gen_random_uuid()"`
    UserID    uuid.UUID `gorm:"type:uuid;not null;uniqueIndex"`
    
    // Notification Settings
    NotificationEmail          bool `gorm:"default:true"`
    NotificationPush           bool `gorm:"default:true"`
    NotificationMarketing      bool `gorm:"default:false"`
    NotificationUpdates        bool `gorm:"default:true"`
    NotificationProjectInvites bool `gorm:"default:true"`
    NotificationComments       bool `gorm:"default:true"`
    NotificationMentions       bool `gorm:"default:true"`
    
    // Appearance Settings
    AppearanceTheme       string `gorm:"default:'system'"`
    AppearanceLanguage    string `gorm:"default:'en'"`
    AppearanceCompactMode bool   `gorm:"default:false"`
    AppearanceAnimations  bool   `gorm:"default:true"`
    
    // Privacy Settings
    PrivacyShowProfile  bool `gorm:"default:true"`
    PrivacyShowActivity bool `gorm:"default:true"`
    PrivacyShowProjects bool `gorm:"default:true"`
    
    CreatedAt time.Time
    UpdatedAt time.Time
}

func (UserSettings) TableName() string {
    return "user_settings"
}

// GetDefaultSettings returns default settings for new users
func GetDefaultSettings() UserSettings {
    return UserSettings{
        NotificationEmail:          true,
        NotificationPush:           true,
        NotificationMarketing:      false,
        NotificationUpdates:        true,
        NotificationProjectInvites: true,
        NotificationComments:       true,
        NotificationMentions:       true,
        AppearanceTheme:            "system",
        AppearanceLanguage:         "en",
        AppearanceCompactMode:      false,
        AppearanceAnimations:       true,
        PrivacyShowProfile:         true,
        PrivacyShowActivity:        true,
        PrivacyShowProjects:        true,
    }
}
```

### Handler

```go
// handlers/user_settings_handler.go

package handlers

import (
    "github.com/gofiber/fiber/v2"
    // import your packages
)

// GetUserSettings godoc
// @Summary Get user settings
// @Description Get settings for the authenticated user
// @Tags Settings
// @Accept json
// @Produce json
// @Security BearerAuth
// @Success 200 {object} dto.UserSettingsResponse
// @Failure 401 {object} dto.ErrorResponse
// @Router /users/settings [get]
func (h *UserHandler) GetUserSettings(c *fiber.Ctx) error {
    userID := c.Locals("userID").(uuid.UUID)
    
    settings, err := h.settingsService.GetSettings(userID)
    if err != nil {
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "success": false,
            "message": "Failed to get settings",
            "error":   err.Error(),
        })
    }
    
    return c.JSON(fiber.Map{
        "success": true,
        "message": "Settings retrieved successfully",
        "data":    settings,
    })
}

// UpdateUserSettings godoc
// @Summary Update user settings
// @Description Update settings for the authenticated user
// @Tags Settings
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param request body dto.UpdateUserSettingsRequest true "Settings to update"
// @Success 200 {object} dto.UserSettingsResponse
// @Failure 400 {object} dto.ErrorResponse
// @Failure 401 {object} dto.ErrorResponse
// @Router /users/settings [put]
func (h *UserHandler) UpdateUserSettings(c *fiber.Ctx) error {
    userID := c.Locals("userID").(uuid.UUID)
    
    var req dto.UpdateUserSettingsRequest
    if err := c.BodyParser(&req); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "success": false,
            "message": "Invalid request body",
            "error":   err.Error(),
        })
    }
    
    // Validate request
    if err := h.validator.Struct(req); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "success": false,
            "message": "Validation failed",
            "errors":  formatValidationErrors(err),
        })
    }
    
    settings, err := h.settingsService.UpdateSettings(userID, &req)
    if err != nil {
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "success": false,
            "message": "Failed to update settings",
            "error":   err.Error(),
        })
    }
    
    return c.JSON(fiber.Map{
        "success": true,
        "message": "Settings updated successfully",
        "data":    settings,
    })
}
```

### Service

```go
// services/user_settings_service.go

package services

import (
    "github.com/google/uuid"
    // import your packages
)

type UserSettingsService struct {
    repo *repositories.UserSettingsRepository
}

func NewUserSettingsService(repo *repositories.UserSettingsRepository) *UserSettingsService {
    return &UserSettingsService{repo: repo}
}

func (s *UserSettingsService) GetSettings(userID uuid.UUID) (*dto.UserSettingsResponse, error) {
    settings, err := s.repo.FindByUserID(userID)
    if err != nil {
        // Return default settings if not found
        defaultSettings := models.GetDefaultSettings()
        return s.toResponse(&defaultSettings), nil
    }
    return s.toResponse(settings), nil
}

func (s *UserSettingsService) UpdateSettings(userID uuid.UUID, req *dto.UpdateUserSettingsRequest) (*dto.UserSettingsResponse, error) {
    // Get or create settings
    settings, err := s.repo.FindByUserID(userID)
    if err != nil {
        // Create new settings with defaults
        settings = &models.UserSettings{
            UserID: userID,
        }
        defaultSettings := models.GetDefaultSettings()
        *settings = defaultSettings
        settings.UserID = userID
    }
    
    // Apply updates (partial update support)
    if req.Notifications != nil {
        settings.NotificationEmail = req.Notifications.Email
        settings.NotificationPush = req.Notifications.Push
        settings.NotificationMarketing = req.Notifications.Marketing
        settings.NotificationUpdates = req.Notifications.Updates
        settings.NotificationProjectInvites = req.Notifications.ProjectInvites
        settings.NotificationComments = req.Notifications.Comments
        settings.NotificationMentions = req.Notifications.Mentions
    }
    
    if req.Appearance != nil {
        if req.Appearance.Theme != "" {
            settings.AppearanceTheme = req.Appearance.Theme
        }
        if req.Appearance.Language != "" {
            settings.AppearanceLanguage = req.Appearance.Language
        }
        settings.AppearanceCompactMode = req.Appearance.CompactMode
        settings.AppearanceAnimations = req.Appearance.Animations
    }
    
    if req.Privacy != nil {
        settings.PrivacyShowProfile = req.Privacy.ShowProfile
        settings.PrivacyShowActivity = req.Privacy.ShowActivity
        settings.PrivacyShowProjects = req.Privacy.ShowProjects
    }
    
    // Save to database
    if err := s.repo.Upsert(settings); err != nil {
        return nil, err
    }
    
    return s.toResponse(settings), nil
}

func (s *UserSettingsService) toResponse(settings *models.UserSettings) *dto.UserSettingsResponse {
    return &dto.UserSettingsResponse{
        Notifications: dto.NotificationSettings{
            Email:          settings.NotificationEmail,
            Push:           settings.NotificationPush,
            Marketing:      settings.NotificationMarketing,
            Updates:        settings.NotificationUpdates,
            ProjectInvites: settings.NotificationProjectInvites,
            Comments:       settings.NotificationComments,
            Mentions:       settings.NotificationMentions,
        },
        Appearance: dto.AppearanceSettings{
            Theme:       settings.AppearanceTheme,
            Language:    settings.AppearanceLanguage,
            CompactMode: settings.AppearanceCompactMode,
            Animations:  settings.AppearanceAnimations,
        },
        Privacy: dto.PrivacySettings{
            ShowProfile:  settings.PrivacyShowProfile,
            ShowActivity: settings.PrivacyShowActivity,
            ShowProjects: settings.PrivacyShowProjects,
        },
    }
}
```

### Repository

```go
// repositories/user_settings_repository.go

package repositories

import (
    "github.com/google/uuid"
    "gorm.io/gorm"
    // import your packages
)

type UserSettingsRepository struct {
    db *gorm.DB
}

func NewUserSettingsRepository(db *gorm.DB) *UserSettingsRepository {
    return &UserSettingsRepository{db: db}
}

func (r *UserSettingsRepository) FindByUserID(userID uuid.UUID) (*models.UserSettings, error) {
    var settings models.UserSettings
    err := r.db.Where("user_id = ?", userID).First(&settings).Error
    if err != nil {
        return nil, err
    }
    return &settings, nil
}

func (r *UserSettingsRepository) Upsert(settings *models.UserSettings) error {
    return r.db.Save(settings).Error
}
```

### Routes

```go
// routes/user_routes.go

// Add these routes to your user routes
api.Get("/users/settings", authMiddleware, userHandler.GetUserSettings)
api.Put("/users/settings", authMiddleware, userHandler.UpdateUserSettings)
```

---

## Usage in Frontend

Frontend akan memanggil endpoints ini saat:

1. **Mount Settings Page** - Fetch `GET /users/settings` untuk mendapatkan current settings
2. **Toggle Setting** - Send `PUT /users/settings` dengan full settings object setiap kali ada perubahan

### Contoh Request dari Frontend

```typescript
// Fetch settings
const response = await api.get('/users/settings');
const settings = response.data.data;

// Update single notification setting
const updateSettings = async (newSettings: UserSettings) => {
    await api.put('/users/settings', newSettings);
};
```

---

## Error Codes

| HTTP Code | Error | Description |
|-----------|-------|-------------|
| 200 | Success | Request berhasil |
| 400 | Bad Request | Validation error atau request body invalid |
| 401 | Unauthorized | Token tidak valid atau expired |
| 500 | Internal Server Error | Server error |

---

## Testing

### cURL Examples

**Get Settings:**
```bash
curl -X GET "http://localhost:8005/api/v1/users/settings" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json"
```

**Update Settings:**
```bash
curl -X PUT "http://localhost:8005/api/v1/users/settings" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "notifications": {
      "email": true,
      "push": true,
      "marketing": false,
      "updates": true,
      "project_invites": true,
      "comments": true,
      "mentions": true
    },
    "appearance": {
      "theme": "dark",
      "language": "id",
      "compact_mode": false,
      "animations": true
    },
    "privacy": {
      "show_profile": true,
      "show_activity": true,
      "show_projects": true
    }
  }'
```

---

## Notes

1. **Auto-create on first access**: Jika user belum memiliki settings record, backend harus membuat dengan default values saat pertama kali diakses atau di-update.

2. **Partial Updates**: Frontend selalu mengirim full object untuk memastikan konsistensi, tapi backend harus support partial updates untuk fleksibilitas.

3. **No separate notification preferences per project**: Untuk MVP, notification settings berlaku global. Per-project settings bisa ditambahkan nanti.

4. **Theme sync**: Theme setting di-sync antara backend dan frontend ThemeContext. Saat user login di device berbeda, theme akan di-load dari backend.

5. **Language preference**: Language setting akan di-sync dengan LanguageContext di frontend.
