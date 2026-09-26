# Vanish.so API Documentation

Vanish.so is a client-side encrypted, vanishing note service that supports secure sharing with AES-GCM-256, custom password key-derivation via PBKDF2, and One-Time Pads (OTP). This document details the server-side API endpoints, database schema, and cryptographic flows.

---

## 🔐 Security Architecture

Vanish.so employs a "Zero-Knowledge" server architecture:
1. **Client-Side Encryption:** All note contents are encrypted on the client side using AES-GCM (256-bit keys) before transmission. The raw decryption keys are *never* sent to the server.
2. **Key Derivation (PBKDF2):** In Password mode, keys are derived client-side from user passwords using PBKDF2 with 100,000 iterations of SHA-256 and a random client-side salt.
3. **Double Salted Authentication Hash:** To verify authority to fetch/delete a note, a client-derived authentication hash (`h`) is sent to the server. The server generates a unique server-side salt (`ss`), hashes the client's `h` with `ss`, and stores the resulting hash (`sh`) in the database.
4. **Vanish-on-Read / Expiry:** Notes set to expire "after viewing" are deleted from the database immediately upon being read. Otherwise, notes are automatically deleted after they pass their scheduled expiration timestamp (1 hour, 24 hours, 7 days, or 30 days).

---

## 🛠️ Endpoints

### 1. Create a Note (`POST /api/new`)

Generates a new note record on the server.

*   **URL:** `/api/new`
*   **Method:** `POST`
*   **Headers:** `Content-Type: application/json`

#### Request Payload Schema

```typescript
interface NewNote {
  mode: "p" | "k" | "otp";
  encrypted: string;
  exp: "viewing" | "1h" | "24h" | "7d" | "30d";
  h: string;
  s: string;
  confirmBeforeViewing: boolean;
}
```

| Field | Type | Description |
| :--- | :--- | :--- |
| `mode` | `string` | The decryption mode: `"p"` (Password), `"k"` (Key), or `"otp"` (One-Time Pad). |
| `encrypted` | `string` | Base64-encoded encrypted note payload. |
| `exp` | `string` | Expiry rule: `"viewing"` (vanish on read), `"1h"`, `"24h"`, `"7d"`, or `"30d"`. |
| `h` | `string` | Hashed password/identifier (client-side generated) used for verification on read. |
| `s` | `string` | Client-side salt (Base64-encoded) used for PBKDF2 key derivation. |
| `confirmBeforeViewing` | `boolean` | If true, prompts the recipient to confirm before revealing the note (highly recommended for single-view notes to prevent pre-fetching/preview links from burning the note). |

*Note: The server will force `confirmBeforeViewing` to `true` if `mode === "p"` or if `exp !== "viewing"`.*

#### Response Payload (200 OK)

```json
{
  "noteid": "A1b2C3d4E5"
}
```

*   `noteid`: A randomly-generated 10-character unique identifier for the created note.

#### Error Responses

*   **`400 Bad Request`**: Returned if `exp` or `mode` does not match the allowed lists of values.

---

### 2. Read a Note (`POST /api/read`)

Retrieves the encrypted content of a note. Reading a note with the `"viewing"` expiry policy will immediately and permanently delete it from the database.

*   **URL:** `/api/read`
*   **Method:** `POST`
*   **Headers:** `Content-Type: application/json`

#### Request Payload Schema

```json
{
  "id": "A1b2C3d4E5",
  "auth": "client_derived_auth_string"
}
```

| Field | Type | Description |
| :--- | :--- | :--- |
| `id` | `string` | The 10-character note identifier returned during creation. |
| `auth` | `string` | The raw client-side password hash/identity used to compute the server match hash. (Not required/ignored in `"otp"` mode). |

#### Response Payload (200 OK)

```json
{
  "content": "Base64_encrypted_payload"
}
```

#### Error Responses

*   **`400 Bad Request`**: Returned if:
    *   The note does not exist (already deleted, expired, or incorrect ID).
    *   The note has expired (notes are checked for expiration on-demand and deleted if `Date.now() > exp`).
    *   The password verification fails (the SHA-256 hash of `auth` + server salt `ss` does not match the stored hash `h`).

---

## 🗄️ Database Schema

Vanish.so uses SQLite (via Drizzle ORM). The `notes` table is structured as follows:

| Column | Type | Nullable | Description |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT (Primary Key)` | No | 10-character unique note ID. |
| `confirmBeforeViewing` | `INTEGER` | No | SQLite boolean representation (0/1). |
| `mode` | `TEXT` | No | `"p"`, `"k"`, or `"otp"`. |
| `encrypted` | `TEXT` | No | Base64 encrypted cipher text. |
| `exp` | `INTEGER` | No | Expiration timestamp in milliseconds (or 0 for vanish-on-read). |
| `h` | `TEXT` | No | Server-salted SHA-256 hash of client auth code (`sh = hash(auth, ss)`). |
| `cs` | `TEXT` | No | Client salt (for key derivation). |
| `ss` | `TEXT` | No | Server salt (randomly generated on creation for server-side hashing). |

---

## ⚙️ Client Cryptographic Flows

### 1. Generating a Note

1.  **Generate a Random Secret Key:**
    *   For Key mode: Generate a 256-bit AES-GCM key.
    *   For Password mode: Prompt for password, generate a random 16-byte `cs` salt, and derive a 256-bit key via PBKDF2 with 100,000 iterations.
2.  **Encrypt Note Content:**
    *   Generate a random 12-byte Initialization Vector (IV).
    *   Encrypt the note content using AES-GCM.
    *   Concatenate `IV (Base64)` + `Ciphertext (Base64)` to create the `encrypted` field.
3.  **Generate Authentication Code (`h`):**
    *   Construct an auth code to prove decryption capability without sending the actual key. For password mode, this is a hash of the password; for key mode, it is a SHA-256 hash of the key itself.
4.  **Send Payload to `/api/new`.**
5.  **Construct URL:**
    *   `https://vanish.so/n/<noteid>#<key_or_password>` (The key hash is placed in the URL fragment `#` so that it is never sent to the server in HTTP requests).

### 2. Decrypting a Note

1.  **Fetch Metadata:** Read the note metadata using the `noteid` in the SvelteKit loader (which fetches schema fields except `encrypted` and `h` / `sh` to determine parameters like `mode`, `confirmBeforeViewing`, and `cs`).
2.  **Request Note Content:**
    *   Call `POST /api/read` with the `id` and the client's auth identifier.
    *   If authorized and not expired, the server returns the `content`.
3.  **Decrypt Content:**
    *   Extract the 12-byte IV (first 16 Base64 chars) and the ciphertext (remaining characters).
    *   Using the key derived from the URL fragment (and optionally the client salt `cs`), decrypt the ciphertext using `AES-GCM` with the extracted IV.
