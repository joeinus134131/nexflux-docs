# Backend Integrations API - Implementation Guide

Panduan ini berisi spesifikasi lengkap untuk mengimplementasikan backend API untuk fitur Integrations di NexFlux.

## Quick Start

Buat file-file berikut di backend Go Anda:

```
internal/
├── models/
│   └── integration.go       # Database models
├── repository/
│   └── integration_repository.go
├── service/
│   └── integration_service.go
├── handler/
│   └── integration_handler.go
├── dto/
│   └── integration_dto.go
└── routes/
    └── integration_routes.go
```

---

## 1. Database Migration

### `migrations/XXXXXX_create_user_integrations.up.sql`

```sql
CREATE TABLE IF NOT EXISTS user_integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider VARCHAR(50) NOT NULL,
    enabled BOOLEAN DEFAULT true,
    config JSONB NOT NULL DEFAULT '{}',
    access_token TEXT,
    refresh_token TEXT,
    token_expires_at TIMESTAMP,
    metadata JSONB DEFAULT '{}',
    connected_at TIMESTAMP DEFAULT NOW(),
    last_used_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(user_id, provider)
);

CREATE INDEX IF NOT EXISTS idx_user_integrations_user_id ON user_integrations(user_id);
CREATE INDEX IF NOT EXISTS idx_user_integrations_provider ON user_integrations(provider);

COMMENT ON TABLE user_integrations IS 'Stores user integrations with external services';
COMMENT ON COLUMN user_integrations.provider IS 'Integration type: mqtt, github, arduino_cloud, google, blynk';
COMMENT ON COLUMN user_integrations.config IS 'Provider-specific configuration as JSON';
```

### Down Migration

```sql
DROP TABLE IF EXISTS user_integrations;
```

---

## 2. Models

### `internal/models/integration.go`

```go
package models

import (
    "encoding/json"
    "time"

    "github.com/google/uuid"
)

// IntegrationProvider types
const (
    ProviderMQTT         = "mqtt"
    ProviderGitHub       = "github"
    ProviderArduinoCloud = "arduino_cloud"
    ProviderGoogle       = "google"
    ProviderBlynk        = "blynk"
)

// UserIntegration represents a connected service for a user
type UserIntegration struct {
    ID             uuid.UUID       `json:"id" gorm:"type:uuid;primary_key;default:gen_random_uuid()"`
    UserID         uuid.UUID       `json:"user_id" gorm:"type:uuid;not null;index"`
    Provider       string          `json:"provider" gorm:"size:50;not null;index"`
    Enabled        bool            `json:"enabled" gorm:"default:true"`
    Config         json.RawMessage `json:"config" gorm:"type:jsonb;default:'{}'"`
    AccessToken    *string         `json:"-" gorm:"type:text"` // encrypted, never exposed
    RefreshToken   *string         `json:"-" gorm:"type:text"` // encrypted, never exposed  
    TokenExpiresAt *time.Time      `json:"token_expires_at"`
    Metadata       json.RawMessage `json:"metadata" gorm:"type:jsonb;default:'{}'"`
    ConnectedAt    time.Time       `json:"connected_at" gorm:"default:now()"`
    LastUsedAt     *time.Time      `json:"last_used_at"`
    CreatedAt      time.Time       `json:"created_at" gorm:"default:now()"`
    UpdatedAt      time.Time       `json:"updated_at" gorm:"default:now()"`
}

func (UserIntegration) TableName() string {
    return "user_integrations"
}

// MQTTConfig represents MQTT-specific configuration
type MQTTConfig struct {
    BrokerURL         string `json:"broker_url"`
    Username          string `json:"username,omitempty"`
    Password          string `json:"password,omitempty"` // stored encrypted
    UseAuthentication bool   `json:"use_authentication"`
    DeviceID          string `json:"device_id"`
    TopicPrefix       string `json:"topic_prefix"`
}

// GitHubConfig represents GitHub-specific configuration
type GitHubConfig struct {
    Username      string   `json:"username"`
    AvatarURL     string   `json:"avatar_url"`
    Email         string   `json:"email"`
    Scopes        []string `json:"scopes"`
    AutoSync      bool     `json:"auto_sync"`
    SelectedRepos []int64  `json:"selected_repos"`
}

// ArduinoCloudConfig represents Arduino Cloud configuration
type ArduinoCloudConfig struct {
    Username     string `json:"username"`
    Organization string `json:"organization"`
}

// OAuthConfig represents generic OAuth configuration
type OAuthConfig struct {
    Email     string   `json:"email"`
    Name      string   `json:"name"`
    AvatarURL string   `json:"avatar_url"`
    Scopes    []string `json:"scopes"`
}
```

