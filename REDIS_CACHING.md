# Redis Caching Implementation Summary

Dokumen ini menjelaskan implementasi Redis caching pada NexFlux backend untuk mengurangi beban database.

## 📋 Overview

Redis caching diimplementasikan untuk:
1. **Mengurangi database hits** - Data yang sering diakses di-cache
2. **Meningkatkan response time** - Cache hit lebih cepat dari database query
3. **Mengurangi server load** - Lebih sedikit query ke PostgreSQL

---

## 🗂️ File Yang Dimodifikasi/Dibuat

### File Baru:
| File | Deskripsi |
|------|-----------|
| `pkg/redis/cache.go` | Cache helper dengan key builders, TTL configurations, dan invalidation helpers |

### File Dimodifikasi:
| File | Perubahan |
|------|-----------|
| `api/services/acl.service.go` | Caching untuk user ACL profile, subscription tiers |
| `api/services/component.service.go` | Caching untuk component categories, component detail |
| `api/services/documentation.service.go` | Caching untuk doc categories, popular/featured articles |
| `api/services/gamification.service.go` | Caching untuk leaderboard, user streak, user stats |
| `api/services/lab.service.go` | Caching untuk lab detail, lab by slug |
| `api/services/user_settings.service.go` | Caching untuk user notification, appearance, privacy settings |

---

## 📊 Cached Endpoints Summary

### 🔐 ACL Module
| Endpoint | Cache Key | TTL | Keterangan |
|----------|-----------|-----|------------|
| `GET /api/v1/users/me` (ACL) | `user:acl:{userID}` | 2 min | User profile dengan subscription & feature info |
| `GET /api/v1/subscription/tiers` | `acl:subscription_tiers` | 1 hour | Subscription tiers (rarely changes) |
| `GET /api/v1/users/me/settings` | `user:settings:{userID}` | 2 min | User notification, appearance, & privacy settings |

### 🔧 Component Module
| Endpoint | Cache Key | TTL | Keterangan |
|----------|-----------|-----|------------|
| `GET /api/v1/components/categories` | `component:categories` | 1 hour | Component categories |
| `GET /api/v1/components/:id` | `component:detail:{id}` | 30 min | Component detail (favorite status added on retrieval) |

### 📚 Documentation Module
| Endpoint | Cache Key | TTL | Keterangan |
|----------|-----------|-----|------------|
| `GET /api/v1/docs/categories` | `doc:categories` | 1 hour | Documentation categories |
| `GET /api/v1/docs/popular` | `doc:popular:{limit}` | 15 min | Popular articles (view count based) |
| `GET /api/v1/docs/featured` | `doc:featured:{limit}` | 1 hour | Featured articles |

### 🏆 Gamification Module
| Endpoint | Cache Key | TTL | Keterangan |
|----------|-----------|-----|------------|
| `GET /api/v1/leaderboard` | `leaderboard:{type}` | 5 min | Leaderboard entries |
| `GET /api/v1/streak` | `user:streak:{userID}` | 5 min | User streak info |
| `GET /api/v1/users/:id/stats` | `user:stats:{userID}` | 2 min | Comprehensive user stats |

### 🧪 Lab Module
| Endpoint | Cache Key | TTL | Keterangan |
|----------|-----------|-----|------------|
| `GET /api/v1/labs/:id` | `lab:detail:{labID}` | 10 min | Lab detail (dynamic fields updated on cache hit) |
| `GET /api/v1/labs/slug/:slug` | `lab:slug:{slug}` | 10 min | Lab by slug |

---

## ⏱️ Cache TTL Configuration

### Short-lived (1-5 minutes) - Frequently changing data
```go
UserProfile:    2 * time.Minute
UserStats:      2 * time.Minute
UserStreak:     5 * time.Minute
LabList:        1 * time.Minute
LabQueue:       30 * time.Second
LabSensorData:  5 * time.Second  // Real-time data
Leaderboard:    5 * time.Minute
DailyChallenge: 15 * time.Minute
UserSession:    1 * time.Minute
```

### Medium-lived (10-30 minutes) - Moderately changing data
```go
LabDetail:        10 * time.Minute
ChallengeList:    15 * time.Minute
ChallengeDetail:  15 * time.Minute
UserAchievements: 10 * time.Minute
ComponentList:    30 * time.Minute
ComponentDetail:  30 * time.Minute
```

