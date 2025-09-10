# HipnoApp Backend API - Technical README

**Base URL (example):** `https://api.hipnoapp.example.com/v1`

Authentication: All protected endpoints require a valid Firebase ID Token passed in `Authorization: Bearer <ID_TOKEN>` header. Backend verifies token with Firebase Admin SDK. Some endpoints also require a server-issued session JWT for additional session validation.

**Flags dinámicos de usuario:**
En cada request autenticada, el backend consulta los flags dinámicos del usuario en Firestore (`/users/{uid}`), como `premium`, `trial`, `banned`, etc. Estos flags están disponibles en los handlers vía `c.Locals("flags")`.

**Ejemplo de uso en Go:**
```go
flags := c.Locals("flags").(map[string]interface{})
if flags["premium"] == true {
  // Permitir acceso a contenido premium
}
if flags["banned"] == true {
  return c.Status(403).JSON(fiber.Map{"error": "Usuario baneado"})
}
```
/v1/sessions/{sessionId}/audios
---
## Authentication & Session
### POST /v1/auth/validate
- **Description:** Validate Firebase ID token, create or update server session, return server JWT and user profile.
- **Auth:** None (accepts Firebase ID token)
- **Body:**
```json
{ "idToken": "<firebase_id_token>" }
```
- **Response:**
```json
{ "jwt": "<server_jwt>", "user": { ... } }
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
- **Description:** Invalidate server session JWT and optionally revoke Firebase refresh token.
- **Auth:** Bearer server JWT

---

## Users & Profile
> **Nota de roles:** Los endpoints bajo esta sección solo pueden ser accedidos por usuarios con rol `user` (Bearer server JWT con `role: "user"`). El rol `admin` no tiene acceso a estos endpoints, excepto donde se indique explícitamente (por ejemplo, consulta de perfiles).



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
    ...otros campos...
  }
}
```

### PATCH /v1/users/{userId}
- **Description:** Actualiza el perfil completo del usuario. Permite modificar campos como `displayName`, `email`, `gender`, `birthDate`, `age`, `language`, `country`, además de información de suscripción (`plan`, `planStatus`, `planExpiration`, `premium`, `trial`, `trialEndsAt`, `trialDaysLeft`, `subscriptionSource`), flags (`banned`), y fechas de creación/actualización.
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
  - 400: Body inválido o campos faltantes
  - 401: No autenticado
  - 403: Usuario baneado o sin permisos
  - 500: Error interno al actualizar usuario

### POST /v1/users/{userId}/device/register
- **Description:** Register a device for push notifications and analytics. Stores deviceId, platform, osVersion, fcmToken.
- **Auth:** Bearer server JWT
- **Body:** `{ "deviceId": "...", "platform": "iOS|Android|Web", "osVersion": "...", "fcmToken": "..." }`

### POST /v1/users/{userId}/device/unregister
- **Description:** Desregistrar un dispositivo del usuario (elimina el deviceId de la colección de dispositivos del usuario en Firestore)
				   														
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
  - 400: Falta el userId o deviceId
  - 500: Error al desregistrar el dispositivo
- **Curl Example:**
```sh
curl -X POST "http://localhost:8080/v1/users/{userId}/device/unregister" \
  -H "Authorization: Bearer <SERVER_JWT>" \
  -H "Content-Type: application/json" \
  -d '{ "deviceId": "device-123" }'
```												 
### POST /v1/users/:userId/profile-image
- **Description:** Genera una URL firmada para que el frontend suba la imagen de perfil del usuario directamente a Cloudflare R2.
				   														
- **Auth:** Bearer server JWT
- **Parámetros:**
  - userId (path): ID del usuario.
  - contentType (query, opcional): image/jpeg (por defecto) o image/png.

- **Response:**
```json
{
  "uploadUrl": "https://<bucket>.r2.cloudflarestorage.com/users/USERID/profile.jpg?...",
  "objectKey": "users/USERID/profile.jpg"
}
```												  
- **Errors:**
  - 400: Falta el userId o deviceId
  - 500: Error al desregistrar el dispositivo
- **Curl Example:**
```sh
curl -X POST "http://localhost:8080/v1/users/ZooCaNaIkQQR4FVG5Ywyb81F9ww1/profile-image?contentType=image/jpeg" \
  -H "Authorization: Bearer <JWT>" \
  -H "Content-Type: application/json"
```													 
---

### PUT <uploadUrl>
- **Description:** Sube la imagen al URL firmado en cloudflare S2 recibido en el paso anterior..

- **Headers:**
  - Content-Type: image/jpeg o image/png

- **Response:**
  - 200 OK
											  
- **Errors:**
  - 500: Error al desregistrar el dispositivo

