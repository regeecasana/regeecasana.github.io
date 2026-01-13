# OAuth Callback Page - Technical Overview

## Purpose
Static HTML page that receives OAuth authorization codes from Azure B2C and securely relays them back to the Zendesk app via `postMessage`.

## How It Works

### 1. **Receives Authorization Code**
```
Azure B2C redirects to:
https://yourdomain.com/callback.html?code=ABC123&state=XYZ789
```

### 2. **Validates Origin** (Security Layer)
```javascript
// Attempts to access window.opener.location.origin
if (canAccess && origin.match(WHITELIST_REGEX)) {
    targetOrigin = specific_origin;  // e.g., "https://1124196.apps.zdusercontent.com"
} else {
    targetOrigin = '*';  // Fallback for cross-origin iframes
}
```

**Whitelist patterns:**
- `*.zendesk.com` - Zendesk instances
- `[0-9]+.apps.zdusercontent.com` - Zendesk app hosting
- `*.zdassets.com` - Zendesk CDN

### 3. **Sends Message Back**
```javascript
window.opener.postMessage({
    type: 'AZURE_B2C_CALLBACK',
    success: true,
    code: 'ABC123',
    state: 'XYZ789'
}, targetOrigin);
```

### 4. **Auto-closes**
Window closes after 2 seconds on success, stays open on error.

## Security Features

| Feature | Implementation |
|---------|---------------|
| **Origin Validation** | Regex whitelist blocks non-Zendesk domains |
| **Targeted postMessage** | Uses specific origin when accessible, wildcard as fallback |
| **Audit Logging** | All actions logged to console |
| **Error Handling** | Shows user-friendly errors, doesn't send on validation failure |
| **No Secrets** | Only relays authorization code (useless without PKCE verifier) |

## Key Files
- **Callback Page:** `oauth-callback.html` (hosted at `auth.ixwisdom.com`)
- **Zendesk App:** Listens for `AZURE_B2C_CALLBACK` messages via `addEventListener('message')`

## Why Wildcard Fallback?
Cross-origin iframe restrictions in Zendesk prevent accessing `window.opener.location.origin`. Wildcard is necessary but still secure because:
- Authorization code requires PKCE `code_verifier` to exchange for tokens
- Code is single-use and expires in 5-10 minutes
- State parameter provides CSRF protection in the Zendesk app
