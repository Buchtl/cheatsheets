No, using OAuth2 Proxy does not automatically add authorization tokens to your GraphQL requests sent from your React app using Apollo. Instead, OAuth2 Proxy primarily handles the authentication at the **reverse proxy level**, which means it manages access to your backend services but does not modify the behavior of your front-end app or automatically include authentication headers in its API requests.

### How OAuth2 Proxy Works with Your React App:
- **OAuth2 Proxy as a Reverse Proxy**: It sits in front of your application and checks if the user is authenticated before granting access to your React app and other services (like your GraphQL server).
- **Session Management**: It manages authentication using **cookies** (by default). After a user is authenticated through the OAuth2 provider, OAuth2 Proxy sets a secure cookie that indicates that the user is logged in.

### What Happens with Your React App and Apollo:
- If your React app uses **Apollo Client** to send GraphQL queries to a backend service (e.g., an Express server running GraphQL):
  - **GraphQL Server Behind OAuth2 Proxy**: The proxy can protect access to the GraphQL server by requiring authentication before a request reaches the server. This ensures that only authenticated users can access the GraphQL API.
  - **React App Requests**: The Apollo Client in your React app won’t automatically have the required authorization header or tokens unless you explicitly configure it to do so.

### How to Add Authentication to Apollo Client:
If your GraphQL server expects an `Authorization` header with a Bearer token, you’ll still need to configure your Apollo Client to include it. Typically, you do this by using an **Apollo Link** for authentication:

#### Example: Adding an Auth Token in Apollo Client
Here's an example of how you might set up an Apollo Client to include a token:

```javascript
import { ApolloClient, InMemoryCache, createHttpLink } from '@apollo/client';
import { setContext } from '@apollo/client/link/context';

// Create an HTTP link to your GraphQL server
const httpLink = createHttpLink({
  uri: 'http://localhost:3000/graphql', // Replace with your GraphQL endpoint
});

// Set up the auth link to include the token in the headers
const authLink = setContext((_, { headers }) => {
  // Get the authentication token from cookies or local storage
  const token = localStorage.getItem('auth_token');

  // Return the headers with the token if it exists
  return {
    headers: {
      ...headers,
      authorization: token ? `Bearer ${token}` : "",
    }
  };
});

// Combine the auth link and http link
const client = new ApolloClient({
  link: authLink.concat(httpLink),
  cache: new InMemoryCache(),
});

export default client;
```

### How to Use Cookies from OAuth2 Proxy:
- If OAuth2 Proxy is using cookies for authentication, you might not need to manually include the `Authorization` header in your GraphQL requests **if**:
  - The cookies are automatically sent with each request due to the **same-origin policy**.
  - Your GraphQL server can read the cookie and verify the session.

However, this is more typical for session-based authentication and less common when using tokens like JWTs, which are often included in the `Authorization` header.

### Summary:
1. **OAuth2 Proxy** manages user authentication and access to services at the reverse proxy level.
2. **Apollo Client** in the React app does not automatically send OAuth tokens unless you explicitly configure it to do so.
3. You may need to handle the **retrieval** and **inclusion** of tokens in GraphQL requests using methods like Apollo Link.
4. If OAuth2 Proxy uses **cookies** for authentication, these cookies might be sent automatically, depending on your setup, but that depends on how your GraphQL server handles authentication.

For a more seamless experience with tokens, you can configure your React app to read the tokens from a secure storage mechanism (like `localStorage` or `sessionStorage`) and set the header appropriately for each GraphQL request.


# Combining apollo and axa-fr/react-oidc

To obtain the OIDC authentication token using the `@axa-fr/react-oidc` framework and use it for your Apollo GraphQL requests, you need to access the OIDC token from the OIDC context provided by the framework. Then, you can use this token in the `Authorization` header when making GraphQL requests.

Here's a step-by-step guide:

### Step 1: Set Up `@axa-fr/react-oidc` Provider
Ensure that your React app is wrapped with the `OidcProvider` from the `@axa-fr/react-oidc` library. This makes the OIDC context available throughout your app.

```javascript
import { OidcProvider } from '@axa-fr/react-oidc';
import { BrowserRouter as Router } from 'react-router-dom';

const oidcConfiguration = {
  client_id: '<your-client-id>',
  redirect_uri: 'http://localhost:3000/auth/callback',
  scope: 'openid profile email',
  authority: 'https://your-oidc-provider',
  response_type: 'code',
};

function App() {
  return (
    <OidcProvider configuration={oidcConfiguration}>
      <Router>
        {/* Your routes and components */}
      </Router>
    </OidcProvider>
  );
}
```

Make sure you replace `client_id` and `authority` with the appropriate values for your OIDC provider.

