# HipnoApp Backend API - Technical README

**Base URL (example):** `https://api.hipnoapp.example.com/v1`

Authentication: All protected endpoints require a valid Firebase ID Token passed in `Authorization: Bearer <ID_TOKEN>` header. Backend verifies token with Firebase Admin SDK. Some endpoints also require a server-issued session JWT for additional session validation.
**Dynamic User Flags:**
For each authenticated request, the backend fetches dynamic user flags from Firestore (`/users/{uid}`), such as `premium`, `trial`, `banned`, etc. These flags are available in handlers via `c.Locals("flags")`.

**Usage example in Go:**
```go
flags := c.Locals("flags").(map[string]interface{})
if flags["premium"] == true {
  // Permitir acceso a contenido premium
}
if flags["banned"] == true {
  return c.Status(403).JSON(fiber.Map{"error": "Usuario baneado"})
}
```
---

## Authentication & Session
### POST /v1/auth/login
- **Description:** Login via email and password validates the Firebase token and registers or updates the user's profile in Firestore.
Dynamic fields such as `premium`, `trial`, `trialEndsAt`, `banned`, and others are automatically initialized or updated based on backend logic (e.g., when activating a trial, registering a subscription, or applying a ban). The user cannot modify these flags directly.
- **Auth:** None (accepts Firebase ID token)
- **Body:**
```json
{
    "idToken": "<FIREBASE-TOKEN>",
    "displayName": "Gabriela Molina",
    "gender": "male",
    "birthDate": "2020-01-01T00:00:00Z",
    "age": 30,
    "language": "es",
    "country": "CO",
    "plan": "free",
    "planStatus": "active",
    "planExpiration": "2025-12-31T23:59:59Z",
    "trialDaysLeft": 7,
    "subscriptionSource": "GooglePlay"
  }```
- **Response:**
```json
{
    "jwt": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3NTc2NDc2OTAsImlhdCI6MTc1NzU2MTI5MCwicm9sZSI6InVzZXIiLCJ1aWQiOiIxbTRxb0JNYWh1WTllUVNuN2lCYTlzMlRuVXYxIn0.DpwWw-jI-GF2k3p9e7l-Xj0Lz7rvL-dOiVIo-T972wA",
    "user": {
        "UID": "1m4qoBMahuY9eQSn7iBa9s2TnUv1",
        "Email": "faus_tito29@hotmail.com",
        "DisplayName": "Gabriela Molina",
        "Gender": "male",
        "BirthDate": "2020-01-01T00:00:00Z",
        "Age": 30,
        "Language": "es",
        "Country": "CO",
        "Role": "",
        "Plan": "free",
        "PlanStatus": "active",
        "PlanExpiration": "2025-12-31T23:59:59Z",
        "Premium": false,
        "Trial": true,
        "TrialEndsAt": "2025-09-17T22:28:09.4911412-05:00",
        "Banned": false,
        "trialDaysLeft": 7,
        "SubscriptionSource": "GooglePlay",
        "CreatedAt": "2025-09-10T22:28:09.4911412-05:00",
        "UpdatedAt": "2025-09-10T22:28:09.4911412-05:00"
    }
}
```

## Authentication & Session
### POST /v1/auth/social-login
- **Description:** Authenticates a user using a Google or Apple ID token. Returns a JWT token and user role.
- **Body:**
```json
{
  "provider": "google", // or "apple"
  "token": "<ID_TOKEN_FROM_PROVIDER>"
}
```
- **Response:**
```json
{
    "jwt": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6ImFkbWluQGhpcG5vYXBwLmNvbSIsImV4cCI6MTc1NzgxOTk0OSwicm9sZSI6InVzZXIifQ.Idi9tfYj2JuSp79bD4PZjI4KppEZnq4fDWc8GXUUlEg",
    "user": {
        "UID": "104643996698677810244",
        "Email": "admin@hipnoapp.com",
        "DisplayName": "edwin javier Lamiña",
        "Gender": "",
        "BirthDate": "0001-01-01T00:00:00Z",
        "Age": 0,
        "Language": "",
        "Country": "",
        "Role": "user",
        "Plan": "free",
        "PlanStatus": "",
        "PlanExpiration": "2025-12-10T22:19:08.4545475-05:00",
        "Premium": false,
        "Trial": true,
        "TrialEndsAt": "2025-09-17T22:19:08.4545475-05:00",
        "Banned": false,
        "trialDaysLeft": 0,
        "SubscriptionSource": "",
        "CreatedAt": "2025-09-10T22:19:08.4545475-05:00",
        "UpdatedAt": "2025-09-10T22:19:08.4545475-05:00"
    }
}
```
- **Example cURL:**
```sh
curl -X POST https://your-domain.up.railway.app/v1/auth/social-login \
  -H "Content-Type: application/json" \
  -d '{
    "provider": "google",
    "token": "eyJhbGciOiJSUzI1NiIsImtpZCI6Ij..."
  }'
```


### POST /v1/auth/refresh
- **Description:** Refresh server JWT using a valid refresh token. This endpoint allows the client to obtain a new server JWT when the current one is about to expire, maintaining the session without requiring the user to re-authenticate.
- **Auth:** Bearer server JWT (can be expired if refreshToken is valid)
- **Body:**
  ```json
  { "refreshToken": "valid_refresh_token" }
  ```
- **Response:**
  ```json
  {
    "jwt": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3NTY3NDk2MTMsImlhdCI6MTc1NjY2MzIxMywicm9sZSI6InVzZXIiLCJ1aWQiOiJ1c2VyMTIzIn0.Ibo6-21LCIaQy-ZJEqZN8uRVPU_EiBegARBMElQSyn8"
  }
  ```
- **Notes:**
  - The response contains only the new JWT. User profile and flags must be fetched separately if needed.
  - The JWT payload includes fields like `exp` (expiration), `iat` (issued at), `role`, and `uid`.
- **Curl Example:**
  ```sh
  curl -X POST "http://localhost:8080/v1/auth/refresh" \
    -H "Authorization: Bearer <EXPIRED_OR_VALID_SERVER_JWT>" \
    -H "Content-Type: application/json" \
    -d '{ "refreshToken": "valid_refresh_token" }'
  ```
- **Errors:**
  - 400: Refresh token missing or invalid
  - 401: Invalid or expired JWT and refresh token
  - 403: User banned or session revoked
  - 500: Internal server error

- **Curl Example:**
  ```sh
  curl -X POST "http://localhost:8080/v1/auth/refresh" \
    -H "Authorization: Bearer <EXPIRED_OR_VALID_SERVER_JWT>" \
    -H "Content-Type: application/json" \
    -d '{ "refreshToken": "<valid_refresh_token>" }'
  ```

### POST /v1/auth/signout
- **Description:** Invalidates the server session JWT and optionally revokes the Firebase refresh token.
- **Authentication:** Requires Bearer server JWT in the Authorization header.

**Request:**

No body required.

**Response:**

```
{
  "message": "Session successfully invalidated"
}
```

**Errors:**
- 401: Missing or invalid JWT

---

> **Role Note:** The endpoints in this section can only be accessed by users with the `user` role (Bearer server JWT with `role: "user"`). The `admin` role does not have access to these endpoints, except where explicitly indicated (for example, profile queries).


### GET /v1/users/{userId}
- **Description:** Get user profile, subscription status, flags dinámicos y rol.
- **Auth:** Bearer server JWT (user or admin)
- **Response:**
```json
{
  "user": {
    "uid": "abc123",
    "email": "usuario@email.com",
    "displayName": "Usuario Ejemplo",
    "role": "user",
    "plan": "free",
    "premium": false,
    "trial": true,
    "trialEndsAt": "2025-09-01T00:00:00Z",
    "planExpiresAt": "2025-12-31T00:00:00Z",
    "banned": false,
    "trialDaysLeft": 7,
    "SubscriptionSource": "GooglePlay",
    "CreatedAt": "2025-09-08T01:52:28.705391Z",
    "UpdatedAt": "2025-09-08T01:53:11.127021Z"
  }
}
```

