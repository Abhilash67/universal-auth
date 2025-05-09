# Auth0 Simple Client

A lightweight, framework-agnostic Auth0 client that handles login, logout, and user profile functionality without any UI components. This package is designed to work with Angular, React, .NET, and any JavaScript application.

## Installation

```bash
npm install auth0-simple-client
```

## Features

- Simple Auth0 authentication flow
- Login and logout functionality
- User profile retrieval
- Token management
- Framework-agnostic (works with Angular, React, .NET, etc.)
- No UI dependencies

## Usage

### Initializing the Client

```javascript
// Import the package
import Auth0SimpleClient from 'auth0-simple-client';
// Or using CommonJS
// const Auth0SimpleClient = require('auth0-simple-client');

// Initialize with your Auth0 configuration
const auth0Client = new Auth0SimpleClient({
  domain: 'your-auth0-domain.auth0.com',
  clientId: 'your-auth0-client-id',
  audience: 'https://your-audience-identifier/',
  redirectUri: window.location.origin, // Optional, defaults to window.location.origin
  scope: 'openid profile email' // Optional, defaults to 'openid profile email'
});
```

### Login

```javascript
// Redirect to Auth0 login page
auth0Client.login();
```

### Checking Authentication Status

```javascript
// Check if user is authenticated
const isAuthenticated = auth0Client.isAuthenticated();
console.log('Is authenticated:', isAuthenticated);
```

### Getting User Profile

```javascript
// Get user profile (returns a promise)
auth0Client.getUserProfile()
  .then(profile => {
    console.log('User profile:', profile);
  })
  .catch(error => {
    console.error('Error getting profile:', error);
  });

// Or using async/await
async function getUserInfo() {
  try {
    const profile = await auth0Client.getUserProfile();
    console.log('User profile:', profile);
  } catch (error) {
    console.error('Error getting profile:', error);
  }
}
```

### Getting Access Token

```javascript
// Get the access token (throws error if not authenticated)
try {
  const accessToken = auth0Client.getAccessToken();
  console.log('Access token:', accessToken);
} catch (error) {
  console.error('Error getting access token:', error);
}
```

### Logout

```javascript
// Logout and redirect to Auth0 logout endpoint
auth0Client.logout();
```

## Integration Examples

### React Integration

```javascript
import React, { useEffect, useState } from 'react';
import Auth0SimpleClient from 'auth0-simple-client';

const auth0Client = new Auth0SimpleClient({
  domain: 'your-auth0-domain.auth0.com',
  clientId: 'your-auth0-client-id',
  audience: 'https://your-audience-identifier/'
});

function App() {
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [userProfile, setUserProfile] = useState(null);

  useEffect(() => {
    // Check if user is authenticated
    setIsAuthenticated(auth0Client.isAuthenticated());
    
    // If authenticated, get user profile
    if (auth0Client.isAuthenticated()) {
      auth0Client.getUserProfile()
        .then(profile => setUserProfile(profile))
        .catch(error => console.error(error));
    }
  }, []);

  const handleLogin = () => {
    auth0Client.login();
  };

  const handleLogout = () => {
    auth0Client.logout();
  };

  return (
    <div>
      {isAuthenticated ? (
        <div>
          <h2>Welcome, {userProfile?.name}</h2>
          <button onClick={handleLogout}>Logout</button>
        </div>
      ) : (
        <button onClick={handleLogin}>Login</button>
      )}
    </div>
  );
}

export default App;
```

### Angular Integration

```typescript
// auth.service.ts
import { Injectable } from '@angular/core';
import Auth0SimpleClient from 'auth0-simple-client';
import { BehaviorSubject } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class AuthService {
  private auth0Client = new Auth0SimpleClient({
    domain: 'your-auth0-domain.auth0.com',
    clientId: 'your-auth0-client-id',
    audience: 'https://your-audience-identifier/'
  });

  private isAuthenticatedSubject = new BehaviorSubject<boolean>(this.auth0Client.isAuthenticated());
  public isAuthenticated$ = this.isAuthenticatedSubject.asObservable();

  private userProfileSubject = new BehaviorSubject<any>(null);
  public userProfile$ = this.userProfileSubject.asObservable();

  constructor() {
    this.checkAuth();
  }

  private async checkAuth(): Promise<void> {
    const isAuthenticated = this.auth0Client.isAuthenticated();
    this.isAuthenticatedSubject.next(isAuthenticated);

    if (isAuthenticated) {
      try {
        const profile = await this.auth0Client.getUserProfile();
        this.userProfileSubject.next(profile);
      } catch (error) {
        console.error('Error getting user profile', error);
      }
    }
  }

  public login(): void {
    this.auth0Client.login();
  }

  public logout(): void {
    this.auth0Client.logout();
  }

  public getAccessToken(): string {
    return this.auth0Client.getAccessToken();
  }
}
```

### .NET Integration (using JavaScript)

```csharp
// In your .cshtml or .razor file
<script src="~/lib/auth0-simple-client/dist/auth0-simple-client.js"></script>
<script>
    // Initialize the client
    var auth0Client = new Auth0SimpleClient({
        domain: 'your-auth0-domain.auth0.com',
        clientId: 'your-auth0-client-id',
        audience: 'https://your-audience-identifier/',
        redirectUri: window.location.origin
    });

    // Check if user is authenticated
    var isAuthenticated = auth0Client.isAuthenticated();
    
    function login() {
        auth0Client.login();
    }
    
    function logout() {
        auth0Client.logout();
    }
    
    // If authenticated, get user profile
    if (isAuthenticated) {
        auth0Client.getUserProfile()
            .then(function(profile) {
                console.log('User profile:', profile);
                document.getElementById('userName').innerText = profile.name;
            })
            .catch(function(error) {
                console.error('Error getting profile:', error);
            });
    }
</script>
```

## Advanced Usage

### Handling Authentication Tokens for API Calls

```javascript
// Example of making an authenticated API call
async function fetchProtectedResource() {
  try {
    // Get the access token
    const accessToken = auth0Client.getAccessToken();
    
    // Use the token for an API call
    const response = await fetch('https://your-api.com/protected-resource', {
      headers: {
        Authorization: `Bearer ${accessToken}`
      }
    });
    
    return await response.json();
  } catch (error) {
    console.error('API call failed:', error);
    // Handle the error (e.g., redirect to login if token is invalid)
    if (!auth0Client.isAuthenticated()) {
      auth0Client.login();
    }
    throw error;
  }
}
```

## Configuration Options

| Option | Description | Required | Default |
|--------|-------------|----------|---------|
| domain | Your Auth0 domain | Yes | - |
| clientId | Your Auth0 client ID | Yes | - |
| audience | The unique identifier for your API | Yes | - |
| redirectUri | The URL Auth0 should redirect to after authentication | No | `window.location.origin` |
| scope | The scopes to request during authentication | No | `openid profile email` |

## License

MIT
