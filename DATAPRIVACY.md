# Data Privacy Backend API Documentation

This document outlines the backend API requirements for the Data Privacy & Settings feature in NexFlux.

## Overview

The Data Privacy section allows users to:
- Control profile visibility
- Manage data collection preferences
- Export their data
- Clear activity history
- Delete their account

## Database Schema

### User Settings Table (Enhancement)

The `user_settings` table needs to be extended or the privacy settings stored:

```sql
-- If using JSONB for settings
ALTER TABLE user_settings 
ADD COLUMN IF NOT EXISTS privacy JSONB DEFAULT '{
  "show_profile": true,
  "show_activity": true,
  "show_projects": true,
  "analytics_enabled": true,
  "personalization_enabled": true
}'::jsonb;

-- Or as a separate table
CREATE TABLE IF NOT EXISTS user_privacy_settings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    show_profile BOOLEAN DEFAULT true,
    show_activity BOOLEAN DEFAULT true,
    show_projects BOOLEAN DEFAULT true,
    analytics_enabled BOOLEAN DEFAULT true,
    personalization_enabled BOOLEAN DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(user_id)
);

CREATE INDEX idx_user_privacy_user_id ON user_privacy_settings(user_id);
```

### Data Export Requests Table

```sql
CREATE TABLE IF NOT EXISTS data_export_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    export_type VARCHAR(50) NOT NULL, -- 'projects', 'account', 'circuits', 'all'
    status VARCHAR(20) DEFAULT 'pending', -- 'pending', 'processing', 'completed', 'failed'
    file_url TEXT, -- URL to download the export (S3, etc.)
    file_size BIGINT, -- Size in bytes
    expires_at TIMESTAMP WITH TIME ZONE, -- When the download link expires
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    completed_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_data_export_user_id ON data_export_requests(user_id);
CREATE INDEX idx_data_export_status ON data_export_requests(status);
```

### Account Deletion Requests Table

```sql
CREATE TABLE IF NOT EXISTS account_deletion_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status VARCHAR(20) DEFAULT 'pending', -- 'pending', 'confirmed', 'processing', 'completed', 'cancelled'
    reason TEXT,
    confirmation_token VARCHAR(255),
    confirmed_at TIMESTAMP WITH TIME ZONE,
    scheduled_deletion_at TIMESTAMP WITH TIME ZONE, -- Usually 30 days after confirmation
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(user_id)
);

CREATE INDEX idx_account_deletion_user_id ON account_deletion_requests(user_id);
CREATE INDEX idx_account_deletion_status ON account_deletion_requests(status);
```

---

## API Endpoints

### 1. Privacy Settings

#### GET /api/v1/users/me/settings/privacy
Get current privacy settings.

**Response:**
```json
{
  "success": true,
  "data": {
    "show_profile": true,
    "show_activity": true,
    "show_projects": true,
    "analytics_enabled": true,
    "personalization_enabled": true
  }
}
```

#### PUT /api/v1/users/me/settings/privacy
Update privacy settings.

**Request:**
```json
{
  "show_profile": true,
  "show_activity": false,
  "show_projects": true,
  "analytics_enabled": false,
  "personalization_enabled": true
}
```

**Response:**
```json
{
  "success": true,
  "message": "Privacy settings updated",
  "data": {
    "show_profile": true,
    "show_activity": false,
    "show_projects": true,
    "analytics_enabled": false,
    "personalization_enabled": true
  }
}
```

---

### 2. Data Export

#### POST /api/v1/users/me/export
Request a data export.

**Request:**
```json
{
  "type": "projects" // 'projects' | 'account' | 'circuits' | 'all'
}
```

**Response:**
```json
{
  "success": true,
  "message": "Export request submitted. You will receive an email when ready.",
  "data": {
    "request_id": "uuid",
    "type": "projects",
    "status": "pending",
    "estimated_time": "5-10 minutes"
  }
}
```

#### GET /api/v1/users/me/export
List all export requests for the user.