- **Curl Example:**
```sh
curl -X PUT "<uploadUrl>" \
  -H "Content-Type: image/jpeg" \
  --data-binary "@ruta/a/tu/imagen.jpg"
```		

---
### PATCH /v1/users/:userId/profile-image-url
- **Description:** Guarda la URL pública de la imagen de perfil en el documento del usuario en Firestore..
				   														
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
  - 400: Falta el userId o deviceId
  - 500: Error al desregistrar el dispositivo
- **Curl Example:**
```sh
curl -X PATCH "http://localhost:8080/v1/users/ZooCaNaIkQQR4FVG5Ywyb81F9ww1/profile-image-url" \
  -H "Authorization: Bearer <JWT>" \
  -H "Content-Type: application/json" \
  -d '{"imageUrl": "https://<bucket>.r2.cloudflarestorage.com/users/ZooCaNaIkQQR4FVG5Ywyb81F9ww1/profile.jpg"}'
```	
---

### GET /v1/users/:userId/profile-image/download
- **Description:** Genera una URL firmada temporal para descargar la imagen de perfil del usuario desde Cloudflare R2. La URL expira después de unos minutos por seguridad.
				   														
- **Auth:** Bearer server JWT

- **Parámetros:**
  - userId (path): ID del usuario
						  
- **Response:**
```json
{
  "signedUrl": "https://r2.example.com/bucket/user123/profile.jpg?...",
  "expiresAt": "2025-09-07T12:00:00Z"
}
```											  
- **Errors:**
 - 400: Faltan parámetros o URL inválida
 - 404: Usuario o imagen no encontrada
 - 500: Error interno (Firestore, R2, etc)

- **Curl Example:**
```sh
curl -X GET "https://api.hipnoapp.com/v1/users/USER_ID/profile-image/download" -H "Authorization: Bearer <token>"
```	
---

### POST /v1/auth/forgot-password
- **Description:** Genera un código de verificación de 6 dígitos, lo almacena en Firestore y lo envía al correo registrado. El código expira tras 5 minutos (configurable).
				   														

- **Parámetros:**
```json
{ "email": "usuario@email.com" }
```

- **Response:**
```json
{ "message": "Código enviado", "expiresAt": "2025-09-07T12:05:00Z" }
```											  
- **Errors:**
 - 400: Email requerido
 - 404: Usuario no encontrado
 - 500: Error interno

- **Curl Example:**
```sh
curl -X POST "http://localhost:8080/v1/auth/forgot-password" \
  -H "Content-Type: application/json" \
  -d '{ "email": "usuario@email.com" }'
```	
---

### POST /v1/auth/resend-code
- **Description:**  Reenvía el código de verificación al correo si no ha expirado.
				   														

- **Parámetros:**
```json
{ "email": "usuario@email.com" }
```

- **Response:**
```json
{ "message": "Código enviado", "expiresAt": "2025-09-07T12:05:00Z" }
```											  
- **Errors:**
 - 400: Email requerido
 - 404: Usuario no encontrado
 - 500: Error interno

- **Curl Example:**
```sh
curl -X POST "http://localhost:8080/v1/auth/resend-code" \
  -H "Content-Type: application/json" \
  -d '{ "email": "usuario@email.com" }'
```	
---


## Admin / CMS Endpoints
> **Nota de roles:** Los endpoints bajo esta sección solo pueden ser accedidos por usuarios con rol `admin` (Bearer server JWT con `role: "admin"`). Los usuarios normales (rol `user`) no tienen acceso a estos endpoints.

### POST /v1/admin/assign-role
- **Description:** Asigna el rol `admin` o `user` a un usuario en Firebase Auth (solo admin puede usar este endpoint).
- **Auth:** Bearer server JWT (admin)
- **Body:**
```json
{
  "uid": "UID_DEL_USUARIO",
  "role": "admin" // o "user"
}
```
- **Response:**
```json
{ "message": "Rol asignado", "uid": "UID_DEL_USUARIO", "role": "admin" }
```
- **Errors:**
  - 400: Faltan uid o role, o rol inválido
  - 500: Error al asignar el rol
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
- **Descripción:** Crear nueva sesión de audio (admin). La sesión solo contiene la imagen general y los metadatos. Los audios se gestionan en una subcolección aparte.
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
- **Descripción:** Agrega un audio a la sesión indicada. Cada audio tiene su propio archivo, imagen, título, duración y orden.
- **Auth:** Bearer server JWT (admin)
- **Path Params:**

  - `sessionId`: ID de la sesión a la que se agrega el audio.
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