### Step 2: Access the OIDC Token
You can use the `useOidcAccessToken` hook provided by `@axa-fr/react-oidc` to access the OIDC access token. Here’s how you can use this in a custom Apollo Client setup:

```javascript
import { ApolloClient, InMemoryCache, createHttpLink } from '@apollo/client';
import { setContext } from '@apollo/client/link/context';
import { useOidcAccessToken } from '@axa-fr/react-oidc';

const httpLink = createHttpLink({
  uri: 'http://localhost:3000/graphql', // Replace with your GraphQL endpoint
});

const useAuthenticatedApolloClient = () => {
  const { accessToken } = useOidcAccessToken();

  const authLink = setContext((_, { headers }) => {
    // Include the OIDC token in the headers
    return {
      headers: {
        ...headers,
        authorization: accessToken ? `Bearer ${accessToken}` : "",
      },
    };
  });

  const client = new ApolloClient({
    link: authLink.concat(httpLink),
    cache: new InMemoryCache(),
  });

  return client;
};

export default useAuthenticatedApolloClient;
```

### Step 3: Use the Apollo Client in Your App
Now you can use the `useAuthenticatedApolloClient` hook to get an instance of the Apollo Client that includes the OIDC access token in the headers:

```javascript
import React from 'react';
import { ApolloProvider } from '@apollo/client';
import useAuthenticatedApolloClient from './useAuthenticatedApolloClient';

function MyApp() {
  const client = useAuthenticatedApolloClient();

  return (
    <ApolloProvider client={client}>
      {/* Your components that use Apollo Client */}
    </ApolloProvider>
  );
}

export default MyApp;
```

### Summary:
- Use `@axa-fr/react-oidc`’s `useOidcAccessToken` to get the OIDC access token.
- Create an Apollo Client instance that adds this token to the `Authorization` header using an Apollo `authLink`.
- Use this customized Apollo Client in your React application to make authenticated GraphQL requests.

This approach ensures that your GraphQL queries include the OIDC access token for authorization, allowing your server to verify the user's identity and permissions.

# Oauth2 and Apollo Sesseion Timeout

You're experiencing a session timeout issue where Apollo doesn't correctly handle the expired authentication state. Since your React app accesses the API through an OAuth2 proxy and the API itself is secured by Keycloak (OIDC), when the session expires, the proxy likely returns a **401 Unauthorized** or a **302 Redirect** response instead of the expected API response, which Apollo interprets as a **network error**.

### ✅ **Steps to Fix the Issue**

1. **Handle Authentication Errors in Apollo**
   - Use an Apollo Link to intercept errors and refresh the session or redirect to login when necessary.

2. **Detect Session Expiration and Refresh Tokens**
   - Since you are using OAuth2-proxy, token refresh is typically handled by the proxy. Ensure that it properly refreshes the session using a refresh token.
   - If refresh is not working, you might need to handle it manually by redirecting to the login page.

---

## **Solution: Implement an Apollo Link Error Handler**
Modify your Apollo client configuration to handle `401` responses properly.

### **1️⃣ Use `apollo-link-error` to Detect Unauthorized Errors**
Modify your Apollo client setup:

```javascript
import { ApolloClient, InMemoryCache, HttpLink } from "@apollo/client";
import { onError } from "@apollo/client/link/error";
import { setContext } from "@apollo/client/link/context";

// API Endpoint (proxied through OAuth2-proxy)
const API_URL = "/graphql"; // Adjust based on your setup

// Handle authentication errors
const errorLink = onError(({ graphQLErrors, networkError, response }) => {
  if (networkError) {
    console.error("Apollo Network Error:", networkError);

    if (networkError.statusCode === 401) {
      console.warn("Session expired. Redirecting to login...");
      window.location.href = "/oauth2/sign_in"; // Adjust as needed
    }
  }
});

// Apollo HTTP Link (calls the GraphQL API)
const httpLink = new HttpLink({ uri: API_URL, credentials: "include" });

// Apollo Client
const client = new ApolloClient({
  link: errorLink.concat(httpLink),
  cache: new InMemoryCache(),
});

export default client;
```

---

### **2️⃣ Ensure OAuth2 Proxy Handles Token Refresh**
If your OAuth2 proxy doesn't refresh tokens automatically, you might need to configure `cookie_refresh` and `cookie_expire`.

**Example OAuth2 Proxy Config:**
```yaml
cookie_refresh: "5m"  # Refresh session every 5 minutes
cookie_expire: "1h"    # Session expires after 1 hour
```

---

### **3️⃣ Alternative: Use Silent Authentication (If Supported)**
If your OAuth2 setup allows refreshing sessions silently, you could try triggering an API call to refresh the session before it expires.

---

## **Expected Behavior After Fix**
- If the session expires, Apollo intercepts the `401` error and redirects to login.
- If OAuth2-proxy refreshes tokens correctly, the session will be maintained.