### PATCH /v1/users/{userId}
- **Description:** Updates the user's complete profile. Allows modifying fields such as `displayName`, `email`, `gender`, `birthDate`, `age`, `language`, `country`, as well as subscription information (`plan`, `planStatus`, `planExpiration`, `premium`, `trial`, `trialEndsAt`, `trialDaysLeft`, `subscriptionSource`), flags (`banned`), and creation/update dates.
- **Auth:** Bearer server JWT (rol: user)
- **Body:**
  ```json
  {
    "UID": "ZooCaNaIkQQR4FVG5Ywyb81F9ww1",
    "Email": "tourogabi@gmail.com",
    "DisplayName": "Gaby molina",
    "Gender": "male",
    "BirthDate": "2020-01-01T00:00:00Z",
    "Age": 30,
    "Language": "es",
    "Country": "CO",
    "Role": "",
    "Plan": "free",
    "PlanStatus": "active",
    "PlanExpiration": "2025-12-31T23:59:59Z",
    "Premium": false,
    "Trial": true,
    "TrialEndsAt": "2025-09-15T01:52:28.705391Z",
    "Banned": false,
    "trialDaysLeft": 7,
    "SubscriptionSource": "GooglePlay",
    "CreatedAt": "2025-09-08T01:52:28.705391Z",
    "UpdatedAt": "2025-09-08T01:53:11.127021Z"
  }
  ```
- **Response:**
  ```json
  { "message": "User updated" }
  ```
- **Curl Example:**
  ```sh
  curl -X PATCH "http://localhost:8080/v1/users/ZooCaNaIkQQR4FVG5Ywyb81F9ww1" \
    -H "Authorization: Bearer <SERVER_JWT>" \
    -H "Content-Type: application/json" \
    -d '{
      "UID": "ZooCaNaIkQQR4FVG5Ywyb81F9ww1",
      "Email": "tourogabi@gmail.com",
      "DisplayName": "Gaby molina",
      "Gender": "male",
      "BirthDate": "2020-01-01T00:00:00Z",
      "Age": 30,
      "Language": "es",
      "Country": "CO",
      "Role": "",
      "Plan": "free",
      "PlanStatus": "active",
      "PlanExpiration": "2025-12-31T23:59:59Z",
      "Premium": false,
      "Trial": true,
      "TrialEndsAt": "2025-09-15T01:52:28.705391Z",
      "Banned": false,
      "trialDaysLeft": 7,
      "SubscriptionSource": "GooglePlay",
      "CreatedAt": "2025-09-08T01:52:28.705391Z",
      "UpdatedAt": "2025-09-08T01:53:11.127021Z"
    }'
  ```
- **Errors:**
  - 400: Invalid body or missing fields
  - 401: Not authenticated
  - 403: User banned or lacks permissions
  - 500: Internal error updating user

### POST /v1/users/{userId}/device/register
- **Description:** Register a device for push notifications and analytics. Stores deviceId, platform, osVersion, fcmToken.
- **Auth:** Bearer server JWT
- **Body:** `{ "deviceId": "...", "platform": "iOS|Android|Web", "osVersion": "...", "fcmToken": "..." }`

### POST /v1/users/{userId}/device/unregister
- **Description:** Unregister a user's device (removes the deviceId from the user's devices collection in Firestore)
				   														
- **Auth:** Bearer server JWT
- **Body:**
```json
{ "deviceId": "device-123" }
							  
```
- **Response:**
```json
{ "message": "Dispositivo desregistrado" }
```												  
- **Errors:**
  - 400: Missing userId or deviceId
  - 500: Error unregistering device
- **Curl Example:**
```sh
curl -X POST "http://localhost:8080/v1/users/{userId}/device/unregister" \
  -H "Authorization: Bearer <SERVER_JWT>" \
  -H "Content-Type: application/json" \
  -d '{ "deviceId": "device-123" }'
```												 
### POST /v1/users/:userId/profile-image
- **Description:** Generates a signed URL so the frontend can upload the user's profile image directly to Cloudflare R2.
				   														
- **Auth:** Bearer server JWT
- **Parámetros:**
  - userId (path): User ID.
  - contentType (query, optional): image/jpeg (default) or image/png.

- **Response:**
```json
{
  "uploadUrl": "https://<bucket>.r2.cloudflarestorage.com/users/USERID/profile.jpg?...",
  "objectKey": "users/USERID/profile.jpg"
}
```												  
- **Errors:**
  - 400: Missing userId or deviceId
  - 500: Error unregistering device
- **Curl Example:**
```sh
curl -X POST "http://localhost:8080/v1/users/ZooCaNaIkQQR4FVG5Ywyb81F9ww1/profile-image?contentType=image/jpeg" \
  -H "Authorization: Bearer <JWT>" \
  -H "Content-Type: application/json"
```													 
---

### PUT <uploadUrl>
- **Description:** Upload the image to the signed Cloudflare R2 URL received in the previous step.

- **Headers:**
  - Content-Type: image/jpeg o image/png

- **Response:**
  - 200 OK
											  
- **Errors:**
  - 500: Error unregistering device

- **Curl Example:**
```sh
curl -X PUT "<uploadUrl>" \
  -H "Content-Type: image/jpeg" \
  --data-binary "@ruta/a/tu/imagen.jpg"
```		

---
### PATCH /v1/users/:userId/profile-image-url
- **Description:** Saves the public profile image URL in the user's document in Firestore.
				   														
- **Auth:** Bearer server JWT

- **Body:**
```json
{
  "imageUrl": "https://<bucket>.r2.cloudflarestorage.com/users/USERID/profile.jpg"
}
							  
```
- **Response:**
```json
{
  "message": "Profile image URL updated",
  "imageUrl": "https://<bucket>.r2.cloudflarestorage.com/users/USERID/profile.jpg"
}
```											  
- **Errors:**
  - 400: Missing userId or deviceId
  - 500: Error unregistering device
- **Curl Example:**
```sh
curl -X PATCH "http://localhost:8080/v1/users/ZooCaNaIkQQR4FVG5Ywyb81F9ww1/profile-image-url" \
  -H "Authorization: Bearer <JWT>" \
  -H "Content-Type: application/json" \
  -d '{"imageUrl": "https://<bucket>.r2.cloudflarestorage.com/users/ZooCaNaIkQQR4FVG5Ywyb81F9ww1/profile.jpg"}'
```	
---

### GET /v1/users/:userId/profile-image/download
- **Description:** Generates a temporary signed URL to download the user's profile image from Cloudflare R2. The URL expires after a few minutes for security.
				   														
- **Auth:** Bearer server JWT
- **Parameters:**
  - userId (path): User ID
						  
- **Response:**
```json
{
  "signedUrl": "https://r2.example.com/bucket/user123/profile.jpg?...",
  "expiresAt": "2025-09-07T12:00:00Z"
}
```											  
- 400: Missing parameters or invalid URL
- 404: User or image not found
- 500: Internal error (Firestore, R2, etc)

- **Curl Example:**
```sh
curl -X GET "https://api.hipnoapp.com/v1/users/USER_ID/profile-image/download" -H "Authorization: Bearer <token>"
```	
---

### POST /v1/auth/forgot-password
- **Description:** Generates a 6-digit verification code, stores it in Firestore, and sends it to the registered email. The code expires after 5 minutes (configurable).

- **Params:**
```json
{ "email": "usuario@email.com" }
```

- **Response:**
```json
{ "message": "Código enviado", "expiresAt": "2025-09-07T12:05:00Z" }
```											  
- **Errors:**
 - 400: Email required
 - 404: User not found
 - 500: Internal error

- **Curl Example:**
```sh
curl -X POST "http://localhost:8080/v1/auth/forgot-password" \
  -H "Content-Type: application/json" \
  -d '{ "email": "usuario@email.com" }'
```	
---

### POST /v1/auth/resend-code
- **Description:**  Reenvía el código de verificación al correo si no ha expirado.

- **Params:**
```json
{ "email": "usuario@email.com" }
```

- **Response:**
```json
{ "message": "Código enviado", "expiresAt": "2025-09-07T12:05:00Z" }
```											  
- **Errors:**
- 400: Email required
- 404: User not found
- 500: Internal error

- **Curl Example:**
```sh
curl -X POST "http://localhost:8080/v1/auth/resend-code" \
  -H "Content-Type: application/json" \
  -d '{ "email": "usuario@email.com" }'
```	
---


## Admin / CMS Endpoints
> **Role Note:** The endpoints in this section can only be accessed by users with the `admin` role (Bearer server JWT with `role: "admin"`). Regular users (`user` role) do not have access to these endpoints.

