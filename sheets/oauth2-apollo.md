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