---

## 3. DTOs

### `internal/dto/integration_dto.go`

```go
package dto

import "time"

// ========== MQTT DTOs ==========

type MQTTConfigRequest struct {
    Enabled           bool   `json:"enabled"`
    BrokerURL         string `json:"broker_url" validate:"required_if=Enabled true,url"`
    Username          string `json:"username,omitempty"`
    Password          string `json:"password,omitempty"`
    UseAuthentication bool   `json:"use_authentication"`
    DeviceID          string `json:"device_id" validate:"required_if=Enabled true"`
    TopicPrefix       string `json:"topic_prefix" validate:"required_if=Enabled true"`
}

type MQTTConfigResponse struct {
    Enabled           bool       `json:"enabled"`
    BrokerURL         string     `json:"broker_url"`
    Username          string     `json:"username,omitempty"`
    UseAuthentication bool       `json:"use_authentication"`
    DeviceID          string     `json:"device_id"`
    TopicPrefix       string     `json:"topic_prefix"`
    LastConnectedAt   *time.Time `json:"last_connected_at,omitempty"`
    Status            string     `json:"status"` // connected, disconnected, error
}

type MQTTTestRequest struct {
    BrokerURL         string `json:"broker_url" validate:"required,url"`
    Username          string `json:"username,omitempty"`
    Password          string `json:"password,omitempty"`
    UseAuthentication bool   `json:"use_authentication"`
}

type MQTTTestResponse struct {
    Connected  bool   `json:"connected"`
    LatencyMs  int64  `json:"latency_ms,omitempty"`
    Error      string `json:"error,omitempty"`
    BrokerInfo struct {
        Version  string `json:"version,omitempty"`
        Protocol string `json:"protocol,omitempty"`
    } `json:"broker_info,omitempty"`
}

// ========== GitHub DTOs ==========

type GitHubIntegrationResponse struct {
    Connected    bool               `json:"connected"`
    Username     string             `json:"username,omitempty"`
    AvatarURL    string             `json:"avatar_url,omitempty"`
    Email        string             `json:"email,omitempty"`
    ConnectedAt  *time.Time         `json:"connected_at,omitempty"`
    Repositories []GitHubRepository `json:"repositories,omitempty"`
    AutoSync     bool               `json:"auto_sync"`
    Scopes       []string           `json:"scopes,omitempty"`
}

type GitHubRepository struct {
    ID       int64      `json:"id"`
    Name     string     `json:"name"`
    FullName string     `json:"full_name"`
    Private  bool       `json:"private"`
    Synced   bool       `json:"synced"`
    LastSync *time.Time `json:"last_sync,omitempty"`
}

type GitHubSyncRequest struct {
    RepositoryID int64    `json:"repository_id" validate:"required"`
    ProjectIDs   []string `json:"project_ids" validate:"required,min=1"`
}

// ========== Arduino Cloud DTOs ==========

type ArduinoCloudConnectRequest struct {
    ClientID     string `json:"client_id" validate:"required"`
    ClientSecret string `json:"client_secret" validate:"required"`
}

type ArduinoCloudResponse struct {
    Connected    bool           `json:"connected"`
    Username     string         `json:"username,omitempty"`
    Organization string         `json:"organization,omitempty"`
    ConnectedAt  *time.Time     `json:"connected_at,omitempty"`
    Devices      []ArduinoDevice `json:"devices,omitempty"`
}

type ArduinoDevice struct {
    ID           string     `json:"id"`
    Name         string     `json:"name"`
    Type         string     `json:"type"`
    Status       string     `json:"status"` // online, offline
    LastActivity *time.Time `json:"last_activity,omitempty"`
}

// ========== Generic OAuth DTOs ==========

type OAuthIntegrationResponse struct {
    Connected   bool       `json:"connected"`
    Email       string     `json:"email,omitempty"`
    Name        string     `json:"name,omitempty"`
    AvatarURL   string     `json:"avatar_url,omitempty"`
    ConnectedAt *time.Time `json:"connected_at,omitempty"`
    Scopes      []string   `json:"scopes,omitempty"`
}

// ========== Generic Responses ==========

type IntegrationStatusResponse struct {
    Provider    string     `json:"provider"`
    Connected   bool       `json:"connected"`
    Enabled     bool       `json:"enabled"`
    ConnectedAt *time.Time `json:"connected_at,omitempty"`
    LastUsedAt  *time.Time `json:"last_used_at,omitempty"`
}
```