### POST /v1/admin/assign-role
- **Description:** Assigns the `admin` or `user` role to a user in Firebase Auth (only admins can use this endpoint).
- **Auth:** Bearer server JWT (admin)
- **Body:**
```json
{
  "uid": "UID_USER",
  "role": "admin" // o "user"
}
```
- **Response:**
```json
{ "message": "Rol asignado", "uid": "UID_DEL_USUARIO", "role": "admin" }
```
- 400: Missing uid or role, or invalid role
- 500: Error assigning role
- **Curl Example:**
```sh
# Usando JWT de sesión admin
curl -X POST "http://localhost:8080/v1/admin/assign-role" \
  -H "Authorization: Bearer <SERVER_JWT_ADMIN>" \
  -H "Content-Type: application/json" \
  -d '{ "uid": "UID_DEL_USUARIO", "role": "admin" }'

# Usando ID Token de Firebase admin
curl -X POST "http://localhost:8080/v1/admin/assign-role" \
  -H "Authorization: Bearer <ID_TOKEN_ADMIN>" \
  -H "Content-Type: application/json" \
  -d '{ "uid": "UID_DEL_USUARIO", "role": "admin" }'
```

### POST /v1/admin/sessions
- **Description:** Create new audio session (admin). The session only contains the general image and metadata. Audios are managed in a separate subcollection.
- **Auth:** Bearer server JWT (admin)
- **Body:**
```json
{
  "title": "Relajación profunda",
  "category": "relax",
  "duration": 600,
  "isPro": true,
  "tags": ["relax", "sleep"],
  "level": "beginner",
  "releaseDate": "2025-08-17",
  "imageUrl": "https://r2.cdn/sessions/session123.jpg",
  "description": "Sesión guiada para relajación profunda."
}
```
- **Response:**
```json
{ "id": "sessionId", "message": "Sesión creada" }
```
- **Notas:**
  - Para agregar audios, usa el endpoint `/v1/admin/sessions/{sessionId}/audios`.
- **Curl Example:**
```sh
curl -X POST "http://localhost:8080/v1/admin/sessions" \
  -H "Authorization: Bearer <SERVER_JWT_ADMIN>" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Relajación profunda",
    "category": "relax",
    "duration": 600,
    "isPro": true,
    "tags": ["relax", "sleep"],
    "level": "beginner",
    "releaseDate": "2025-08-17",
    "imageUrl": "https://r2.cdn/sessions/session123.jpg",
    "description": "Sesión guiada para relajación profunda."
  }'
```

### PATCH /v1/admin/sessions/{sessionId}
- **Description:** Update session metadata.
- **Auth:** Bearer server JWT (admin)
- **Body:**
```json
{
  "title": "Nuevo título",
  "category": "relax",
  "duration": 700,
  "isPro": false,
  "tags": ["relax", "focus"],
  "level": "intermediate",
  "releaseDate": "2025-09-01",
  "audioUrl": "",
  "imageUrl": "",
  "description": "Sesión actualizada."
}
```
- **Response:**
```json
{ "id": "sessionId", "message": "Sesión actualizada" }
```
- **Curl Example:**
```sh
curl -X PATCH "http://localhost:8080/v1/admin/sessions/{sessionId}" \
  -H "Authorization: Bearer <ADMIN_JWT>" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Nuevo título",
    "category": "relax",
    "duration": 700,
    "isPro": false,
    "tags": ["relax", "focus"],
    "level": "intermediate",
    "releaseDate": "2025-09-01",
    "audioUrl": "",
    "imageUrl": "",
    "description": "Sesión actualizada."
  }'
```
### POST /v1/admin/sessions/{sessionId}/audios
- **Description:** Adds an audio track to the specified session. Each audio has its own file, image, title, duration, and order.
- **Auth:** Bearer server JWT (admin)
- **Path Params:**
  - `sessionId`: ID of the session to which the audio is being added.
  **Body:**
  ```json
  {
    "title": "Intro",
    "audioUrl": "https://r2.cdn/sessions/session123/audio1.mp3", // Referencia lógica, no se usa para descarga directa
    "imageUrl": "https://r2.cdn/sessions/session123/audio1.jpg",
    "duration": 300,
    "order": 1,
    "description": "Audio introductorio."
  }
  ```
**Response:**
  ```json
  { "audioId": "audio1", "message": "Audio agregado" }
  ```
**Curl Example:**
  ```sh
  curl -X POST "http://localhost:8080/v1/admin/sessions/session123/audios" \
    -H "Authorization: Bearer <SERVER_JWT_ADMIN>" \
    -H "Content-Type: application/json" \
    -d '{
      "title": "Intro",
      "audioUrl": "https://r2.cdn/sessions/session123/audio1.mp3",
      "imageUrl": "https://r2.cdn/sessions/session123/audio1.jpg",
      "duration": 300,
      "order": 1,
      "description": "Audio introductorio."
    }'
  ```

> **Important Note:**
> The `audioUrl` field is only a logical reference to the object in R2. To download or play the audio, the client must request the signed URL using the `/v1/sessions/{sessionId}/download` endpoint, sending both the `sessionId` and the corresponding `audioId`. The backend validates access and generates a temporary URL for that specific audio.

### DELETE /v1/admin/sessions/{sessionId}
- **Description:** Delete session metadata (soft-delete recommended).
- **Auth:** Bearer server JWT (admin)
- **Response:**
```json
{ "id": "sessionId", "message": "Session deleted" }
```
- **Curl Example:**
```sh
curl -X DELETE "http://localhost:8080/v1/admin/sessions/{sessionId}" \
  -H "Authorization: Bearer <ADMIN_JWT>"
```

### POST /v1/admin/upload
**Description:** Requests a signed upload URL for Cloudflare R2. Allows uploading audio files (`.mp3`) or images (`.jpg`) for a specific session or audio. The object path is automatically generated based on the file type.
**Auth:** Bearer server JWT (admin)
**Body:**
```json
{
  "sessionId": "rZ8id6DFPyYbEFDGPmuf",
  "audioId": "CqJDS3QHrpWnmR15T9sv",
  "type": "audio", // o "image"
  "contentType": "audio/mpeg" // o "image/jpeg"
}
```
**Response:**
```json
{
  "uploadUrl": "...",
  "objectKey": "sessions/rZ8id6DFPyYbEFDGPmuf/CqJDS3QHrpWnmR15T9sv.mp3" // o .jpg
}
```
**Curl Example (audio):**
```sh
curl -X POST "http://localhost:8080/v1/admin/upload" \
  -H "Authorization: Bearer <SERVER_JWT_ADMIN>" \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "rZ8id6DFPyYbEFDGPmuf",
    "audioId": "CqJDS3QHrpWnmR15T9sv",
    "type": "audio",
    "contentType": "audio/mpeg"
  }'
```
**Curl Example (imagen asociada a audio):**
```sh
curl -X POST "http://localhost:8080/v1/admin/upload" \
  -H "Authorization: Bearer <SERVER_JWT_ADMIN>" \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "rZ8id6DFPyYbEFDGPmuf",
    "audioId": "CqJDS3QHrpWnmR15T9sv",
    "type": "image",
    "contentType": "image/jpeg"
  }'
```
**Curl Example (imagen de portada de sesión):**
```sh
curl -X POST "http://localhost:8080/v1/admin/upload" \
  -H "Authorization: Bearer <SERVER_JWT_ADMIN>" \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "rZ8id6DFPyYbEFDGPmuf",
    "type": "image",
    "contentType": "image/jpeg"
  }'
```

### PUT <uploadUrl> (Actual file upload to Cloudflare R2)
- **Description:** Upload the audio or image file directly to Cloudflare R2 using the pre-signed `uploadUrl` obtained from the backend.
- **Auth:** No authentication required (the URL already includes temporary permissions).
- **Headers:**
  - `Content-Type: audio/mpeg` (for audio)
  - `Content-Type: image/jpeg` (for image)
- **Body:** Binary file content.
- **Notes:**
  - The `uploadUrl` is only valid for a limited time.
  - The backend does not receive the file directly; it only manages the URL generation.
- **Ejemplo de uso (audio):**
  ```sh
  curl -X PUT "<uploadUrl>" \
    -H "Content-Type: audio/mpeg" \
    --data-binary "@/ruta/al/audio.mp3"
  ```
**Ejemplo de uso (imagen asociada a audio):**
  ```sh
  curl -X PUT "<uploadUrl>" \
    -H "Content-Type: image/jpeg" \
    --data-binary "@C:/ruta/a/imagen.jpg"
  ```