**Response:**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "type": "projects",
      "status": "completed",
      "file_url": "https://storage.example.com/exports/...",
      "file_size": 1048576,
      "expires_at": "2024-01-20T00:00:00Z",
      "created_at": "2024-01-15T10:00:00Z",
      "completed_at": "2024-01-15T10:05:00Z"
    }
  ]
}
```

#### GET /api/v1/users/me/export/:id
Get specific export request status.

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "type": "account",
    "status": "completed",
    "file_url": "https://storage.example.com/exports/...",
    "file_size": 2048,
    "expires_at": "2024-01-20T00:00:00Z",
    "created_at": "2024-01-15T10:00:00Z"
  }
}
```

#### GET /api/v1/users/me/export/:id/download
Download the exported file (redirect to signed URL).

**Response:**
- `302 Redirect` to signed download URL
- Or stream the file directly with appropriate headers

---

### 3. Activity Management

#### DELETE /api/v1/users/me/activity
Clear all activity history for the user.

**Response:**
```json
{
  "success": true,
  "message": "Activity history cleared",
  "data": {
    "deleted_count": 150
  }
}
```

#### GET /api/v1/users/me/activity
Get activity history (paginated).

**Query Parameters:**
- `page` (default: 1)
- `limit` (default: 20, max: 100)
- `type` (optional): Filter by activity type

**Response:**
```json
{
  "success": true,
  "data": {
    "activities": [
      {
        "id": "uuid",
        "type": "project_created",
        "description": "Created project 'Arduino LED Blink'",
        "metadata": {},
        "created_at": "2024-01-15T10:00:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 150,
      "total_pages": 8
    }
  }
}
```

---

### 4. Settings Reset

#### POST /api/v1/users/me/settings/reset
Reset all settings to defaults.

**Response:**
```json
{
  "success": true,
  "message": "Settings reset to defaults",
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
      "show_projects": true,
      "analytics_enabled": true,
      "personalization_enabled": true
    }
  }
}
```

---

### 5. Account Deletion

#### POST /api/v1/users/me/delete-account
Initiate account deletion process.

**Request:**
```json
{
  "password": "current_password",
  "reason": "Optional reason for leaving" // optional
}
```

**Response:**
```json
{
  "success": true,
  "message": "Account deletion initiated. Check your email for confirmation.",
  "data": {
    "request_id": "uuid",
    "status": "pending",
    "confirmation_required": true,
    "scheduled_deletion_at": null // Will be set after confirmation
  }
}
```

#### POST /api/v1/users/me/delete-account/confirm
Confirm account deletion (from email link).

**Request:**
```json
{
  "token": "confirmation_token_from_email"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Account deletion confirmed. Your account will be deleted in 30 days.",
  "data": {
    "scheduled_deletion_at": "2024-02-15T00:00:00Z"
  }
}
```

#### POST /api/v1/users/me/delete-account/cancel
Cancel pending account deletion.

**Response:**
```json
{
  "success": true,
  "message": "Account deletion cancelled"
}
```

#### GET /api/v1/users/me/delete-account/status
Check account deletion status.

**Response:**
```json
{
  "success": true,
  "data": {
    "has_pending_deletion": true,
    "status": "confirmed",
    "scheduled_deletion_at": "2024-02-15T00:00:00Z",
    "can_cancel": true
  }
}
```

---

## Go Implementation

### DTOs

