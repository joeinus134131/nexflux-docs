# Community Forum - Backend Requirements

## Overview

Dokumen ini berisi spesifikasi API backend yang dibutuhkan untuk fitur **Community Forum** di NexFlux. Forum ini mirip dengan Stack Overflow, memungkinkan user untuk tanya jawab, diskusi, dan berbagi pengetahuan seputar IoT, Arduino, dan embedded systems.

---

## Database Schema

### 1. `forum_posts` - Tabel Posts

```sql
CREATE TABLE forum_posts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    content TEXT NOT NULL,
    content_html TEXT, -- Rendered markdown content
    category VARCHAR(50) NOT NULL CHECK (category IN ('question', 'discussion', 'tutorial', 'bug-report', 'showcase')),
    votes INT DEFAULT 0,
    views INT DEFAULT 0,
    answers_count INT DEFAULT 0,
    has_accepted_answer BOOLEAN DEFAULT FALSE,
    is_featured BOOLEAN DEFAULT FALSE,
    is_pinned BOOLEAN DEFAULT FALSE,
    is_closed BOOLEAN DEFAULT FALSE,
    is_deleted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    last_activity_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_forum_posts_user ON forum_posts(user_id);
CREATE INDEX idx_forum_posts_category ON forum_posts(category);
CREATE INDEX idx_forum_posts_created ON forum_posts(created_at DESC);
CREATE INDEX idx_forum_posts_votes ON forum_posts(votes DESC);
CREATE INDEX idx_forum_posts_last_activity ON forum_posts(last_activity_at DESC);
```

### 2. `forum_answers` - Tabel Answers

```sql
CREATE TABLE forum_answers (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    post_id UUID NOT NULL REFERENCES forum_posts(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    content TEXT NOT NULL,
    content_html TEXT,
    votes INT DEFAULT 0,
    is_accepted BOOLEAN DEFAULT FALSE,
    is_deleted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_forum_answers_post ON forum_answers(post_id);
CREATE INDEX idx_forum_answers_user ON forum_answers(user_id);
CREATE INDEX idx_forum_answers_votes ON forum_answers(votes DESC);
```

### 3. `forum_tags` - Tabel Tags

```sql
CREATE TABLE forum_tags (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(50) NOT NULL UNIQUE,
    slug VARCHAR(50) NOT NULL UNIQUE,
    color VARCHAR(7) DEFAULT '#6366F1', -- Hex color
    description TEXT,
    usage_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_forum_tags_name ON forum_tags(name);
CREATE INDEX idx_forum_tags_usage ON forum_tags(usage_count DESC);
```

### 4. `forum_post_tags` - Junction Table Posts-Tags

```sql
CREATE TABLE forum_post_tags (
    post_id UUID NOT NULL REFERENCES forum_posts(id) ON DELETE CASCADE,
    tag_id UUID NOT NULL REFERENCES forum_tags(id) ON DELETE CASCADE,
    PRIMARY KEY (post_id, tag_id)
);

CREATE INDEX idx_post_tags_post ON forum_post_tags(post_id);
CREATE INDEX idx_post_tags_tag ON forum_post_tags(tag_id);
```

### 5. `forum_votes` - Tabel Votes (untuk tracking siapa yang vote)

```sql
CREATE TABLE forum_votes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    post_id UUID REFERENCES forum_posts(id) ON DELETE CASCADE,
    answer_id UUID REFERENCES forum_answers(id) ON DELETE CASCADE,
    vote_type SMALLINT NOT NULL CHECK (vote_type IN (-1, 1)), -- -1 for downvote, 1 for upvote
    created_at TIMESTAMP DEFAULT NOW(),
    CONSTRAINT vote_target CHECK (
        (post_id IS NOT NULL AND answer_id IS NULL) OR 
        (post_id IS NULL AND answer_id IS NOT NULL)
    ),
    UNIQUE(user_id, post_id),
    UNIQUE(user_id, answer_id)
);
```

### 6. `forum_bookmarks` - Saved Posts

```sql
CREATE TABLE forum_bookmarks (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    post_id UUID NOT NULL REFERENCES forum_posts(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (user_id, post_id)
);
```

### 7. `forum_comments` - Comments pada Answers (optional)

```sql
CREATE TABLE forum_comments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    answer_id UUID NOT NULL REFERENCES forum_answers(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    content TEXT NOT NULL,
    is_deleted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);
```

---

## API Endpoints

### Base URL: `/api/v1/community`

