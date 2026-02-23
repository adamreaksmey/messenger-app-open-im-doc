# OpenIM Chat App — Implementation Guide

> **Stack:** OpenIM · Go/Gin · DH Auth (mobile) · JWT Auth (admin) · No E2E Encryption
> **Target Scale:** 200k users
> **App Type:** Telegram-like messenger with admin dashboard

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Infrastructure & Dependencies](#2-infrastructure--dependencies)
3. [Gateway — Auth Verification & Token Injection](#3-gateway--auth-verification--token-injection)
4. [Authentication](#4-authentication)
5. [OpenIM Integration](#5-openim-integration)
6. [Core Features](#6-core-features)
7. [Admin Dashboard](#7-admin-dashboard)
8. [Media & File Handling](#8-media--file-handling)
9. [Push Notifications](#9-push-notifications)
10. [Environment & Deployment](#10-environment--deployment)
11. [Team Responsibilities](#11-team-responsibilities)

**See also:** [README.md](./README.md) and the [sections/](./sections/) folder for the **migration plan** from the current chat app to this OpenIM-based design. The sections map current behavior to OpenIM and give a phased implementation order (see [sections/00-INDEX.md](./sections/00-INDEX.md) and [sections/08-migration-phases.md](./sections/08-migration-phases.md)).

---

## 1. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER                                │
│   Mobile App (iOS/Android)            Admin Dashboard (Web)          │
│   - DH Auth (X25519 + HMAC)           - JWT Auth                     │
│   - Carries: DH sessionId only        - Carries: JWT only            │
│   - WebSocket: imToken (one-time)     - REST API                     │
└──────────────────────────┬───────────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────────┐
│                    YOUR GATEWAY (Go/Gin)                             │
│                                                                      │
│  1. Verify DH session (HMAC) OR verify JWT                           │
│  2. Resolve user's imToken from Redis                                │
│  3. Inject `token` header into proxied request                       │
│  4. Forward to OpenIM via gRPC                                       │
│                                                                      │
│  ⚠️  WebSocket is handled differently — see Section 3                │
└────────────┬─────────────────────────────┬───────────────────────────┘
             │ REST                        │ WebSocket (special path)
┌────────────▼────────────┐   ┌────────────▼──────────────────────────┐
│     YOUR BACKEND        │   │          OPENIM SERVER                │
│  - Device Registration  │   │  - Message routing                    │
│  - OTP / DH Auth        │   │  - Group management                   │
│  - User profiles        │   │  - WebSocket connections              │
│  - Admin JWT auth       │   │  - Message history                    │
│  - Webhook receiver     │   │  - Friend/contact management          │
│  - imToken store/cache  │   │                                       │
└────────────┬────────────┘   └────────────┬──────────────────────────┘
             │                             │
┌────────────▼─────────────────────────────▼──────────────────────────┐
│                          DATA LAYER                                 │
│   PostgreSQL          MongoDB            Redis              MinIO   │
│   (users, devices)    (messages)         (imToken cache,   (media,  │
│                                          sessions)          files)  │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Design Decisions

- **Your gateway owns authentication.** It verifies DH/JWT, resolves the user's OpenIM token from Redis, and injects it transparently before proxying to OpenIM. OpenIM never sees your auth protocol.
- **The client never manages two tokens.** Mobile clients carry only their DH session. The gateway handles the imToken exchange invisibly. The only exception is the WebSocket connection — see Section 3.
- **No E2E encryption.** All messages pass through OpenIM in plaintext. The admin dashboard can read any message via OpenIM's admin APIs.
- **OpenIM is the messaging engine.** Don't rebuild message routing, group logic, or WebSocket infrastructure — that's what OpenIM is for.
- **Your backend handles webhooks.** OpenIM fires webhook callbacks on message events. Your backend listens for moderation, logging, and notification triggers.
- **Service-to-service uses gRPC.** All communication between your backend and OpenIM (API calls, admin operations, token minting, proxied messaging) is done via gRPC. Client-to-backend remains REST/HTTP or WebSocket as appropriate.

---

## 2. Infrastructure & Dependencies

### Required Services

| Service | Purpose | Notes |
|---|---|---|
| **OpenIM Server** | Core messaging engine | Run via Docker Compose |
| **ETCD** | OpenIM service discovery | Bundled in OpenIM Docker setup |
| **MongoDB** | Message storage | Managed by OpenIM |
| **Redis** | imToken cache, session store | Shared between OpenIM and your backend |
| **Kafka** | OpenIM message queue | Bundled in OpenIM Docker setup |
| **MinIO** | Object storage (media) | Can swap for S3 in production |
| **PostgreSQL** | Your user/device data | You manage this |
| **Your Backend (Go/Gin)** | Auth, profiles, webhooks, gateway | Main service |
| **Nginx** | Reverse proxy, SSL termination | Routes to your backend and OpenIM WS |

### Getting OpenIM Running

```bash
# Clone OpenIM Docker setup
git clone https://github.com/openimsdk/openim-docker
cd openim-docker

# Copy and configure environment
cp .env.example .env
# Edit .env — set SECRET, MINIO credentials, MONGO password, etc.

# Start everything
docker compose up -d

# Verify OpenIM is healthy
curl http://localhost:10002/healthz
```

> **Critical `.env` values to set before anything else:**
> - `SECRET` — the OpenIM admin secret (used to mint user tokens)
> - `MINIO_ACCESS_KEY` / `MINIO_SECRET_KEY`
> - `MONGO_PASSWORD`
> - `API_OPENIM_PORT` (default: `10002`)
> - `OPENIM_WS_PORT` (default: `10001`)

### OpenIM Key Ports

| Port | Service |
|---|---|
| `10001` | WebSocket (client connections) |
| `10002` | gRPC API (your backend calls this for all OpenIM operations) |
| `10008` | Admin API (gRPC; your backend uses this for admin operations) |

---

## 3. Gateway — Auth Verification & Token Injection

This is the most critical piece of the architecture. **Read this section carefully.**

### How It Works (REST client → backend → OpenIM via gRPC)

```
Mobile Client                  Your Gateway              OpenIM (gRPC)
     │                              │                        │
     │  POST /im/msg/send_msg       │                        │
     │  Authorization: Session xxx  │                        │
     │─────────────────────────────>│                        │
     │                              │ 1. Verify HMAC session │
     │                              │ 2. Get userID          │
     │                              │ 3. Redis GET imToken   │
     │                              │ 4. Inject token        │
     │                              │ 5. Call OpenIM via gRPC│
     │                              │──────────────────────->│
     │                              │  gRPC SendMsg          │
     │                              │  (token, operationID)  │
     │<─────────────────────────────│<───────────────────────│
     │  200 OK                      │                        │
```

### Gateway Middleware

```go
// middleware/gateway.go

package middleware

import (
    "context"
    "net/http"
    "net/http/httputil"
    "net/url"
    "strings"

    "github.com/gin-gonic/gin"
    "github.com/google/uuid"
    "github.com/redis/go-redis/v9"
)

const imTokenPrefix = "im:token:"

// OpenIMProxy creates a reverse proxy middleware that:
// 1. Verifies the caller (DH session or JWT) — done by auth middleware before this
// 2. Injects the imToken into the forwarded request from Redis
// 3. Forwards the request to OpenIM via gRPC (service-to-service)
func OpenIMProxy(rdb *redis.Client, openIMBaseURL string) gin.HandlerFunc {
    target, _ := url.Parse(openIMBaseURL)
    proxy := httputil.NewSingleHostReverseProxy(target)

    proxy.Director = func(req *http.Request) {
        req.URL.Scheme = target.Scheme
        req.URL.Host = target.Host
        // Strip our path prefix: /im/msg/send_msg → /msg/send_msg
        req.URL.Path = strings.TrimPrefix(req.URL.Path, "/im")
        req.Host = target.Host
    }

    return func(c *gin.Context) {
        // userID is set by whichever auth middleware ran before this
        userID := c.GetString("userID")
        if userID == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
            return
        }

        // Fetch the user's OpenIM token from Redis
        imToken, err := rdb.Get(context.Background(), imTokenPrefix+userID).Result()
        if err != nil {
            // Not in cache — user needs to re-authenticate
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "im session expired, please re-login"})
            return
        }

        // Inject OpenIM-required headers
        c.Request.Header.Set("token", imToken)
        c.Request.Header.Set("operationID", uuid.New().String())

        // Strip our own auth headers — OpenIM doesn't know about them
        c.Request.Header.Del("Authorization")
        c.Request.Header.Del("X-Signature")
        c.Request.Header.Del("X-Timestamp")
        c.Request.Header.Del("X-Nonce")

        proxy.ServeHTTP(c.Writer, c.Request)
    }
}
```

### Auth Middleware

```go
// middleware/auth.go

package middleware

import (
    "net/http"
    "strings"

    "github.com/gin-gonic/gin"
    "github.com/golang-jwt/jwt/v5"
)

// DHSessionAuth verifies the HMAC session from mobile clients
// and sets userID in the Gin context.
func DHSessionAuth(sessionStore SessionStore) gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")
        if !strings.HasPrefix(authHeader, "Session ") {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "missing session"})
            return
        }

        sessionID := strings.TrimPrefix(authHeader, "Session ")
        timestamp := c.GetHeader("X-Timestamp")
        nonce := c.GetHeader("X-Nonce")
        signature := c.GetHeader("X-Signature")

        userID, err := sessionStore.Verify(sessionID, timestamp, nonce, signature)
        if err != nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid session"})
            return
        }

        c.Set("userID", userID)
        c.Next()
    }
}

// JWTAuth verifies admin JWT tokens and sets adminID + role in context.
func JWTAuth(secret string) gin.HandlerFunc {
    return func(c *gin.Context) {
        tokenStr := strings.TrimPrefix(c.GetHeader("Authorization"), "Bearer ")
        if tokenStr == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "missing token"})
            return
        }

        token, err := jwt.Parse(tokenStr, func(t *jwt.Token) (interface{}, error) {
            return []byte(secret), nil
        })
        if err != nil || !token.Valid {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid token"})
            return
        }

        claims := token.Claims.(jwt.MapClaims)
        c.Set("adminID", claims["adminId"])
        c.Set("role", claims["role"])
        c.Next()
    }
}

func RequireRole(role string) gin.HandlerFunc {
    return func(c *gin.Context) {
        r := c.GetString("role")
        if r != role && r != "superadmin" {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "forbidden"})
            return
        }
        c.Next()
    }
}
```

### Wiring the Router

```go
// main.go

func setupRouter(rdb *redis.Client, cfg *Config) *gin.Engine {
    r := gin.Default()

    openIMProxy := middleware.OpenIMProxy(rdb, cfg.OpenIMAPIURL)

    // Mobile clients: DH session verified, imToken injected, proxied to OpenIM
    mobileIM := r.Group("/im", middleware.DHSessionAuth(sessionStore))
    mobileIM.Any("/*path", openIMProxy)

    // Your own backend routes
    api := r.Group("/api/v1")
    api.POST("/device/register", handlers.RegisterDevice)
    api.POST("/auth/otp/send", handlers.SendOTP)
    api.POST("/auth/otp/verify", handlers.VerifyOTP)
    api.POST("/user/profile/setup", middleware.DHSessionAuth(sessionStore), handlers.SetupProfile)
    api.GET("/stickers/packs", middleware.DHSessionAuth(sessionStore), handlers.GetStickerPacks)
    api.GET("/gifs/search", middleware.DHSessionAuth(sessionStore), handlers.SearchGIFs)
    api.POST("/user/push-token", middleware.DHSessionAuth(sessionStore), handlers.RegisterPushToken)

    // Admin dashboard routes — JWT protected
    admin := r.Group("/api/v1/admin")
    admin.POST("/login", handlers.AdminLogin) // no auth on login itself
    adminAuthed := admin.Group("", middleware.JWTAuth(cfg.JWTSecret))
    adminAuthed.GET("/users", handlers.AdminListUsers)
    adminAuthed.DELETE("/users/:userID", middleware.RequireRole("moderator"), handlers.AdminBanUser)
    adminAuthed.GET("/messages", handlers.AdminGetMessages)
    adminAuthed.DELETE("/messages/:msgID", middleware.RequireRole("moderator"), handlers.AdminDeleteMessage)
    adminAuthed.GET("/groups", handlers.AdminListGroups)
    adminAuthed.POST("/groups/:groupID/dismiss", middleware.RequireRole("moderator"), handlers.AdminDismissGroup)
    adminAuthed.POST("/stickers/packs", middleware.RequireRole("superadmin"), handlers.AdminCreateStickerPack)

    // Webhook receiver from OpenIM (internal — validate via secret header)
    r.POST("/webhooks/openim", handlers.OpenIMWebhook)

    return r
}
```

---

### ⚠️ WebSocket: The Exception to the Gateway Pattern

> **This is critical and must be understood by the whole team before implementation begins.**

**The REST gateway pattern cannot be applied to WebSocket connections.**

WebSocket is a persistent, stateful, binary-framed connection that stays open for hours. Once the handshake completes, your gateway cannot intercept and re-sign each frame the way it can with discrete REST requests. Putting your gateway in the middle of every WebSocket frame would add latency, create a single point of failure for all active sessions, and massively complicate the code.

**The solution: deliver the imToken once at login, then let the client connect directly.**

```
Auth Flow (REST, through gateway):
  Client ──► POST /api/v1/auth/otp/verify
          ◄── { sessionId, imToken, wsURL }
                         │
                         └─ imToken is returned ONCE here.
                            Client stores it securely (Keychain / Keystore).
                            Gateway also caches it in Redis for REST injection.

WebSocket Flow (direct, NO gateway):
  Client ──► wss://your-domain.com/ws?token=<imToken>
          ◄── OpenIM WebSocket handshake
          ◄──► Messages flow bidirectionally, persistently
               Your backend is NOT in this path at all.
```

**imToken lifecycle:**
- Minted by your backend after successful OTP verification
- Stored in Redis (`im:token:<userID>`) — used by the gateway for REST calls
- Also returned to the client once — used only for the WebSocket connection
- Long-lived (default ~7 days in OpenIM). Refreshed when the user re-authenticates
- On ban: delete from Redis AND call `/auth/force_logout` so the WS connection is killed by OpenIM

**Summary of the two paths:**

| | REST API calls | WebSocket |
|---|---|---|
| **Client sends** | DH session headers | imToken (query param) |
| **Goes through your gateway?** | ✅ Yes | ❌ No — direct Nginx passthrough |
| **Token injected by gateway?** | ✅ Yes, from Redis | N/A — client presents it directly |
| **Connection type** | Stateless, per-request | Stateful, persistent (hours) |

**Nginx configuration — REST through backend, WebSocket direct passthrough:**

```nginx
server {
    listen 443 ssl;
    server_name your-domain.com;

    # All REST — goes to your Go/Gin backend
    # This includes /api/* (your routes) and /im/* (proxied to OpenIM by your gateway)
    location /api/ {
        proxy_pass http://your-backend:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /im/ {
        proxy_pass http://your-backend:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # ─────────────────────────────────────────────────────────────────
    # WebSocket: DIRECT passthrough to OpenIM. Your backend is NOT here.
    # The client authenticates with its imToken as a query param or header.
    # Nginx passes it straight through to OpenIM which validates it natively.
    # Do NOT route this through your Go backend.
    # ─────────────────────────────────────────────────────────────────
    location /ws {
        proxy_pass http://openim-server:10001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 3600s;  # keep long-lived connections alive
        proxy_send_timeout 3600s;
    }

    # Admin dashboard static files
    location /admin/ {
        root /var/www;
        try_files $uri $uri/ /admin/index.html;
    }
}
```

---

## 4. Authentication

### 4.1 Mobile Auth — DH (X25519 + HMAC)

The `RegistrationClient` (TypeScript, client-side) is the approved implementation. Your Go backend must implement the matching server-side handshake. **Do not modify the crypto protocol.**

#### Device Registration Handler

```go
// handlers/auth.go

package handlers

import (
    "crypto/hmac"
    "crypto/rand"
    "crypto/sha256"
    "encoding/base64"
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/google/uuid"
    "golang.org/x/crypto/curve25519"
)

type RegisterDeviceRequest struct {
    ClientPublicKey string `json:"clientPublicKey" binding:"required"`
    DeviceInfo      string `json:"deviceInfo" binding:"required"`
    Platform        string `json:"platform" binding:"required"` // ios|android|web
    DeviceName      string `json:"deviceName"`
}

func RegisterDevice(c *gin.Context) {
    var req RegisterDeviceRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // Generate server X25519 keypair
    var serverPrivKey [32]byte
    rand.Read(serverPrivKey[:])
    serverPrivKey[0] &= 248
    serverPrivKey[31] &= 127
    serverPrivKey[31] |= 64

    serverPubKey, _ := curve25519.X25519(serverPrivKey[:], curve25519.Basepoint)

    // Decode client public key and compute shared secret
    clientPubKey, _ := base64.StdEncoding.DecodeString(req.ClientPublicKey)
    sharedSecret, _ := curve25519.X25519(serverPrivKey[:], clientPubKey)

    // Derive keys (match the client-side derivation exactly)
    deviceSecret := deriveDeviceSecret(sharedSecret, req.DeviceInfo)

    deviceID := uuid.New().String()
    db.StoreDevice(deviceID, req.Platform, req.DeviceInfo, deviceSecret)

    c.JSON(http.StatusOK, gin.H{
        "deviceId":        deviceID,
        "serverPublicKey": base64.StdEncoding.EncodeToString(serverPubKey),
    })
}
```

#### OTP Verification + imToken Minting

```go
type VerifyOTPRequest struct {
    PhoneNumber string `json:"phoneNumber" binding:"required"`
    OTP         string `json:"otp" binding:"required"`
    DeviceID    string `json:"deviceId" binding:"required"`
}

func VerifyOTP(c *gin.Context) {
    var req VerifyOTPRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    timestamp  := c.GetHeader("X-Timestamp")
    nonce      := c.GetHeader("X-Nonce")
    sessionID  := strings.TrimPrefix(c.GetHeader("Authorization"), "Session ")
    signature  := c.GetHeader("X-Signature")

    device, err := db.GetDevice(req.DeviceID)
    if err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "unknown device"})
        return
    }

    serverHMACKey := deriveServerHMACKey(device.Secret)

    // Verify session: HMAC(serverHMACKey, "{deviceID}:{timestamp}:{nonce}")
    expectedSession := computeHMAC(serverHMACKey, req.DeviceID+":"+timestamp+":"+nonce)
    if !hmac.Equal([]byte(sessionID), []byte(expectedSession)) {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid session"})
        return
    }

    // Verify signature: HMAC(serverHMACKey, "otp-verify:{phone}:{timestamp}:{nonce}")
    expectedSig := computeHMAC(serverHMACKey, "otp-verify:"+req.PhoneNumber+":"+timestamp+":"+nonce)
    if !hmac.Equal([]byte(signature), []byte(expectedSig)) {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid signature"})
        return
    }

    if !otpStore.Verify(req.PhoneNumber, req.OTP) {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid OTP"})
        return
    }

    user, isNew := db.FindOrCreateUserByPhone(req.PhoneNumber)

    // Mint OpenIM token and cache it in Redis
    platformID := platformIDFromString(device.Platform) // ios=1, android=2, web=5
    imToken, err := openimClient.MintUserToken(c.Request.Context(), user.ID, platformID)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to mint im token"})
        return
    }
    rdb.Set(c.Request.Context(), "im:token:"+user.ID, imToken, 7*24*time.Hour)

    // Issue your own session token
    sessionToken := sessionStore.Issue(user.ID, req.DeviceID)

    c.JSON(http.StatusOK, gin.H{
        "sessionId": sessionToken,
        "imToken":   imToken,                    // client uses this for WebSocket only
        "wsURL":     "wss://your-domain.com/ws", // direct to OpenIM via Nginx
        "isNewUser": isNew,
        "user":      user,
    })
}

func computeHMAC(key []byte, message string) string {
    mac := hmac.New(sha256.New, key)
    mac.Write([]byte(message))
    return base64.StdEncoding.EncodeToString(mac.Sum(nil))
}
```

#### OpenIM Client (Admin Token + User Token Minting)

```go
// openim/client.go

package openim

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "sync"
    "time"
)

type Client struct {
    baseURL    string
    secret     string
    adminToken string
    tokenExp   time.Time
    mu         sync.Mutex
    http       *http.Client
}

func NewClient(baseURL, secret string) *Client {
    return &Client{
        baseURL: baseURL,
        secret:  secret,
        http:    &http.Client{Timeout: 10 * time.Second},
    }
}

// getAdminToken fetches and caches the OpenIM admin token.
// This is a backend-only token — never sent to clients.
func (c *Client) getAdminToken(ctx context.Context) (string, error) {
    c.mu.Lock()
    defer c.mu.Unlock()

    if time.Now().Before(c.tokenExp) {
        return c.adminToken, nil
    }

    body, _ := json.Marshal(map[string]interface{}{
        "secret":     c.secret,
        "platformID": 1,
        "userID":     "imAdmin",
    })

    resp, err := c.post(ctx, "/auth/user_token", body, "")
    if err != nil {
        return "", err
    }

    token := resp["data"].(map[string]interface{})["token"].(string)
    c.adminToken = token
    c.tokenExp = time.Now().Add(6 * time.Hour)
    return token, nil
}

func (c *Client) MintUserToken(ctx context.Context, userID string, platformID int) (string, error) {
    adminToken, err := c.getAdminToken(ctx)
    if err != nil {
        return "", err
    }

    body, _ := json.Marshal(map[string]interface{}{
        "userID":     userID,
        "platformID": platformID,
    })

    resp, err := c.post(ctx, "/auth/user_token", body, adminToken)
    if err != nil {
        return "", err
    }

    return resp["data"].(map[string]interface{})["token"].(string), nil
}

func (c *Client) PostWithAdminToken(ctx context.Context, path string, body []byte) (map[string]interface{}, error) {
    adminToken, err := c.getAdminToken(ctx)
    if err != nil {
        return nil, err
    }
    return c.post(ctx, path, body, adminToken)
}

func (c *Client) post(ctx context.Context, path string, body []byte, token string) (map[string]interface{}, error) {
    req, _ := http.NewRequestWithContext(ctx, "POST", c.baseURL+path, bytes.NewReader(body))
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("operationID", fmt.Sprintf("%d", time.Now().UnixNano()))
    if token != "" {
        req.Header.Set("token", token)
    }

    resp, err := c.http.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var result map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&result)
    return result, nil
}
```

---

### 4.2 Admin Dashboard Auth — JWT

```go
// handlers/admin_auth.go

package handlers

import (
    "net/http"
    "os"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/golang-jwt/jwt/v5"
    "golang.org/x/crypto/bcrypt"
)

type AdminLoginRequest struct {
    Username string `json:"username" binding:"required"`
    Password string `json:"password" binding:"required"`
}

func AdminLogin(c *gin.Context) {
    var req AdminLoginRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    admin, err := db.FindAdminByUsername(req.Username)
    if err != nil || bcrypt.CompareHashAndPassword([]byte(admin.PasswordHash), []byte(req.Password)) != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid credentials"})
        return
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
        "adminId": admin.ID,
        "role":    admin.Role, // "superadmin" | "moderator"
        "exp":     time.Now().Add(8 * time.Hour).Unix(),
    })

    signed, _ := token.SignedString([]byte(os.Getenv("JWT_SECRET")))
    c.JSON(http.StatusOK, gin.H{"token": signed})
}
```

---

## 5. OpenIM Integration

### 5.1 Registering Users in OpenIM

Call this the first time a user completes registration:

```go
func (c *Client) RegisterUser(ctx context.Context, userID, nickname, avatarURL string) error {
    adminToken, err := c.getAdminToken(ctx)
    if err != nil {
        return err
    }

    body, _ := json.Marshal(map[string]interface{}{
        "secret": c.secret,
        "users": []map[string]interface{}{
            {"userID": userID, "nickname": nickname, "faceURL": avatarURL},
        },
    })

    _, err = c.post(ctx, "/user/user_register", body, adminToken)
    return err
}
```

### 5.2 Webhook Receiver

Configure in OpenIM's `config/webhooks.yaml`:

```yaml
url: "http://your-backend:8080/webhooks/openim"
afterSendSingleMsg:
  enable: true
afterSendGroupMsg:
  enable: true
afterCreateGroup:
  enable: true
```

```go
// handlers/webhook.go

type OpenIMWebhookPayload struct {
    CallbackCommand string      `json:"callbackCommand"`
    SendID          string      `json:"sendID"`
    RecvID          string      `json:"recvID"`
    GroupID         string      `json:"groupID"`
    Content         interface{} `json:"content"`
}

func OpenIMWebhook(c *gin.Context) {
    if c.GetHeader("X-OpenIM-Secret") != os.Getenv("OPENIM_WEBHOOK_SECRET") {
        c.AbortWithStatus(http.StatusUnauthorized)
        return
    }

    var payload OpenIMWebhookPayload
    if err := c.ShouldBindJSON(&payload); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    switch payload.CallbackCommand {
    case "callbackAfterSendSingleMsgCommand":
        go handleDirectMessage(payload)
    case "callbackAfterSendGroupMsgCommand":
        go handleGroupMessage(payload)
    case "callbackAfterCreateGroupCommand":
        go handleGroupCreated(payload)
    }

    // OpenIM requires this exact response shape to continue processing
    c.JSON(http.StatusOK, gin.H{"actionCode": 0, "errCode": 0, "errMsg": ""})
}
```

---

## 6. Core Features

### 6.1 Direct Messages & Group Chats

Handled entirely by the OpenIM SDK on the mobile client. Your gateway receives REST from the client and forwards to OpenIM via **gRPC** — no custom message routing needed.

```
Client SDK → POST /im/msg/send_msg  (DH session headers)
           → Gateway verifies session, looks up imToken from Redis, injects it
           → OpenIM receives properly authenticated request
           → Message delivered to recipient
```

### 6.2 GIFs & Stickers

**Sticker pack list endpoint:**

```go
// handlers/stickers.go

func GetStickerPacks(c *gin.Context) {
    packs, err := db.GetActiveStickerPacks()
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusOK, gin.H{"packs": packs})
}
```

Stickers are sent as OpenIM custom messages by the mobile SDK. The client builds a custom message with ContentType `1400` and a JSON body like `{"type":"sticker","stickerID":"xxx","url":"https://..."}`.

**GIF proxy (keeps your API key server-side):**

```go
// handlers/gifs.go

func SearchGIFs(c *gin.Context) {
    query := c.Query("q")
    limit := c.DefaultQuery("limit", "20")

    resp, err := http.Get(fmt.Sprintf(
        "https://tenor.googleapis.com/v2/search?q=%s&limit=%s&key=%s",
        url.QueryEscape(query), limit, os.Getenv("TENOR_API_KEY"),
    ))
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "gif search failed"})
        return
    }
    defer resp.Body.Close()

    var result interface{}
    json.NewDecoder(resp.Body).Decode(&result)
    c.JSON(http.StatusOK, result)
}
```

### 6.3 Why a Wrapper (Your Backend) Instead of Patching OpenIM

When you need to **build on top** of OpenIM — for example, enriching messages with data from **your own DB** (e.g. a media table keyed by `media_id`, so the client gets full media URLs and metadata) — you have two options:

- **Option A:** Add the feature **inside OpenIM’s code** (fork/patch: OpenIM calls your backend or DB to enrich before returning).
- **Option B:** **Wrapper:** client calls **your** backend; your backend calls OpenIM **via gRPC**, then enriches using **your DB** (no separate media service — straight DB access), and returns the combined result.

We recommend **Option B (wrapper)**. Rationale:

1. **Latency is small and bounded.** The wrapper adds one hop (client → your backend) and one DB read (your backend → your DB). Your backend talks to OpenIM over **gRPC** (efficient, same-region). With backend and DB in the same region, that’s typically tens of ms. For “fetch message list” or similar, that’s acceptable. You can optimize (caching, indexing) and still keep the wrapper.

2. **Patching OpenIM doesn’t remove enrichment latency; it only moves it.** To attach your data (e.g. media URLs), *some* component must read from your DB. With the wrapper, your backend does it. With a patch, OpenIM would call your backend or DB. The RTT to your DB (or to a backend that talks to your DB) is similar either way. So Option A doesn’t eliminate latency — it just puts the dependency inside OpenIM’s code path.

3. **Patching OpenIM is ongoing cost, not one-time.** You’re maintaining a fork. Every OpenIM upgrade requires re-applying or re-porting your change, re-testing, and re-deploying. That’s recurring effort. The wrapper stays in your codebase; OpenIM upgrades don’t touch it.

4. **Failure and ownership.** With a patch, OpenIM’s message-fetch path depends on your backend or DB. If your DB is slow or down, OpenIM’s API can time out or fail and this will require us to implement some sort of fallback feature so we don't confuse the users. With the wrapper, *your* backend controls fallback (e.g. return messages without enriched media, or use cached data). Clear ownership: enrichment behavior lives in your service.

5. **No separate media service.** Enrichment is “your backend → OpenIM via **gRPC** + your DB.” Your backend reads from your media table (or any other table) directly; no extra microservice required.

**Summary:** For anything that needs your own data (message enrichment with media from your DB, custom admin queries, etc.), keep it in **your backend**: call OpenIM **via gRPC**, then enrich from your DB and return. Avoid patching OpenIM so you don’t take on long-term fork maintenance and coupling.

---

## 7. Admin Dashboard

### 7.1 Overview

The admin dashboard is a React web app that talks to your Go backend using JWT. Your backend calls OpenIM's admin API **via gRPC**. Admins never communicate with OpenIM directly.

### 7.2 Admin Handlers

#### User Management

```go
// handlers/admin_users.go

func AdminListUsers(c *gin.Context) {
    search := c.Query("search")
    page, _ := strconv.Atoi(c.DefaultQuery("page", "1"))
    limit, _ := strconv.Atoi(c.DefaultQuery("limit", "50"))

    users, total, err := db.SearchUsers(search, page, limit)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusOK, gin.H{"users": users, "total": total})
}

func AdminBanUser(c *gin.Context) {
    userID := c.Param("userID")

    if err := db.BanUser(userID); err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }

    // Force logout all OpenIM sessions
    body, _ := json.Marshal(map[string]interface{}{
        "userID":     userID,
        "platformID": -1, // -1 = all platforms
    })
    openimClient.PostWithAdminToken(c.Request.Context(), "/auth/force_logout", body)

    // Evict imToken from Redis — gateway will deny all further REST calls immediately
    rdb.Del(c.Request.Context(), "im:token:"+userID)

    c.JSON(http.StatusOK, gin.H{"success": true})
}
```

#### Message Viewing & Deletion

```go
// handlers/admin_messages.go

func AdminGetMessages(c *gin.Context) {
    conversationID := c.Query("conversationID")
    seq, _ := strconv.Atoi(c.DefaultQuery("seq", "0"))
    count, _ := strconv.Atoi(c.DefaultQuery("count", "20"))

    seqs := make([]int, count)
    for i := range seqs {
        seqs[i] = seq - i
    }

    body, _ := json.Marshal(map[string]interface{}{
        "conversationID": conversationID,
        "seqs":           seqs,
    })

    resp, err := openimClient.PostWithAdminToken(c.Request.Context(), "/msg/get_conversation_msgs_by_seq", body)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusOK, resp)
}

func AdminDeleteMessage(c *gin.Context) {
    msgID  := c.Param("msgID")
    userID := c.Query("userID")

    body, _ := json.Marshal(map[string]interface{}{
        "userID": userID,
        "seqs":   []string{msgID},
    })

    openimClient.PostWithAdminToken(c.Request.Context(), "/msg/del_msgs", body)
    c.JSON(http.StatusOK, gin.H{"success": true})
}
```

#### Group Management

```go
// handlers/admin_groups.go

func AdminListGroups(c *gin.Context) {
    page, _ := strconv.Atoi(c.DefaultQuery("page", "1"))
    body, _ := json.Marshal(map[string]interface{}{
        "pagination": map[string]int{"pageNumber": page, "showNumber": 50},
    })
    resp, _ := openimClient.PostWithAdminToken(c.Request.Context(), "/group/get_groups", body)
    c.JSON(http.StatusOK, resp)
}

func AdminDismissGroup(c *gin.Context) {
    body, _ := json.Marshal(map[string]interface{}{"groupID": c.Param("groupID")})
    openimClient.PostWithAdminToken(c.Request.Context(), "/group/dismiss_group", body)
    c.JSON(http.StatusOK, gin.H{"success": true})
}
```

#### Sticker Management

```go
// handlers/admin_stickers.go

type CreateStickerPackRequest struct {
    Name         string `json:"name" binding:"required"`
    ThumbnailURL string `json:"thumbnailUrl" binding:"required"`
}

func AdminCreateStickerPack(c *gin.Context) {
    var req CreateStickerPackRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    pack, err := db.CreateStickerPack(req.Name, req.ThumbnailURL)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusCreated, gin.H{"pack": pack})
}
```

### 7.3 Admin Dashboard Frontend Checklist

| Section | Features |
|---|---|
| **Users** | Search by phone/name, view profile, ban/unban, force logout |
| **Messages** | Search by user/group, view DM threads, view group threads, delete messages |
| **Groups** | List all groups, view members, disband group, remove members |
| **Stickers** | Upload packs, add/remove stickers, toggle pack visibility |
| **GIFs** | Configure Tenor/GIPHY key, block specific GIF IDs |
| **Analytics** | DAU, MAU, message volume, new registrations |
| **Admins** | Create accounts, assign roles (superadmin only) |

---

## 8. Media & File Handling

OpenIM uses MinIO for all chat media automatically. The SDK handles upload/download for image, video, audio, and file messages without custom code on your part.

For **sticker and GIF assets**, host them in a separate MinIO bucket you manage:

```go
// storage/minio.go

package storage

import (
    "bytes"
    "context"
    "fmt"
    "os"

    "github.com/minio/minio-go/v7"
)

func UploadSticker(ctx context.Context, client *minio.Client, data []byte, filename string) (string, error) {
    bucket := "stickers"
    _, err := client.PutObject(
        ctx, bucket, filename,
        bytes.NewReader(data), int64(len(data)),
        minio.PutObjectOptions{ContentType: "image/webp"},
    )
    if err != nil {
        return "", err
    }
    return fmt.Sprintf("https://%s/%s/%s", os.Getenv("MINIO_ENDPOINT"), bucket, filename), nil
}
```

---

## 9. Push Notifications

OpenIM supports APNs (iOS) and FCM (Android) natively. Configure credentials in OpenIM's `config/notification.yaml`.

Store device push tokens after login:

```go
// handlers/push_token.go

type RegisterPushTokenRequest struct {
    Token    string `json:"token" binding:"required"`
    Platform string `json:"platform" binding:"required"` // "ios" | "android"
}

func RegisterPushToken(c *gin.Context) {
    userID := c.GetString("userID")

    var req RegisterPushTokenRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    if err := db.SavePushToken(userID, req.Token, req.Platform); err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }

    c.JSON(http.StatusOK, gin.H{"success": true})
}
```

The mobile SDK also registers its push token with OpenIM directly after the WebSocket connection is established (OpenIM handles APNs/FCM delivery for chat messages natively). Your push token storage above is for your own system notifications (OTP confirmations, account alerts, etc.).

---

## 10. Environment & Deployment

### Environment Variables

```env
# Server
PORT=8080

# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/chatapp

# OpenIM (backend talks to OpenIM via gRPC at this address)
OPENIM_API_URL=http://openim-server:10002
OPENIM_SECRET=your-openim-secret-here
OPENIM_WEBHOOK_SECRET=your-webhook-secret

# Auth
JWT_SECRET=your-jwt-secret-min-32-chars

# Redis (shared with OpenIM)
REDIS_URL=redis://localhost:6379

# MinIO
MINIO_ENDPOINT=localhost:9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin

# External APIs
TENOR_API_KEY=your-tenor-key
SMS_PROVIDER_API_KEY=your-sms-key
```

### docker-compose.override.yml

```yaml
services:
  your-backend:
    build: ./backend
    ports:
      - "8080:8080"
    environment:
      - PORT=8080
      - DATABASE_URL=${DATABASE_URL}
      - OPENIM_SECRET=${SECRET}
      - OPENIM_API_URL=http://openim-api:10002
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=${JWT_SECRET}
      - OPENIM_WEBHOOK_SECRET=${OPENIM_WEBHOOK_SECRET}
    depends_on:
      - openim-api
      - redis
      - postgres
    networks:
      - openim-network

  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: chatapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - openim-network

volumes:
  postgres_data:
```

---

## 11. Team Responsibilities

| Area | Who | Key Tasks |
|---|---|---|
| **Gateway & Middleware** | Backend lead | DH session verification, JWT middleware, Redis imToken injection, reverse proxy setup |
| **Auth Handlers** | Backend team | Device registration, OTP flow, OpenIM user creation, token minting |
| **Admin Handlers** | Backend team | All `/api/v1/admin/*` routes — users, messages, groups, stickers |
| **OpenIM Client** | Backend | `openim.Client` wrapper (gRPC), admin token caching, all OpenIM API calls |
| **OpenIM Infra** | DevOps / Backend | Docker Compose, Nginx config, environment setup, MinIO buckets |
| **Mobile (iOS/Android)** | Mobile team | OpenIM SDK integration, `RegistrationClient` wiring, WebSocket connection with imToken, push token registration |
| **Admin Dashboard** | Frontend | React app, JWT auth, all admin UI sections |

### Critical Path (Do These First)

1. Get OpenIM running locally and verify the `/healthz` endpoint responds
2. Implement `RegisterDevice` and `VerifyOTP` — confirm the full DH handshake works end-to-end
3. Call `RegisterUser` in OpenIM and mint an imToken — verify it lands in Redis
4. Implement the gateway middleware — test that it injects the token correctly on a proxied request
5. Connect the mobile SDK to the WebSocket using the returned `imToken` and `wsURL`
6. Send a test message between two users end-to-end
7. Only then: build admin routes, stickers, GIFs, and everything else on top

---
◊◊
## Appendix — OpenIM API Reference (gRPC)

Your backend communicates with OpenIM **via gRPC** for all service-to-service calls. The following operations map to OpenIM gRPC services:

| Action | gRPC service / method |
|--|--|
| Register user | `user.user_register` |
| Get user info | `user.get_users_info` |
| Mint user token | `auth.user_token` |
| Force logout | `auth.force_logout` |
| Send message (server-side) | `msg.send_msg` |
| Get messages by seq | `msg.get_conversation_msgs_by_seq` |
| Delete messages | `msg.del_msgs` |
| Create group | `group.create_group` |
| Get all groups | `group.get_groups` |
| Dismiss group | `group.dismiss_group` |
| Remove group member | `group.kick_group` |

> ⚠️ **Reminder:** Every OpenIM gRPC call requires a unique `operationID` in the request context. The gateway and `openim.Client` wrapper set this automatically. For direct calls from your admin handlers, use the OpenIM gRPC client and set `operationID` in the metadata.

REST API docs (for reference; backend uses gRPC): [https://docs.openim.io/restapi](https://docs.openim.io/restapi)
OpenIM Docker repo: [https://github.com/openimsdk/openim-docker](https://github.com/openimsdk/openim-docker)
OpenIM SDK (React Native): [https://github.com/openimsdk/open-im-sdk-reactnative](https://github.com/openimsdk/open-im-sdk-reactnative)