# Savile — Core API & Zero-Knowledge Architecture Plan

> **Reviewed & rewritten by senior architect. Original draft by Gemini 3.6/3.1 Pro.**

---

## 1. Zero-Knowledge Strategy

* **On-Device Preferences:** User choices (color palettes, top 3 font styles, custom widget layouts) are encrypted end-to-end (E2EE) on the iOS client.
* **Zero Database Visibility:** The backend database contains **zero tables** tracking user-created wallpaper layouts, user design configurations, or any personally identifiable information.
* **Asset-Driven Filtering:** The database only manages system assets (wallpapers and widgets). Matching happens locally on-device or via ephemeral, client-side preference hashing through the Python data engine.
* **Stateless API:** The backend acts as a pure content delivery layer — no sessions, no auth tokens, no user tracking.

> **Note:** As per Odytom's rules, strictly NO MongoDB. We are using PostgreSQL (recommended) containerized via Docker.

---

## 2. Relational Database Schema (PostgreSQL)

### Table: `wallpapers`

The core asset table. Stores **only** public system wallpapers — zero user data.

| Column         | Type            | Constraints                      | Description                                           |
|----------------|-----------------|----------------------------------|-------------------------------------------------------|
| `id`           | `UUID`          | PK, DEFAULT `gen_random_uuid()`  | Unique asset identifier                               |
| `asset_url`    | `VARCHAR(512)`  | NOT NULL                         | CDN URL for full-resolution wallpaper                 |
| `thumbnail_url`| `VARCHAR(512)`  | NOT NULL                         | Compressed preview image URL                          |
| `aspect_ratio` | `VARCHAR(10)`   | NOT NULL, DEFAULT `'9:19.5'`     | Screen ratio: `9:16`, `9:19.5`, `3:4` (iPad)         |
| `width_px`     | `INTEGER`       | NOT NULL                         | Source width in pixels                                |
| `height_px`    | `INTEGER`       | NOT NULL                         | Source height in pixels                               |
| `is_active`    | `BOOLEAN`       | NOT NULL, DEFAULT `true`         | Soft-delete / unpublish flag                          |
| `sort_order`   | `INTEGER`       | DEFAULT `0`                      | Manual ordering for featured content                  |
| `created_at`   | `TIMESTAMPTZ`   | NOT NULL, DEFAULT `NOW()`        | Upload timestamp                                      |
| `updated_at`   | `TIMESTAMPTZ`   | NOT NULL, DEFAULT `NOW()`        | Last modification (triggers client delta sync)        |

**Index:** `idx_wallpapers_active_updated` on `(is_active, updated_at DESC)`

---

### Table: `tags`

A flat, reusable tag system for both wallpapers and widgets.

| Column     | Type           | Constraints         | Description                                        |
|------------|----------------|---------------------|----------------------------------------------------|
| `id`       | `SERIAL`       | PK                  | Auto-increment ID                                  |
| `slug`     | `VARCHAR(50)`  | UNIQUE, NOT NULL    | URL-safe identifier: `minimalist`, `dark-mode`     |
| `label`    | `VARCHAR(100)` | NOT NULL            | Human-readable display name                        |
| `category` | `VARCHAR(30)`  | NOT NULL            | Grouping: `style`, `mood`, `scene`, `texture`      |

**Seed examples:** `(minimalist, style)`, `(dark-mode, mood)`, `(nature, scene)`, `(gradient, texture)`, `(abstract, style)`

---

### Table: `wallpaper_tags` (join — many-to-many)

A wallpaper can be `minimalist` + `dark-mode` + `nature` simultaneously.

| Column         | Type      | Constraints                                  |
|----------------|-----------|----------------------------------------------|
| `wallpaper_id` | `UUID`    | FK → `wallpapers.id` ON DELETE CASCADE       |
| `tag_id`       | `INTEGER` | FK → `tags.id` ON DELETE CASCADE             |

**Primary Key:** `(wallpaper_id, tag_id)`

---

### Table: `colors`

