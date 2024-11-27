# Oauth2-Proxy Access Token

To configure Nginx to forward HTTPS requests while replacing the `Authorization` header with the value from the `X-Forwarded-Access-Token` header, you can use the following approach:

### Steps:

1. **Install the Required Modules**: Ensure your Nginx has the `ngx_http_headers_module` enabled. This module is generally included in most Nginx installations.

2. **Edit Your Nginx Configuration**:
   Update the configuration to include a server block that handles the forwarding logic.

   Here is an example configuration:

   ```nginx
   server {
       listen 443 ssl;
       server_name your-domain.com;

       ssl_certificate /path/to/your/certificate.crt;
       ssl_certificate_key /path/to/your/private.key;

       location / {
           # Extract the value of X-Forwarded-Access-Token
           set $access_token $http_x_forwarded_access_token;

           # Replace Authorization header with the token
           proxy_set_header Authorization "Bearer $access_token";

           # Forward other headers as-is
           proxy_pass http://backend-service;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
   }
   ```

3. **Explanation**:
   - `set $access_token $http_x_forwarded_access_token`: Extracts the value of the `X-Forwarded-Access-Token` header and assigns it to the `$access_token` variable.
   - `proxy_set_header Authorization "Bearer $access_token";`: Sets the `Authorization` header to `Bearer <value of X-Forwarded-Access-Token>`.
   - `proxy_pass http://backend-service;`: Forwards the request to the backend service.

4. **Test Your Configuration**:
   After updating the configuration file:
   ```bash
   sudo nginx -t
   ```
   This will check for syntax errors. If the test passes, reload Nginx:
   ```bash
   sudo systemctl reload nginx
   ```

5. **Verify Behavior**:
   Send a test request with the `X-Forwarded-Access-Token` header to ensure that the `Authorization` header is set correctly when forwarded to the backend service.

---

### Notes:
- Replace `http://backend-service` with the actual URL or IP of your backend service.
- If you are using a reverse proxy for multiple routes, ensure that you configure this behavior for the specific paths that require it.
- If the `X-Forwarded-Access-Token` header is missing, the `Authorization` header will be empty. You can add additional logic to handle this case if needed.

# When backend-service is only accessible via https

To configure Nginx to forward HTTPS requests to a backend that is also accessible via HTTPS, ensure that the `proxy_pass` directive specifies `https://` instead of `http://`. Additionally, configure Nginx to trust the backend's SSL/TLS certificate.

Here’s the updated configuration:

### Nginx Configuration for HTTPS to HTTPS Proxy with Header Replacement

```nginx
server {
    listen 443 ssl;
    server_name your-domain.com;

    ssl_certificate /path/to/your/certificate.crt;
    ssl_certificate_key /path/to/your/private.key;

    location / {
        # Extract the value of X-Forwarded-Access-Token
        set $access_token $http_x_forwarded_access_token;

        # Replace Authorization header with the token
        proxy_set_header Authorization "Bearer $access_token";

        # Forward other headers as-is
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Specify HTTPS backend
        proxy_pass https://backend-service;

        # SSL settings for the proxy
        proxy_ssl_server_name on; # Allows Nginx to use SNI to connect to the backend
        proxy_ssl_verify off;     # Optional: disables SSL verification (useful for self-signed certificates)
    }
}
```

### Key Points:
1. **`proxy_pass https://backend-service`**:
   - Ensures the connection to the backend is over HTTPS.

2. **`proxy_ssl_server_name on`**:
   - Enables Server Name Indication (SNI) when connecting to the backend, ensuring compatibility with multi-domain backends.

3. **`proxy_ssl_verify off`** (Optional):
   - Disables SSL certificate verification for the backend. Use this only if the backend uses self-signed certificates or untrusted certs.
   - If the backend's certificate is trusted, remove this line or replace it with proper certificate verification using `proxy_ssl_trusted_certificate`.

4. **SSL/TLS Certificates**:
   - Ensure that both the Nginx proxy and the backend have valid SSL/TLS certificates. If the backend uses self-signed certificates, consider providing Nginx with the certificate for verification:
     ```nginx
     proxy_ssl_verify on;
     proxy_ssl_trusted_certificate /path/to/backend-ca.crt;
     ```

5. **Testing and Reloading**:
   After updating the configuration, test and reload:
   ```bash
   sudo nginx -t
   sudo systemctl reload nginx
   ```

### Additional Notes:
- Replace `backend-service` with the actual hostname or IP of your backend service.
- If the backend uses a custom port (e.g., `4433`), specify it explicitly in `proxy_pass`, like `https://backend-service:4433`.
- Ensure the backend allows the proxy's requests and supports the expected `Authorization` header format.
