// Auth0 specific implementation
import { AuthProvider } from '../auth-provider.js';

export default class Auth0Provider extends AuthProvider {
  constructor(config) {
    super();
    if (!config || !config.domain || !config.clientId || !config.audience) {
      throw new Error('Missing required Auth0 configuration parameters');
    }
    
    this.config = {
      domain: config.domain,
      clientId: config.clientId,
      audience: config.audience,
      redirectUri: config.redirectUri || window.location.origin,
      scope: config.scope || 'openid profile email',
      responseType: 'code',
      cacheLocation: 'localstorage'
    };
    
    this.accessToken = null;
    this.userProfile = null;
    this.expiresAt = null;
    this.authenticated = false;
    
    // Initialize on creation
    this._initializeAuth();
  }
  
  _initializeAuth() {
    // Check if token exists in localStorage
    const storedAuth = localStorage.getItem('authClient');
    if (storedAuth) {
      try {
        const authData = JSON.parse(storedAuth);
        if (authData.expiresAt && new Date().getTime() < authData.expiresAt) {
          this.accessToken = authData.accessToken;
          this.userProfile = authData.userProfile;
          this.expiresAt = authData.expiresAt;
          this.authenticated = true;
        } else {
          // Token expired, clear storage
          this._clearStorage();
        }
      } catch (e) {
        this._clearStorage();
      }
    }
    
    // Check for authentication callback
    if (!this.authenticated && window.location.search.includes('code=')) {
      this._handleAuthCallback();
    }
  }
  
  _clearStorage() {
    localStorage.removeItem('authClient');
    this.accessToken = null;
    this.userProfile = null;
    this.expiresAt = null;
    this.authenticated = false;
  }
  
  async login() {
    try {
      // Create Auth0 URL
      const authUrl = `https://${this.config.domain}/authorize?` +
        `client_id=${this.config.clientId}&` +
        `redirect_uri=${encodeURIComponent(this.config.redirectUri)}&` +
        `response_type=${this.config.responseType}&` +
        `scope=${encodeURIComponent(this.config.scope)}&` +
        `audience=${encodeURIComponent(this.config.audience)}`;
      
      // Redirect to Auth0 login page
      window.location.assign(authUrl);
    } catch (error) {
      console.error('Login failed:', error);
      throw error;
    }
  }
  
  async _handleAuthCallback() {
    try {
      // Get authorization code from URL
      const urlParams = new URLSearchParams(window.location.search);
      const code = urlParams.get('code');
      
      if (!code) {
        throw new Error('No authorization code found in URL');
      }
      
      // Exchange code for token
      const tokenResponse = await fetch(`https://${this.config.domain}/oauth/token`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          grant_type: 'authorization_code',
          client_id: this.config.clientId,
          code,
          redirect_uri: this.config.redirectUri
        })
      });
      
      if (!tokenResponse.ok) {
        throw new Error('Failed to exchange code for token');
      }
      
      const tokenData = await tokenResponse.json();
      
      // Set auth data
      this.accessToken = tokenData.access_token;
      this.expiresAt = new Date().getTime() + (tokenData.expires_in * 1000);
      this.authenticated = true;
      
      // Get user profile
      await this.getUserProfile();
      
      // Save to localStorage
      this._saveAuthData();
      
      // Remove code from URL without reloading page
      const url = new URL(window.location.href);
      url.search = '';
      window.history.replaceState({}, document.title, url.toString());
      
      return true;
    } catch (error) {
      console.error('Authentication callback handling failed:', error);
      this._clearStorage();
      throw error;
    }
  }
  
  async getUserProfile() {
    if (!this.authenticated || !this.accessToken) {
      throw new Error('Not authenticated');
    }
    
    try {
      const userInfoResponse = await fetch(`https://${this.config.domain}/userinfo`, {
        headers: {
          Authorization: `Bearer ${this.accessToken}`
        }
      });
      
      if (!userInfoResponse.ok) {
        throw new Error('Failed to fetch user profile');
      }
      
      this.userProfile = await userInfoResponse.json();
      this._saveAuthData();
      
      return this.userProfile;
    } catch (error) {
      console.error('Failed to get user profile:', error);
      throw error;
    }
  }
  
  logout() {
    // Clear local storage and auth state
    this._clearStorage();
    
    // Redirect to Auth0 logout
    const logoutUrl = `https://${this.config.domain}/v2/logout?` +
      `client_id=${this.config.clientId}&` +
      `returnTo=${encodeURIComponent(this.config.redirectUri)}`;
    
    window.location.assign(logoutUrl);
  }
  
  isAuthenticated() {
    return this.authenticated && this.expiresAt && new Date().getTime() < this.expiresAt;
  }
  
  getAccessToken() {
    if (!this.isAuthenticated()) {
      throw new Error('Not authenticated');
    }
    return this.accessToken;
  }
  
  _saveAuthData() {
    localStorage.setItem('authClient', JSON.stringify({
      accessToken: this.accessToken,
      userProfile: this.userProfile,
      expiresAt: this.expiresAt
    }));
  }
}