Normalized color lookup for palette-based filtering.

| Column     | Type          | Constraints         | Description                                        |
|------------|---------------|---------------------|----------------------------------------------------|
| `id`       | `SERIAL`      | PK                  | Auto-increment ID                                  |
| `hex_code` | `VARCHAR(7)`  | UNIQUE, NOT NULL    | e.g., `#FF5733`                                    |
| `name`     | `VARCHAR(50)` | NOT NULL            | Human-readable: `Coral`, `Midnight Blue`           |
| `warmth`   | `VARCHAR(10)` | NOT NULL            | `warm`, `cool`, `neutral` — enables palette matching |

---

### Table: `wallpaper_colors` (join — many-to-many)

A wallpaper's dominant color palette (extracted during upload).

| Column         | Type       | Constraints                                  |
|----------------|------------|----------------------------------------------|
| `wallpaper_id` | `UUID`     | FK → `wallpapers.id` ON DELETE CASCADE       |
| `color_id`     | `INTEGER`  | FK → `colors.id` ON DELETE CASCADE           |
| `dominance`    | `SMALLINT` | `1` = primary, `2` = secondary, `3` = accent |

**Primary Key:** `(wallpaper_id, color_id)`

---

### Table: `widgets`

Widget **templates** only. No user configuration data stored here.

| Column            | Type           | Constraints                      | Description                                              |
|-------------------|----------------|----------------------------------|----------------------------------------------------------|
| `id`              | `UUID`         | PK, DEFAULT `gen_random_uuid()`  | Unique widget identifier                                 |
| `name`            | `VARCHAR(100)` | NOT NULL                         | Display name: `Minimal Clock`, `Quote Card`              |
| `widget_type`     | `VARCHAR(30)`  | NOT NULL                         | Category: `clock`, `weather`, `calendar`, `quote`, `shortcut` |
| `preview_url`     | `VARCHAR(512)` |                                  | Preview image showing default widget appearance          |
| `config_template` | `JSONB`        | NOT NULL                         | Structural template with default fonts, layout, slots    |
| `is_active`       | `BOOLEAN`      | NOT NULL, DEFAULT `true`         | Soft-delete / unpublish flag                             |
| `sort_order`      | `INTEGER`      | DEFAULT `0`                      | Manual ordering                                          |
| `created_at`      | `TIMESTAMPTZ`  | NOT NULL, DEFAULT `NOW()`        | Upload timestamp                                         |
| `updated_at`      | `TIMESTAMPTZ`  | NOT NULL, DEFAULT `NOW()`        | Last modification                                        |

**Index:** `idx_widgets_type_active` on `(widget_type, is_active)`

---

### Table: `widget_tags` (join — many-to-many)

| Column      | Type      | Constraints                              |
|-------------|-----------|------------------------------------------|
| `widget_id` | `UUID`    | FK → `widgets.id` ON DELETE CASCADE      |
| `tag_id`    | `INTEGER` | FK → `tags.id` ON DELETE CASCADE         |

**Primary Key:** `(widget_id, tag_id)`

---

### Table: `collections` *(optional — discuss with Jerry)*

Curated wallpaper packs like "Midnight Elegance" or "Summer Vibes".

| Column        | Type           | Constraints                      | Description                     |
|---------------|----------------|----------------------------------|---------------------------------|
| `id`          | `UUID`         | PK, DEFAULT `gen_random_uuid()`  | Collection identifier           |
| `slug`        | `VARCHAR(100)` | UNIQUE, NOT NULL                 | URL-safe: `midnight-elegance`   |
| `title`       | `VARCHAR(200)` | NOT NULL                         | Display title                   |
| `description` | `TEXT`         |                                  | Short description for iOS UI    |
| `cover_url`   | `VARCHAR(512)` |                                  | Cover image for collection card |
| `is_active`   | `BOOLEAN`      | NOT NULL, DEFAULT `true`         | Publish flag                    |
| `sort_order`  | `INTEGER`      | DEFAULT `0`                      | Display ordering                |
| `created_at`  | `TIMESTAMPTZ`  | NOT NULL, DEFAULT `NOW()`        | Created timestamp               |

