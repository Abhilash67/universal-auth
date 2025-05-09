// Okta specific implementation
import { AuthProvider } from '../auth-provider.js';

export default class OktaProvider extends AuthProvider {
  constructor(config) {
    super();
    if (!config || !config.orgUrl || !config.clientId) {
      throw new Error('Missing required Okta configuration parameters');
    }
    
    this.config = {
      orgUrl: config.orgUrl,
      clientId: config.clientId,
      redirectUri: config.redirectUri || window.location.origin,
      scopes: config.scopes || ['openid', 'profile', 'email']
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
      // Create Okta authorization URL
      const queryParams = new URLSearchParams({
        client_id: this.config.clientId,
        redirect_uri: this.config.redirectUri,
        response_type: 'code',
        scope: this.config.scopes.join(' '),
        state: this._generateRandomState(),
        nonce: this._generateNonce()
      });
      
      const authUrl = `${this.config.orgUrl}/oauth2/v1/authorize?${queryParams.toString()}`;
      
      // Redirect to Okta login page
      window.location.assign(authUrl);
    } catch (error) {
      console.error('Login failed:', error);
      throw error;
    }
  }
  
  _generateRandomState() {
    // Generate a random state value to prevent CSRF
    const array = new Uint32Array(8);
    window.crypto.getRandomValues(array);
    const state = Array.from(array, dec => ('0' + dec.toString(16)).substr(-2)).join('');
    localStorage.setItem('okta_state', state);
    return state;
  }
  
  _generateNonce() {
    const array = new Uint32Array(8);
    window.crypto.getRandomValues(array);
    return Array.from(array, dec => ('0' + dec.toString(16)).substr(-2)).join('');
  }
  
  async _handleAuthCallback() {
    try {
      // Get authorization code from URL
      const urlParams = new URLSearchParams(window.location.search);
      const code = urlParams.get('code');
      const state = urlParams.get('state');
      
      if (!code) {
        throw new Error('No authorization code found in URL');
      }
      
      // Verify state to prevent CSRF
      const savedState = localStorage.getItem('okta_state');
      if (state !== savedState) {
        throw new Error('Invalid state parameter');
      }
      
      // Exchange code for token
      const tokenResponse = await fetch(`${this.config.orgUrl}/oauth2/v1/token`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded'
        },
        body: new URLSearchParams({
          grant_type: 'authorization_code',
          client_id: this.config.clientId,
          redirect_uri: this.config.redirectUri,
          code
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
      localStorage.removeItem('okta_state');
      
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
      const userInfoResponse = await fetch(`${this.config.orgUrl}/oauth2/v1/userinfo`, {
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
    
    // Redirect to Okta logout
    const logoutUrl = `${this.config.orgUrl}/oauth2/v1/logout?` +
      `id_token_hint=${this.idToken}&` +
      `post_logout_redirect_uri=${encodeURIComponent(this.config.redirectUri)}`;
    
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