---

### 1. Posts

#### `GET /posts` - List Posts

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | int | 1 | Page number |
| `limit` | int | 20 | Items per page (max 50) |
| `sort` | string | "hot" | Sort by: `hot`, `new`, `top`, `unanswered` |
| `category` | string | null | Filter by category |
| `tag` | string | null | Filter by tag slug |
| `search` | string | null | Search in title and content |
| `user_id` | uuid | null | Filter by author |

**Response:**
```json
{
  "success": true,
  "data": {
    "posts": [
      {
        "id": "uuid",
        "title": "string",
        "content_preview": "string (first 200 chars)",
        "category": "question|discussion|tutorial|bug-report|showcase",
        "author": {
          "id": "uuid",
          "name": "string",
          "avatar_url": "string|null",
          "level": 15,
          "reputation": 2450,
          "badges": { "gold": 3, "silver": 12, "bronze": 28 }
        },
        "tags": [
          { "id": "uuid", "name": "Arduino", "slug": "arduino", "color": "#00979D" }
        ],
        "votes": 42,
        "answers_count": 8,
        "views": 1250,
        "has_accepted_answer": true,
        "is_featured": false,
        "created_at": "2026-01-12T10:30:00Z",
        "last_activity_at": "2026-01-13T05:15:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total_items": 150,
      "total_pages": 8
    }
  }
}
```

---