```go
// pkg/dto/privacy.go

package dto

// PrivacySettingsRequest represents privacy settings update request
type PrivacySettingsRequest struct {
    ShowProfile            *bool `json:"show_profile,omitempty"`
    ShowActivity           *bool `json:"show_activity,omitempty"`
    ShowProjects           *bool `json:"show_projects,omitempty"`
    AnalyticsEnabled       *bool `json:"analytics_enabled,omitempty"`
    PersonalizationEnabled *bool `json:"personalization_enabled,omitempty"`
}

// PrivacySettingsResponse represents privacy settings response
type PrivacySettingsResponse struct {
    ShowProfile            bool `json:"show_profile"`
    ShowActivity           bool `json:"show_activity"`
    ShowProjects           bool `json:"show_projects"`
    AnalyticsEnabled       bool `json:"analytics_enabled"`
    PersonalizationEnabled bool `json:"personalization_enabled"`
}

// DataExportRequest represents data export request
type DataExportRequest struct {
    Type string `json:"type" validate:"required,oneof=projects account circuits all"`
}

// DataExportResponse represents data export response
type DataExportResponse struct {
    ID          string     `json:"id"`
    Type        string     `json:"type"`
    Status      string     `json:"status"`
    FileURL     *string    `json:"file_url,omitempty"`
    FileSize    *int64     `json:"file_size,omitempty"`
    ExpiresAt   *time.Time `json:"expires_at,omitempty"`
    CreatedAt   time.Time  `json:"created_at"`
    CompletedAt *time.Time `json:"completed_at,omitempty"`
}

// DeleteAccountRequest represents account deletion request
type DeleteAccountRequest struct {
    Password string `json:"password" validate:"required"`
    Reason   string `json:"reason,omitempty"`
}

// DeleteAccountConfirmRequest represents account deletion confirmation
type DeleteAccountConfirmRequest struct {
    Token string `json:"token" validate:"required"`
}

// DeleteAccountStatusResponse represents account deletion status
type DeleteAccountStatusResponse struct {
    HasPendingDeletion  bool       `json:"has_pending_deletion"`
    Status              string     `json:"status,omitempty"`
    ScheduledDeletionAt *time.Time `json:"scheduled_deletion_at,omitempty"`
    CanCancel           bool       `json:"can_cancel"`
}
```

### Models

```go
// internal/models/privacy.go

package models

import (
    "time"
    "github.com/google/uuid"
)

// UserPrivacySettings represents user privacy settings
type UserPrivacySettings struct {
    ID                     uuid.UUID `gorm:"type:uuid;primary_key;default:gen_random_uuid()"`
    UserID                 uuid.UUID `gorm:"type:uuid;not null;uniqueIndex"`
    ShowProfile            bool      `gorm:"default:true"`
    ShowActivity           bool      `gorm:"default:true"`
    ShowProjects           bool      `gorm:"default:true"`
    AnalyticsEnabled       bool      `gorm:"default:true"`
    PersonalizationEnabled bool      `gorm:"default:true"`
    CreatedAt              time.Time
    UpdatedAt              time.Time
}

// DataExportRequest represents a data export request
type DataExportRequest struct {
    ID          uuid.UUID  `gorm:"type:uuid;primary_key;default:gen_random_uuid()"`
    UserID      uuid.UUID  `gorm:"type:uuid;not null;index"`
    ExportType  string     `gorm:"type:varchar(50);not null"`
    Status      string     `gorm:"type:varchar(20);default:'pending'"`
    FileURL     *string    `gorm:"type:text"`
    FileSize    *int64
    ExpiresAt   *time.Time
    CreatedAt   time.Time
    CompletedAt *time.Time
}

// AccountDeletionRequest represents an account deletion request
type AccountDeletionRequest struct {
    ID                  uuid.UUID  `gorm:"type:uuid;primary_key;default:gen_random_uuid()"`
    UserID              uuid.UUID  `gorm:"type:uuid;not null;uniqueIndex"`
    Status              string     `gorm:"type:varchar(20);default:'pending'"`
    Reason              *string    `gorm:"type:text"`
    ConfirmationToken   *string    `gorm:"type:varchar(255)"`
    ConfirmedAt         *time.Time
    ScheduledDeletionAt *time.Time
    CreatedAt           time.Time
}
```

### Handler

