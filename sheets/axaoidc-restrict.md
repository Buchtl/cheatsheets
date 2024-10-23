To restrict access to your React app based on roles using `@axa-fr/react-oidc`, you can utilize role-based access control (RBAC) by checking the roles present in the user's ID token or access token. Here’s how you can do it:

### 1. Retrieve User Roles
When a user logs in using `@axa-fr/react-oidc`, the library automatically handles the authentication process and stores the user information, including their ID token, in the OIDC context. Typically, roles are included in the ID token as claims (e.g., `roles`, `role`, `groups`, or a custom claim).

To retrieve the user roles, you can use the `useOidcUser` hook provided by `@axa-fr/react-oidc`.

```javascript
import React from 'react';
import { useOidcUser } from '@axa-fr/react-oidc';

const App = () => {
  const { oidcUser } = useOidcUser();

  // Assuming roles are in `oidcUser.profile.roles`
  const userRoles = oidcUser?.profile?.roles || [];

  return (
    <div>
      <h1>Welcome to the App</h1>
      <p>User Roles: {userRoles.join(', ')}</p>
    </div>
  );
};

export default App;
```

### 2. Create a Role-Based Route Component
You can create a higher-order component (HOC) or a custom route component that checks if the user has the required role(s). This component will wrap around the protected components or routes and ensure that only users with the necessary roles can access them.

Here’s an example of a `ProtectedRoute` component:

```javascript
import React from 'react';
import { Route, Redirect } from 'react-router-dom';
import { useOidcUser } from '@axa-fr/react-oidc';

const ProtectedRoute = ({ component: Component, requiredRoles, ...rest }) => {
  const { oidcUser, isLoading } = useOidcUser();

  if (isLoading) {
    return <div>Loading...</div>;
  }

  const userRoles = oidcUser?.profile?.roles || [];

  // Check if user has one of the required roles
  const hasAccess = requiredRoles.some((role) => userRoles.includes(role));

  return (
    <Route
      {...rest}
      render={(props) =>
        hasAccess ? <Component {...props} /> : <Redirect to="/unauthorized" />
      }
    />
  );
};

export default ProtectedRoute;
```

### 3. Use the `ProtectedRoute` in Your App
Now, use the `ProtectedRoute` component for any route you want to protect based on roles:

```javascript
import React from 'react';
import { BrowserRouter as Router, Switch, Route } from 'react-router-dom';
import ProtectedRoute from './ProtectedRoute';
import HomePage from './HomePage';
import AdminPage from './AdminPage';
import UnauthorizedPage from './UnauthorizedPage';

const App = () => {
  return (
    <Router>
      <Switch>
        <Route exact path="/" component={HomePage} />
        <ProtectedRoute 
          path="/admin" 
          component={AdminPage} 
          requiredRoles={['admin']} 
        />
        <Route path="/unauthorized" component={UnauthorizedPage} />
      </Switch>
    </Router>
  );
};

export default App;
```

In this example:
- If a user tries to access `/admin` and has the `admin` role, they will be granted access to the `AdminPage`.
- If they don't have the `admin` role, they will be redirected to the `UnauthorizedPage`.

### 4. Customize Role Checking Logic
Depending on how roles are structured in your ID token, you may need to adjust the way you extract roles from the `oidcUser` object. For example, if roles are stored under a different claim like `role` or `groups`, make sure to modify the retrieval logic:

```javascript
const userRoles = oidcUser?.profile?.role || []; // Example if roles are in `role`
```

With these steps, you can effectively restrict access to parts of your app based on user roles using `@axa-fr/react-oidc`.