> **Nota importante:**
> El campo `audioUrl` es solo una referencia lógica al objeto en R2. Para descargar o reproducir el audio, el cliente debe solicitar la URL firmada mediante el endpoint `/v1/sessions/{sessionId}/download`, enviando tanto el `sessionId` como el `audioId` correspondiente. El backend valida el acceso y genera una URL temporal para ese audio específico.

### DELETE /v1/admin/sessions/{sessionId}
- **Description:** Delete session metadata (soft-delete recommended).
- **Auth:** Bearer server JWT (admin)
- **Response:**
```json
{ "id": "sessionId", "message": "Sesión eliminada" }
```
- **Curl Example:**
```sh
curl -X DELETE "http://localhost:8080/v1/admin/sessions/{sessionId}" \
  -H "Authorization: Bearer <ADMIN_JWT>"
```

### POST /v1/admin/upload
**Description:** Solicita una URL de subida firmada para Cloudflare R2. Permite subir archivos de audio (`.mp3`) o imagen (`.jpg`) para una sesión o audio específico. La ruta se genera automáticamente según el tipo.
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

### PUT <uploadUrl> (Subida de archivo real a Cloudflare R2)
- **Descripción:** Sube el archivo de audio o imagen directamente a Cloudflare R2 usando la URL pre-firmada (`uploadUrl`) obtenida previamente del backend.
- **Auth:** No requiere autenticación (la URL ya incluye permisos temporales).
- **Headers:**
  - `Content-Type: audio/mpeg` (para audio)
  - `Content-Type: image/jpeg` (para imagen)
- **Body:** Binario del archivo.
- **Notas:**
  - La URL `uploadUrl` es válida solo por tiempo limitado.
  - El backend no recibe el archivo directamente; solo gestiona la generación de la URL.
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
- **Descripción:** Obtiene la metadata de una sesión específica, incluyendo título, descripción, duración, referencia de audio, imagen, si es Pro y etiquetas.
- **Auth:** Opcional (si se envía JWT, la respuesta puede incluir información sobre disponibilidad según el plan del usuario).
- **Path Params:**
  - `sessionId`: ID de la sesión a consultar.
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
  - 404: Sesión no encontrada
  - 403: Acceso denegado (si la sesión es Pro y el usuario no tiene acceso)
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

### 3. Consultar audios activos en una sesión
#### GET /v1/sessions/{sessionId}/audios
<!--
  Devuelve los audios de una sesión y determina cuáles están activos para el usuario según su fecha de inscripción y el intervalo de activación configurado.

  ### Lógica de activación de audios:
  1. Se obtiene la fecha de inscripción del usuario a la sesión desde `users/{userId}/sessions/{sessionId}`.
  2. El intervalo de activación de audios se consulta desde la configuración global (`config/audioActivationInterval` en Firestore, valor por defecto: 3 días).
  3. Los audios de la sesión se listan ordenados por el campo `order`.
  4. Para cada audio, se calcula si está activo:
    - El audio en la posición N (`order=N`) se activa cuando han transcurrido al menos N * intervalo días desde la fecha de inscripción.
    - Ejemplo: Si la inscripción fue el 1 de agosto y el intervalo es 3 días:
      - Audio 1 (`order=1`): activo desde el 1 de agosto.
      - Audio 2 (`order=2`): activo desde el 4 de agosto.
      - Audio 3 (`order=3`): activo desde el 7 de agosto.
  5. El campo `isActive` se determina para cada audio y se incluye en la respuesta.

  Esta lógica permite que los audios se desbloqueen progresivamente conforme el usuario avanza en la sesión, promoviendo un acceso escalonado al contenido.

  ### Autenticación:
  - Requiere autenticación Bearer con JWT del servidor.
-->
- **Descripción:** Devuelve los audios de la sesión y cuáles están activos para el usuario según la fecha de inscripción y el intervalo configurado.
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
  - 404: Sesión no encontrada
  - 403: Acceso denegado (si el usuario no tiene acceso a los audios activos)
- **Curl Example:**
  ```sh
  curl -X GET "http://localhost:8080/v1/sessions/{sessionId}/audios?userId={userId}" \
    -H "Authorization: Bearer <TU_TOKEN_JWT>"
  ```

---


### 4. Finalizar sesión
#### POST /v1/sessions/{sessionId}/complete
- **Descripción:** Marca la sesión como completada para el usuario. Actualiza el estado de la sesión inscrita en Firestore (`estado: completada`), registra la fecha de finalización y permite inscribirse a una nueva sesión si tiene menos de 3 activas.
- **Lógica:**
  1. Verifica que el usuario esté inscrito en la sesión.
  2. Cambia el estado de la sesión a `completada` y registra la fecha de finalización.
  3. Permite inscribirse a una nueva sesión si el usuario tiene menos de 3 activas.
  4. Actualiza el progreso y estadísticas del usuario.