```go
// internal/handlers/privacy_handler.go

package handlers

import (
    "github.com/gofiber/fiber/v2"
)

type PrivacyHandler struct {
    privacyService services.PrivacyService
}

func NewPrivacyHandler(ps services.PrivacyService) *PrivacyHandler {
    return &PrivacyHandler{privacyService: ps}
}

// GetPrivacySettings godoc
// @Summary Get privacy settings
// @Tags Privacy
// @Security BearerAuth
// @Success 200 {object} dto.APIResponse{data=dto.PrivacySettingsResponse}
// @Router /users/me/settings/privacy [get]
func (h *PrivacyHandler) GetPrivacySettings(c *fiber.Ctx) error {
    userID := c.Locals("userID").(string)
    
    settings, err := h.privacyService.GetPrivacySettings(c.Context(), userID)
    if err != nil {
        return utils.ErrorResponse(c, fiber.StatusInternalServerError, "Failed to get privacy settings", err)
    }
    
    return utils.SuccessResponse(c, "Privacy settings retrieved", settings)
}

// UpdatePrivacySettings godoc
// @Summary Update privacy settings
// @Tags Privacy
// @Security BearerAuth
// @Param request body dto.PrivacySettingsRequest true "Privacy settings"
// @Success 200 {object} dto.APIResponse{data=dto.PrivacySettingsResponse}
// @Router /users/me/settings/privacy [put]
func (h *PrivacyHandler) UpdatePrivacySettings(c *fiber.Ctx) error {
    userID := c.Locals("userID").(string)
    
    var req dto.PrivacySettingsRequest
    if err := c.BodyParser(&req); err != nil {
        return utils.ErrorResponse(c, fiber.StatusBadRequest, "Invalid request body", err)
    }
    
    settings, err := h.privacyService.UpdatePrivacySettings(c.Context(), userID, &req)
    if err != nil {
        return utils.ErrorResponse(c, fiber.StatusInternalServerError, "Failed to update privacy settings", err)
    }
    
    return utils.SuccessResponse(c, "Privacy settings updated", settings)
}

// RequestDataExport godoc
// @Summary Request data export
// @Tags Privacy
// @Security BearerAuth
// @Param request body dto.DataExportRequest true "Export request"
// @Success 200 {object} dto.APIResponse{data=dto.DataExportResponse}
// @Router /users/me/export [post]
func (h *PrivacyHandler) RequestDataExport(c *fiber.Ctx) error {
    userID := c.Locals("userID").(string)
    
    var req dto.DataExportRequest
    if err := c.BodyParser(&req); err != nil {
        return utils.ErrorResponse(c, fiber.StatusBadRequest, "Invalid request body", err)
    }
    
    export, err := h.privacyService.RequestDataExport(c.Context(), userID, req.Type)
    if err != nil {
        return utils.ErrorResponse(c, fiber.StatusInternalServerError, "Failed to request export", err)
    }
    
    return utils.SuccessResponse(c, "Export request submitted", export)
}

// ClearActivityHistory godoc
// @Summary Clear activity history
// @Tags Privacy
// @Security BearerAuth
// @Success 200 {object} dto.APIResponse
// @Router /users/me/activity [delete]
func (h *PrivacyHandler) ClearActivityHistory(c *fiber.Ctx) error {
    userID := c.Locals("userID").(string)
    
    count, err := h.privacyService.ClearActivityHistory(c.Context(), userID)
    if err != nil {
        return utils.ErrorResponse(c, fiber.StatusInternalServerError, "Failed to clear activity", err)
    }
    
    return utils.SuccessResponse(c, "Activity history cleared", fiber.Map{
        "deleted_count": count,
    })
}

// InitiateAccountDeletion godoc
// @Summary Initiate account deletion
// @Tags Privacy
// @Security BearerAuth
// @Param request body dto.DeleteAccountRequest true "Deletion request"
// @Success 200 {object} dto.APIResponse
// @Router /users/me/delete-account [post]
func (h *PrivacyHandler) InitiateAccountDeletion(c *fiber.Ctx) error {
    userID := c.Locals("userID").(string)
    
    var req dto.DeleteAccountRequest
    if err := c.BodyParser(&req); err != nil {
        return utils.ErrorResponse(c, fiber.StatusBadRequest, "Invalid request body", err)
    }
    
    result, err := h.privacyService.InitiateAccountDeletion(c.Context(), userID, &req)
    if err != nil {
        return utils.ErrorResponse(c, fiber.StatusInternalServerError, "Failed to initiate deletion", err)
    }
    
    return utils.SuccessResponse(c, "Account deletion initiated. Check your email for confirmation.", result)
}
```

### Routes