### Long-lived (1+ hour) - Rarely changing data
```go
ComponentCategories: 1 * time.Hour
DocCategories:       1 * time.Hour
DocArticle:          1 * time.Hour
DocVideoList:        1 * time.Hour
FeatureGates:        5 * time.Minute
SubscriptionTiers:   1 * time.Hour
AllAchievements:     1 * time.Hour
CircuitTemplates:    1 * time.Hour
```

---

## 🔄 Cache Invalidation

### Per-Module Invalidation Functions
```go
// User-related caches
redis.InvalidateUserCaches(userID)

// Lab caches (when lab data changes)
redis.InvalidateLabCaches(labID)
redis.InvalidateLabSlugCache(slug)

// Component caches
redis.InvalidateComponentCaches()

// Documentation caches
redis.InvalidateDocCaches()

// Challenge caches
redis.InvalidateChallengeCaches()

// Leaderboard caches
redis.InvalidateLeaderboardCache(leaderboardType)
redis.InvalidateAllLeaderboards()

// Achievement caches
redis.InvalidateAchievementsCaches()

// ACL caches
redis.InvalidateACLCaches()
```

---

## 🔑 Cache Key Patterns

```
# User-related
user:profile:{userID}
user:acl:{userID}
user:stats:{userID}
user:streak:{userID}
user:achievements:{userID}
user:settings:{userID}

# ACL
acl:feature_gates
acl:subscription_tiers
acl:subscription:{userID}
acl:session:{userID}

# Lab
lab:list:{platform}:{status}:{page}
lab:detail:{labID}
lab:slug:{slug}
lab:queue:{labID}
lab:sensors:{labID}

# Component
component:categories
component:list:{category}:{page}
component:detail:{componentID}

# Documentation
doc:categories
doc:category:{slug}
doc:article:{slug}
doc:articles:{category}:{page}
doc:popular:{limit}
doc:featured:{limit}
doc:videos:{category}:{page}
doc:video:{videoID}

# Challenge
challenge:list:{type}:{difficulty}:{page}
challenge:detail:{challengeID}
challenge:daily
challenge:progress:{userID}

# Gamification
leaderboard:{type}
achievements:all

# Circuit
circuit:templates
```

---

## ⚠️ Data Yang TIDAK Di-cache

Data berikut **TIDAK** di-cache karena bersifat real-time atau personalized:

1. **Lab Session Status** - Status sesi berubah terus-menerus
2. **Queue Position** - Posisi antrian berubah saat ada user join/leave
3. **Sensor Data** - Data sensor real-time
4. **Active Session Info** - Status sesi aktif
5. **User Favorites** - Di-add saat retrieve, bukan di-cache
6. **Lab Availability** - Status ketersediaan lab
7. **Search Results** - Terlalu variatif untuk di-cache
8. **Audit Logs** - Harus selalu fresh

---

## 🎯 Best Practices Implemented

1. **Cache-Aside Pattern** - Check cache first, fetch from DB on miss
2. **Async Cache Updates** - Use `go redis.SetJSON()` to not block response
3. **Dynamic Field Updates** - Update volatile fields (status, count) on cache hit
4. **Separate User-Specific Data** - Cache base data, add user-specific on retrieval
5. **Short TTL for Dynamic Data** - Shorter TTL for frequently changing data
6. **Pattern-Based Invalidation** - Use patterns for bulk invalidation

---

## 📈 Expected Performance Improvement

| Scenario | Without Cache | With Cache | Improvement |
|----------|---------------|------------|-------------|
| Get Subscription Tiers | ~50ms | ~1ms | 50x faster |
| Get Component Categories | ~30ms | ~1ms | 30x faster |
| Get Leaderboard | ~100ms | ~5ms | 20x faster |
| Get User Stats | ~80ms | ~2ms | 40x faster |
| Get Doc Categories | ~40ms | ~1ms | 40x faster |

---

## 🛠️ Maintenance Notes

### Adding New Cached Endpoints
1. Add cache key to `pkg/redis/cache.go` → `CacheKeys`
2. Add TTL to `pkg/redis/cache.go` → `CacheTTL`
3. Implement cache logic in service layer
4. Add invalidation logic where data is modified

### Monitoring Cache Performance
Use Redis CLI commands:
```bash
# Check cache hit rate
redis-cli INFO stats | grep keyspace

# Monitor cache keys
redis-cli KEYS "cache:*"

# Check memory usage
redis-cli INFO memory
```

---

## 🔒 Security Considerations

1. **No sensitive data cached without encryption**
2. **User-specific data keyed by userID**
3. **Cache invalidation on permission changes**
4. **TTL ensures stale data expires**