---

## 4. Repository

### `internal/repository/integration_repository.go`

```go
package repository

import (
    "context"
    "encoding/json"
    
    "github.com/google/uuid"
    "gorm.io/gorm"
    "your-app/internal/models"
)

type IntegrationRepository interface {
    GetByUserAndProvider(ctx context.Context, userID uuid.UUID, provider string) (*models.UserIntegration, error)
    GetAllByUser(ctx context.Context, userID uuid.UUID) ([]models.UserIntegration, error)
    Upsert(ctx context.Context, integration *models.UserIntegration) error
    Delete(ctx context.Context, userID uuid.UUID, provider string) error
    UpdateLastUsed(ctx context.Context, userID uuid.UUID, provider string) error
}

type integrationRepository struct {
    db *gorm.DB
}

func NewIntegrationRepository(db *gorm.DB) IntegrationRepository {
    return &integrationRepository{db: db}
}

func (r *integrationRepository) GetByUserAndProvider(ctx context.Context, userID uuid.UUID, provider string) (*models.UserIntegration, error) {
    var integration models.UserIntegration
    err := r.db.WithContext(ctx).
        Where("user_id = ? AND provider = ?", userID, provider).
        First(&integration).Error
    if err != nil {
        return nil, err
    }
    return &integration, nil
}

func (r *integrationRepository) GetAllByUser(ctx context.Context, userID uuid.UUID) ([]models.UserIntegration, error) {
    var integrations []models.UserIntegration
    err := r.db.WithContext(ctx).
        Where("user_id = ?", userID).
        Find(&integrations).Error
    return integrations, err
}

func (r *integrationRepository) Upsert(ctx context.Context, integration *models.UserIntegration) error {
    return r.db.WithContext(ctx).Save(integration).Error
}

func (r *integrationRepository) Delete(ctx context.Context, userID uuid.UUID, provider string) error {
    return r.db.WithContext(ctx).
        Where("user_id = ? AND provider = ?", userID, provider).
        Delete(&models.UserIntegration{}).Error
}

func (r *integrationRepository) UpdateLastUsed(ctx context.Context, userID uuid.UUID, provider string) error {
    return r.db.WithContext(ctx).
        Model(&models.UserIntegration{}).
        Where("user_id = ? AND provider = ?", userID, provider).
        Update("last_used_at", gorm.Expr("NOW()")).Error
}
```

---

## 5. Service

### `internal/service/integration_service.go`