**Notas importantes:**
- Verifica que el archivo fuente existe y tiene contenido antes de subir.
- En Windows PowerShell, usa la ruta completa y asegúrate de que el archivo no esté vacío.
- Si el objeto aparece con tamaño 0 B en R2, revisa el método de subida y el archivo fuente.

---

## Sessions (Content)

### GET /v1/sessions
- **Description:** List sessions with filters (category, tags, level, releaseDate, isPro, freeToday).
- **Auth:** Optional (returns availability based on user plan if auth provided)
- **Query params:**
  - `category`: string (ej: "relax")
  - `tags`: string, etiquetas separadas por coma (ej: "sueño,calma")
  - `level`: string (ej: "beginner")
  - `releaseDate`: string (YYYY-MM-DD)
  - `isPro`: boolean
  - `freeToday`: boolean
  - `page`: integer (default: 1)
  - `pageSize`: integer (default: 20)
- **Curl Example:**
```sh
curl -X GET "http://localhost:8080/v1/sessions?category=relax&tags=sueño,calma&level=beginner&page=1&pageSize=10" \
  -H "Authorization: Bearer <TU_TOKEN_JWT>"
```

### GET /v1/sessions/{sessionId}
- **Description:** Retrieves the metadata for a specific session, including title, description, duration, audio reference, image, Pro status, and tags.
- **Auth:** Optional (if a JWT is provided, the response may include availability information based on the user's plan).
- **Path Params:**
  - `sessionId`: ID of the session to query.
- **Response:**
  ```json
  {
    "session": {
      "id": "session123",
      "title": "Relajación profunda",
      "description": "Sesión guiada para relajación profunda.",
      "duration": 600,
      "audioUrl": "https://r2.cdn/sessions/session123.mp3",
      "imageUrl": "https://r2.cdn/sessions/session123.jpg",
      "isPro": true,
      "tags": ["relax", "sleep"],
      "level": "beginner",
      "releaseDate": "2025-08-17"
    }
  }
  ```
- **Errores:**
  - 404: Session not found
  - 403: Access denied (if the session is Pro and the user does not have access)
- **Curl Example:**
  ```sh
  curl -X GET "http://localhost:8080/v1/sessions/session123" \
    -H "Authorization: Bearer <TU_TOKEN_JWT>"
  ```

## Sesiones Inscritas y Activación de Audios

### 1. Session registration
#### POST /v1/sessions/{sessionId}/register
- **Description:** Registers the user in a session. Validates that the user does not have more than 3 active sessions. Records the registration date and saves the main metadata (`title`, `progress`, `completedAudios`, etc.) in the user's session document. If `title` is not sent, it is automatically obtained from the session metadata.
- **Auth:** Bearer server JWT
- **Path Params:**
  - `sessionId`: ID of the session to register.
- **Response:**
  ```json
  {
    "message": "Session registered successfully",
    "sessionId": "session123",
    "title": "Deep relaxation",
    "progress": 0.0,
    "enrolledAt": "...",
    "status": "active",
    "completedAudios": []
  }
  ```
- **Errors:**
  - 400: Maximum 3 active sessions allowed
  - 500: Connection or registration error
- **Curl Example:**
  ```sh
  curl -X POST "http://localhost:8080/v1/sessions/session123/register" \
    -H "Authorization: Bearer <TU_TOKEN_JWT>" \
    -H "Content-Type: application/json" \
    -d '{ "title": "Deep relaxation" }'
  ```

---

### 2. Listar sesiones inscritas
#### GET /v1/users/{userId}/sessions
- **Description (EN):** Returns all sessions for a given user.
- **Description (ES):** Devuelve todas las sesiones de un usuario.
- **Auth:** Bearer server JWT (user)
- **Response:**
```json
{
  "sessions": [
    {
      "sessionId": "string",
      "title": "string",
      "progress": 0.0,
      "enrolledAt": "string",
      "status": "string",
      "completedAudios": ["string"]
    }
  ]
}
```
- **Errors:**
  - 400: Missing userId / Falta userId
  - 500: Internal error / Error interno
  - **Curl Example:**
  ```sh
  curl -X GET "http://localhost:8080/v1/users/{userId}/sessions" \
    -H "Authorization: Bearer <TU_TOKEN_JWT>"
  ```

### 3. Query active audios in a session
#### GET /v1/sessions/{sessionId}/audios
<!--
  Returns the audios of a session and determines which are active for the user based on their enrollment date and the configured activation interval.

  ### Audio activation logic:
  1. The user's enrollment date for the session is obtained from `users/{userId}/sessions/{sessionId}`.
  2. The audio activation interval is fetched from the global configuration (`config/audioActivationInterval` in Firestore, default value: 3 days).
  3. Session audios are listed ordered by the `order` field.
  4. For each audio, it is calculated whether it is active:
    - The audio at position N (`order=N`) becomes active when at least N * interval days have passed since the enrollment date.
    - Example: If enrollment was on August 1 and the interval is 3 days:
      - Audio 1 (`order=1`): active from August 1.
      - Audio 2 (`order=2`): active from August 4.
      - Audio 3 (`order=3`): active from August 7.
  5. The `isActive` field is determined for each audio and included in the response.

  This logic allows audios to be progressively unlocked as the user advances through the session, promoting staggered access to content.

  ### Authentication:
  - Requires Bearer authentication with server JWT.
-->
- **Description:** Returns the session audios and which are active for the user based on enrollment date and configured interval.
- **Auth:** Bearer server JWT
- **Query params:** `userId`
- **Response:**
  ```json
  {
    "audios": [
      {
        "audioId": "G53bywy6NQAKHopdtFPa",
        "isActive": true,
        "order": 1,
        "title": "Amor 1"
      }
      // ...otros audios...
    ]
  }
  ```
- **Errores:**
  - 404: Session not found
  - 403: Access denied (if the user does not have access to active audios)
- **Curl Example:**
  ```sh
  curl -X GET "http://localhost:8080/v1/sessions/{sessionId}/audios?userId={userId}" \
    -H "Authorization: Bearer <TU_TOKEN_JWT>"
  ```

---


### Complete Session
#### POST /v1/sessions/{sessionId}/complete
- **Description:** Marks the session as completed for the user. Updates the enrolled session status in Firestore (`status: completed`), records the completion date, and allows the user to enroll in a new session if they have fewer than 3 active sessions.
- **Logic:**
  1. Verifies that the user is enrolled in the session.
  2. Changes the session status to `completed` and records the completion date.
  3. Allows enrollment in a new session if the user has fewer than 3 active sessions.
  4. Updates the user's progress and statistics.
- **Auth:** Bearer server JWT
- **Body:**
  ```json
  { "userId": "abc123" }
  ```
- **Response:**
  ```json
  { "message": "Sesión completada", "sessionId": "session123" }
  ```
- **Errors:**
  - 404: Session not found or user not enrolled
  - 403: Access denied
  - 500: Error updating session
- **Curl Example:**
  ```sh
  curl -X POST "http://localhost:8080/v1/sessions/session123/complete" \
    -H "Authorization: Bearer <TU_TOKEN_JWT>" \
    -H "Content-Type: application/json" \
    -d '{ "userId": "abc123" }'
  ```
---

#### POST /v1/admin/config/audio-activation-interval
- **Description:** Updates the global configuration for progressive audio activation and the maximum number of active sessions allowed for enrollment.
- **Auth:** Bearer server JWT (admin)
- **Body:**
  ```json
  {
    "intervalDays": 5,
    "maxActiveSessions": 3
  }
  ```
- **Response:**
  ```json
  {
    "message": "Configuration updated",
    "intervalDays": 5,
    "maxActiveSessions": 3
    }
    ```
  - **Errors:**
    - 400: Invalid body or missing fields
    - 401: Unauthorized (invalid JWT or missing admin role)
    - 500: Internal error or missing credentials
- **Curl Example:**
  ```sh
  curl -X POST "http://localhost:8080/v1/admin/config/audio-activation-interval" \
    -H "Authorization: Bearer <SERVER_JWT_ADMIN>" \
    -H "Content-Type: application/json" \
    -d '{ "intervalDays": 5, "maxActiveSessions": 3 }'
  ```
  ### POST /v1/sessions/{sessionId}/start

  - **Description:** Marks the start of a session for the user. Creates a preliminary entry in the session history (`sessions_history`) that records when the user started the session, from which device and location. This record is used to calculate total listening time, statistics, and achievements.
    - **Authentication:** Requires server JWT (Bearer).
    - **Body:**
    ```json
    {
      "userId": "abc123",
      "deviceInfo": {
        "deviceId": "device-123",
        "platform": "iOS",
        "osVersion": "17.5"
      },
      "location": {
        "lat": -34.6037,
        "lng": -58.3816
      }
    }
    ```
  - **Response:**
    ```json
    {
      "message": "Session start recorded",
      "sessionId": "session123",
      "startedAt": "2025-08-10T12:34:56Z"
    }
    ```
    - **Notes:**
    - The backend records the start time and associates device and location if provided.
    - The entry in `sessions_history` will be updated later with progress and completion.
    - Can be used for statistics, achievements, and usage tracking.
    - **Errors:**
    - 400: Missing or invalid data
    - 401: Not authenticated
    - 404: Session not found
    - 500: Internal error
    - **Curl Example:**
    ```sh
    curl -X POST "http://localhost:8080/v1/sessions/session123/start" \
      -H "Authorization: Bearer <TU_TOKEN_JWT>" \
      -H "Content-Type: application/json" \
      -d '{
        "userId": "abc123",
        "deviceInfo": {
          "deviceId": "device-123",
          "platform": "iOS",
          "osVersion": "17.5"
        },
        "location": {
          "lat": -34.6037,
          "lng": -58.3816
        }
      }'
    ```

### POST /v1/sessions/{sessionId}/heartbeat

- **Description:** This endpoint receives periodic "heartbeat" updates from the client while the user is playing an audio session. It supports sessions with multiple audios: each heartbeat can include the `audioId` field to identify the specific audio being played. The backend records real-time progress, detects interruptions, buffering issues, and network connection type. The collected data is used for usage statistics, session resumption, quality monitoring, and user experience analysis. If `audioId` is sent, the heartbeat is stored under the corresponding audio's subcollection; if not, it is saved in the general heartbeats collection for the session.

- **Authentication:** Requires server JWT (Bearer).

- **Body:**
  ```json
  {
    "userId": "abc123",                // User ID (optional if present in JWT)
    "audioId": "audio1",               // (optional) ID of the audio being played
    "position": 123,                   // Current position in seconds within the audio
    "duration": 600,                   // Total duration of the audio in seconds
    "connectionType": "wifi",          // Connection type ("wifi", "cellular", "offline")
    "buffering": false,                // Whether the user is experiencing buffering
    "deviceInfo": {                    // Optional device information
      "deviceId": "device-123",
      "platform": "iOS",
      "osVersion": "17.5"
    },
    "timestamp": "2025-08-10T12:35:00Z" // Heartbeat timestamp (ISO8601, optional)
  }
  ```

- **Response:**
  ```json
  {
    "message": "Heartbeat registrado",
    "sessionId": "session123",
    "position": 123,
    "buffering": false
  }
  ```

- **Notes:**
  - The backend can use this data to update session progress, detect disconnections, save resume points, and analyze network issues.
  - It is recommended to send heartbeats every 10-30 seconds during active playback.
  - If the user loses connection, heartbeats can be stored locally and sent when connectivity is restored.

- **Errors:**
  - 400: Invalid body or missing fields
  - 401: Not authenticated
  - 404: Session not found
  - 500: Internal error

- **Curl Example:**
  ```sh
  curl -X POST "http://localhost:8080/v1/sessions/session123/heartbeat" \
    -H "Authorization: Bearer <TU_TOKEN_JWT>" \
    -H "Content-Type: application/json" \
    -d '{
          "userId": "edJpSZ76WPNHJ1rPoafv6rgYMKy2",
          "audioId": "CqJDS3QHrpWnmR15T9sv",
          "position": 15,
          "duration": 600,
          "connectionType": "wifi",
          "buffering": false,
          "timestamp": "2025-08-30T12:34:56Z",
          "deviceInfo": {
            "deviceId": "device-123",
            "platform": "iOS",
            "osVersion": "17.5"
          }
    }'
  ```


### POST /v1/sessions/{sessionId}/download
- **Description:** Requests a signed URL from the server to download the session audio for offline use. The backend validates the user's plan and daily download limits.
- **Auth:** Bearer server JWT
- **Body:**
  ```json
  {
    "userId": "abc123",
    "deviceId": "device-123"
  }
  ```
- **Response:**
  ```json
  {
    "signedUrl": "https://r2.cdn/sessions/session123/audio1.mp3?signature=...",
    "expiresAt": "2025-08-10T23:59:59Z"
  }
  ```
- **Notes:**
  - The signed URL allows downloading the specific audio or image file (`audioId`) directly from Cloudflare R2 for a limited time.
  - The backend logs the download to enforce daily limits and track usage per device.
  - If the user exceeds the download limit, a 403 error is returned.
  - You must include `audioId` in the request body to obtain the signed URL for the corresponding session resource.
  - The `type` parameter specifies whether you want to download the audio (`type: "audio"`) or the image (`type: "image"`).

- **Errors:**
  - 400: Missing data (userId, deviceId, audioId)
  - 403: Download limit reached or insufficient plan
  - 404: Session or audio not found
  - 500: Internal error

- **Curl Example:**
  ```sh
  # Download audio
  curl -X POST "http://localhost:8080/v1/sessions/session123/download" \
    -H "Authorization: Bearer <TU_TOKEN_JWT>" \
    -H "Content-Type: application/json" \
    -d '{
      "userId": "abc123",
      "deviceId": "device-123",
      "audioId": "CqJDS3QHrpWnmR15T9sv",
      "type": "audio"
    }'

    # Download image associated with audio
  curl -X POST "http://localhost:8080/v1/sessions/session123/download" \
    -H "Authorization: Bearer <TU_TOKEN_JWT>" \
    -H "Content-Type: application/json" \
    -d '{
      "userId": "abc123",
      "deviceId": "device-123",
      "audioId": "CqJDS3QHrpWnmR15T9sv",
      "type": "image"
    }'
    # Download session cover image
  curl -X POST "http://localhost:8080/v1/sessions/session123/download" \
    -H "Authorization: Bearer <TU_TOKEN_JWT>" \
    -H "Content-Type: application/json" \
    -d '{
      "userId": "abc123",
      "deviceId": "device-123",
      "type": "image"
    }'

  # Download image associated with audio
  curl -X POST "http://localhost:8080/v1/sessions/session123/download" \
    -H "Authorization: Bearer <TU_TOKEN_JWT>" \
    -H "Content-Type: application/json" \
    -d '{
      "userId": "abc123",
      "deviceId": "device-123",
      "audioId": "CqJDS3QHrpWnmR15T9sv",
      "type": "image"
    }'
  ```

### GET /v1/users/{userId}/downloads  
**Description:** Lists all sessions and audios downloaded by the user. Returns information about each download, including session, audio, type, and date.

- **Auth:** Bearer server JWT (role: user)
- **Path Params:**
  - `userId`: User ID

- **Response:**
```json
{
  "downloads": [
    {
      "sessionId": "session123",
      "audioId": "audio456",
      "title": "Relajación profunda",
      "downloadedAt": "2025-09-01T12:34:56Z",
      "type": "audio"
    },
    {
      "sessionId": "session789",
      "title": "Imagen portada",
      "downloadedAt": "2025-08-30T10:20:00Z",
      "type": "image"
    }
  ]
}
```
- **Errors:**
  - 400: Missing userId
  - 401: Not authenticated
  - 403: User banned or lacks permissions
  - 500: Internal error retrieving downloads

- **Curl Example:**
```sh
curl -X GET "http://localhost:8080/v1/users/abc123/downloads" \
  -H "Authorization: Bearer <TU_TOKEN_JWT>"
```
- **Extended details:**
  - The endpoint queries the `downloads` subcollection in Firestore under the user's document (`users/{userId}/downloads`).
  - Each document represents a download made by the user, either audio or image.
  - Fields include session identifier (`sessionId`), audio identifier (`audioId`, optional), title, download date (`downloadedAt`, ISO8601 format), and type (`audio` or `image`).
  - The backend validates the JWT and user flags before returning the information. If the user is banned, it returns 403.
  - The endpoint is protected and only accessible to authenticated users with the `user` role.

### DELETE /v1/users/{userId}/downloads/{sessionId}/{audioId}  ✅ *New*
**Description:** Deletes the download record of a specific audio for the user, allowing them to download it again.

- **Auth:** Bearer server JWT (role: user)
- **Path Params:**
  - `userId`: User ID
  - `sessionId`: Session ID
  - `audioId`: Audio ID

- **Response:**
```json
{
  "message": "Download deleted",
  "sessionId": "session123",
  "audioId": "audio456"
}
```

- **Errors:**
  - 400: Missing userId, sessionId, or audioId
  - 401: Not authenticated
  - 403: User banned or lacks permissions
  - 404: Download not found
  - 500: Internal error deleting download

- **Curl Example:**
```sh
curl -X DELETE "http://localhost:8080/v1/users/abc123/downloads/session123/audio456" \
  -H "Authorization: Bearer <TU_TOKEN_JWT>"
```

- **Notes:**
  - The endpoint deletes the corresponding document in the `downloads` subcollection under the user in Firestore.
  - If the download does not exist, it returns 404.
  - After deletion, the user can download the audio again using the download endpoint.
**Description:** Lists all sessions and audios downloaded by the user. Returns information about each download, including session, audio, type, and date.

- **Auth:** Bearer server JWT (rol: user)
- **Path Params:**
  - `userId`: ID del usuario

- **Response:**
```json
{
  "downloads": [
    {
      "sessionId": "session123",
      "audioId": "audio456",
      "title": "Relajación profunda",
      "downloadedAt": "2025-09-01T12:34:56Z",
      "type": "audio"
    },
    {
      "sessionId": "session789",
      "title": "Imagen portada",
      "downloadedAt": "2025-08-30T10:20:00Z",
      "type": "image"
    }
  ]
}
```

- **Errors:**
  - 400: Falta userId
  - 401: No autenticado
  - 403: Usuario baneado o sin permisos
  - 500: Error interno al consultar descargas

- **Curl Example:**
```sh
curl -X GET "http://localhost:8080/v1/users/abc123/downloads" \
  -H "Authorization: Bearer <TU_TOKEN_JWT>"
```

- **Extended details:**
  - The endpoint queries the `downloads` subcollection in Firestore under the user's document (`users/{userId}/downloads`).
  - Each document represents a download made by the user, either audio or image.
  - Fields include session identifier (`sessionId`), audio identifier (`audioId`, optional), title, download date (`downloadedAt`, ISO8601 format), and type (`audio` or `image`).
  - The backend validates the JWT and user flags before returning the information. If the user is banned, it returns 403.
  - The endpoint is protected and only accessible to authenticated users with the `user` role.
---


## Library & Search
### GET /v1/library/search
**Description:** Performs full-text and filtered search over available sessions in the library. Allows searching by title, category, tags, level, release date, and Pro status. Returns paginated results.
**Auth:** Bearer server JWT (required, role: user or admin)
**Query Params:**
  - `q`: string. Text for searching session titles (full-text, case-insensitive).
  - `category`: string. Filter by category (e.g., "relax", "focus").
  - `tags`: string. Comma-separated list of tags (e.g., "sleep,calm").
  - `level`: string. Difficulty level (e.g., "beginner", "intermediate", "advanced").
  - `releaseDate`: string. Release date (YYYY-MM-DD).
  - `isPro`: boolean. Filter by Pro sessions (true/false).
  - `page`: integer. Results page (default: 1).
  - `pageSize`: integer. Number of results per page (default: 20).

**Request Example:**
```sh
curl -X GET "http://localhost:8080/v1/library/search?q=relajación&category=relax&tags=sueño,calma&level=beginner&page=1&pageSize=10" \
  -H "Authorization: Bearer <TU_TOKEN_JWT>"
```

**Response Example:**
```json
{
  "sessions": [
    {
      "id": "session123",
      "title": "Relajación profunda",
      "description": "Sesión guiada para relajación profunda.",
      "category": "relax",
      "tags": ["relax", "sleep"],
      "level": "beginner",
      "releaseDate": "2025-08-17",
      "isPro": true,
      "duration": 600,
      "imageUrl": "https://r2.cdn/sessions/session123.jpg"
    },
    // ...otros resultados...
  ]
}
```
**Errors:**
  - 400: Invalid parameters
  - 401: Not authenticated
  - 403: User banned or lacks permissions
  - 500: Internal error searching sessions

**Extended details:**
  - The endpoint performs search and filtering over the `sessions` collection in Firestore.
  - Supports combining text search and advanced filters.
  - Results are returned paginated according to the `page` and `pageSize` parameters.
  - The backend validates the JWT on each request and only allows access to authenticated users (role: user or admin).
  - If the user is banned, returns 403.
  - Useful for building the app's main search and displaying personalized results.
  - **Note:** This endpoint is not available for unauthenticated users; the app requires login to access search.

### POST /v1/library/categories
**Description:** Creates a new category in the library.
- **Auth:** Bearer JWT (authenticated user)
- **Body:**
```json
{
  "name": "relax"
}
```
- **Response:**
```json
{
  "message": "Categoría creada",
  "name": "relax"
}
```
- **Errors:**
  - 400: Category name required
  - 401: Authentication token required
  - 409: Category already exists
  - 500: Connection or creation error

**Request Example:**
```sh
curl -X POST "http://localhost:8080/v1/library/categories" \
  -H "Authorization: Bearer <TU_TOKEN_JWT>" \
  -H "Content-Type: application/json" \
  -d '{ "name": "relax" }'
  ```


### GET /v1/library/categories
**Description:** Lists all categories and the number of sessions in each.
- **Auth:** Bearer JWT (authenticated user)
- **Response:**
```json
{
  "categories": [
    { "name": "relax", "sessionCount": 5 },
    { "name": "focus", "sessionCount": 3 }
  ]
}
```
- **Errors:**
  - 401: Authentication token required
  - 500: Connection or query error

**Request Example:**
```sh
curl -X GET "http://localhost:8080/v1/library/categories" \
  -H "Authorization: Bearer <TU_TOKEN_JWT>"
```



### POST /v1/users/{userId}/onboarding  
**Description:** Register the user's initial answers when installing the app for the first time and completing the onboarding flow.
**Auth:** Bearer server JWT
**Body:**
```json
{
  "goals": ["sleep", "focus"],        // Main goals (improve sleep, focus, etc.)
  "experienceLevel": "beginner"       // Experience level with meditation/hypnosis (beginner, intermediate, advanced)
}
```  
**Response:**            
```json
{
    "message": "Onboarding saved successfully"
}
```            
- **Errors:**
  - 400: Invalid body or missing fields
  - 401: Not authenticated
  - 403: User banned or lacks permissions
  - 500: Internal error registering onboarding

**Request Example:**
```sh
curl -X GET "http://localhost:8080/v1/library/categories" \
  -H "Authorization: Bearer <TU_TOKEN_JWT>"
```																				 

# Favorites & Playlists (User Library)

### POST /v1/users/{userId}/favorites/{sessionId}
**Description:** Adds a session to the user's favorites.
- **Auth:** Bearer server JWT
- **Path Params:**
  - userId: User ID
  - sessionId: Session ID

**Response:**
```json
{
  "message": "Session added to favorites",
  "sessionId": "session123"
}
```
- **Errors:**
  - 400: Missing userId or sessionId
  - 401: Not authenticated
  - 403: User banned or lacks permissions
  - 500: Internal error adding favorite

**Request Example:**
```sh
curl -X POST "http://localhost:8080/v1/users/abc123/favorites/session123" ^
  -H "Authorization: Bearer <SERVER_JWT>"
```		

### DELETE /v1/users/{userId}/favorites/{sessionId}  
**Description:** Removes a session from the user's favorites list.
- **Auth:** Bearer server JWT
- **Path Params:**
  - userId: User ID
  - sessionId: Session ID

**Response:**			
```json
{
  "message": "Session removed from favorites",
  "sessionId": "session123"
}
```			
- **Errors:**
  - 400: Missing userId or sessionId
  - 401: Not authenticated
  - 403: User banned or lacks permissions
  - 404: Favorite not found
  - 500: Internal error deleting favorite

**Request Example:**
```sh
curl -X DELETE "http://localhost:8080/v1/users/abc123/favorites/session123" \
  -H "Authorization: Bearer <SERVER_JWT>"
```
```sh
curl -X DELETE "http://localhost:8080/v1/users/abc123/favorites/session123" ^
  -H "Authorization: Bearer <SERVER_JWT>"
```		


### GET /v1/users/{userId}/favorites 
**Description:** Lists the user's favorite sessions.
- **Auth:** Bearer server JWT (role: user)
- **Path Params:**
  - userId: User ID

**Response:**			
```json
{
  "favorites": [
    {
      "sessionId": "session123",
      "title": "Deep Relaxation",
      "category": "relax",
      "imageUrl": "https://r2.cdn/sessions/session123.jpg",
      "isPro": true,
      "addedAt": "2025-09-06T12:34:56Z"
    }
    // ...other favorites...
  ]
}
```			
- **Errors:**
  - 400: Missing userId
  - 401: Not authenticated
  - 403: User banned or lacks permissions
  - 500: Internal error retrieving favorites

**Request Example:**
```sh
curl --location 'http://localhost:8080/v1/users/edJpSZ76WPNHJ1rPoafv6rgYMKy2/favorites'\
--header 'Authorization: Bearer <SERVER_JWT>'
```		

---

## Subscriptions & Transactions

### POST /v1/receipt/verify

**Description:** Verifies a purchase receipt from Google Play or App Store. The backend validates the receipt with the official API of each platform, updates the user's plan, and records the transaction.
- **Auth:** Bearer server JWT (role: user)
- **Body:**
 ```json
{
  "userId": "abc123",
  "receipt": {
    "source": "GooglePlay", // or "AppStore"
    "data": "<base64_receipt>"
  }
}
```

**Response:**			
```json
{
  "message": "Receipt verified and plan updated",
  "plan": "premium"
}
```

**Description:** Verifies a purchase receipt from Google Play or App Store. The backend validates the receipt with the official API of each platform, updates the user's plan, and records the transaction.

- **Transaction control:**
Only one active transaction per user is allowed.
  - If the user already has an active transaction that has not expired, the expiration is updated.
  - If the active transaction has expired, it is marked as expired and a new one is recorded.

- **Fields updated in the user:**
  - plan, premium, planExpiration, planStatus, subscriptionSource, trial, trialDaysLeft, role

- **Transaction record:**
  - Each transaction stores: source, receiptData, plan, purchaseDate, expirationDate, planStatus, createdAt

- **Auth:** Bearer server JWT (role: user)
- **Body:**
  - For Google Play:
    - `data` must be a base64-encoded JSON with the fields `packageName`, `productId`, `purchaseToken`.
  - For App Store:
    - `data` is the base64 receipt provided by Apple.

```json
{
  "userId": "abc123",
  "receipt": {
    "source": "GooglePlay",
    "data": "<base64 de '{\"packageName\":\"com.app\",\"productId\":\"premium_month\",\"purchaseToken\":\"token123\"}'>"
  }
```

```json
{
  "userId": "abc123",
  "receipt": {
    "source": "AppStore",
    "data": "<recibo_base64_de_apple>"
  }
}
```

**Response:**
```json
{
  "message": "Receipt verified and plan updated",
  "plan": "premium"
}
```
- **Errors:**
  - 400: Invalid body or missing fields
  - 401: Invalid or unverified receipt
  - 403: User banned or lacks permissions
  - 500: Error updating transaction

**Curl Example (Google Play):**
```sh
curl -X POST "http://localhost:8080/v1/receipt/verify" \
  -H "Authorization: Bearer <SERVER_JWT>" \
  -H "Content-Type: application/json" \
  -d '{
    "userId": "abc123",
    "receipt": {
      "source": "GooglePlay",
      "data": "eyJwYWNrYWdlTmFtZSI6ImNvbS5hcHAiLCJwcm9kdWN0SWQiOiJwcmVtaXVtX21vbnRoIiwicHVyY2hhc2VUb2tlbiI6InRva2VuMTIzIn0="
    }
  }'
```

**Example curl (App Store):**
```sh
curl -X POST "http://localhost:8080/v1/receipt/verify" \
  -H "Authorization: Bearer <SERVER_JWT>" \
  -H "Content-Type: application/json" \
  -d '{
    "userId": "abc123",
    "receipt": {
      "source": "AppStore",
      "data": "<recibo_base64_de_apple>"
    }
  }'
```

### POST /v1/events/batch
- **Description:** This endpoint allows sending a batch of user events (e.g., play, pause, etc.) to be efficiently processed and stored in the backend. It is ideal for apps that generate many events in a short period (high frequency).

- **Auth:** Bearer server JWT (role: user)

- **Body:**			
```json
{
  "events": [
    {
      "userId": "edJpSZ76WPNHJ1rPoafv6rgYMKy2",
      "eventType": "play",
      "timestamp": "2025-09-06T12:34:56Z",
      "data": {
        "sessionId": "rZ8id6DFPyYbEFDGPmuf",
        "metadata": { "device": "android", "version": "1.2.3" }
      }
    },
    {
      "userId": "edJpSZ76WPNHJ1rPoafv6rgYMKy2",
      "eventType": "play",
       "timestamp": "2025-09-06T12:34:56Z",
      "data": {
        "sessionId": "rZ8id6DFPyYbEFDGPmuf",
        "audioId": "CqJDS3QHrpWnmR15T9s",
        "metadata": { "device": "android", "version": "1.2.3" }
      }
    },
    {
      "userId": "edJpSZ76WPNHJ1rPoafv6rgYMKy2",
      "eventType": "pause",
       "timestamp": "2025-09-06T12:34:56Z",
      "data": {
        "sessionId": "rZ8id6DFPyYbEFDGPmuf",
        "audioId": "CqJDS3QHrpWnmR15T9s",
        "metadata": { "device": "android", "version": "1.2.3" }
      }
    }
  ]
}
```
- **Response:**			
```json
{
  "message": "Batch events ingested successfully",
  "count": 2
}
```			
- **Errors:**
  - 400: Invalid body or missing fields
  - 401: Not authenticated
  - 403: User banned or lacks permissions
  - 500: Internal error retrieving favorites

**Request Example:**
```sh
curl -X POST "http://localhost:8080/v1/events/batch" \
  -H "Authorization: Bearer <SERVER_JWT>" \
  -H "Content-Type: application/json" \
  -d '{
  "events":[
    {
      "userId": "edJpSZ76WPNHJ1rPoafv6rgYMKy2",
      "eventType": "play",
      "timestamp": "2025-09-06T12:34:56Z",
      "data": {
        "sessionId": "rZ8id6DFPyYbEFDGPmuf",
        "metadata": { "device": "android", "version": "1.2.3" }
      }
    },
    {
      "userId": "edJpSZ76WPNHJ1rPoafv6rgYMKy2",
      "eventType": "play",
       "timestamp": "2025-09-06T12:34:56Z",
        "data":{
          "sessionId": "rZ8id6DFPyYbEFDGPmuf",
          "audioId": "CqJDS3QHrpWnmR15T9s",
          "metadata": { "device": "android", "version": "1.2.3" }
      }
    },
    {
      "userId": "edJpSZ76WPNHJ1rPoafv6rgYMKy2",
      "eventType": "pause",
       "timestamp": "2025-09-06T12:34:56Z",
      "data": {
        "sessionId": "rZ8id6DFPyYbEFDGPmuf",
        "audioId": "CqJDS3QHrpWnmR15T9s",
        "metadata": { "device": "android", "version": "1.2.3" }
      }
    }
  ]
}'

```	

----
## Alerts & Reminders - Advanced Configuration

### POST /v1/alerts
**Description:** Creates or schedules an alert or reminder for the user.
- **Auth:** Bearer server JWT (role: user)
- **Body:**
 ```json
{
    "userId": "edJpSZ76WPNHJ1rPoafv6rgYMKy2",
    "message": "Recuerda tu sesión de meditación a las 21:00",
    "scheduledFor": "2025-09-06T21:00:00Z",
    "type": "reminder"
  }
```	

**Response:**			
```json
{
    "message": "Alert scheduled"
}
```			
- **Errores:**
  - 400: Invalid body or missing fields
  - 500: Internal server error

**Request Example:**
```sh
curl -X POST "http://localhost:8080/v1/alerts" \
  -H "Authorization: Bearer <SERVER_JWT>" \
  -H "Content-Type: application/json" \
  -d '{
    "userId": "abc123",
    "message": "Recuerda tu sesión de meditación a las 21:00",
    "scheduledFor": "2025-09-06T21:00:00Z",
    "type": "reminder"
  }'s
```		

---
### POST /v1/alerts/{alertId}/send
- **Description:** Trigger sending an alert now (admin or scheduled function).
- **Auth:** Bearer server JWT (admin or system)
- **Path Params:**
  - alertId: ID of the alert to send

- **Query Params (optional):**
  - userId: User ID (if the alert is in the user's subcollection)

**Response:**
```json
{ "message": "Alert sent", "alertId": "alert123" }
```
- **Errors:**
  - 400: Missing alertId
  - 404: Alert not found
  - 500: Failed to update alert status

**Request Example:**
TODO: This curl is still to be defined
```sh
curl -X POST "http://localhost:8080/v1/alerts/alert123/send" \
  -H "Authorization: Bearer <SERVER_JWT_ADMIN>"
```	

(If the alert is under a specific user)
	
```sh
curl -X POST "http://localhost:8080/v1/alerts/alert123/send?userId=abc123" \
  -H "Authorization: Bearer <SERVER_JWT_ADMIN>"
```	
---

### POST /v1/alerts/reminder
- **Description:** Sets the time, frequency, and active days for the user's meditation reminder. Saves the configuration in Firestore under `users/{userId}/reminders/meditation`.
- **Auth:** Bearer server JWT
- **Body:**
  ```json
  {
    "userId": "abc123",
    "reminderTime": "21:00",
    "frequency": "daily", // "daily", "weekdays", "custom"
    "activeDays": ["M", "T", "W", "T", "F"]
  }
  ```
- **Response:**
  ```json
  { "message": "Reminder configuration saved" }
  ```
- **Errors:**
  - 400: Invalid body or missing required fields
  - 500: Error saving reminder in Firestore
- **Details:**
  - `reminderTime`: Time in HH:mm format (example: "21:00").
  - `frequency`: Can be "daily" (every day), "weekdays" (Monday to Friday), or "custom" (custom days).
  - `activeDays`: Array of active days. Use abbreviations: "M" (Monday), "T" (Tuesday), "W" (Wednesday), "T" (Thursday), "F" (Friday), "S" (Saturday), "U" (Sunday).
  - The configuration is stored in Firestore and can be queried by the app to display or modify the reminder.
- **Curl Example:**
  ```bash
  curl -X POST "http://localhost:8080/v1/alerts/reminder" \
    -H "Authorization: Bearer <JWT>" \
    -H "Content-Type: application/json" \
    -d '{
      "userId": "abc123",
      "reminderTime": "21:00",
      "frequency": "daily",
      "activeDays": ["M", "T", "W", "T", "F"]
    }'
  ```
---
### PATCH /v1/alerts/preferences
- **Description:** Configures the toggles for each type of notification.
- **Auth:** Bearer server JWT
- **Body:**
```json
{
  "userId": "abc123",
  "motivationalMessages": true,
  "achievementNotifications": true,
  "sessionReminders": false
}
```
- **Response:**
```json
{ "message": "Notification preferences updated" }
```
- **Curl Example:**
  ```bash
  curl -X PATCH https://api.tuapp.com/v1/alerts/preferences \
  -H "Authorization: Bearer <SERVER_JWT>" \
  -H "Content-Type: application/json" \
  -d '{
    "userId": "abc123",
    "motivationalMessages": true,
    "achievementNotifications": true,
    "sessionReminders": false
  }'
  ```

---
### PATCH /v1/alerts/message-style
- **Description:** Sets the preferred style for personalized messages for the authenticated user.
- **Auth:** Bearer server JWT
- **Body:**
```json
{
    "userId": "edJpSZ76WPNHJ1rPoafv6rgYMKy2",
    "messageStyle": "motivational"
  }
```
- **Response:**
```json
{ "message": "Message style updated" }
```
- **Curl Example:**
  ```bash
  curl -X PATCH https://api.tuapp.com/v1/alerts/message-style \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
       "userId": "edJpSZ76WPNHJ1rPoafv6rgYMKy2",
       "messageStyle": "motivational"
  }'
  ```
---
### GET /v1/alerts/preview
- **Description:** Retrieves a preview of the personalized message for the user.
- **Auth:** Bearer server JWT
- **Headers:**
  - Authorization: Bearer <token>
  - Accept-Language: en (optional, values: en, es, etc.)
- **Query params:** 
  - `userId`
- **Response:**
```json
{
    "preview": "You are unstoppable! Your session awaits you at 9:00 PM."
}
```
- **Errors:**
  - 400: Missing userId query param
  - 401: Not authenticated
  - 500: Internal error retrieving favorites


- **Curl Example:**
  ```bash
  curl -X GET https://api.tuapp.com/v1/alerts/preview?userId=abc123" \
  -H "Authorization: Bearer <token>" \
  -H "Accept-Language: es"
  ```
---
### GET /v1/alerts/upcoming
- **Description:** Returns the list of scheduled notifications for the upcoming days.
- **Auth:** Bearer server JWT
- **Query params:** 
  - `userId`
- **Response:**
```json
{
  "upcoming": [
    {
      "type": "session",
      "title": "Focus on breathing",
      "scheduledFor": "2025-09-06T21:00:00Z",
      "tags": ["Focus", "Breathing"]
    },
    {
      "type": "achievement",
      "title": "Weekly Goal",
      "scheduledFor": "2025-09-08T21:00:00Z",
      "tags": ["Weekly Goal"]
    }
  ]
}
```

- **Errors:**
  - 400: Missing userId
  - 401: Not authenticated
  - 500: Internal error retrieving favorites

**Request Example:**
```sh
curl -X GET "https://api.tuapp.com/v1/alerts/upcoming?userId=abc123" \
  -H "Authorization: Bearer <server JWT>"
```		

---
### PATCH /v1/alerts/advanced-options
- **Description:** Configures snooze options and Do Not Disturb (DND) respect.
- **Auth:** Bearer server JWT
- **Body:**
```json
{
  "userId": "abc123",
  "smartSnooze": true,
  "doNotDisturbRespect": true
}
```
- **Response:**
```json
{ "message": "Advanced options updated" }
```
- **Errors:**
  - 400: Missing userId or invalid body
  - 401: Not authenticated
  - 500: Internal error updating options

- **Request Example:**
```sh
curl --location --request PATCH 'http://localhost:8080/v1/alerts/advanced-options' \
--header 'Authorization: Bearer <SERVER_JWT>' \
--header 'Content-Type: application/json' \
--data '{
  "userId": "abc123",
  "smartSnooze": true,
  "doNotDisturbRespect": true
}'
```	

---

### GET /v1/alerts/config
- **Description:** Returns all alert and reminder configuration for the user.
- **Auth:** Bearer server JWT
- **Query params:**
  - `userId`
- **Response:**
```json
{
  "reminderTime": "21:00",
  "frequency": "daily",
  "activeDays": ["M", "T", "W", "T", "F"],
  "motivationalMessages": true,
  "achievementNotifications": true,
  "sessionReminders": false,
  "messageStyle": "gentle",
  "smartSnooze": true,
  "doNotDisturbRespect": true
}
```
- **Errors:**
  - 400: Missing userId
  - 401: Not authenticated
  - 500: Internal error retrieving configuration

**Request Example:**

```sh
curl --location 'http://localhost:8080/v1/alerts/config?userId=abc123' \
--header 'Authorization: Bearer <SERVER_JWT>'
```		

---

### PATCH /v1/alerts/config
- **Description:** Updates the user's alert and reminder configuration.
- **Auth:** Bearer server JWT (role: user)
- **Body Params:** (you can send only the fields you want to update)
```json
{
    "userId": "edJpSZ76WPNHJ1rPoafv6rgYMKy2",
    "reminderTime": "21:00",
    "frequency": "daily",
    "activeDays": ["M", "T", "W", "T", "F"],
    "motivationalMessages": true,
    "achievementNotifications": true,
    "sessionReminders": false,
    "messageStyle": "gentle",
    "smartSnooze": true,
    "doNotDisturbRespect": true
  }
```
- **Response:**

```json
{
  "message": "Alert configuration updated"
}
```

- **Errors:**
  - 400: Missing userId or invalid body
  - 401: Not authenticated
  - 500: Internal error updating configuration

- **Request Example:**

```sh
curl --location --request PATCH 'http://localhost:8080/v1/alerts/config' \
--header 'Authorization: Bearer <SERVER_JWT>' \
--header 'Content-Type: application/json' \
--data ''{
  "userId": "abc123",
  "reminderTime": "21:00",
  "frequency": "daily",
  "activeDays": ["M", "T", "W", "T", "F"],
  "motivationalMessages": true,
  "achievementNotifications": true,
  "sessionReminders": false,
  "messageStyle": "gentle",
  "smartSnooze": true,
  "doNotDisturbRespect": true
}'

```