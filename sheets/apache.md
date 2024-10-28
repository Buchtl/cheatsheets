To configure Apache to redirect all HTTP traffic on port 80 to HTTPS (port 443) without specifying a `ServerName`, you can use the following configuration in your `httpd.conf`:

```apache
<VirtualHost *:80>
    # Redirect all HTTP traffic to HTTPS
    RewriteEngine On
    RewriteCond %{HTTPS} off
    RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
</VirtualHost>
```

### Explanation:
- **`<VirtualHost *:80>`**: This block listens for incoming HTTP requests on port 80.
- **`RewriteEngine On`**: Enables the rewrite engine.
- **`RewriteCond %{HTTPS} off`**: Checks if the incoming request is not using HTTPS.
- **`RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]`**: Redirects all requests to the HTTPS version of the URL, preserving the host and URI, with a `301 Moved Permanently` status.

### Notes:
- This configuration does not require you to set a `ServerName` and will work for any domain or hostname reaching this server on port 80.
- Make sure you have the `mod_rewrite` module enabled in Apache to use these rewrite rules:
  ```bash
  a2enmod rewrite
  ```
- After updating `httpd.conf`, restart the Apache server to apply the changes:
  ```bash
  sudo systemctl restart apache2
  ```

This setup ensures that any requests coming in on HTTP will be redirected to the HTTPS version of the same URL.