```go
package service

import (
    "context"
    "encoding/json"
    "fmt"
    "net"
    "time"
    
    "github.com/google/uuid"
    mqtt "github.com/eclipse/paho.mqtt.golang"
    "your-app/internal/dto"
    "your-app/internal/models"
    "your-app/internal/repository"
)

type IntegrationService interface {
    // MQTT
    GetMQTTConfig(ctx context.Context, userID uuid.UUID) (*dto.MQTTConfigResponse, error)
    SaveMQTTConfig(ctx context.Context, userID uuid.UUID, req dto.MQTTConfigRequest) error
    TestMQTTConnection(ctx context.Context, req dto.MQTTTestRequest) (*dto.MQTTTestResponse, error)
    DeleteMQTTConfig(ctx context.Context, userID uuid.UUID) error
    
    // GitHub
    GetGitHubIntegration(ctx context.Context, userID uuid.UUID) (*dto.GitHubIntegrationResponse, error)
    DeleteGitHubIntegration(ctx context.Context, userID uuid.UUID) error
    
    // Arduino Cloud
    GetArduinoIntegration(ctx context.Context, userID uuid.UUID) (*dto.ArduinoCloudResponse, error)
    ConnectArduinoCloud(ctx context.Context, userID uuid.UUID, req dto.ArduinoCloudConnectRequest) error
    DeleteArduinoIntegration(ctx context.Context, userID uuid.UUID) error
    
    // Google
    GetGoogleIntegration(ctx context.Context, userID uuid.UUID) (*dto.OAuthIntegrationResponse, error)
    DeleteGoogleIntegration(ctx context.Context, userID uuid.UUID) error
    
    // Blynk
    GetBlynkIntegration(ctx context.Context, userID uuid.UUID) (*dto.OAuthIntegrationResponse, error)
    DeleteBlynkIntegration(ctx context.Context, userID uuid.UUID) error
}

type integrationService struct {
    repo repository.IntegrationRepository
}

func NewIntegrationService(repo repository.IntegrationRepository) IntegrationService {
    return &integrationService{repo: repo}
}

// ========== MQTT Implementation ==========

func (s *integrationService) GetMQTTConfig(ctx context.Context, userID uuid.UUID) (*dto.MQTTConfigResponse, error) {
    integration, err := s.repo.GetByUserAndProvider(ctx, userID, models.ProviderMQTT)
    if err != nil {
        return nil, err
    }
    
    var config models.MQTTConfig
    if err := json.Unmarshal(integration.Config, &config); err != nil {
        return nil, err
    }
    
    return &dto.MQTTConfigResponse{
        Enabled:           integration.Enabled,
        BrokerURL:         config.BrokerURL,
        Username:          config.Username,
        UseAuthentication: config.UseAuthentication,
        DeviceID:          config.DeviceID,
        TopicPrefix:       config.TopicPrefix,
        LastConnectedAt:   integration.LastUsedAt,
        Status:            "disconnected", // Could check actual connection status
    }, nil
}

func (s *integrationService) SaveMQTTConfig(ctx context.Context, userID uuid.UUID, req dto.MQTTConfigRequest) error {
    config := models.MQTTConfig{
        BrokerURL:         req.BrokerURL,
        Username:          req.Username,
        Password:          req.Password, // Should be encrypted before storing
        UseAuthentication: req.UseAuthentication,
        DeviceID:          req.DeviceID,
        TopicPrefix:       req.TopicPrefix,
    }
    
    configJSON, err := json.Marshal(config)
    if err != nil {
        return err
    }
    
    // Try to get existing integration
    existing, _ := s.repo.GetByUserAndProvider(ctx, userID, models.ProviderMQTT)
    
    integration := &models.UserIntegration{
        UserID:   userID,
        Provider: models.ProviderMQTT,
        Enabled:  req.Enabled,
        Config:   configJSON,
    }
    
    if existing != nil {
        integration.ID = existing.ID
        integration.ConnectedAt = existing.ConnectedAt
    }
    
    return s.repo.Upsert(ctx, integration)
}

func (s *integrationService) TestMQTTConnection(ctx context.Context, req dto.MQTTTestRequest) (*dto.MQTTTestResponse, error) {
    startTime := time.Now()
    
    opts := mqtt.NewClientOptions()
    opts.AddBroker(req.BrokerURL)
    opts.SetClientID(fmt.Sprintf("nexflux-test-%d", time.Now().UnixNano()))
    opts.SetConnectTimeout(5 * time.Second)
    
    if req.UseAuthentication {
        opts.SetUsername(req.Username)
        opts.SetPassword(req.Password)
    }
    
    client := mqtt.NewClient(opts)
    token := client.Connect()
    
    if token.WaitTimeout(5 * time.Second) {
        if token.Error() != nil {
            return &dto.MQTTTestResponse{
                Connected: false,
                Error:     token.Error().Error(),
            }, nil
        }
        
        latency := time.Since(startTime).Milliseconds()
        client.Disconnect(100)
        
        return &dto.MQTTTestResponse{
            Connected: true,
            LatencyMs: latency,
        }, nil
    }
    
    return &dto.MQTTTestResponse{
        Connected: false,
        Error:     "Connection timeout",
    }, nil
}

func (s *integrationService) DeleteMQTTConfig(ctx context.Context, userID uuid.UUID) error {
    return s.repo.Delete(ctx, userID, models.ProviderMQTT)
}

// ... implement other methods similarly
```

