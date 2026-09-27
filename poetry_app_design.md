# Poetry Sharing App — Design Document

A cross-platform (mobile + web) app where users write, upload, and store poems, and
browse profiles that show each poem as a **preview card** (2–3 lines + "Read more").

---

## 1. Tech Stack

| Layer            | Choice                                   | Why |
|-------------------|-------------------------------------------|-----|
| Client            | **Flutter / Dart**                        | Single codebase for iOS, Android, Web |
| Backend API       | **Python (FastAPI)**                      | Fast, async, auto-generates OpenAPI docs, plays well with Postgres via SQLAlchemy/asyncpg |
| Database          | **PostgreSQL**                            | Relational, strong text search (`tsvector`), good for tags/likes/comments later |
| Auth              | JWT (access + refresh tokens)             | Stateless, works cleanly with mobile clients |
| File storage      | Local disk (dev) → S3 / Cloud Storage (prod) | For avatars and optional scanned/handwritten poem images |
| State mgmt (Flutter) | Riverpod (or Provider/Bloc)            | Testable, avoids widget-tree coupling |

---

## 2. High-Level Architecture

```
┌────────────────┐        HTTPS/JSON        ┌───────────────────┐
│  Flutter App    │ ───────────────────────► │  FastAPI Backend   │
│ (iOS/Android/   │ ◄─────────────────────── │  (Python)          │
│  Web)           │                          │                    │
└────────────────┘                          └─────────┬──────────┘
                                                         │
                                              ┌──────────▼──────────┐
                                              │   PostgreSQL DB      │
                                              └───────────────────────┘
                                                         │
                                              ┌──────────▼──────────┐
                                              │  File/Object Storage │
                                              │ (avatars, images)    │
                                              └───────────────────────┘
```

---

## 3. Database Design (PostgreSQL)

```sql
-- Users
CREATE TABLE users (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username      VARCHAR(50)  UNIQUE NOT NULL,
    email         VARCHAR(255) UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    display_name  VARCHAR(100),
    bio           TEXT,
    avatar_url    TEXT,
    created_at    TIMESTAMPTZ DEFAULT now(),
    updated_at    TIMESTAMPTZ DEFAULT now()
);

-- Poems
CREATE TABLE poems (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id      UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title        VARCHAR(200) NOT NULL,
    content      TEXT NOT NULL,          -- full poem, newline-separated
    preview      TEXT GENERATED ALWAYS AS (
                     substring(content from 1 for 200)
                 ) STORED,                -- cheap fallback preview
    cover_image_url TEXT,                 -- optional uploaded scan/photo
    is_public    BOOLEAN DEFAULT TRUE,
    like_count   INT DEFAULT 0,
    created_at   TIMESTAMPTZ DEFAULT now(),
    updated_at   TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_poems_user_id ON poems(user_id);
CREATE INDEX idx_poems_created_at ON poems(created_at DESC);

-- Optional: tags
CREATE TABLE tags (
    id   SERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL
);

CREATE TABLE poem_tags (
    poem_id UUID REFERENCES poems(id) ON DELETE CASCADE,
    tag_id  INT REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (poem_id, tag_id)
);

-- Optional: likes
CREATE TABLE likes (
    user_id  UUID REFERENCES users(id) ON DELETE CASCADE,
    poem_id  UUID REFERENCES poems(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (user_id, poem_id)
);
```

> Note: `preview` as a generated column is a simple fallback. For a true
> "first 2–3 lines" preview (respecting line breaks, not just characters),
> compute it in the API layer instead — see §5.2.

---

## 4. Backend (FastAPI) Structure

```
backend/
├── app/
│   ├── main.py               # FastAPI app entrypoint
│   ├── core/
│   │   ├── config.py         # env vars, settings
│   │   └── security.py       # JWT, password hashing
│   ├── db/
│   │   ├── session.py        # async SQLAlchemy engine/session
│   │   └── models.py         # ORM models (User, Poem, Tag, Like)
│   ├── schemas/
│   │   ├── user.py           # Pydantic request/response models
│   │   └── poem.py
│   ├── api/
│   │   ├── routes_auth.py
│   │   ├── routes_users.py
│   │   └── routes_poems.py
│   └── services/
│       └── preview.py        # preview-generation logic
├── alembic/                  # DB migrations
├── requirements.txt
└── Dockerfile
```

### 4.1 Key Endpoints

| Method | Path                       | Purpose |
|--------|-----------------------------|---------|
| POST   | `/auth/register`            | Create account |
| POST   | `/auth/login`                | Get JWT tokens |
| POST   | `/auth/refresh`              | Refresh access token |
| GET    | `/users/{username}`          | Public profile info |
| PUT    | `/users/me`                  | Update own profile (bio, avatar) |
| POST   | `/poems`                     | Create a poem |
| GET    | `/poems/{poem_id}`           | Full poem detail |
| PUT    | `/poems/{poem_id}`           | Edit poem |
| DELETE | `/poems/{poem_id}`           | Delete poem |
| GET    | `/users/{username}/poems`    | Paginated list of preview cards |
| POST   | `/poems/{poem_id}/cover`     | Upload cover/scanned image |
| POST   | `/poems/{poem_id}/like`      | Like/unlike |

### 4.2 Preview generation (2–3 lines)