---

Try this setup and let me know if you need adjustments! 🚀

# Setup with access token 2m
Yes, you **can** set the **Access Token Lifespan** to **2 minutes** while keeping the **SSO session** and **Refresh Token lifespan** at **8 hours** in Keycloak.

However, doing this means that your **Access Token** (used in API requests) will expire **every 2 minutes**, requiring frequent token refreshes. This is fine **if OAuth2 Proxy handles refresh tokens properly**.

---

## **🔧 Steps to Set This Up in Keycloak**
1. **Log in to Keycloak Admin Console**
   - Navigate to `http://<your-keycloak-domain>/admin/`

2. **Go to Realm Settings → Tokens Tab**
   - **SSO Session Idle Timeout** → `28800` (8 hours)
   - **SSO Session Max Lifespan** → `28800` (8 hours)
   - **Refresh Token Max Lifespan** → `28800` (8 hours)
   - **Access Token Lifespan** → `120` (2 minutes)

3. **Save Changes** ✅

---

## **🔄 Handling Frequent Token Expiration**
Since the **Access Token expires every 2 minutes**, OAuth2 Proxy must automatically refresh it.

### **1️⃣ Ensure OAuth2 Proxy Refreshes Tokens**
Modify `oauth2-proxy.yaml` to allow refresh before token expiration:
```yaml
cookie_refresh: "1m30s"  # Refresh the session every 1 minute 30 seconds
cookie_expire: "8h"      # Keep session alive for 8 hours
```

---

### **2️⃣ Update Apollo Client to Handle Expired Tokens**
Since the Access Token expires every 2 minutes, API requests will start failing with `401 Unauthorized`. You need **Apollo Link** to detect token expiration and force OAuth2 Proxy to refresh it.

#### **Modify `apolloClient.js`**
```javascript
import { ApolloClient, InMemoryCache, HttpLink } from "@apollo/client";
import { onError } from "@apollo/client/link/error";

// API Endpoint
const API_URL = "/graphql";

// Handle Authentication Errors
const errorLink = onError(({ networkError }) => {
  if (networkError && networkError.statusCode === 401) {
    console.warn("Access token expired. Redirecting to refresh...");
    window.location.href = "/oauth2/sign_in"; // OAuth2 Proxy refresh
  }
});

// Apollo HTTP Link
const httpLink = new HttpLink({
  uri: API_URL,
  credentials: "include", // Send cookies with requests
});

// Apollo Client
const client = new ApolloClient({
  link: errorLink.concat(httpLink),
  cache: new InMemoryCache(),
});

export default client;
```
✅ This ensures that expired tokens **trigger a silent refresh**.

---

## **✅ Final Behavior**
- 🔹 **Access Token expires every 2 minutes**.
- 🔹 OAuth2 Proxy automatically **refreshes** the session **before expiration**.
- 🔹 Apollo Client **detects expired tokens** and triggers a refresh.
- 🔹 The **session remains active for 8 hours** unless the user logs out.

---

### **⚠️ Potential Issues**
- If OAuth2 Proxy **does not refresh the session correctly**, users may get logged out every 2 minutes.
- If your API requires **manual token renewal**, consider setting the **Access Token Lifespan** to a slightly higher value (e.g., **5–10 minutes**).

Would you like me to help debug token refresh behavior? 🚀

Great question! The **Client settings** in Keycloak **do have a purpose**, but they **only apply in specific scenarios**. Here's a breakdown of when and how **Client settings** work, and why they often seem overridden by **Realm settings**.

---

## **🔍 How Do Client Settings Work?**
By default, **Realm settings take precedence** over Client settings **unless explicitly overridden**.

However, **Client session settings** allow you to:
1. **Override Realm defaults for specific clients** (if enabled).
2. **Enforce stricter timeouts** for some clients while keeping others at realm defaults.
3. **Ensure different apps have different session lifespans** while still sharing the same Keycloak instance.

---

## **🔧 When Do Client Settings Apply?**
Client settings **only take effect if**:
1. **The realm allows overriding defaults**
   - Some versions of Keycloak have an **"Override Realm Defaults"** toggle in **Client settings**.
   - If this is not enabled, the **Realm settings always win**.

2. **The client uses its own session settings**
   - If the client is configured **to allow a shorter or longer session**, it will override the Realm.
   - Example: You want **your React app's session to last 8 hours**, but another client (e.g., mobile app) should **expire after 1 hour**.

---

## **🔧 How to Make Client Settings Work**
If you want your **Client settings to actually take effect**, do the following:

### **1️⃣ Enable Client-Specific Session Settings**
1. **Log in to Keycloak Admin Console** → `http://<your-keycloak-domain>/admin/`
2. **Go to** `Clients` → Select your client (e.g., `oauth2-proxy` or your React App client)
3. **Go to the "Advanced Settings" (or "Sessions" tab, depending on Keycloak version)**
4. **Check for an option to override Realm defaults**:
   - **If available:** Toggle **ON** (e.g., `Override Realm Defaults: Yes`)
   - **If not available:** The realm settings will always apply.
5. **Set session values for your client:**
   - **Client Session Idle Timeout** → `28800` (8 hours)
   - **Client Session Max Lifespan** → `28800` (8 hours)
   - **Access Token Lifespan** → `120` (2 minutes)
6. **Save Changes** ✅

### **2️⃣ Restart Keycloak (If Necessary)**
Some Keycloak versions require a restart for session settings to take effect:
```bash
docker restart keycloak
# or if running Keycloak standalone
systemctl restart keycloak
```

---

## **🔎 When Should You Use Realm vs. Client Settings?**
| **Scenario** | **Use Realm Settings?** | **Use Client Settings?** |
|-------------|----------------|----------------|
| **All clients should have the same session timeout** | ✅ Yes | ❌ No |
| **Different apps need different session lifespans** (e.g., 8h for Web, 1h for Mobile) | ❌ No | ✅ Yes |
| **You want Keycloak to handle all session policies centrally** | ✅ Yes | ❌ No |
| **A specific client should have longer/shorter sessions than the default** | ❌ No | ✅ Yes (but ensure override is enabled) |

---

## **✅ Summary**
- **Realm settings** are **global** and apply by default.
- **Client settings** only work if **Keycloak allows overriding** realm defaults.
- If your session expires after **30 minutes** despite setting **8 hours in the client**, it's because the **Realm setting still enforces 30 minutes**.
- To fix it, either **update the Realm settings** or **enable client-specific session overrides**.

Would you like help debugging whether your client settings are actually being applied? 🚀

To log out a Keycloak/OIDC session via `oauth2-proxy` in a React (Vite) app, you need to ensure that:

1. **OAuth2 Proxy is configured** to handle logout requests properly.
2. **Your React app** correctly initiates the logout process by redirecting the user to the appropriate logout endpoint.

---

## **1. OAuth2 Proxy Logout Configuration**
OAuth2 Proxy supports logout via the `/oauth2/sign_out` endpoint. However, to fully log out from Keycloak, you need to configure it properly.

### **Key OAuth2 Proxy Configurations** (Example)
```ini
provider=oidc
oidc_issuer_url=https://keycloak.example.com/realms/myrealm
client_id=my-client-id
client_secret=my-client-secret
cookie_secret=your-random-secret
cookie_secure=true
cookie_domains=.example.com
set_xauthrequest=true
```
Ensure `set_xauthrequest=true` is enabled so that the authentication session is properly handled.

### **Modify Logout URL Handling** (Optional)
If Keycloak doesn't automatically log the user out, you may need to specify a `post_logout_redirect_uri`. OAuth2 Proxy does not provide this option directly, so you may need to manually configure Keycloak’s logout URL.

---

## **2. React (Vite) App - Logout Implementation**
### **Logout Flow**
1. Call `/oauth2/sign_out` to clear OAuth2 Proxy session.
2. Redirect the user to Keycloak's logout endpoint to clear their OIDC session.
3. Redirect the user back to the app after logout.

### **Example Logout Function**
```tsx
const logout = () => {
  const oauthProxyLogoutUrl = "/oauth2/sign_out";
  const keycloakLogoutUrl = `https://keycloak.example.com/realms/myrealm/protocol/openid-connect/logout?redirect_uri=${encodeURIComponent(window.location.origin)}`;

  // Step 1: Logout from OAuth2 Proxy
  fetch(oauthProxyLogoutUrl, { credentials: "include" })
    .then(() => {
      // Step 2: Redirect to Keycloak logout endpoint
      window.location.href = keycloakLogoutUrl;
    })
    .catch((err) => {
      console.error("Logout failed", err);
      window.location.href = keycloakLogoutUrl; // Fallback redirect
    });
};
```

### **Call the Function on Logout Button**
```tsx
<button onClick={logout}>Logout</button>
```

---

## **3. Redirect Back to Your App**
Ensure Keycloak allows redirection after logout by setting up the valid redirect URI.
- Go to **Keycloak Admin Panel** → Clients → Your Client → **Advanced Settings**
- Set `Valid post logout redirect URIs` to match your app's URL (e.g., `https://yourapp.example.com`).

---

### **Additional Notes**
- If `oauth2-proxy` is behind an **ingress controller**, make sure `/oauth2/sign_out` is properly exposed.
- Test in an **incognito browser** to ensure the logout fully clears the session.

Would you like help debugging any specific issues? 🚀
