# JWT & Token-Based Authentication

## 1. Real-World Scenario

In a super-app ecosystem serving 3,000,000 (3 million) daily active users across 20 distinct microservices (Order, Payment, User Profile, Notifications, Logistics, etc.):

If the platform relies on **Traditional Stateful Sessions (Cookie + Centralized Session Store)**:
- Whenever a user navigates or clicks a feature, every downstream microservice must issue a remote network call to a centralized session store (Redis or PostgreSQL) asking: *"Is this session ID valid, and who owns it?"*.
- At peak traffic (60,000 RPS), the central session store endures 60,000 read queries every single second purely to verify login state. When that session database experiences network latency, all 20 dependent microservices stall and collapse (*Single Point of Failure*).

**Solution with JWT (JSON Web Token):**
The architecture transitions to a self-contained, stateless authentication standard:
1. Upon login, the Auth Service validates user credentials and issues a cryptographically signed JWT using HMAC-SHA256 or asymmetric RSA keys.
2. The token encapsulates non-sensitive claims (such as `user_id`, `role`, and expiration `exp`) directly inside its encoded payload.
3. Downstream microservices verify authenticity independently and instantaneously **without touching a central session database**. Each service evaluates the cryptographic signature using a local in-memory public key or shared secret.

This introduces a classic industry dilemma: *If JWTs are stateless and unrecorded in databases, how can an administrator revoke an active session (force logout) if a token is compromised?*
The enterprise standard resolves this using: **Short-Lived Access Tokens (15 minutes)** paired with **Refresh Token Rotation (7 days)**.

## 2. What Is It and What Is It For?

JSON Web Token (JWT - RFC 7519) is an open industry standard defining a compact, self-contained mechanism for securely transmitting identity claims between parties as a JSON object.

A JWT consists of three segments delimited by periods (`.`):
```
header.payload.signature
```
1. **Header**: Declares the token type (`JWT`) and cryptographic signing algorithm (e.g., `HS256` or `RS256`).
2. **Payload (Claims)**: Carries identity attributes such as `sub` (Subject/User ID), `role`, `iat` (issued at), and `exp` (expiration). **CRITICAL: The payload is merely Base64-encoded, not encrypted! Anyone can inspect its contents.**
3. **Signature**: A cryptographic hash computed across the Header, Payload, and server Secret Key. It prevents third parties from tampering with payload claims (e.g., escalating privilege from `user` to `admin`).

## 3. Why Is It Needed at High Throughput?

At high operational scale:
1. **Eliminates Database I/O Contention**: Authentication checks occur via local mathematical CPU verification inside each application worker.
2. **Enables True Horizontal Elasticity**: Backend services remain strictly stateless; requests routed to Server A, B, or Z process seamlessly without cross-node session replication.
3. **Cross-Domain & Mobile Compatibility**: Operates independently of browser cookie domains, integrating seamlessly across iOS, Android, desktop, and third-party API clients.

## 4. How It Works (Access Token & Refresh Token Rotation Flow)

```
[Client / Mobile App]               [API Gateway / Service]            [Auth Service & Redis]
         |                                     |                                 |
         | 1. POST /login (email + password) ----------------------------------> |
         |                                     |                           (Validate DB)
         | <-------------------------------------------------------------------- |
         |    Returns:                         |                                 |
         |    * Access Token (Exp: 15 Mins)    |                                 |
         |    * Refresh Token (Exp: 7 Days)    |                                 |
         |                                     |                                 |
         | 2. GET /orders (Bearer AccessToken) |                                 |
         | ----------------------------------> |                                 |
         |                               [Verify Signature                       |
         |                                Locally via CPU]                       |
         | <---------------------------------- |                                 |
         |    HTTP 200 OK (Orders Data)        |                                 |
         |                                     |                                 |
         | ... 15 Minutes Later (Access Token Expired) ...                       |
         |                                     |                                 |
         | 3. GET /orders (Bearer AccessToken) |                                 |
         | ----------------------------------> |                                 |
         | <---------------------------------- |                                 |
         |    HTTP 401 Unauthorized (Expired)  |                                 |
         |                                     |                                 |
         | 4. POST /auth/refresh (Send Refresh Token) -------------------------> |
         |                                     |                           [Verify Refresh Token
         |                                     |                            in Redis / DB]
         | <-------------------------------------------------------------------- |
         |    Returns: New Access Token + New Refresh Token (Rotation Complete!) |
```

## 5. Core Implementation Logic (JWT Generator & Middleware Verifier in Go)

Below is an enterprise implementation in Go using the official `github.com/golang-jwt/jwt/v5` package:

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"net/http"
	"strings"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

var jwtSecretKey = []byte("super_secret_signing_key_production_2026")

// CustomClaims encapsulates application-specific token payload fields
type CustomClaims struct {
	UserID string `json:"user_id"`
	Role   string `json:"role"`
	jwt.RegisteredClaims
}

