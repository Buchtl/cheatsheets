Ja — dafür ist bei Keycloak typischerweise der **Client-Credentials-Flow** gedacht. Deine Spring-Boot-Anwendung authentifiziert sich dabei **als technischer Client**, ohne Benutzer.

### Architektur

```text
Spring Boot App
      |
      |  Client ID + Client Secret
      |  OAuth2 Client Credentials
      v
   Keycloak
      |
      |  Access Token
      v
    Andere API
```

Die Spring-Boot-App holt sich bei Keycloak ein **Access Token** und schickt dieses anschließend an die andere API:

```http
Authorization: Bearer eyJhbGciOi...
```

## 1. Keycloak: Client für deine Spring-Boot-App anlegen

In Keycloak legst du z. B. einen Client an:

```text
Client ID: my-spring-app
Client authentication: ON
Authorization: OFF
Standard flow: OFF
Service accounts roles: ON
```

Wichtig ist insbesondere **Service Accounts Roles**. Damit bekommt dein Client eine technische Identität, die ohne Benutzer verwendet werden kann.

Das Secret findest du anschließend unter:

```text
Clients
  → my-spring-app
  → Credentials
  → Client Secret
```

Der Token-Endpunkt sieht typischerweise so aus:

```text
https://keycloak.example.com/realms/myrealm/protocol/openid-connect/token
```

## 2. Spring Boot als OAuth2 Client konfigurieren

Mit Spring Security kannst du den Client-Credentials-Flow direkt verwenden.

Dependency:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

`application.yml`:

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          my-api:
            provider: keycloak
            client-id: my-spring-app
            client-secret: ${KEYCLOAK_CLIENT_SECRET}
            authorization-grant-type: client_credentials
        provider:
          keycloak:
            token-uri: https://keycloak.example.com/realms/myrealm/protocol/openid-connect/token
```

Damit weiß Spring:

> Wenn ich für `my-api` ein Token brauche, verwende Client ID + Secret und den Client-Credentials-Grant.

### 3. Token automatisch verwenden

Mit Spring Security kannst du beispielsweise einen `RestClient` entsprechend konfigurieren:

```java
@Bean
RestClient restClient(OAuth2AuthorizedClientManager authorizedClientManager) {

    var interceptor = new OAuth2ClientHttpRequestInterceptor(
        authorizedClientManager
    );

    interceptor.setClientRegistrationIdResolver(
        request -> "my-api"
    );

    return RestClient.builder()
        .requestInterceptor(interceptor)
        .build();
}
```

Dann:

```java
@Service
public class MyService {

    private final RestClient restClient;

    public MyService(RestClient restClient) {
        this.restClient = restClient;
    }

    public String callApi() {
        return restClient
            .get()
            .uri("https://api.example.com/orders")
            .retrieve()
            .body(String.class);
    }
}
```

Spring kümmert sich dann darum, ein Access Token zu besorgen und es als

```http
Authorization: Bearer <token>
```

mitzusenden.

### 4. Was passiert dabei genau?

Beim ersten API-Aufruf ungefähr:

```text
Spring Boot
    |
    | POST /token
    | grant_type=client_credentials
    | client_id=my-spring-app
    | client_secret=******
    |
    v
Keycloak
    |
    | access_token
    v
Spring Boot
    |
    | GET /orders
    | Authorization: Bearer <access_token>
    v
Andere API
```

Die andere API muss anschließend **nicht** mit deinem Client Secret kommunizieren. Sie validiert lediglich das Access Token.

---

### 5. Rollen/Berechtigungen

Ein wichtiger Punkt ist noch die Frage:

**Woher weiß die andere API, was deine Spring-Boot-Anwendung darf?**

Dafür kannst du in Keycloak dem Service Account des Clients Rollen geben, z. B.:

```text
my-spring-app
└── Service Account
    ├── api-read
    └── api-write
```

Keycloak stellt diese Informationen dann typischerweise als Claims im JWT bereit.

Die andere API kann beispielsweise prüfen:

```text
iss  = https://keycloak.example.com/realms/myrealm
aud  = my-api
roles = ["api-read"]
```

Damit hast du eine saubere Trennung:

```text
Authentifizierung:
    "Wer ist der Aufrufer?"
    → my-spring-app

Autorisierung:
    "Was darf er?"
    → api-read / api-write
```

**Für eine reine Machine-to-Machine-Kommunikation ist Client Credentials normalerweise genau der passende OAuth2-Flow.** Ein Benutzer, Login-Formular, Authorization Code Flow oder Benutzer-Session ist dafür nicht erforderlich.

Wenn du mir sagst, **welche Spring-Boot-Version (z. B. 3.x) und welche Keycloak-Version** du verwendest und ob du `RestClient`, `WebClient` oder OpenFeign benutzt, kann ich dir die konkrete Konfiguration inkl. `SecurityConfig` zeigen.