```python
# app/services/preview.py
def make_preview(content: str, max_lines: int = 3, max_chars: int = 160) -> str:
    lines = [ln for ln in content.strip().splitlines() if ln.strip()]
    preview_lines = lines[:max_lines]
    preview = "\n".join(preview_lines)
    if len(preview) > max_chars:
        preview = preview[:max_chars].rsplit(" ", 1)[0] + "…"
    elif len(lines) > max_lines:
        preview += "…"
    return preview
```

This is called when serializing poems for list/profile endpoints, so the
Flutter client just renders whatever string it receives — no truncation
logic needed on-device.

### 4.3 Example: `GET /users/{username}/poems` response

```json
{
  "items": [
    {
      "id": "b3f1...",
      "title": "Autumn Light",
      "preview": "The maples burn in amber flame,\nA hush falls where the sparrows came,\nAnd evening wears a quieter name.",
      "created_at": "2026-08-01T10:00:00Z",
      "like_count": 12
    }
  ],
  "page": 1,
  "page_size": 20,
  "total": 47
}
```

---

## 5. Flutter App Structure

```
flutter_app/
├── lib/
│   ├── main.dart
│   ├── models/
│   │   ├── user.dart
│   │   └── poem.dart
│   ├── services/
│   │   └── api_service.dart     # http client wrapping the FastAPI backend
│   ├── providers/               # Riverpod providers
│   │   ├── auth_provider.dart
│   │   └── poems_provider.dart
│   ├── screens/
│   │   ├── login_screen.dart
│   │   ├── feed_screen.dart
│   │   ├── profile_screen.dart
│   │   ├── poem_editor_screen.dart
│   │   └── poem_detail_screen.dart
│   └── widgets/
│       ├── poem_card.dart       # the preview card
│       └── app_scaffold.dart
└── pubspec.yaml
```

### 5.1 `Poem` model

```dart
class Poem {
  final String id;
  final String title;
  final String preview;
  final DateTime createdAt;
  final int likeCount;

  Poem({
    required this.id,
    required this.title,
    required this.preview,
    required this.createdAt,
    required this.likeCount,
  });

  factory Poem.fromJson(Map<String, dynamic> json) => Poem(
        id: json['id'],
        title: json['title'],
        preview: json['preview'],
        createdAt: DateTime.parse(json['created_at']),
        likeCount: json['like_count'] ?? 0,
      );
}
```

### 5.2 `PoemCard` widget (profile grid/list item)

```dart
import 'package:flutter/material.dart';

class PoemCard extends StatelessWidget {
  final String title;
  final String preview;
  final VoidCallback onTap;

  const PoemCard({
    super.key,
    required this.title,
    required this.preview,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      elevation: 2,
      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
      child: InkWell(
        borderRadius: BorderRadius.circular(12),
        onTap: onTap,
        child: Padding(
          padding: const EdgeInsets.all(16),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(
                title,
                style: Theme.of(context).textTheme.titleMedium,
                maxLines: 1,
                overflow: TextOverflow.ellipsis,
              ),
              const SizedBox(height: 8),
              Text(
                preview,
                style: Theme.of(context)
                    .textTheme
                    .bodyMedium
                    ?.copyWith(fontStyle: FontStyle.italic, height: 1.4),
                maxLines: 3,
                overflow: TextOverflow.ellipsis,
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

### 5.3 Profile screen (grid of cards)

```dart
ListView.builder(
  itemCount: poems.length,
  itemBuilder: (context, index) {
    final poem = poems[index];
    return PoemCard(
      title: poem.title,
      preview: poem.preview,
      onTap: () => Navigator.pushNamed(
        context,
        '/poem-detail',
        arguments: poem.id,
      ),
    );
  },
);
```

---

## 6. Auth Flow

1. User registers/logs in → backend returns `access_token` (short-lived) + `refresh_token`.
2. Flutter stores tokens with `flutter_secure_storage`.
3. `api_service.dart` attaches `Authorization: Bearer <token>` to requests, and
   silently refreshes on 401 using the refresh token.

---

## 7. Uploading Poems (Two Modes)

- **Typed poem**: user writes directly in a rich-text-lite editor
  (`TextField` with multiline + optional Markdown formatting) → `POST /poems`.
- **Photo/scan upload**: user snaps or picks an image of a handwritten poem →
  `image_picker` in Flutter → upload to `POST /poems/{id}/cover` (or a
  dedicated OCR step later if you want to extract text automatically).

---

## 8. Deployment Notes

- **Backend**: Dockerize FastAPI + run behind Uvicorn/Gunicorn; deploy to
  Fly.io / Render / a small VPS; use Alembic for migrations.
- **Database**: managed Postgres (Supabase, Neon, RDS) simplifies backups/TLS.
- **Flutter Web**: can be hosted as static files (Firebase Hosting, Netlify)
  hitting the same API.
- **Images**: start with local disk + Nginx static serving; move to S3-compatible
  storage (e.g., Cloudflare R2) when you need scale.

---

## 9. Possible Enhancements

- Full-text search on poems (`tsvector` + GIN index in Postgres).
- Tags/genres and a discovery feed.
- Comments and likes (tables already sketched above).
- Draft vs. published states (`status` enum column).
- Rich formatting (stanza breaks, italics) stored as Markdown, rendered with
  `flutter_markdown`.
- Offline draft support with local SQLite (`drift`) synced to the backend.

---

This gives you a working skeleton: Postgres schema → FastAPI CRUD + preview
logic → Flutter cards. From here, the natural next step is scaffolding the
actual FastAPI project and the Flutter `pubspec.yaml`/routes if you want to
start coding.
