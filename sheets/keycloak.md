# Keycloak 14
Here's a `docker-compose.yml` configuration that sets up a Keycloak instance based on `bitnami/keycloak:14.0.0-debian-10-r5` and imports a `realm-export.json` file during startup. 

### Steps:

1. Place your `realm-export.json` file in the same directory as the `docker-compose.yml`.
2. Modify the environment variables like `KEYCLOAK_USER` and `KEYCLOAK_PASSWORD` according to your requirements.

```yaml
version: '3.8'

services:
  keycloak:
    image: bitnami/keycloak:14.0.0-debian-10-r5
    container_name: keycloak
    environment:
      - KEYCLOAK_USERNAME=admin         # Admin username
      - KEYCLOAK_PASSWORD=admin         # Admin password
      - KEYCLOAK_IMPORT=/opt/bitnami/keycloak/data/import/realm-export.json
      - KEYCLOAK_LOGLEVEL=DEBUG         # Optional: Adjust log level
      - KEYCLOAK_HTTP_PORT=8080         # Optional: Adjust port
      - JAVA_OPTS=-Dkeycloak.profile.feature.upload_scripts=enabled
    volumes:
      - ./realm-export.json:/opt/bitnami/keycloak/data/import/realm-export.json:ro
    ports:
      - "8080:8080"
    restart: unless-stopped
    networks:
      - keycloak-net

networks:
  keycloak-net:
    driver: overlay # Use 'bridge' if running standalone, 'overlay' for Swarm
```

### Notes:
1. **Realm Import Path:** 
   - The `KEYCLOAK_IMPORT` environment variable specifies the location of the `realm-export.json` file inside the container. The file must be in the `/opt/bitnami/keycloak/data/import/` directory.
   - The `:ro` in the volume ensures the file is mounted read-only.

2. **Swarm Mode:** 
   - If you're deploying this in a Swarm cluster, ensure that the `keycloak-net` overlay network exists, or Docker will create it automatically.

3. **Realm Export JSON:**
   - The `realm-export.json` file must be in the correct format for the Keycloak version you're using.

4. **Startup Logs:**
   - Check the container logs to verify that the realm import is successful during startup:
     ```bash
     docker logs keycloak
     ```