#### `GET /posts/:id` - Get Post Detail

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "title": "string",
    "content": "string (full markdown)",
    "content_html": "string (rendered HTML)",
    "category": "question",
    "author": { ... },
    "tags": [ ... ],
    "votes": 42,
    "views": 1251,
    "has_accepted_answer": true,
    "is_closed": false,
    "created_at": "2026-01-12T10:30:00Z",
    "updated_at": "2026-01-12T10:30:00Z",
    "user_vote": 1, // Current user's vote: -1, 0, or 1
    "is_bookmarked": false,
    "answers": [
      {
        "id": "uuid",
        "content": "string",
        "content_html": "string",
        "author": { ... },
        "votes": 15,
        "is_accepted": true,
        "user_vote": 0,
        "created_at": "2026-01-12T11:00:00Z"
      }
    ]
  }
}
```

---

#### `POST /posts` - Create Post

**Request Body:**
```json
{
  "title": "string (required, max 255)",
  "content": "string (required, markdown)",
  "category": "question|discussion|tutorial|bug-report|showcase (required)",
  "tags": ["tag-slug-1", "tag-slug-2"] // max 5 tags
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "title": "string",
    ...
  },
  "message": "Post created successfully"
}
```

**XP Reward:** +10 XP untuk membuat post

---

#### `PUT /posts/:id` - Update Post

**Request Body:**
```json
{
  "title": "string",
  "content": "string",
  "category": "string",
  "tags": ["tag-slug-1"]
}
```

**Authorization:** Hanya author atau admin

---

#### `DELETE /posts/:id` - Delete Post (Soft Delete)

**Authorization:** Hanya author atau admin

---

#### `POST /posts/:id/vote` - Vote Post

**Request Body:**
```json
{
  "vote": 1  // 1 for upvote, -1 for downvote, 0 to remove vote
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "votes": 43,
    "user_vote": 1
  }
}
```

**XP Reward:** 
- +2 XP untuk post author jika upvote
- -1 XP untuk post author jika downvote

---

#### `POST /posts/:id/bookmark` - Bookmark/Unbookmark Post

**Response:**
```json
{
  "success": true,
  "data": {
    "is_bookmarked": true
  }
}
```

---

#### `POST /posts/:id/close` - Close Post (Mark as Resolved)

**Authorization:** Hanya author atau admin

---

### 2. Answers

#### `POST /posts/:post_id/answers` - Create Answer

**Request Body:**
```json
{
  "content": "string (required, markdown)"
}
```

**XP Reward:** +5 XP untuk memberikan jawaban

---

#### `PUT /answers/:id` - Update Answer

**Authorization:** Hanya author atau admin

---

#### `DELETE /answers/:id` - Delete Answer

**Authorization:** Hanya author atau admin

---

#### `POST /answers/:id/vote` - Vote Answer

**Request Body:**
```json
{
  "vote": 1
}
```

---

#### `POST /answers/:id/accept` - Accept Answer

**Authorization:** Hanya post author

**Response:**
```json
{
  "success": true,
  "message": "Answer accepted"
}
```

**XP Reward:** 
- +15 XP untuk answer author ketika accepted
- +2 XP untuk post author

---

### 3. Tags

#### `GET /tags` - List All Tags

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | int | 20 | Number of tags |
| `sort` | string | "popular" | Sort by: `popular`, `name`, `new` |

**Response:**
```json
{
  "success": true,
  "data": {
    "tags": [
      {
        "id": "uuid",
        "name": "Arduino",
        "slug": "arduino",
        "color": "#00979D",
        "description": "Arduino microcontroller platform",
        "usage_count": 1250
      }
    ]
  }
}
```

---

#### `GET /tags/:slug` - Get Tag Detail with Posts

**Response:**
```json
{
  "success": true,
  "data": {
    "tag": { ... },
    "posts": [ ... ],
    "pagination": { ... }
  }
}
```

---

### 4. Users/Contributors

#### `GET /users/top` - Top Contributors

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `period` | string | "all" | Period: `week`, `month`, `year`, `all` |
| `limit` | int | 10 | Number of users |

**Response:**
```json
{
  "success": true,
  "data": {
    "users": [
      {
        "id": "uuid",
        "name": "string",
        "avatar_url": "string|null",
        "level": 22,
        "reputation": 5670,
        "badges": { "gold": 8, "silver": 24, "bronze": 45 },
        "posts_count": 45,
        "answers_count": 128,
        "accepted_answers_count": 67
      }
    ]
  }
}
```

---

#### `GET /users/:id/posts` - User's Posts

Same format as `GET /posts` with automatic `user_id` filter

---

#### `GET /users/me/bookmarks` - Current User's Bookmarks

Same format as `GET /posts`

---

### 5. Statistics

#### `GET /stats` - Forum Statistics

**Response:**
```json
{
  "success": true,
  "data": {
    "total_posts": 12500,
    "total_answers": 45000,
    "total_users": 8200,
    "resolved_rate": 92,
    "posts_today": 45,
    "answers_today": 156
  }
}
```

---

## Gamification Integration

### XP Rewards

| Action | XP Earned |
|--------|-----------|
| Create a post | +10 |
| Answer a question | +5 |
| Answer accepted | +15 |
| Post gets upvoted | +2 |
| Answer gets upvoted | +2 |
| Post gets downvoted | -1 |
| Answer gets downvoted | -1 |

### Reputation System

Reputation dihitung dari:
- Total upvotes dari posts × 5
- Total upvotes dari answers × 10
- Total accepted answers × 15
- Total downvotes × -2

### Badges (Achievements)

| Badge | Criteria | Type |
|-------|----------|------|
| **First Question** | Ask first question | Bronze |
| **First Answer** | Post first answer | Bronze |
| **Helper** | 10 answers | Bronze |
| **Curious Mind** | 10 questions | Bronze |
| **Problem Solver** | 5 accepted answers | Silver |
| **Expert Helper** | 25 accepted answers | Silver |
| **Community Star** | 100 upvotes received | Silver |
| **Knowledge Guru** | 50 accepted answers | Gold |
| **Legend** | 1000 reputation | Gold |

---

## Real-time Features (Optional - WebSocket)

### Events

```typescript
// Subscribe to post updates
socket.on('post:updated', (data) => { ... })
socket.on('post:new_answer', (data) => { ... })
socket.on('answer:accepted', (data) => { ... })

// Subscribe to user notifications
socket.on('notification:new', (data) => { ... })
```

---

## Notifications

Trigger notifikasi untuk:
1. **Answer received** - Seseorang menjawab pertanyaan kamu
2. **Answer accepted** - Jawaban kamu diterima
3. **Mentioned** - Seseorang mention @username kamu
4. **Reply** - Seseorang reply comment kamu
5. **Upvote milestone** - Post/Answer mencapai milestone votes (10, 25, 50, 100)

---

## Search

Implementasi full-text search menggunakan PostgreSQL:

```sql
-- Add search vector column
ALTER TABLE forum_posts ADD COLUMN search_vector tsvector;

-- Create trigger to update search vector
CREATE FUNCTION update_post_search_vector() RETURNS trigger AS $$
BEGIN
  NEW.search_vector := 
    setweight(to_tsvector('indonesian', COALESCE(NEW.title, '')), 'A') ||
    setweight(to_tsvector('indonesian', COALESCE(NEW.content, '')), 'B');
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER post_search_vector_update
  BEFORE INSERT OR UPDATE ON forum_posts
  FOR EACH ROW EXECUTE FUNCTION update_post_search_vector();