---

## 6. Handler

### `internal/handler/integration_handler.go`

```go
package handler

import (
    "net/http"
    
    "github.com/gin-gonic/gin"
    "github.com/google/uuid"
    "your-app/internal/dto"
    "your-app/internal/service"
    "your-app/internal/utils"
)

type IntegrationHandler struct {
    service service.IntegrationService
}

func NewIntegrationHandler(service service.IntegrationService) *IntegrationHandler {
    return &IntegrationHandler{service: service}
}

// ========== MQTT Handlers ==========

// GET /users/me/integrations/mqtt
func (h *IntegrationHandler) GetMQTTConfig(c *gin.Context) {
    userID := c.MustGet("userID").(uuid.UUID)
    
    config, err := h.service.GetMQTTConfig(c.Request.Context(), userID)
    if err != nil {
        utils.NotFoundResponse(c, "MQTT configuration not found")
        return
    }
    
    utils.SuccessResponse(c, config, "MQTT configuration retrieved")
}

// PUT /users/me/integrations/mqtt
func (h *IntegrationHandler) SaveMQTTConfig(c *gin.Context) {
    userID := c.MustGet("userID").(uuid.UUID)
    
    var req dto.MQTTConfigRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        utils.BadRequestResponse(c, err.Error())
        return
    }
    
    if err := h.service.SaveMQTTConfig(c.Request.Context(), userID, req); err != nil {
        utils.InternalServerErrorResponse(c, "Failed to save MQTT configuration")
        return
    }
    
    utils.SuccessResponse(c, nil, "MQTT configuration saved")
}

// POST /users/me/integrations/mqtt/test
func (h *IntegrationHandler) TestMQTTConnection(c *gin.Context) {
    var req dto.MQTTTestRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        utils.BadRequestResponse(c, err.Error())
        return
    }
    
    result, err := h.service.TestMQTTConnection(c.Request.Context(), req)
    if err != nil {
        utils.InternalServerErrorResponse(c, "Failed to test MQTT connection")
        return
    }
    
    utils.SuccessResponse(c, result, "MQTT connection test completed")
}

// DELETE /users/me/integrations/mqtt
func (h *IntegrationHandler) DeleteMQTTConfig(c *gin.Context) {
    userID := c.MustGet("userID").(uuid.UUID)
    
    if err := h.service.DeleteMQTTConfig(c.Request.Context(), userID); err != nil {
        utils.InternalServerErrorResponse(c, "Failed to delete MQTT configuration")
        return
    }
    
    utils.SuccessResponse(c, nil, "MQTT configuration deleted")
}

// ========== GitHub Handlers ==========

// GET /users/me/integrations/github
func (h *IntegrationHandler) GetGitHubIntegration(c *gin.Context) {
    userID := c.MustGet("userID").(uuid.UUID)
    
    integration, err := h.service.GetGitHubIntegration(c.Request.Context(), userID)
    if err != nil {
        utils.NotFoundResponse(c, "GitHub not connected")
        return
    }
    
    utils.SuccessResponse(c, integration, "GitHub integration retrieved")
}

// DELETE /users/me/integrations/github
func (h *IntegrationHandler) DeleteGitHubIntegration(c *gin.Context) {
    userID := c.MustGet("userID").(uuid.UUID)
    
    if err := h.service.DeleteGitHubIntegration(c.Request.Context(), userID); err != nil {
        utils.InternalServerErrorResponse(c, "Failed to disconnect GitHub")
        return
    }
    
    utils.SuccessResponse(c, nil, "GitHub disconnected")
}

// Similar handlers for Arduino Cloud, Google, Blynk...
```

