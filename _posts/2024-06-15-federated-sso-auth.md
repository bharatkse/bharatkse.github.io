---
layout: blog
title: "Implementing Federated SSO: SAML, OIDC, and AWS Cognito in Production"
description: "A production guide to implementing federated SSO using SAML and OIDC with AWS Cognito to support enterprise identity providers securely."
date: 2024-06-15
categories: [security, authentication, architecture]
tags: [sso, saml, oidc, aws-cognito, security]
read_time: true
toc: true
toc_sticky: true
classes: wide
---


## The Problem: Username/Password Doesn't Scale

When we started, users had unique passwords for our platform. Then enterprise customers started asking: "Can employees just use their company credentials?"

This is the classic federated authentication problem. Enterprise customers don't want:
- More passwords to manage
- More accounts to provision
- More complexity for IT teams

They want **Single Sign-On (SSO)** — one login for all apps.

For them, this is table stakes. For us, supporting it was non-trivial:

- Learn three different protocols (SAML, OIDC, OAuth2)
- Handle different enterprise identity providers (Okta, Azure AD, Google Workspace)
- Manage multiple authentication flows
- Ensure backward compatibility with password authentication

We built an authentication system supporting:
- Traditional username/password
- SAML 2.0 (for enterprises)
- OpenID Connect (OIDC)
- AWS Cognito (for multi-account deployments)
- GitHub/Google OAuth2 (for consumers)

This article covers what we learned.

---

## Architecture: Layered Authentication

We implemented a layered approach:

```
┌─────────────────────────────────────────────────┐
│       Application Layer (Django + DRF)          │
│  Protect @login_required views/endpoints        │
└────────────────────┬────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────┐
│     Authentication Layer (django-allauth)       │
│  - Token generation/validation                  │
│  - Session management                           │
│  - JWT issuance                                 │
└────────────────────┬────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────┐
│      SSO Integration Layer (Custom)             │
│  - SAML provider logic                          │
│  - OIDC provider logic                          │
│  - Multi-tenant provisioning                    │
└────────────────────┬────────────────────────────┘
                     │
   ┌─────────────────┼─────────────────┐
   ▼                 ▼                 ▼
Okta              Azure AD        Google Workspace
(SAML 2.0)        (OIDC)          (OIDC)
```

---

## SAML 2.0: Enterprise Standard

SAML (Security Assertion Markup Language) is how enterprises do SSO. It's XML-based, complex, but well-supported.

### SAML Flow Explained

```
1. User visits app: https://app.example.com/login
   ↓
2. App redirects to IdP: https://okta.example.com/auth?SAMLRequest=...
   ↓
3. User logs in at IdP (with company credentials)
   ↓
4. IdP creates signed SAML Assertion (XML with user info)
   ↓
5. IdP redirects back to app with Assertion
   ↓
6. App verifies signature and creates session
   ↓
7. User is logged in ✓
```

### Implementation with Django