-- Create GIN index
CREATE INDEX idx_forum_posts_search ON forum_posts USING GIN(search_vector);
```

---

## Rate Limiting

| Endpoint | Rate Limit |
|----------|------------|
| Create Post | 5 per hour |
| Create Answer | 30 per hour |
| Vote | 60 per hour |
| Search | 60 per minute |

---

## Content Moderation

### Spam Detection
- Detect duplicate content
- Rate limit new users
- Flag posts with excessive links

### Report System

```sql
CREATE TABLE forum_reports (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    reporter_id UUID NOT NULL REFERENCES users(id),
    post_id UUID REFERENCES forum_posts(id),
    answer_id UUID REFERENCES forum_answers(id),
    reason VARCHAR(50) NOT NULL, -- spam, inappropriate, off-topic, etc.
    description TEXT,
    status VARCHAR(20) DEFAULT 'pending', -- pending, reviewed, actioned, dismissed
    created_at TIMESTAMP DEFAULT NOW()
);
```

---

## Seed Data (Default Tags)

```sql
INSERT INTO forum_tags (name, slug, color, description) VALUES
('Arduino', 'arduino', '#00979D', 'Arduino microcontroller platform'),
('ESP32', 'esp32', '#E7352C', 'ESP32 WiFi/Bluetooth microcontroller'),
('Raspberry Pi', 'raspberry-pi', '#C51A4A', 'Raspberry Pi single-board computers'),
('IoT', 'iot', '#6366F1', 'Internet of Things projects'),
('Sensor', 'sensor', '#10B981', 'Various sensors and modules'),
('LED', 'led', '#F59E0B', 'LED and lighting projects'),
('Motor Control', 'motor-control', '#EC4899', 'DC motors, stepper, servo'),
('Power Supply', 'power-supply', '#8B5CF6', 'Power management and batteries'),
('PCB Design', 'pcb-design', '#14B8A6', 'PCB design and fabrication'),
('Programming', 'programming', '#3B82F6', 'Code and software development'),
('3D Printing', '3d-printing', '#F97316', '3D printing for makers'),
('MQTT', 'mqtt', '#22C55E', 'MQTT protocol and messaging'),
('Home Automation', 'home-automation', '#A855F7', 'Smart home projects'),
('Robotics', 'robotics', '#EF4444', 'Robot and automation projects');
```

---

## Frontend Integration Notes

### API Hook Example

```typescript
// hooks/useCommunity.ts
import api from './api';

export const useCommunityPosts = (params?: {
  page?: number;
  sort?: 'hot' | 'new' | 'top' | 'unanswered';
  category?: string;
  tag?: string;
  search?: string;
}) => {
  return useQuery(['community-posts', params], () => 
    api.get('/community/posts', { params })
  );
};

export const useCreatePost = () => {
  return useMutation((data: CreatePostDTO) =>
    api.post('/community/posts', data)
  );
};

export const useVotePost = () => {
  return useMutation(({ postId, vote }: { postId: string; vote: number }) =>
    api.post(`/community/posts/${postId}/vote`, { vote })
  );
};
```

---

## Error Codes

| Code | Message |
|------|---------|
| `COMMUNITY_001` | Post not found |
| `COMMUNITY_002` | Answer not found |
| `COMMUNITY_003` | Tag not found |
| `COMMUNITY_004` | You cannot vote your own post |
| `COMMUNITY_005` | You already voted this post |
| `COMMUNITY_006` | Rate limit exceeded |
| `COMMUNITY_007` | Post is closed |
| `COMMUNITY_008` | Unauthorized to perform this action |
| `COMMUNITY_009` | Invalid category |
| `COMMUNITY_010` | Too many tags (max 5) |

---

## Summary Checklist

### Phase 1 - Core Features
- [x] Database migrations
- [x] CRUD Posts
- [x] CRUD Answers
- [x] Voting system
- [x] Tags system
- [x] Basic search

### Phase 2 - Engagement
- [x] Accept answer
- [x] Bookmarks
- [x] User stats/profile
- [x] Top contributors
- [x] XP integration

### Phase 3 - Advanced
- [ ] Notifications
- [x] Full-text search
- [x] Report system
- [ ] Rate limiting
- [ ] Real-time updates (WebSocket)

---

*Document Version: 1.1*
*Last Updated: 2026-01-13*
*Backend Implementation: Complete (Phase 1 & 2, partial Phase 3)*
