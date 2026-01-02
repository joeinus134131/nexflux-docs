# Upload API Documentation

This document describes the file upload API endpoints and bucket configuration.

## Storage Buckets

The following Supabase storage buckets are used:

| Bucket Name | Purpose | Max Size | Allowed Types |
|-------------|---------|----------|---------------|
| `nexflux-avatars` | User profile pictures | 2MB | JPEG, PNG, GIF, WebP |
| `nexflux-components` | Electronic component images | 5MB | JPEG, PNG, GIF, WebP, SVG |
| `nexflux-documents` | Documentation files | 10MB | PDF, JPEG, PNG, DOC, DOCX |
| `nexflux-projects` | Project files (schemas, code) | 10MB | JPEG, PNG, GIF, WebP, JSON |
| `nexflux-thumbnails` | Project thumbnail images | 5MB | JPEG, PNG, GIF, WebP |

---

## API Endpoints

### 1. Upload File

**Endpoint:** `POST /api/v1/upload`

**Description:** Upload a file to the specified bucket.

**Headers:**
```
Authorization: Bearer <token>
Content-Type: multipart/form-data
```

**Request Body (FormData):**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `file` | File | Yes | The file to upload |
| `bucket` | String | Yes | Target bucket name |
| `path` | String | No | Custom path within bucket |

**Example Request:**
```javascript
const formData = new FormData();
formData.append('file', file);
formData.append('bucket', 'nexflux-thumbnails');
formData.append('path', 'projects/123/thumbnail.jpg');

const response = await api.post('/upload', formData, {
    headers: { 'Content-Type': 'multipart/form-data' }
});
```

**Success Response:**
```json
{
    "success": true,
    "data": {
        "url": "https://xxx.supabase.co/storage/v1/object/public/nexflux-thumbnails/projects/123/thumbnail.jpg",
        "path": "projects/123/thumbnail.jpg",
        "bucket": "nexflux-thumbnails",
        "size": 102400,
        "type": "image/jpeg"
    }
}
```

**Error Response:**
```json
{
    "success": false,
    "message": "File type not allowed",
    "code": "INVALID_FILE_TYPE"
}
```

---

### 2. Delete File

**Endpoint:** `DELETE /api/v1/upload`

**Description:** Delete a file from storage.

**Headers:**
```
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**
```json
{
    "bucket": "nexflux-thumbnails",
    "path": "projects/123/thumbnail.jpg"
}
```

**Success Response:**
```json
{
    "success": true,
    "message": "File deleted successfully"
}
```

---

### 3. Upload Avatar (Convenience Endpoint)

**Endpoint:** `POST /api/v1/users/me/avatar`

**Description:** Upload avatar for current user (uses `nexflux-avatars` bucket).

**Headers:**
```
Authorization: Bearer <token>
Content-Type: multipart/form-data
```

**Request Body (FormData):**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `avatar` | File | Yes | The avatar image file |

**Success Response:**
```json
{
    "success": true,
    "avatar_url": "https://xxx.supabase.co/storage/v1/object/public/nexflux-avatars/user-123/avatar.jpg"
}
```

---

## Frontend Upload Service

The frontend uses a centralized upload service located at `src/services/upload.ts`.

### Available Functions:

```typescript
import { 
    uploadFile,
    uploadAvatar,
    uploadProjectThumbnail,
    uploadProjectFile,
    uploadComponentImage,
    uploadDocument,
    deleteFile,
    validateFile,
    STORAGE_BUCKETS 
} from '../services/upload';

// Upload avatar
const result = await uploadAvatar(file, userId, (progress) => {
    console.log(`Upload progress: ${progress}%`);
});

// Upload project thumbnail
const result = await uploadProjectThumbnail(file, projectId);

// Upload component image
const result = await uploadComponentImage(file, componentId);

// General file upload
const result = await uploadFile({
    bucket: STORAGE_BUCKETS.PROJECTS,
    file: myFile,
    path: 'custom/path/file.json',
    onProgress: (progress) => console.log(progress)
});

// Validate file before upload
const validation = validateFile(file, STORAGE_BUCKETS.AVATARS);
if (!validation.valid) {
    console.error(validation.error);
}
```

### Upload Result:
```typescript
interface UploadResult {
    success: boolean;
    url?: string;      // Public URL of uploaded file
    path?: string;     // Path within bucket
    error?: string;    // Error message if failed
}
```

---

## Backend Implementation Notes

### Supabase Storage Setup

1. Create buckets in Supabase Dashboard ✅
2. Set bucket policies for public access (thumbnails, components) or authenticated access (documents, projects)
3. Configure CORS settings for file uploads

### Required Environment Variables:

```env
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_SERVICE_KEY=your-service-key
SUPABASE_ANON_KEY=your-anon-key
```

### Backend Upload Handler Example (Go):

```go
func (h *Handler) UploadFile(c *fiber.Ctx) error {
    file, err := c.FormFile("file")
    if err != nil {
        return c.Status(400).JSON(fiber.Map{"error": "No file provided"})
    }

    bucket := c.FormValue("bucket")
    path := c.FormValue("path")

    // Validate bucket
    validBuckets := []string{"nexflux-avatars", "nexflux-components", "nexflux-documents", "nexflux-projects", "nexflux-thumbnails"}
    if !contains(validBuckets, bucket) {
        return c.Status(400).JSON(fiber.Map{"error": "Invalid bucket"})
    }

    // Upload to Supabase Storage
    url, err := h.storage.Upload(bucket, path, file)
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "Upload failed"})
    }

    return c.JSON(fiber.Map{
        "success": true,
        "data": fiber.Map{
            "url":  url,
            "path": path,
        },
    })
}
```

---

## Security Considerations

1. **File Type Validation**: Always validate file MIME types on both frontend and backend
2. **File Size Limits**: Enforce max file sizes per bucket
3. **Path Sanitization**: Sanitize file paths to prevent directory traversal
4. **Authentication**: All upload endpoints require valid JWT token
5. **Rate Limiting**: Implement rate limiting for upload endpoints
6. **Virus Scanning**: Consider implementing virus scanning for uploaded files