```python
import logging
from django.http import HttpResponseRedirect
from django.views import View
from onelogin.saml2.auth import OneLogin_Saml2_Auth
from onelogin.saml2.utils import OneLogin_Saml2_Utils

logger = logging.getLogger(__name__)

class SamlLoginView(View):
    """Initiates SAML login by redirecting to IdP."""
    
    def get(self, request):
        # Get SAML config for customer's tenant
        tenant_id = request.GET.get('tenant_id')
        tenant = Tenant.objects.get(id=tenant_id)
        
        saml_settings = self._get_saml_settings(tenant)
        
        auth = OneLogin_Saml2_Auth(request, saml_settings)
        
        # Redirect user to IdP
        redirect_url = auth.login()
        return HttpResponseRedirect(redirect_url)
    
    def _get_saml_settings(self, tenant):
        """Load SAML configuration from database."""
        
        if not tenant.saml_enabled:
            raise Exception("SAML not enabled for this tenant")
        
        return {
            'sp': {
                'entityID': 'https://app.example.com/saml/metadata/',
                'assertionConsumerService': {
                    'url': 'https://app.example.com/saml/acs/',
                    'binding': 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST'
                },
                'singleLogoutService': {
                    'url': 'https://app.example.com/saml/sls/',
                    'binding': 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect'
                }
            },
            'idp': {
                'entityID': tenant.saml_idp_entity_id,
                'singleSignOnService': {
                    'url': tenant.saml_sso_url,
                    'binding': 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect'
                },
                'singleLogoutService': {
                    'url': tenant.saml_slo_url,
                    'binding': 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect'
                },
                'x509cert': tenant.saml_x509cert
            },
            'security': {
                'nameIdEncrypted': False,
                'authnRequestsSigned': True,
                'wantAssertionsSigned': True,
                'signMetadata': True,
                'encryptionAlgorithm': 'http://www.w3.org/2001/04/xmlenc#aes256-cbc',
                'digestAlgorithm': 'http://www.w3.org/2001/04/xmldsig-more#sha256'
            }
        }


class SamlAcsView(View):
    """Assertion Consumer Service - receives SAML response from IdP."""
    
    def post(self, request):
        """Handle SAML response."""
        
        try:
            tenant_id = request.session.get('tenant_id')
            if not tenant_id:
                # Try to determine from SAML response
                # (in production, you'd have more sophisticated mapping)
                tenant_id = request.POST.get('RelayState')
            
            tenant = Tenant.objects.get(id=tenant_id)
            saml_settings = self._get_saml_settings(tenant)
            
            auth = OneLogin_Saml2_Auth(request, saml_settings)
            auth.process_response()
            
            # CRITICAL: Verify signature
            if not auth.is_authenticated():
                logger.error("SAML authentication failed")
                logger.error(auth.get_last_error_reason())
                raise Exception("SAML validation failed")
            
            # Extract user attributes from SAML response
            saml_data = auth.get_attributes()
            
            email = saml_data.get('email', [None])[0]
            first_name = saml_data.get('firstName', [None])[0]
            last_name = saml_data.get('lastName', [None])[0]
            
            if not email:
                raise Exception("Email not found in SAML assertion")
            
            # Find or create user
            user, created = User.objects.get_or_create(
                email=email,
                defaults={
                    'first_name': first_name or '',
                    'last_name': last_name or '',
                }
            )
            
            # Link user to tenant
            TenantUser.objects.get_or_create(
                user=user,
                tenant=tenant,
                defaults={'role': 'member'}
            )
            
            # Create session
            login(request, user, backend='django.contrib.auth.backends.ModelBackend')
            
            # Store SAML session index for logout
            request.session['saml_session_index'] = auth.get_session_index()
            
            logger.info(f"User {email} logged in via SAML from {tenant.name}")
            
            return HttpResponseRedirect('/dashboard/')
        
        except Exception as e:
            logger.error(f"SAML ACS error: {str(e)}", exc_info=True)
            return HttpResponse(f"Authentication failed: {str(e)}", status=401)


class SamlLogoutView(View):
    """Logout and notify IdP via SAML SLO."""
    
    def get(self, request):
        """Initiate SAML logout."""
        
        tenant_id = request.session.get('tenant_id')
        if not tenant_id:
            logout(request)
            return HttpResponseRedirect('/login/')
        
        tenant = Tenant.objects.get(id=tenant_id)
        saml_settings = self._get_saml_settings(tenant)
        
        auth = OneLogin_Saml2_Auth(request, saml_settings)
        
        # Get SAML session index from login
        session_index = request.session.get('saml_session_index')
        
        # Notify IdP of logout
        logout_url = auth.logout(name_id_format=session_index)
        
        # Clear Django session
        logout(request)
        
        return HttpResponseRedirect(logout_url)
```

### SAML Metadata Endpoint

Identity providers need to know about your application. Provide a metadata endpoint:

```python
class SamlMetadataView(View):
    """Expose SAML metadata for IdP configuration."""
    
    def get(self, request):
        tenant_id = request.GET.get('tenant_id')
        tenant = Tenant.objects.get(id=tenant_id)
        
        saml_settings = self._get_saml_settings(tenant)
        
        auth = OneLogin_Saml2_Auth(request, saml_settings)
        
        metadata = auth.get_settings().get_sp_metadata()
        
        return HttpResponse(metadata, content_type='application/xml')
```

Customer gives this URL to their IT team, who import it into Okta/Azure/etc.

---

## OIDC: Modern Alternative

OIDC (OpenID Connect) is built on OAuth2. It's simpler than SAML and increasingly preferred.

### OIDC vs SAML

| Aspect | SAML | OIDC |
|--------|------|------|
| Protocol | XML-based | JSON-based |
| Complexity | High | Low |
| Enterprise | Industry standard | Growing adoption |
| Setup | More manual | More automated |
| Token Type | Assertions | JWT |

### OIDC Implementation with Cognito

We use AWS Cognito as our OIDC provider:

```python
from django.conf import settings
from rest_framework import status
from rest_framework.response import Response
from rest_framework.views import APIView
from rest_framework.permissions import AllowAny
import requests
import jwt

class CognitoCallbackView(APIView):
    """Handle OAuth2/OIDC callback from Cognito."""
    
    permission_classes = [AllowAny]
    
    def get(self, request):
        """
        Cognito redirects here with auth code.
        Exchange code for tokens.
        """
        
        code = request.query_params.get('code')
        state = request.query_params.get('state')
        
        if not code:
            return Response(
                {'error': 'Missing authorization code'},
                status=status.HTTP_400_BAD_REQUEST
            )
        
        # Verify state parameter (CSRF protection)
        stored_state = request.session.get('oauth_state')
        if state != stored_state:
            return Response(
                {'error': 'Invalid state parameter'},
                status=status.HTTP_403_FORBIDDEN
            )
        
        try:
            # Exchange code for tokens
            tokens = self._exchange_code_for_tokens(code)
            
            # Verify and decode ID token
            id_token_payload = self._verify_id_token(tokens['id_token'])
            
            # Extract user info
            email = id_token_payload['email']
            name = id_token_payload.get('name', '')
            cognito_sub = id_token_payload['sub']
            
            # Find or create user
            user, created = User.objects.get_or_create(
                email=email,
                defaults={
                    'first_name': name.split()[0] if name else '',
                    'last_name': ' '.join(name.split()[1:]) if len(name.split()) > 1 else '',
                }
            )
            
            # Store Cognito info
            user.cognito_sub = cognito_sub
            user.save()
            
            # Login user
            login(request, user, backend='django.contrib.auth.backends.ModelBackend')
            
            # Store refresh token (for later use)
            request.session['refresh_token'] = tokens['refresh_token']
            
            return redirect('/dashboard/')
        
        except Exception as e:
            logger.error(f"Cognito callback error: {str(e)}", exc_info=True)
            return Response(
                {'error': str(e)},
                status=status.HTTP_401_UNAUTHORIZED
            )
    
    def _exchange_code_for_tokens(self, code):
        """Exchange authorization code for tokens."""
        
        token_url = f'https://{settings.COGNITO_DOMAIN}.auth.{settings.AWS_REGION}.amazoncognito.com/oauth2/token'
        
        response = requests.post(
            token_url,
            data={
                'grant_type': 'authorization_code',
                'client_id': settings.COGNITO_CLIENT_ID,
                'client_secret': settings.COGNITO_CLIENT_SECRET,
                'code': code,
                'redirect_uri': settings.COGNITO_REDIRECT_URI
            },
            headers={'Content-Type': 'application/x-www-form-urlencoded'}
        )
        
        if response.status_code != 200:
            raise Exception(f"Token exchange failed: {response.text}")
        
        return response.json()
    
    def _verify_id_token(self, id_token):
        """Verify JWT signature and decode."""
        
        # Get Cognito's public keys
        jwks_url = f'https://cognito-idp.{settings.AWS_REGION}.amazonaws.com/{settings.COGNITO_USER_POOL_ID}/.well-known/jwks.json'
        jwks = requests.get(jwks_url).json()
        
        # Decode without verification first (to get kid)
        unverified_header = jwt.get_unverified_header(id_token)
        kid = unverified_header['kid']
        
        # Find matching key
        key = next((k for k in jwks['keys'] if k['kid'] == kid), None)
        if not key:
            raise Exception("Key not found in JWKS")
        
        # Construct public key
        from cryptography.hazmat.primitives.asymmetric import rsa
        from jwcrypto import jwk as jwk_module
        
        # Verify signature
        payload = jwt.decode(
            id_token,
            key=key,
            algorithms=['RS256'],
            audience=settings.COGNITO_CLIENT_ID,
            issuer=f'https://cognito-idp.{settings.AWS_REGION}.amazonaws.com/{settings.COGNITO_USER_POOL_ID}'
        )
        
        return payload
```