---

### Table: `collection_items` (join)

| Column          | Type      | Constraints                                  |
|-----------------|-----------|----------------------------------------------|
| `collection_id` | `UUID`    | FK → `collections.id` ON DELETE CASCADE      |
| `wallpaper_id`  | `UUID`    | FK → `wallpapers.id` ON DELETE CASCADE       |
| `position`      | `INTEGER` | Order within the collection                  |

**Primary Key:** `(collection_id, wallpaper_id)`

---

## 3. Core API Endpoints (Express or Laravel)

### Public Asset Feeds

| Method | Endpoint                        | Query Params                                                    | Description                              |
|--------|---------------------------------|-----------------------------------------------------------------|------------------------------------------|
| `GET`  | `/api/v1/wallpapers`            | `?tags=minimalist,dark-mode&warmth=cool&page=1&per_page=20&sort=newest` | Paginated, filtered wallpaper catalog    |
| `GET`  | `/api/v1/wallpapers/:id`        | —                                                               | Single wallpaper with full metadata      |
| `GET`  | `/api/v1/widgets`               | `?type=clock&tags=minimalist&page=1&per_page=20`               | Paginated widget templates               |
| `GET`  | `/api/v1/widgets/:id`           | —                                                               | Single widget with full config template  |
| `GET`  | `/api/v1/collections`           | `?page=1&per_page=10`                                          | Browse curated collections               |
| `GET`  | `/api/v1/collections/:slug`     | —                                                               | Collection detail with wallpapers        |
| `GET`  | `/api/v1/tags`                  | `?category=style`                                               | List all available tags (for filter UI)  |
| `GET`  | `/api/v1/colors`                | —                                                               | List all colors (for palette matching)   |

### Delta Sync (Critical for Mobile)

| Method | Endpoint         | Query Params                                      | Description                                              |
|--------|------------------|---------------------------------------------------|----------------------------------------------------------|
| `GET`  | `/api/v1/sync`   | `?since=2026-07-20T00:00:00Z&type=wallpapers`    | Returns only assets modified after the `since` timestamp |

**Sync Response Shape:**
```json
{
  "data": [ /* new/updated assets */ ],
  "deleted_ids": [ /* UUIDs of deactivated assets */ ],
  "sync_timestamp": "2026-07-23T16:00:00Z"
}
```

The client stores `sync_timestamp` locally and passes it as `?since=` on the next sync call.

### Health & Meta

| Method | Endpoint          | Description                                      |
|--------|-------------------|--------------------------------------------------|
| `GET`  | `/api/v1/health`  | Basic health check for Docker / monitoring       |
| `GET`  | `/api/v1/meta`    | API version, total asset counts, last update time|

---

## 4. Zero-Knowledge Compliance Checklist

- [x] No `users` table
- [x] No `user_preferences` table
- [x] No `sessions` / `auth_tokens` table
- [x] No user ID foreign keys anywhere in the schema
- [x] No endpoint accepts or returns user-specific data
- [x] API is fully stateless — acts as a pure content delivery layer
- [x] All personalization happens client-side via the Python matching engine or on-device E2EE logic

---

## 5. Next Steps for Core API Team (Aiso & Super Jerry)

1. **Agree on Framework:** Decide between Express (Node.js) or Laravel (PHP) for the REST API.
2. **Confirm PostgreSQL:** Verify both team members are comfortable with PostgreSQL over MySQL.
3. **Docker Setup:** Create `docker-compose.yml` for the API server + PostgreSQL database.
4. **Write SQL Migrations:** Convert the schema tables above into numbered migration files in `/backend/migrations/`.
5. **Integration with Python Engine:** Coordinate with Sheriff & Yazn on how the Python preference-matching script interfaces with the API (REST calls vs. shared DB connection).
6. **OpenAPI Spec:** Create `openapi.yaml` in `/backend/docs/` documenting the full API contract.
