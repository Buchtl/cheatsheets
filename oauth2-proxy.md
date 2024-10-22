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