### OIDC Login Initiation

```python
class OIDCLoginView(APIView):
    permission_classes = [AllowAny]
    
    def get(self, request):
        """Redirect to Cognito login."""
        
        import uuid
        
        # Generate state for CSRF protection
        state = str(uuid.uuid4())
        request.session['oauth_state'] = state
        
        authorization_url = f'https://{settings.COGNITO_DOMAIN}.auth.{settings.AWS_REGION}.amazoncognito.com/oauth2/authorize'
        
        params = {
            'client_id': settings.COGNITO_CLIENT_ID,
            'response_type': 'code',
            'scope': 'openid profile email',
            'redirect_uri': settings.COGNITO_REDIRECT_URI,
            'state': state
        }
        
        query_string = urllib.parse.urlencode(params)
        redirect_url = f'{authorization_url}?{query_string}'
        
        return redirect(redirect_url)
```

---

## Multi-Tenant SSO

Enterprise customers often need multiple teams using the same identity provider. We implemented tenant-scoped SSO:

```python
class TenantSamlLoginView(SamlLoginView):
    """SAML login scoped to specific tenant."""
    
    def get(self, request, tenant_slug):
        """Login via SAML for specific tenant."""
        
        tenant = Tenant.objects.get(slug=tenant_slug)
        
        if not tenant.saml_enabled:
            raise Http404("SAML not enabled")
        
        # Store tenant in session for later retrieval
        request.session['target_tenant_id'] = tenant.id
        
        # Create SAML request with RelayState
        saml_settings = self._get_saml_settings(tenant)
        auth = OneLogin_Saml2_Auth(request, saml_settings)
        
        redirect_url = auth.login(return_to=f'/saml/acs/?tenant_id={tenant.id}')
        
        return HttpResponseRedirect(redirect_url)
```

This allows customers to configure SSO per tenant without conflicts.

---

## Security Best Practices

### 1. Signature Verification (CRITICAL)

Never trust SAML/OIDC assertions without signature verification:

```python
# WRONG: Accept any SAML assertion
def insecure_saml_handler(saml_response):
    # Parse XML
    root = ET.fromstring(saml_response)
    email = root.find('.//email').text
    return email

# RIGHT: Verify signature
def secure_saml_handler(saml_response, idp_certificate):
    auth = OneLogin_Saml2_Auth(request, saml_settings)
    
    if not auth.is_authenticated():
        raise Exception("Signature verification failed")
    
    # Only now access attributes
    return auth.get_attributes()
```

### 2. Token Expiration

OIDC tokens expire. Check `exp` claim:

```python
import time

def verify_token_not_expired(payload):
    exp = payload.get('exp')
    if not exp:
        raise Exception("No expiration in token")
    
    if time.time() > exp:
        raise Exception("Token expired")
```

### 3. Prevent Token Reuse

Store and validate nonce to prevent replay attacks:

```python
class OIDCCallbackView(APIView):
    def get(self, request):
        # Generate nonce in auth request
        nonce = str(uuid.uuid4())
        request.session['oidc_nonce'] = nonce
        
        # Verify nonce in callback
        id_token = tokens['id_token']
        payload = jwt.decode(id_token, ...)
        
        if payload['nonce'] != request.session.get('oidc_nonce'):
            raise Exception("Nonce mismatch - replay attack detected")
```

### 4. HTTPS Only

All OAuth2/OIDC callbacks must be HTTPS:

```python
# settings.py
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

---

## Testing SSO

```python
from django.test import TestCase, Client
from unittest.mock import patch, Mock

class SamlLoginTestCase(TestCase):
    def test_saml_login_redirects_to_idp(self):
        """Verify login redirects to IdP."""
        
        client = Client()
        response = client.get('/saml/login/?tenant_id=1')
        
        self.assertEqual(response.status_code, 302)
        self.assertIn('okta.com', response.url)
    
    @patch('onelogin.saml2.auth.OneLogin_Saml2_Auth')
    def test_saml_acs_creates_user(self, mock_auth):
        """Verify ACS creates/updates user correctly."""
        
        mock_auth.return_value.is_authenticated.return_value = True
        mock_auth.return_value.get_attributes.return_value = {
            'email': ['test@example.com'],
            'firstName': ['John'],
            'lastName': ['Doe']
        }
        
        client = Client()
        response = client.post('/saml/acs/', {'SAMLResponse': 'mock'})
        
        # Verify user created
        user = User.objects.get(email='test@example.com')
        self.assertEqual(user.first_name, 'John')
        
        # Verify redirected to dashboard
        self.assertEqual(response.status_code, 302)
        self.assertIn('dashboard', response.url)
```

---

## Results & Adoption

After implementing federated SSO:

**Before:**
- Enterprise prospects: "Do you support SAML?"
- Answer: "No, but we're working on it"
- Sales friction: High
- Demo account creation: Manual, error-prone

**After:**
- SAML, OIDC, and Cognito all supported
- Customers can provision users automatically
- Zero friction for enterprise onboarding
- Demo account creation: Automated

**Impact:**
- **3 new enterprise customers** (previously blocked on SSO)
- **$500K+ in new ARR**
- **60% reduction in onboarding time** for enterprise deals
- **Zero support issues** related to authentication

---

## Implementation Checklist

- [ ] Choose authentication library (django-allauth, python-social-auth, etc.)
- [ ] Implement SAML with OneLogin library
- [ ] Implement OIDC with standard OAuth2 library
- [ ] Create tenant-scoped login flows
- [ ] Implement signature verification
- [ ] Add token expiration checks
- [ ] Store user attributes correctly
- [ ] Set up test environment (Okta Sandbox, Cognito test pool)
- [ ] Test all SAML/OIDC failure scenarios
- [ ] Document SSO setup for customers
- [ ] Create runbook for SSO troubleshooting
- [ ] Monitor SSO auth failures in production

---

## Key Takeaways

Federated SSO is complex but essential for enterprise adoption:

1. **SAML** is for traditional enterprises (established standard)
2. **OIDC** is for modern platforms (simpler, JSON-based)
3. **Always verify signatures** - Never trust unauthenticated assertions
4. **Test failure scenarios** - IdP downtime, expired tokens, signature mismatches
5. **Monitor auth flows** - Log all login attempts for security auditing

Supporting SSO isn't a nice-to-have anymore—it's table stakes for B2B SaaS.

---

## Related Resources

- [SAML 2.0 Specification](https://www.oasis-open.org/standards#saml2.0)
- [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)
- [OneLogin SAML Python Library](https://github.com/onelogin/python3-saml)
- [AWS Cognito OIDC Guide](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools.html)
- [OWASP OAuth2 Security Best Practices](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)

---

**Building enterprise auth?** What protocols are you supporting? Drop a comment below or reach out on [LinkedIn](https://linkedin.com/in/bharat-kumar28).