---

## 7. Routes

### `internal/routes/integration_routes.go`

```go
package routes

import (
    "github.com/gin-gonic/gin"
    "your-app/internal/handler"
    "your-app/internal/middleware"
)

func RegisterIntegrationRoutes(router *gin.RouterGroup, h *handler.IntegrationHandler, authMiddleware *middleware.AuthMiddleware) {
    integrations := router.Group("/users/me/integrations")
    integrations.Use(authMiddleware.JWTAuth())
    {
        // MQTT
        integrations.GET("/mqtt", h.GetMQTTConfig)
        integrations.PUT("/mqtt", h.SaveMQTTConfig)
        integrations.POST("/mqtt/test", h.TestMQTTConnection)
        integrations.DELETE("/mqtt", h.DeleteMQTTConfig)
        
        // GitHub
        integrations.GET("/github", h.GetGitHubIntegration)
        integrations.DELETE("/github", h.DeleteGitHubIntegration)
        // Note: GitHub OAuth connect/callback are handled separately in auth routes
        
        // Arduino Cloud
        integrations.GET("/arduino-cloud", h.GetArduinoIntegration)
        integrations.PUT("/arduino-cloud", h.ConnectArduinoCloud)
        integrations.DELETE("/arduino-cloud", h.DeleteArduinoIntegration)
        
        // Google
        integrations.GET("/google", h.GetGoogleIntegration)
        integrations.DELETE("/google", h.DeleteGoogleIntegration)
        
        // Blynk
        integrations.GET("/blynk", h.GetBlynkIntegration)
        integrations.DELETE("/blynk", h.DeleteBlynkIntegration)
    }
    
    // OAuth routes (separate from integrations)
    auth := router.Group("/auth")
    {
        auth.GET("/github/connect", h.InitiateGitHubOAuth)
        auth.GET("/github/callback", h.HandleGitHubCallback)
        
        auth.GET("/google/connect", h.InitiateGoogleOAuth)
        auth.GET("/google/callback", h.HandleGoogleCallback)
    }
}
```

---

## Environment Variables

Add these to your `.env` file:

```env
# GitHub OAuth
GITHUB_CLIENT_ID=your_github_client_id
GITHUB_CLIENT_SECRET=your_github_client_secret
GITHUB_CALLBACK_URL=http://localhost:8005/api/v1/auth/github/callback

# Google OAuth
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret
GOOGLE_CALLBACK_URL=http://localhost:8005/api/v1/auth/google/callback

# Arduino Cloud API
ARDUINO_API_URL=https://api2.arduino.cc

# MQTT Test Configuration
MQTT_DEFAULT_BROKER=ws://212.85.24.191:9001

# Encryption key for tokens (32 bytes for AES-256)
INTEGRATION_ENCRYPTION_KEY=your-32-byte-encryption-key-here
```

---

## Dependencies

Add to `go.mod`:

```
github.com/eclipse/paho.mqtt.golang v1.4.3
golang.org/x/oauth2 v0.15.0
```

---

## Priority Implementation Order

1. **Week 1:** MQTT Integration (most used for Remote Lab)
   - Database migration
   - Models & DTOs
   - Repository, Service, Handler
   - Routes registration

2. **Week 2:** GitHub Integration (OAuth flow)
   - OAuth configuration
   - Token storage & refresh
   - Basic API integration

3. **Week 3:** Arduino Cloud Integration
   - API key authentication
   - Device listing
   
4. **Week 4:** Google & Blynk
   - OAuth flows
   - Basic integration
