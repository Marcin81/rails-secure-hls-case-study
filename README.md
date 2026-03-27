# Secure HLS Video Streaming – Complete Architecture & Security Overview

## 1. Scope and Threat Model

### Assumptions

- User is **authenticated** via Sorcery (not Devise), with roles/levels stored in `users.wing_chun_roles`.
- User has **browser access only** (no admin rights, no server console, no CLI access).
- Backend is Rails 8.1 (Propshaft, Turbo, Stimulus), PostgreSQL/SQLite, ActiveStorage.
- Budget server: 20GB storage, 4GB RAM.
- No hardware DRM (Widevine/FairPlay) available.
- JS in browser is observable (segment decryption happens in memory).

### Out-of-scope attacks

- Screen recording
- Reverse engineering JS
- Full DRM bypass
- Unauthorized server access

This is **controlled-access streaming**, not a professional DRM solution.

---

## 2. High-Level Architecture

### Components

- **Rails Backend**
    - Auth: Sorcery
    - Authorization: CanCanCan
    - JWT key issuance for frontend requests
    - ActiveStorage encrypted storage for segments and thumbnails
- **Background Jobs**
    - Video transcoding to multiple resolutions
    - HLS segmentation + AES encryption per segment
- **Frontend (JS)**
    - MediaSource API + Stimulus
    - Fetch encrypted segments
    - Decrypt in-memory with JWT-provided key/IV
- **Storage**
    - HLS `.m3u8` playlists per resolution
    - `.m4s` encrypted segments
    - AES key/IV stored encrypted in DB

---

## 3. Video Upload → Transcode → Encrypt Flow

1. User uploads video
2. Background job enqueues
3. Video is transcoded to multiple resolutions (480p, 1080p, 4K)
4. HLS segmentation into `.m4s`
5. AES encryption **per segment per resolution** (unique key + IV for each segment)
6. **Plain video is never persisted**

```text
[Upload]
   ↓
[Transcode Job]
   ↓
[Multiple Resolutions]
   ↓
[HLS Segmentation]
   ↓
[AES Encryption per segment]
   ↓
[Encrypted Storage]
```

## 4. Key Management

- Keys & IVs are stored **encrypted** in the database.
- Indexed via BlindIndex.
- **Each segment has its own key set** (AES key + IV).
- Key rotation occurs automatically **per segment**.
- Short-lived JWT issued for each client request, containing the `key_set_id` for the requested segment.
- Key material is **never logged**.
- This ensures that even if a JWT is intercepted, it expires quickly and is valid only for the specific segment.
---

## 5. Playback Authorization Flow

### Step 1: Request Start Token

- Browser requests a JWT from the backend.
- Rails authenticates the user and authorizes access.
- Backend returns JWT containing:
    - `user_id`
    - `video_id`
    - `key_set_id`
    - AES key (Base64)
    - IV (Base64)
    - expiration timestamp

### Step 2: Fetch Encrypted Segment

- Browser requests segment using the JWT.
- Backend validates JWT and user access.
- Encrypted segment is returned.

### Step 3: In-Memory Decryption

- JS decrypts the segment **in memory**.
- Decrypted data is appended to MediaSource buffer for playback.
- Supports **multiple resolutions dynamically**.

---

## 6. Client-Side Decryption (JS)

- Decryption happens **entirely in memory**.
- No plaintext video is saved to disk.
- Supports multiple resolutions for adaptive playback.
- Ensures secure playback without exposing segments on storage.

```text
Encrypted Segment → JS in-memory decrypt → MediaSource buffer → Video playback
```

## 7. Security Properties

### ✅ Enforced

- User authorization per video.
- JWT key expiration.
- Hotlinking prevention.
- Segments encrypted at rest.
- CanCanCan enforces access rights per user/video.

### ⚠️ Limitations

- Screen recording or JS key capture.
- DRM-level attacks (e.g., intercepting decrypted stream in memory).
- Advanced reverse engineering of JS.

---

## 8. Key Capture & Segment Reuse

- Captured JWT key/IV can decrypt **only the specific segment it was issued for**.
- Each segment has a unique key set; re-fetching segments requires a valid JWT for that segment.
- Per-request re-encryption of all segments is impractical on a budget server.

```text
Encrypted Segment → JS in-memory decrypt → MediaSource buffer → Video playback
```

## 9. Performance & Budget Considerations

- Budget server: 20GB storage, 4GB RAM.
- Multiple resolutions supported: 480p, 1080p, 4K.
- HLS segmentation reduces memory and disk load.
- **Multiple key sets per video** slightly increase DB storage, but CPU/storage overhead is low due to async encryption per segment.
- Background jobs handle transcoding + encryption asynchronously.
---

## 10. Key Specs / Environment

- **Ruby:** 4.0.0
- **Rails:** 8.1.1 (GitHub branch `8-1-stable`)
- **Storage:** ActiveStorage
- **Auth:** Sorcery + CanCanCan
- **Encryption:** Lockbox + AES
- **Background Jobs:** MissionControl Jobs / RailsEventStore
- **Frontend:** Stimulus + MediaSource API + Turbo

### Database schema highlights

- `video_encryption_key_sets`: stores key & IV encrypted.
- `video_metadata`: video info (duration, orientation).
- `video_key_links`: maps key sets to videos.
- `users`: store club_id, roles, auth info.

### Gemfile highlights

Rails, Turbo, Stimulus, Sorcery, CanCanCan, Lockbox, BlindIndex, ActiveStorage validations, image processing, JWT.

---

## 11. Summary

- Secure HLS delivery using encrypted segments + short-lived JWT keys.
- Background jobs handle transcoding + encryption.
- Frontend decrypts in-memory per segment.
- Budget-friendly design: unique key per segment, no per-request re-encryption.
- Threat model protects against unauthorized access, but not against screen capture.
- Authorization enforced via CanCanCan, scoped per video and user.
- Clear separation between backend storage, job processing, and frontend playback.

## 12. Architecture Flow Diagram

```mermaid
flowchart TD
    A[User Browser] -->|Login via Sorcery| B[Rails Backend]
    B -->|Authorize & Generate JWT| C[JWT with Key/IV]
    C -->|Send JWT to JS| D[Frontend (Stimulus + MediaSource)]
    D -->|Request Segment with JWT| E[Backend: Validate JWT & User Access]
    E -->|Return Encrypted Segment| D
    D -->|In-memory AES Decrypt| F[MediaSource Buffer]
    F -->|Playback| G[Video Element]

    subgraph Backend
        B
        E
        H[ActiveStorage Encrypted Segments]
        I[VideoStream::TranscodeAndEncryptJob]
        J[Video Encryption Key Sets]
        I --> H
        I --> J
    end

    subgraph Frontend
        D
        F
        G
    end

    H -. AES Encrypted .-> D
    J -. Key/IV .-> C
```

### Legend

- **JWT**: Short-lived token containing AES key & IV for authorized user/video.
- **AES Encryption**: Segments stored encrypted; decrypted only in memory.
- **MediaSource**: Supports adaptive streaming across multiple resolutions.
- **Backend Jobs**: Handle transcoding and encryption asynchronously.
- **User Access Control**: Enforced via CanCanCan, scoped per user/video.  
