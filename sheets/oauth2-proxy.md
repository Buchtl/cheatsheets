To use Bitnami's OAuth2 Proxy container to secure a React application, you will set up the OAuth2 Proxy as a reverse proxy that requires authentication before allowing access to your React app. The process involves the following steps:

### 1. **Prerequisites**
   - A running React app (served using a web server like NGINX or any other preferred method).
   - A domain name or local setup using `localhost` or custom domains.
   - OAuth2 provider credentials (e.g., Google, GitHub, Okta, etc.).

### 2. **Set Up OAuth2 Provider Credentials**
   - Depending on your provider, follow their documentation to create OAuth credentials:
     - For Google: Go to the [Google Developers Console](https://console.developers.google.com/), create a project, and obtain client ID and secret.
     - For GitHub: Go to your [GitHub Developer Settings](https://github.com/settings/developers), create a new OAuth app, and get client ID and secret.

   You will need:
   - `CLIENT_ID`
   - `CLIENT_SECRET`
   - `REDIRECT_URI` (e.g., `http://localhost:4180/oauth2/callback`)

### 3. **Launch OAuth2 Proxy Using Bitnami's Image**
   You can use Docker Compose to run the OAuth2 Proxy. Create a `docker-compose.yml` file:

   ```yaml
   version: '3'
   services:
     oauth2-proxy:
       image: bitnami/oauth2-proxy:latest
       ports:
         - "4180:4180"
       environment:
         - OAUTH2_PROXY_PROVIDER=google # or github, okta, etc.
         - OAUTH2_PROXY_CLIENT_ID=<your-client-id>
         - OAUTH2_PROXY_CLIENT_SECRET=<your-client-secret>
         - OAUTH2_PROXY_COOKIE_SECRET=<a-random-32-byte-string>
         - OAUTH2_PROXY_REDIRECT_URL=http://localhost:4180/oauth2/callback
         - OAUTH2_PROXY_UPSTREAMS=http://localhost:3000
         - OAUTH2_PROXY_EMAIL_DOMAINS=*
         - OAUTH2_PROXY_COOKIE_SECURE=false
       volumes:
         - ./oauth2_proxy.cfg:/opt/bitnami/oauth2-proxy/conf/oauth2_proxy.cfg
   ```

   Replace `<your-client-id>` and `<your-client-secret>` with the values from your OAuth2 provider.

   - `OAUTH2_PROXY_UPSTREAMS` is set to your React app's backend (e.g., `http://localhost:3000` if your app is served there).
   - `OAUTH2_PROXY_COOKIE_SECRET` must be a random 32-byte string for signing cookies. You can generate one using `openssl rand -base64 32`.

### 4. **Configure OAuth2 Proxy Settings (Optional)**
   If you need more customization, create an `oauth2_proxy.cfg` file with additional settings. Here’s an example:

   ```ini
   # oauth2_proxy.cfg
   http_address = "0.0.0.0:4180"
   upstreams = [
     "http://localhost:3000"
   ]
   provider = "google"
   client_id = "<your-client-id>"
   client_secret = "<your-client-secret>"
   redirect_url = "http://localhost:4180/oauth2/callback"
   cookie_secure = false
   email_domains = [
     "*"
   ]
   ```

   This config file can be mounted into the OAuth2 Proxy container using the volume in `docker-compose.yml`.

### 5. **Run Docker Compose**
   Start the OAuth2 Proxy container:

   ```bash
   docker-compose up -d
   ```

   This will start OAuth2 Proxy on port `4180` and will forward authenticated requests to your React app running on `localhost:3000`.

### 6. **Test the Authentication Flow**
   - Visit `http://localhost:4180`.
   - You should be redirected to the OAuth provider's login page.
   - After authenticating, OAuth2 Proxy will redirect you to your React app.

### 7. **Optional: Deploying Behind NGINX (or Any Web Server)**
   If you want to serve your React app and OAuth2 Proxy behind a web server like NGINX, set up a reverse proxy to handle both services:

   ```nginx
   server {
       listen 80;
       server_name your-domain.com;

       location /oauth2/ {
           proxy_pass http://localhost:4180;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }

       location / {
           auth_request /oauth2/auth;
           error_page 401 = @error401;
           proxy_pass http://localhost:3000;
       }

       location @error401 {
           return 302 /oauth2/start?rd=$scheme://$host$request_uri;
       }
   }
   ```

   Replace `your-domain.com` with your domain. This configuration ensures:
   - All requests are checked by `oauth2/auth` before reaching the React app.
   - Unauthorized users are redirected to the OAuth2 login page.

### Summary
1. Configure OAuth2 Proxy with the proper provider settings.
2. Set the upstream to point to your React app.
3. Use Docker Compose to run the proxy.
4. Optionally configure NGINX for better management in production.

This setup ensures that users must authenticate with the OAuth2 provider before they can access your React app.


# Overwrite Auth Header

By default, if an Authorization header is set explicitly in your React app, the oauth2-proxy can overwrite it depending on its configuration. Specifically, the behavior is controlled by the following environment variables:
Key Configurations for oauth2-proxy

* `OAUTH2_PROXY_SET_AUTHORIZATION_HEADER`:

    If this is set to true, oauth2-proxy will overwrite the Authorization header with its own token (Authorization: Bearer <token>).
    If it is set to false, it should leave any existing Authorization header unchanged.

* `OAUTH2_PROXY_PASS_ACCESS_TOKEN`:

    When this is true, it passes the OAuth2 token to the upstream in either cookies or headers.
    `If OAUTH2_PROXY_SET_AUTHORIZATION_HEADER` is also true, this will add or overwrite the Authorization header with the token.

How to Prevent Overwriting

If you want to ensure that the Authorization header you set in your React app remains unchanged when forwarded through oauth2-proxy, you should configure oauth2-proxy like this in your docker-compose.yml:

```yaml
environment:
  OAUTH2_PROXY_SET_AUTHORIZATION_HEADER: "false" # Prevents overwriting the Authorization header
  OAUTH2_PROXY_PASS_ACCESS_TOKEN: "false" # Optional: prevents the access token from being added automatically
```
By setting OAUTH2_PROXY_SET_AUTHORIZATION_HEADER to false, oauth2-proxy will not modify the Authorization header that comes from your React app.
Summary

* If `OAUTH2_PROXY_SET_AUTHORIZATION_HEADER` is true, oauth2-proxy will overwrite the Authorization header.
    To preserve the Authorization header set in your React app, ensure `OAUTH2_PROXY_SET_AUTHORIZATION_HEADER` is set to false in your docker-compose.yml.

This configuration will make sure that any Authorization header you set manually in the React app is respected when requests are forwarded by the OAuth2 proxy.


# USER-INFO
To access user information like `preferred_username` in your React app when using `oauth2-proxy`, the typical approach is to configure `oauth2-proxy` to include specific user details in the headers of proxied requests. Then, these headers can be read by your backend service or, in some cases, directly by your React app if the information is exposed.

Here’s how you can achieve this:

### Step 1: Configure `oauth2-proxy` to Pass User Information
You need to set up `oauth2-proxy` to include user information as headers in the proxied requests. This is done using the `OAUTH2_PROXY_SET_XAUTHREQUEST` configuration, which adds a series of `X-Auth-Request-*` headers to the forwarded requests. This setting allows the proxy to pass user details such as `preferred_username`.

In your `docker-compose.yml`:

```yaml
services:
  oauth2-proxy:
    image: quay.io/oauth2-proxy/oauth2-proxy:v7.2.1
    environment:
      OAUTH2_PROXY_PROVIDER: "oidc" # or the provider you are using
      OAUTH2_PROXY_CLIENT_ID: "<your-client-id>"
      OAUTH2_PROXY_CLIENT_SECRET: "<your-client-secret>"
      OAUTH2_PROXY_COOKIE_SECRET: "<a-random-secret>"
      OAUTH2_PROXY_UPSTREAMS: "http://your-backend-service:8080"
      OAUTH2_PROXY_HTTP_ADDRESS: "0.0.0.0:4180"
      OAUTH2_PROXY_SET_XAUTHREQUEST: "true" # Enable passing user info as headers
    ports:
      - "4180:4180"
```

With `OAUTH2_PROXY_SET_XAUTHREQUEST` set to `true`, the following headers become available to your upstream (backend service):
- `X-Auth-Request-User`: Typically the user's name or identifier.
- `X-Auth-Request-Email`: The user's email address.
- `X-Auth-Request-Preferred-Username`: The `preferred_username` claim from the OAuth2 token (if available).

### Step 2: Make User Information Available to the React App
Now that `oauth2-proxy` adds the user information as headers to requests, you have a couple of options for making this information accessible to your React app:

#### Option 1: Expose User Info via a Custom Endpoint
The most common way is to have your backend service expose an endpoint that reads these headers and returns the user information to the frontend. Here’s a simple example using an Express server as the backend:

```javascript
// server.js (Express server example)
const express = require('express');
const app = express();

app.get('/user-info', (req, res) => {
  // Extract user information from headers set by oauth2-proxy
  const preferredUsername = req.headers['x-auth-request-preferred-username'];
  const email = req.headers['x-auth-request-email'];
  
  res.json({
    preferred_username: preferredUsername,
    email: email,
  });
});

app.listen(8080, () => {
  console.log('Backend service running on port 8080');
});
```

In this example, the `/user-info` endpoint reads the `X-Auth-Request-Preferred-Username` header and other user details set by `oauth2-proxy` and returns them as JSON.

#### Option 2: Fetch User Information in Your React App
In your React app, you can now make a request to the `/user-info` endpoint to retrieve the user's `preferred_username`:

```javascript
import React, { useEffect, useState } from 'react';

const UserInfo = () => {
  const [userInfo, setUserInfo] = useState(null);

  useEffect(() => {
    const fetchUserInfo = async () => {
      try {
        const response = await fetch('/user-info', {
          method: 'GET',
          credentials: 'include', // Ensure cookies are sent with the request
        });
        const data = await response.json();
        setUserInfo(data);
      } catch (error) {
        console.error('Error fetching user info:', error);
      }
    };

    fetchUserInfo();
  }, []);

  if (!userInfo) {
    return <div>Loading user information...</div>;
  }

  return (
    <div>
      <p>Preferred Username: {userInfo.preferred_username}</p>
      <p>Email: {userInfo.email}</p>
    </div>
  );
};

export default UserInfo;
```

In this code:
- The `fetchUserInfo` function sends a request to the `/user-info` endpoint to retrieve the user details.
- It then updates the state with the user's information, including `preferred_username`, which can be displayed in the React component.

### Summary
- **Step 1**: Set `OAUTH2_PROXY_SET_XAUTHREQUEST` to `true` to pass user information as headers.
- **Step 2**: Create a backend endpoint that extracts user info from these headers.
- **Step 3**: Call this endpoint from your React app to access and display user details like `preferred_username`.

This approach ensures that the OAuth2 proxy handles authentication and the backend forwards the necessary user information securely to the frontend.

If you are getting a `302` (temporary redirect) response instead of a `301` (permanent redirect), the issue may be with the way the `RewriteRule` is configured or a potential conflict with another rule or configuration. To ensure a `301` redirect, you can make a few adjustments to your Apache configuration:

### Updated Configuration:
```apache
<VirtualHost *:80>
    RewriteEngine On
    RewriteCond %{HTTPS} off
    RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
</VirtualHost>
```

### Things to Check when getting 302 instead 301:
1. **Clear Browser Cache**: Browsers often cache redirects, especially `302` responses. Try clearing your browser cache or testing in incognito mode to ensure the new `301` response is applied.
   
2. **Check for Other Redirects**: Make sure there are no other configurations or `.htaccess` files that might contain a `Redirect` or `RewriteRule` causing a `302` response.

3. **Explicitly Set the Status Code**: If the above rule is not forcing the `301`, try using a more explicit form:
    ```apache
    <VirtualHost *:80>
        Redirect permanent / https://%{HTTP_HOST}/
    </VirtualHost>
    ```
   This approach uses the `Redirect` directive, which is simpler and directly sets a `301` status.

4. **Check Apache Logs**: If it's still returning a `302`, check the Apache error and access logs for any clues:
   ```bash
   sudo tail -f /var/log/apache2/error.log
   sudo tail -f /var/log/apache2/access.log
   ```

5. **Restart Apache**: After making these changes, don't forget to restart Apache to ensure the new configuration is applied:
    ```bash
    sudo systemctl restart apache2
    ```

These adjustments should ensure that Apache sends a `301` redirect instead of a `302` when redirecting from HTTP to HTTPS.

# Access Token

To configure an OAuth2 Proxy Docker container (using Swarm or Compose) to include the OIDC access token in the headers, you need to:

1. **Ensure that `--pass-access-token` is enabled in the OAuth2 Proxy configuration.**
   This flag will instruct the proxy to include the OIDC `access_token` as a header in the forwarded request.

2. **Set up `--pass-authorization-header` if needed.**
   If you want to include the token as a Bearer token in the `Authorization` header, this should also be enabled.

3. **Define custom headers for the token if necessary.**
   You can configure OAuth2 Proxy to add the access token to a specific header using `--set-authorization-header` or by defining custom header mappings.

### Example Docker Compose Configuration
Here’s an example of how to configure this in a `docker-compose.yml` file:

```yaml
version: "3.7"
services:
  oauth2-proxy:
    image: quay.io/oauth2-proxy/oauth2-proxy:v7.4.0
    ports:
      - "4180:4180"
    environment:
      OAUTH2_PROXY_PROVIDER: "oidc"
      OAUTH2_PROXY_OIDC_ISSUER_URL: "https://your-issuer-url.com"
      OAUTH2_PROXY_CLIENT_ID: "your-client-id"
      OAUTH2_PROXY_CLIENT_SECRET: "your-client-secret"
      OAUTH2_PROXY_COOKIE_SECRET: "your-cookie-secret"
    command:
      - --http-address=0.0.0.0:4180
      - --upstream=http://example-app:8080
      - --redirect-url=https://your-redirect-url.com/oauth2/callback
      - --pass-access-token=true
      - --set-authorization-header=true
      - --pass-authorization-header=true
      - --email-domain=*
```

### Key Configuration Flags
- **`--pass-access-token=true`**
  Ensures that the `access_token` is passed in the headers.

- **`--set-authorization-header=true`**
  Sets the `Authorization` header with the `access_token` as a Bearer token.

- **`--pass-authorization-header=true`**
  Ensures the `Authorization` header is forwarded as is.

- **Custom Header Mapping (Optional):**
  If you want to map the access token to a custom header, use a proxy or a middleware upstream of the OAuth2 Proxy.

### With Docker Swarm
In Docker Swarm, you can use a `deploy` block to ensure proper scaling and resource allocation. For example:

```yaml
services:
  oauth2-proxy:
    image: quay.io/oauth2-proxy/oauth2-proxy:v7.4.0
    ports:
      - "4180:4180"
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: "0.5"
          memory: "512M"
      restart_policy:
        condition: on-failure
    environment:
      # Same as in the Compose example
    command:
      # Same as in the Compose example
```

### Verify Configuration
After configuring, test the headers with a debugging tool like `curl` or Postman to confirm that the `access_token` is included in the headers. For example:

```bash
curl -v -H "Authorization: Bearer <token>" http://localhost:4180
```

If you don't see the `access_token` being forwarded correctly, double-check your issuer configuration and OAuth2 Proxy logs for issues.

# Difference Between Acess Token and ID Token
The **access token** and **ID token** are both JSON Web Tokens (JWTs) commonly used in OAuth 2.0 and OpenID Connect (OIDC), but they serve different purposes:

---

### **1. Access Token**
**Purpose**: Grants access to APIs and resources.
- **What it contains**:
  - Information required by the resource server (e.g., API) to authorize requests.
  - Often includes user permissions (scopes), expiration time, and information about the client or user.
  - It **does not contain detailed user profile information.**
  - Example claims:
    ```json
    {
      "iss": "https://auth.example.com",
      "sub": "1234567890",
      "aud": "api.example.com",
      "exp": 1698771200,
      "scope": "read write"
    }
    ```

- **Use case**:
  - Sent in the `Authorization` header as a Bearer token:
    ```http
    Authorization: Bearer <access_token>
    ```
  - Allows the client to act on behalf of a user when calling APIs.

- **Audience**: Resource server (e.g., an API).

---

### **2. ID Token**
**Purpose**: Confirms the user's identity to the client application.
- **What it contains**:
  - Information about the authenticated user.
  - Typically includes the user's profile details, such as name, email, and other claims (depending on what the identity provider returns).
  - Example claims:
    ```json
    {
      "iss": "https://auth.example.com",
      "sub": "1234567890",
      "aud": "your-client-id",
      "exp": 1698771200,
      "email": "user@example.com",
      "name": "John Doe",
      "iat": 1698767600
    }
    ```

- **Use case**:
  - Used by the client application to verify that the user has been authenticated by the identity provider.
  - Typically returned during the OpenID Connect login flow.
  - It is **not sent to APIs**; it's used only by the client.

- **Audience**: The client application (e.g., a web or mobile app).

---

### **Key Differences**
| Feature           | Access Token                          | ID Token                             |
|--------------------|---------------------------------------|---------------------------------------|
| **Purpose**        | Authorizes access to APIs            | Confirms user's identity             |
| **Audience**       | Resource server (API)                | Client application                   |
| **Includes User Info?** | No, typically contains scopes and permissions | Yes, user profile info (e.g., name, email) |
| **Use Case**       | Sent in API requests                 | Verifies authentication              |
| **Expires?**       | Yes                                  | Yes                                  |
| **Part of OAuth2?**| Yes                                  | No, specific to OIDC                 |

---

### **Analogy**
Think of a **concert:**
- The **access token** is your ticket that grants entry to different areas (e.g., backstage, VIP). It's given to the event staff (API) to validate your permissions.
- The **ID token** is like your ID card that proves who you are to the ticket issuer (client application). It’s used at check-in but not shown to event staff. 

# Decode Token

Here is a shell script that decodes a JWT access token. The script assumes the token is passed as an argument and requires `jq` for JSON formatting.

```bash
#!/bin/bash

# Check for required utilities
if ! command -v jq &> /dev/null; then
    echo "Error: jq is not installed. Please install jq to use this script."
    exit 1
fi

# Check if token is provided
if [ -z "$1" ]; then
    echo "Usage: $0 <jwt-access-token>"
    exit 1
fi

JWT_TOKEN=$1

# Split the JWT token into its components
IFS='.' read -r HEADER PAYLOAD SIGNATURE <<< "$JWT_TOKEN"

# Decode Base64 (with URL-safe encoding) and parse JSON
decode_base64url() {
    local encoded=$1
    # Convert Base64 URL format to standard Base64
    local decoded=$(echo "$encoded" | tr '_-' '/+' | base64 -d 2>/dev/null)
    if [ $? -ne 0 ]; then
        echo "Error: Invalid JWT token"
        exit 1
    fi
    echo "$decoded"
}

# Decode header
HEADER_JSON=$(decode_base64url "$HEADER" | jq .)
echo "===== HEADER ====="
echo "$HEADER_JSON"

# Decode payload
PAYLOAD_JSON=$(decode_base64url "$PAYLOAD" | jq .)
echo "===== PAYLOAD ====="
echo "$PAYLOAD_JSON"

# Signature can't be decoded directly as it is hashed
echo "===== SIGNATURE ====="
echo "$SIGNATURE"
```

### How to Use the Script
1. Save the script to a file, e.g., `decode_jwt.sh`.
2. Make it executable:
   ```bash
   chmod +x decode_jwt.sh
   ```
3. Run the script with your JWT token as an argument:
   ```bash
   ./decode_jwt.sh <your-jwt-token>
   ```

### Example Output
```plaintext
===== HEADER =====
{
  "alg": "HS256",
  "typ": "JWT"
}
===== PAYLOAD =====
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022
}
===== SIGNATURE =====
abc123signaturepart
```

**Note:** The script does not verify the JWT signature; it only decodes the token for inspection. Signature verification requires the secret or public key used to sign the token.

# Logout

Logging out of a React app that uses **OAuth2 Proxy** involves clearing the user's session in the proxy and potentially in the identity provider (IdP) as well. The process usually looks like this:

---

### Steps to Logout:

1. **Clear the OAuth2 Proxy Cookie/Session**:
   - When using OAuth2 Proxy, the user’s session is stored in a secure cookie (e.g., `oauth2_proxy` or a custom-named session cookie).
   - To log the user out, the session needs to be invalidated. This is typically done by redirecting the user to the OAuth2 Proxy's `/oauth2/sign_out` endpoint.

2. **Redirect to the Identity Provider (IdP) Logout URL**:
   - Some configurations require redirecting the user to the IdP's logout endpoint to terminate their session at the IdP level.
   - If this step is skipped, the user might still appear logged in at the IdP, and a new session could be initiated immediately upon accessing the app.

3. **Clear Application State**:
   - Remove any local state or cached information in your React app that pertains to the user, such as tokens stored in memory or local storage.

---

### Implementation in React App:

#### Step 1: Configure the OAuth2 Proxy Logout URL
Add a logout route in your OAuth2 Proxy configuration. This is usually `/oauth2/sign_out`.

You can also add `--after-sign-out-url=<url>` to specify where the user should be redirected after logout.

#### Step 2: Create a Logout Handler in React
Here’s a basic example of a logout function:

```javascript
const logout = () => {
  // Redirect to the OAuth2 Proxy logout endpoint
  window.location.href = '/oauth2/sign_out';
};
```

#### Step 3: Handle Optional IdP Logout
If you need to log out from the IdP, the OAuth2 Proxy's `/oauth2/sign_out` endpoint can redirect users to the IdP logout endpoint:

- Add the `--oidc-extra-logout-url-parameters` or similar flags (depending on your OAuth2 Proxy version) to include the IdP logout URL.
- Alternatively, manually append the IdP's logout URL as a query parameter in your React app:
  ```javascript
  const logout = () => {
    const idpLogoutUrl = 'https://idp.example.com/logout';
    const appRedirectUrl = encodeURIComponent(window.location.origin); // Redirect back to your app
    window.location.href = `/oauth2/sign_out?rd=${idpLogoutUrl}?returnTo=${appRedirectUrl}`;
  };
  ```

---

### Additional Notes:
- **Session Duration**: Ensure the OAuth2 Proxy session cookie expiration aligns with your app's logout behavior.
- **Token Revocation**: If access or refresh tokens are issued and need to be revoked, ensure the IdP supports this and configure it in your app or proxy.
- **Frontend Cleanup**: After logout, ensure your React app clears any user-related data in the local state, context, or localStorage.