// GenerateAccessToken produces short-lived authentication tokens (15-minute validity)
func GenerateAccessToken(userID, role string) (string, error) {
	claims := CustomClaims{
		UserID: userID,
		Role:   role,
		RegisteredClaims: jwt.RegisteredClaims{
			ExpiresAt: jwt.NewNumericDate(time.Now().Add(15 * time.Minute)),
			IssuedAt:  jwt.NewNumericDate(time.Now()),
			Issuer:    "company-auth-service",
		},
	}

	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return token.SignedString(jwtSecretKey)
}

// JWTMiddleware performs local cryptographic validation without database lookups
func JWTMiddleware(next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		authHeader := r.Header.Get("Authorization")
		if authHeader == "" || !strings.HasPrefix(authHeader, "Bearer ") {
			http.Error(w, "Unauthorized: missing bearer token", http.StatusUnauthorized)
			return
		}

		tokenString := strings.TrimPrefix(authHeader, "Bearer ")

		// Parse and verify signature mathematically via CPU
		token, err := jwt.ParseWithClaims(tokenString, &CustomClaims{}, func(t *jwt.Token) (interface{}, error) {
			// Defend against the "none" algorithm exploit
			if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
				return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
			}
			return jwtSecretKey, nil
		})

		if err != nil || !token.Valid {
			http.Error(w, "Unauthorized: invalid or expired token", http.StatusUnauthorized)
			return
		}

		claims, ok := token.Claims.(*CustomClaims)
		if !ok {
			http.Error(w, "Unauthorized: invalid claims payload", http.StatusUnauthorized)
			return
		}

		// Propagate authenticated context to subsequent business handlers
		ctx := context.WithValue(r.Context(), "user_id", claims.UserID)
		ctx = context.WithValue(ctx, "user_role", claims.Role)
		next(w, r.WithContext(ctx))
	}
}
```

## 6. Trade-offs (Pros & Cons)

### Pros
- **Stateless & High Performance**: Bypasses centralized session database reads on every HTTP request.
- **Decoupled Architecture**: Downstream microservices authenticate clients even during identity provider outages.
- **Cross-Platform Friendly**: Natively integrated across Single-Page Apps (React/Vue/Next.js), native mobile apps, and server-to-server APIs.

### Cons
- **Instant Revocation Challenges**: Compromised tokens remain usable until natural expiration (`exp`) unless expensive blacklist checks are introduced.
- **HTTP Header Overhead**: JWT strings range from 300 to 1,000 bytes, consuming higher bandwidth compared to 32-byte session cookies.
- **Public Visibility of Claims**: Payloads are merely Base64-encoded; sensitive credentials must never reside inside claims.

## 7. When to Use vs When to Avoid

### When to Use
- High-throughput distributed microservices where centralized session querying creates severe infrastructure bottlenecks.
- Mobile APIs and public-facing developer ecosystems requiring stateless token verification.
- Single Sign-On (SSO) federations spanning disparate domain boundaries.

### When to Avoid
- Traditional monolithic web applications serving browser clients (server-managed HTTP-Only cookie sessions provide simpler, instantaneous revocation).
- Strictly regulated financial systems requiring instantaneous sub-millisecond session termination upon threat detection.

## 8. Key Observability Metrics (Monitoring Dashboard)

- **`jwt_verification_failures_total` (Counter)**: Tracks malformed signatures or altered payloads; sudden spikes indicate brute-force attempts or signing key misconfigurations.
- **`jwt_expired_tokens_total` (Counter)**: Frequency of rejected requests bearing expired timestamps.
- **`refresh_token_reuse_detected` (Counter)**: Critical security metric flagging attempts to replay invalidated refresh tokens, signaling client-side token theft.

## 9. Real-World Case Studies

- **Netflix**: Deploys tokenized authentication using a binary variant (*MSL - Media Saling Layer*) across millions of smart TVs and devices worldwide. Edge gateways validate signatures before traffic enters internal microservices.
- **Auth0 / Okta**: Standardizes open enterprise identity using asymmetric (RSA/ECDSA) signed JWTs in OpenID Connect (OIDC), enabling global consumer applications to verify user identity via public keys hosted at `/.well-known/jwks.json`.

## 10. Production Gotchas

1. **Storing Secrets in Payloads**: Developers mistakenly assuming JWTs are encrypted. Anyone pasting an encoded token into `jwt.io` can read all claims instantly.  
   *Mitigation*: Never place passwords, credit cards, or internal security secrets inside JWT claims.
2. **Excessive Access Token Validity Periods**: Setting 30-day access token lifespans to avoid implementing refresh token flows. Stolen tokens grant attackers unrestricted month-long access.  
   *Mitigation*: Restrict access tokens to 15 minutes; implement secure refresh token rotation.
3. **The "none" Algorithm Vulnerability**: Attackers tampering with headers to specify `{"alg": "none"}` and omitting signatures. Unsanitized verifiers accept the token as valid.  
   *Mitigation*: Enforce explicit cryptographic signing algorithm validation in Go middleware verifiers.

## 11. Quick Summary

JWT is a self-contained, stateless authentication standard enabling local cryptographic identity verification without database read bottlenecks. Production architectures combine short-lived access tokens for peak throughput with refresh token rotation for robust long-term security.