```go
// In routes/routes.go or similar

// Privacy & Data routes (authenticated)
privacy := api.Group("/users/me", middleware.JWTAuth())
{
    // Privacy settings
    privacy.Get("/settings/privacy", privacyHandler.GetPrivacySettings)
    privacy.Put("/settings/privacy", privacyHandler.UpdatePrivacySettings)
    
    // Settings reset
    privacy.Post("/settings/reset", settingsHandler.ResetSettings)
    
    // Data export
    privacy.Post("/export", privacyHandler.RequestDataExport)
    privacy.Get("/export", privacyHandler.ListExportRequests)
    privacy.Get("/export/:id", privacyHandler.GetExportRequest)
    privacy.Get("/export/:id/download", privacyHandler.DownloadExport)
    
    // Activity management
    privacy.Get("/activity", activityHandler.GetActivityHistory)
    privacy.Delete("/activity", privacyHandler.ClearActivityHistory)
    
    // Account deletion
    privacy.Post("/delete-account", privacyHandler.InitiateAccountDeletion)
    privacy.Post("/delete-account/confirm", privacyHandler.ConfirmAccountDeletion)
    privacy.Post("/delete-account/cancel", privacyHandler.CancelAccountDeletion)
    privacy.Get("/delete-account/status", privacyHandler.GetAccountDeletionStatus)
}
```

---

## Background Jobs

### Data Export Worker

The data export process should be handled asynchronously:

```go
// internal/workers/export_worker.go

func (w *ExportWorker) ProcessExportRequest(requestID string) error {
    // 1. Get export request from DB
    // 2. Based on type, gather data:
    //    - projects: Fetch all user projects, zip them
    //    - account: Fetch user profile, settings, activity as JSON
    //    - circuits: Fetch all saved circuits
    //    - all: Combine all above
    // 3. Create archive (ZIP)
    // 4. Upload to storage (S3, etc.)
    // 5. Update request with file URL and status
    // 6. Send notification email to user
    return nil
}
```

### Account Deletion Worker

```go
// internal/workers/deletion_worker.go

func (w *DeletionWorker) ProcessScheduledDeletions() error {
    // 1. Find all confirmed deletions where scheduled_deletion_at <= now
    // 2. For each:
    //    - Soft delete (or hard delete based on policy)
    //    - Anonymize any data that must be retained
    //    - Clean up associated files
    //    - Send final notification email
    return nil
}
```

---

## Frontend Integration Summary

The frontend Settings -> Data Privacy section is already implemented with:

| Feature | Frontend Status | Backend Required |
|---------|-----------------|------------------|
| Profile Visibility Toggles | ✅ Implemented | `PUT /users/me/settings/privacy` |
| Analytics Toggle | ✅ Implemented | `PUT /users/me/settings/privacy` |
| Personalization Toggle | ✅ Implemented | `PUT /users/me/settings/privacy` |
| Export Projects | ✅ UI Ready | `POST /users/me/export` |
| Export Account Data | ✅ UI Ready | `POST /users/me/export` |
| Export Circuits | ✅ UI Ready | `POST /users/me/export` |
| Clear Activity | ✅ UI Ready | `DELETE /users/me/activity` |
| Reset Settings | ✅ Implemented | `POST /users/me/settings/reset` |
| Delete Account | ✅ UI Ready | `POST /users/me/delete-account` |

---

## Security Considerations

1. **Password Verification**: Account deletion requires password confirmation
2. **Email Confirmation**: Deletion requires email confirmation with token
3. **Grace Period**: 30-day grace period before permanent deletion
4. **Rate Limiting**: Limit export requests (e.g., 3 per day)
5. **Audit Logging**: Log all privacy-related actions
6. **GDPR Compliance**: Ensure data exports include all user data
7. **Data Retention**: Define retention policy for deleted accounts

---

## Implementation Priority

1. **High Priority**
   - `PUT /users/me/settings/privacy` - Privacy toggles
   - `POST /users/me/settings/reset` - Reset settings
   
2. **Medium Priority**
   - `DELETE /users/me/activity` - Clear activity
   - `POST /users/me/export` - Data export
   
3. **Lower Priority** (Complex, requires background jobs)
   - Account deletion flow
   - Export file generation
