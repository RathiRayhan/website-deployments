# Nginx Reverse Proxy Routing & Path Manipulation Guide

This runbook outlines the exact decision-making process for configuring Nginx reverse proxies based on how frontend and backend applications define their API routes. 

## Step 1: Reconnaissance (Analyze the Application Code)
Before writing any Nginx configuration, you must inspect both the frontend and backend repositories to understand the routing architecture.

1.  **Check the Frontend Base URL:**
    *   Look for `.env` files or API configuration files.
    *   Search for variables like `VITE_APP_BASE_URL`, `REACT_APP_API_URL`, or `NEXT_PUBLIC_API_URL`.
    *   *Expected Outcome:* Determine whether the frontend sends requests with a specific prefix (e.g., `/api/users`) or directly to the root path (e.g., `/client/products`).
2.  **Check the Backend Route Definitions:**
    *   Search the backend code for route initializations (e.g., `grep -R "app.use" server/`).
    *   *Expected Outcome:* Identify the exact route definitions in the backend code, noting whether they are registered with a prefix (e.g., `app.use('/api/users', ...)`) or without one (e.g., `app.use('/client', ...)`).

Based on the findings above, choose one of the following three methods.

---

## Method 1: The Common Prefix WITHOUT Trailing Slash (Exact Match)

**When to use:** Both Frontend and Backend share the EXACT same path structure, and there is a common prefix (like `/api/`). Nginx does not need to modify the URL.

*   **Frontend requests:** `/api/users`
*   **Backend expects:** `/api/users`

**Nginx Configuration:**
```nginx
location /api/ {
    # NO trailing slash at the end of the URL
    proxy_pass http://127.0.0.1:5001; 
}
```

**Behavior:**
Nginx acts as a transparent proxy. The URI `/api/users` is passed exactly as it is to the backend.

---

## Method 2: The Common Prefix WITH Trailing Slash (Path Rewrite)

**When to use:** The Frontend uses a common prefix (like `/api/`) to help Nginx route the traffic, BUT the Backend does NOT include this prefix in its route definitions. Nginx must strip the prefix before forwarding.

*   **Frontend requests:** `/api/users`
*   **Backend expects:** `/users`

**Nginx Configuration:**
```nginx
location /api/ {
    # HAS a trailing slash at the end of the URL
    proxy_pass http://127.0.0.1:5001/; 
}
```

**Behavior:**
Nginx strips the matched `location` block part (`/api/`) from the URI and forwards the remainder (`/users`) to the backend.

> ⚠️ **IMPORTANT NOTE (Common Pitfall):**
> You cannot arbitrarily use this method if the frontend does not natively send the prefix. For Nginx to strip `/api/` using a trailing slash, **the frontend MUST actually include `/api/` in its original request.** 
> If the frontend requests `/client/products` directly, the `location /api/` block will never trigger. A DevOps engineer cannot force this Nginx configuration unless they also have the ability to modify the frontend's API Base URL codebase to append the `/api/` prefix.

---

## Method 3: The Regex Method (Multiple Prefixes / No Common Base)

**When to use:** The application does not use a single unifying prefix (like `/api/`). Instead, the frontend calls multiple distinct root-level endpoints, and the backend expects those exact same endpoints. 

*   **Frontend requests:** `/client/products`, `/sales/reports`, `/management/users`
*   **Backend expects:** `/client/products`, `/sales/reports`, `/management/users`

**Nginx Configuration:**
```nginx
# Use the tilde (~) for case-sensitive regular expression matching
location ~ ^/(client|sales|management|general)/ {
    # MUST NOT have a trailing slash (Nginx strictly forbids URI rewriting inside regex locations)
    proxy_pass http://127.0.0.1:5001;
}
```

**Behavior:**
Nginx groups these specific starting paths together. If a request starts with any of these words, it is forwarded to the backend without any modification.

> **💡 Pro-Tip:** Alternatively, you can write multiple standard `location` blocks (e.g., `location /client/ { ... }`, `location /sales/ { ... }`). The Regex method is simply a cleaner, more concise way to handle this scenario without polluting the Nginx config file.

---

## Decision Matrix Summary

| Frontend Request | Backend Expects | Nginx Block | Proxy Pass Trailing Slash? |
| :--- | :--- | :--- | :--- |
| `/api/users` | `/api/users` | `location /api/` | **No** (`proxy_pass http://...;`) |
| `/api/users` | `/users` | `location /api/` | **Yes** (`proxy_pass http://.../;`) |
| `/users` | `/users` | `location /users/` | **No** (`proxy_pass http://...;`) |
| Multiple unique paths | Exact same unique paths | `location ~ ^/(x|y|z)/` | **No** (`proxy_pass http://...;`) |