- **Auth:** Bearer server JWT
- **Body:**
  ```json
  { "userId": "abc123" }
  ```
- **Response:**
  ```json
  { "message": "Sesión completada", "sessionId": "session123" }
  ```
- **Errores:**
  - 404: Sesión no encontrada o usuario no inscrito
  - 403: Acceso denegado
  - 500: Error al actualizar la sesión
- **Curl Example:**
  ```sh
  curl -X POST "http://localhost:8080/v1/sessions/session123/complete" \
    -H "Authorization: Bearer <TU_TOKEN_JWT>" \
    -H "Content-Type: application/json" \
    -d '{ "userId": "abc123" }'
  ```
---

#### POST /v1/admin/config/audio-activation-interval
- **Descripción:** Actualiza la configuración global para la activación progresiva de audios y el número máximo de sesiones activas permitidas para inscripción.
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
    "message": "Configuración actualizada",
    "intervalDays": 5,
    "maxActiveSessions": 3
  }
  ```
- **Errores:**
  - 400: Body inválido o campos faltantes
  - 401: No autorizado (JWT inválido o sin rol admin)
  - 500: Error interno o credenciales faltantes
- **Curl Example:**
  ```sh
  curl -X POST "http://localhost:8080/v1/admin/config/audio-activation-interval" \
    -H "Authorization: Bearer <SERVER_JWT_ADMIN>" \
    -H "Content-Type: application/json" \
    -d '{ "intervalDays": 5, "maxActiveSessions": 3 }'
  ```
  ### POST /v1/sessions/{sessionId}/start

  - **Descripción:** Marca el inicio de una sesión para el usuario. Crea una entrada preliminar en el historial de sesiones (`sessions_history`) que registra cuándo el usuario comenzó la sesión, desde qué dispositivo y ubicación. Este registro se utiliza para calcular el tiempo total de escucha, estadísticas y logros.
  - **Autenticación:** Requiere JWT del servidor (Bearer).
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
  - **Respuesta:**
    ```json
    {
      "message": "Inicio de sesión registrado",
      "sessionId": "session123",
      "startedAt": "2025-08-10T12:34:56Z"
    }
    ```
  - **Notas:**
    - El backend registra la hora de inicio y asocia el dispositivo y ubicación si se proveen.
    - La entrada en `sessions_history` se actualizará posteriormente con el progreso y la finalización.
    - Puede usarse para estadísticas, logros y control de uso.
  - **Errores:**
    - 400: Datos faltantes o inválidos
    - 401: No autenticado
    - 404: Sesión no encontrada
    - 500: Error interno
  - **Ejemplo Curl:**
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

-- **Descripción:** Este endpoint recibe actualizaciones periódicas ("heartbeat") del cliente mientras el usuario reproduce una sesión de audio. Soporta sesiones con múltiples audios: cada heartbeat puede incluir el campo `audioId` para identificar el audio específico que está siendo reproducido. El backend registra el progreso en tiempo real, detecta interrupciones, problemas de buffering y el tipo de conexión de red. Los datos recopilados se usan para estadísticas de uso, reanudación de sesiones, monitoreo de calidad y análisis de experiencia del usuario. Si se envía `audioId`, el heartbeat se almacena bajo la subcolección del audio correspondiente; si no, se guarda en la colección general de heartbeats de la sesión.

- **Autenticación:** Requiere JWT del servidor (Bearer).

- **Body:**
  ```json
  {
    "userId": "abc123",                // ID del usuario (opcional si está en el JWT)
    "audioId": "audio1",               // (opcional) ID del audio que está siendo reproducido
    "position": 123,                     // Posición actual en segundos dentro del audio
    "duration": 600,                     // Duración total del audio en segundos
    "connectionType": "wifi",           // Tipo de conexión ("wifi", "cellular", "offline")
    "buffering": false,                  // Si el usuario está experimentando buffering
    "deviceInfo": {                      // Información opcional del dispositivo
      "deviceId": "device-123",
      "platform": "iOS",
      "osVersion": "17.5"
    },
    "timestamp": "2025-08-10T12:35:00Z" // Marca de tiempo del heartbeat (ISO8601, opcional)
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

- **Notas:**
  - El backend puede usar estos datos para actualizar el progreso de la sesión, detectar desconexiones, guardar el punto de reanudación y analizar problemas de red.
  - Se recomienda enviar heartbeats cada 10-30 segundos durante la reproducción activa.
  - Si el usuario pierde conexión, los heartbeats pueden almacenarse localmente y enviarse cuando se recupere la conectividad.

- **Errores:**
  - 400: Body inválido o campos faltantes
  - 401: No autenticado
  - 404: Sesión no encontrada
  - 500: Error interno

- **Ejemplo Curl:**
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
- **Descripción:** Solicita al servidor una URL firmada para descargar el audio de la sesión y usarlo offline. El backend valida el plan del usuario y los límites diarios de descargas permitidas.
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
- **Notas:**
  - La URL firmada permite descargar el archivo de audio o imagen específico (`audioId`) directamente desde Cloudflare R2 por tiempo limitado.
  - El backend registra la descarga para controlar límites diarios y uso por dispositivo.
  - Si el usuario supera el límite de descargas, se retorna error 403.
  - Es obligatorio enviar el `audioId` en el body para obtener la URL firmada del recurso correspondiente a la sesión.
  - El parámetro `type` permite especificar si se desea descargar el audio (`type: "audio"`) o la imagen (`type: "image"`).

- **Errores:**
  - 400: Datos faltantes (userId, deviceId, audioId)
  - 403: Límite de descargas alcanzado o plan insuficiente
  - 404: Sesión o audio no encontrado
  - 500: Error interno

- **Ejemplo Curl:**
  ```sh
  # Descargar audio
  curl -X POST "http://localhost:8080/v1/sessions/session123/download" \
    -H "Authorization: Bearer <TU_TOKEN_JWT>" \
    -H "Content-Type: application/json" \
    -d '{
      "userId": "abc123",
      "deviceId": "device-123",
      "audioId": "CqJDS3QHrpWnmR15T9sv",
      "type": "audio"
    }'

  # Descargar imagen asociada a audio
  curl -X POST "http://localhost:8080/v1/sessions/session123/download" \
    -H "Authorization: Bearer <TU_TOKEN_JWT>" \
    -H "Content-Type: application/json" \
    -d '{
      "userId": "abc123",
      "deviceId": "device-123",
      "audioId": "CqJDS3QHrpWnmR15T9sv",
      "type": "image"
    }'

  # Descargar imagen de portada de sesión
  curl -X POST "http://localhost:8080/v1/sessions/session123/download" \
    -H "Authorization: Bearer <TU_TOKEN_JWT>" \
    -H "Content-Type: application/json" \
    -d '{
      "userId": "abc123",
      "deviceId": "device-123",
      "type": "image"
    }'

  # Descargar imagen asociada a audio
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

----Falta desde aqui

### GET /v1/users/{userId}/downloads  ✅ *Nuevo*
**Description:** Lista todas las sesiones y audios descargados por el usuario. Devuelve información sobre cada descarga, incluyendo sesión, audio, tipo y fecha.

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

- **Detalle extendido:**
  - El endpoint consulta la subcolección `downloads` en Firestore bajo el documento del usuario (`users/{userId}/downloads`).
  - Cada documento representa una descarga realizada por el usuario, ya sea de audio o imagen.
  - Los campos incluyen el identificador de sesión (`sessionId`), identificador de audio (`audioId`, opcional), título, fecha de descarga (`downloadedAt`, formato ISO8601) y tipo (`audio` o `image`).
  - El backend valida el JWT y los flags del usuario antes de retornar la información. Si el usuario está baneado, retorna 403.
  - El endpoint está protegido y solo accesible para usuarios autenticados con rol `user`.

---

### DELETE /v1/users/{userId}/downloads/{sessionId}/{audioId}  ✅ *Nuevo*
**Description:** Elimina el registro de descarga de un audio específico para el usuario, permitiendo que pueda volver a descargarlo.

- **Auth:** Bearer server JWT (rol: user)
- **Path Params:**
  - `userId`: ID del usuario
  - `sessionId`: ID de la sesión
  - `audioId`: ID del audio

- **Response:**
```json
{
  "message": "Descarga eliminada",
  "sessionId": "session123",
  "audioId": "audio456"
}
```

- **Errors:**
  - 400: Falta userId, sessionId o audioId
  - 401: No autenticado
  - 403: Usuario baneado o sin permisos
  - 404: Descarga no encontrada
  - 500: Error interno al eliminar la descarga

- **Curl Example:**
```sh
curl -X DELETE "http://localhost:8080/v1/users/abc123/downloads/session123/audio456" \
  -H "Authorization: Bearer <TU_TOKEN_JWT>"
```

- **Notas:**
  - El endpoint elimina el documento correspondiente en la subcolección `downloads` bajo el usuario en Firestore.
  - Si la descarga no existe, retorna 404.
  - Tras eliminar, el usuario puede volver a descargar el audio usando el endpoint de descarga.
**Description:** Lista todas las sesiones y audios descargados por el usuario. Devuelve información sobre cada descarga, incluyendo sesión, audio, tipo y fecha.

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

- **Detalle extendido:**
  - El endpoint consulta la subcolección `downloads` en Firestore bajo el documento del usuario (`users/{userId}/downloads`).
  - Cada documento representa una descarga realizada por el usuario, ya sea de audio o imagen.
  - Los campos incluyen el identificador de sesión (`sessionId`), identificador de audio (`audioId`, opcional), título, fecha de descarga (`downloadedAt`, formato ISO8601) y tipo (`audio` o `image`).
  - El backend valida el JWT y los flags del usuario antes de retornar la información. Si el usuario está baneado, retorna 403.
  - El endpoint está protegido y solo accesible para usuarios autenticados con rol `user`.
---


## Library & Search
### GET /v1/library/search
**Description:** Realiza búsqueda full-text y filtrada sobre las sesiones disponibles en la librería. Permite buscar por título, categoría, etiquetas, nivel, fecha de lanzamiento y si la sesión es Pro. Devuelve resultados paginados.
**Auth:** Bearer server JWT (requerido, rol: user o admin)
**Query Params:**
  - `q`: string. Texto para búsqueda en el título de la sesión (full-text, case-insensitive).
  - `category`: string. Filtra por categoría (ej: "relax", "focus").
  - `tags`: string. Lista de etiquetas separadas por coma (ej: "sueño,calma").
  - `level`: string. Nivel de dificultad (ej: "beginner", "intermediate", "advanced").
  - `releaseDate`: string. Fecha de lanzamiento (YYYY-MM-DD).
  - `isPro`: boolean. Filtra por sesiones Pro (true/false).
  - `page`: integer. Página de resultados (default: 1).
  - `pageSize`: integer. Cantidad de resultados por página (default: 20).

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
  - 400: Parámetros inválidos
  - 401: No autenticado
  - 403: Usuario baneado o sin permisos
  - 500: Error interno al buscar sesiones

**Detalle extendido:**
  - El endpoint realiza búsqueda y filtrado sobre la colección `sessions` en Firestore.
  - Permite combinar búsqueda por texto y filtros avanzados.
  - Los resultados se devuelven paginados según los parámetros `page` y `pageSize`.
  - El backend valida el JWT en cada request y solo permite el acceso a usuarios autenticados (rol: user o admin).
  - Si el usuario está baneado, retorna 403.
  - Útil para construir el buscador principal de la app y mostrar resultados personalizados.
  - **Nota:** Este endpoint no está disponible para usuarios no autenticados; la app requiere login para acceder a la búsqueda.

### POST /v1/library/categories
**Description:**: Crea una nueva categoría en la librería.
- **Auth:** Bearer JWT (usuario autenticado)
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

- **Errores:**
  - 400: Nombre de categoría requerido
  - 401: Token de autenticación requerido
  - 409: La categoría ya existe
  - 500: Error de conexión o creación

**Request Example:**
```sh
curl -X POST "http://localhost:8080/v1/library/categories" \
  -H "Authorization: Bearer <TU_TOKEN_JWT>" \
  -H "Content-Type: application/json" \
  -d '{ "name": "relax" }'
  ```


### GET /v1/library/categories
**Description:**  Lista todas las categorías y el conteo de sesiones por cada una.
- **Auth:** Bearer JWT (usuario autenticado)
- **Response:**
```json
{
  "categories": [
    { "name": "relax", "sessionCount": 5 },
    { "name": "focus", "sessionCount": 3 }
  ]
}
```	
- **Errores:**
  - 401: Token de autenticación requerido
  - 500: Error de conexión o consulta

**Request Example:**
```sh
curl -X GET "http://localhost:8080/v1/library/categories" \
  -H "Authorization: Bearer <TU_TOKEN_JWT>"
```



### POST /v1/users/{userId}/onboarding  
**Description:** Registrar las respuestas iniciales del usuario cuando instala la app por primera vez y completa el flujo de onboarding.
**Auth:** Bearer server JWT
**Body:**
```json
{
  "goals": ["sleep", "focus"],        // Objetivos principales (mejorar sueño, concentración, etc.)
  "experienceLevel": "beginner"       // Nivel de experiencia con meditación/hipnosis (beginner, intermediate, advanced)
}
```	
**Response:**			
```json
{
    "message": "Onboarding guardado correctamente"
}
```			
- **Errores:**
  - 400: Body inválido o campos faltantes
  - 401: No autenticado
  - 403: Usuario baneado o sin permisos
  - 500: Error interno al registrar onboarding

**Request Example:**
```sh
curl -X GET "http://localhost:8080/v1/library/categories" \
  -H "Authorization: Bearer <TU_TOKEN_JWT>"
```																				 

# Favorites & Playlists (User Library)

### POST /v1/users/{userId}/favorites/{sessionId} 
**Description:** Agrega una sesión a favoritos para el usuario.
- **Auth:** Bearer server JWT
- **Path Params:**
  - userId: ID del usuario
  - sessionId: ID de la sesión

**Response:**			
```json
{
  "message": "Sesión agregada a favoritos",
  "sessionId": "session123"
}
```			
- **Errores:**
  - 400: Falta userId o sessionId
  - 401: No autenticado
  - 403: Usuario baneado o sin permisos
  - 500: Error interno al agregar favorito

**Request Example:**
```sh
curl -X POST "http://localhost:8080/v1/users/abc123/favorites/session123" ^
  -H "Authorization: Bearer <SERVER_JWT>"
```		

### DELETE /v1/users/{userId}/favorites/{sessionId}  
**Description:** Elimina una sesión de la lista de favoritos del usuario.
- **Auth:** Bearer server JWT
- **Path Params:**
  - userId: ID del usuario
  - sessionId: ID de la sesión

**Response:**			
```json
{
  "message": "Sesión eliminada de favoritos",
  "sessionId": "session123"
}
```			
- **Errores:**
  - 400: Falta userId o sessionId
  - 401: No autenticado
  - 403: Usuario baneado o sin permisos
  - 404: Favorito no encontrado
  - 500: Error interno al eliminar favorito

**Request Example:**
```sh
curl -X DELETE "http://localhost:8080/v1/users/abc123/favorites/session123" ^
  -H "Authorization: Bearer <SERVER_JWT>"
```		


### GET /v1/users/{userId}/favorites 
**Description:** Lista sesiones favoritas del usuario.
- **Auth:** Bearer server JWT (rol: user)
- **Path Params:**
  - userId: ID del usuario

**Response:**			
```json
{
  "favorites": [
    {
      "sessionId": "session123",
      "title": "Relajación profunda",
      "category": "relax",
      "imageUrl": "https://r2.cdn/sessions/session123.jpg",
      "isPro": true,
      "addedAt": "2025-09-06T12:34:56Z"
    }
    // ...otros favoritos...
  ]
}
```			
- **Errores:**
  - 400: Falta userId
  - 401: No autenticado
  - 403: Usuario baneado o sin permisos
  - 500: Error interno al consultar favoritos

**Request Example:**
```sh
curl --location 'http://localhost:8080/v1/users/edJpSZ76WPNHJ1rPoafv6rgYMKy2/favorites'\
--header 'Authorization: Bearer <SERVER_JWT>'
```		

---

## Subscriptions & Transactions

### POST /v1/receipt/verify

**Description:** Verifica un recibo de compra de Google Play o App Store. El backend valida el recibo con la API correspondiente, actualiza el plan del usuario y registra la transacción.
- **Auth:** Bearer server JWT (rol: user)
- **Body:**
 ```json
{
  "userId": "abc123",
  "receipt": {
    "source": "GooglePlay", // o "AppStore"
    "data": "<recibo_base64>"
  }
}
```	

**Response:**			
```json
{
  "message": "Recibo verificado y plan actualizado",
  "plan": "premium"
}
```			

**Description:** Verifica un recibo de compra de Google Play o App Store. El backend valida el recibo con la API oficial de cada plataforma, actualiza el plan del usuario y registra la transacción.

- **Control de transacciones:**
Solo se permite una transacción activa por usuario.
  - Si el usuario ya tiene una transacción activa y no ha expirado, se actualiza la expiración.
  - Si la transacción activa ya expiró, se marca como expirada y se registra una nueva.

- **Campos actualizados en el usuario:**
  - plan, premium, planExpiration, planStatus, subscriptionSource, trial, trialDaysLeft, role

- **Registro de transacciones:**
  - Cada transacción almacena: source, receiptData, plan, purchaseDate, expirationDate, planStatus, createdAt

- **Auth:** Bearer server JWT (rol: user)
- **Body:**
  - Para Google Play:
    - `data` debe ser un base64 de un JSON con los campos `packageName`, `productId`, `purchaseToken`.
  - Para App Store:
    - `data` es el recibo base64 entregado por Apple.

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
  "message": "Recibo verificado y plan actualizado",
  "plan": "premium"
}
```
- **Errores:**
  - 400: Body inválido o faltan campos
  - 401: Recibo inválido o no verificado
  - 403: Usuario baneado o sin permisos
  - 500: Error al actualizar transacción

**Ejemplo curl (Google Play):**
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

**Ejemplo curl (App Store):**
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
- **Description:** Este endpoint permite enviar un lote (batch) de eventos de usuario (por ejemplo: play, pause, etc.) para ser procesados de forma eficiente y almacenados en el backend. Es ideal para apps que generan muchos eventos en poco tiempo (alta frecuencia).

- **Auth:** Bearer server JWT (rol: user)

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
- **Errores:**
  - 400: Body inválido o faltan campos.
  - 401: No autenticado
  - 403: Usuario baneado o sin permisos
  - 500: Error interno al consultar favoritos

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
## Alerts & Reminders - Configuración avanzada

### POST /v1/alerts
**Description:** Crea o agenda una alerta o recordatorio para el usuario.
- **Auth:** Bearer server JWT (rol: user)
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
  - alertId: ID de la alerta a enviar

- **Query Params (opcional):**
  - userId: ID del usuario (si la alerta está en la subcolección del usuario)

**Response:**			
```json
{ "message": "Alert sent", "alertId": "alert123" }
```			
- **Errores:**
  - 400: Missing alertId
  - 404: Alert not found
  - 500: Failed to update alert status

**Request Example:**
TODO: Este curl aun esta por definirse
```sh
curl -X POST "http://localhost:8080/v1/alerts/alert123/send" \
  -H "Authorization: Bearer <SERVER_JWT_ADMIN>"
```	
(Si la alerta está bajo un usuario específico)
	
```sh
curl -X POST "http://localhost:8080/v1/alerts/alert123/send?userId=abc123" \
  -H "Authorization: Bearer <SERVER_JWT_ADMIN>"
```	
---

### POST /v1/alerts/reminder
- **Descripción:** Configura la hora, frecuencia y días activos del recordatorio de meditación para el usuario. Guarda la configuración en Firestore bajo `users/{userId}/reminders/meditation`.
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
- **Errores:**
  - 400: Body inválido o campos obligatorios faltantes
  - 500: Error guardando recordatorio en Firestore
- **Detalle:**
  - `reminderTime`: Hora en formato HH:mm (ejemplo: "21:00").
  - `frequency`: Puede ser "daily" (todos los días), "weekdays" (lunes a viernes), o "custom" (días personalizados).
  - `activeDays`: Array de días activos. Usar abreviaturas: "M" (lunes), "T" (martes), "W" (miércoles), "T" (jueves), "F" (viernes), "S" (sábado), "U" (domingo).
  - La configuración se almacena en Firestore y puede ser consultada por la app para mostrar o modificar el recordatorio.
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
- **Description:** Configura los toggles de cada tipo de notificación.
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
- **Description:** Configura el estilo preferido para los mensajes personalizados para el usuario autenticado.
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
- **Description:** Obtiene la vista previa del mensaje personalizado para el usuario.
- **Auth:** Bearer server JWT
- **Headers:**
  - Authorization: Bearer <token>
  - Accept-Language: es (opcional, valores: es, en, etc.)
- **Query params:** 
  - `userId`
- **Response:**
```json
{
    "preview": "¡Eres imparable! Tu sesión te espera a las 9:00 PM."
}
```
- **Errores:**
  - 400: Missing userId query param
  - 401: No autenticado
  - 500: Error interno al consultar favoritos


- **Curl Example:**
  ```bash
  curl -X GET https://api.tuapp.com/v1/alerts/preview?userId=abc123" \
  -H "Authorization: Bearer <token>" \
  -H "Accept-Language: es"
  ```
---
### GET /v1/alerts/upcoming
- **Description:** Devuelve la lista de notificaciones programadas para los próximos días.
- **Auth:** Bearer server JWT
- **Query params:** `userId`
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

- **Errores:**
  - 400: Falta userId
  - 401: No autenticado
  - 500: Error interno al consultar favoritos

**Request Example:**
```sh
curl -X GET "https://api.tuapp.com/v1/alerts/upcoming?userId=user123" \
  -H "Authorization: Bearer <server JWT>"
```		

---
### PATCH /v1/alerts/advanced-options
- **Description:** Configura las opciones de snooze y respeto a DND.
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

- **Errores:**
  - 400: Falta userId o body inválido
  - 401: No autenticado
  - 500: Error interno al actualizar opciones

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
- **Description:** Devuelve toda la configuración de alertas y recordatorios del usuario
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
- **Errores:**
- 400: Falta userId
- 401: No autenticado
- 500: Error interno al consultar configuración

**Request Example:**

```sh
curl --location 'http://localhost:8080/v1/alerts/config?userId=abc123' \
--header 'Authorization: Bearer <SERVER_JWT>'
```		

---


### PATCH /v1/alerts/config
- **Description:** Actualiza la configuración de alertas y recordatorios del usuario.
- **Auth:** Bearer server JWT (rol: user)
- **Body Params:**: (puedes enviar solo los campos que deseas actualizar)
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
  "message": "Configuración de alertas actualizada"
}
```

- **Errores:**
  - 400: Falta userId o body inválido
  - 401: No autenticado
  - 500: Error interno al actualizar configuración

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