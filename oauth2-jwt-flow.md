# OAuth2 + JWT Authentication Flow

## Mermaid Diagrams

### Complete OAuth2 + JWT Flow

```mermaid
sequenceDiagram
    participant User
    participant Browser
    participant SpringBoot as Spring Boot App
    participant OAuth2Provider as Google OAuth2
    participant JWTService as JWT Service
    participant API as Protected API

    Note over User,API: OAuth2 Authentication Flow

    User->>Browser: Click "Sign in with Google"
    Browser->>SpringBoot: GET /oauth2/authorization/google
    SpringBoot->>Browser: 302 Redirect to Google
    Browser->>OAuth2Provider: Authorization Request<br/>(client_id, redirect_uri, scope)

    OAuth2Provider->>User: Show Login Page
    User->>OAuth2Provider: Enter credentials
    OAuth2Provider->>User: Show Consent Screen
    User->>OAuth2Provider: Grant permissions

    OAuth2Provider->>Browser: 302 Redirect with auth code<br/>(/login/oauth2/code/google?code=xxx)
    Browser->>SpringBoot: Follow redirect with code

    SpringBoot->>OAuth2Provider: Exchange code for tokens<br/>(code, client_id, client_secret)
    OAuth2Provider->>SpringBoot: Access Token + ID Token

    SpringBoot->>OAuth2Provider: GET /userinfo<br/>(with access token)
    OAuth2Provider->>SpringBoot: User Profile Data<br/>(email, name, picture)

    SpringBoot->>JWTService: Generate JWT<br/>(email, name, provider)
    JWTService->>SpringBoot: JWT Token

    SpringBoot->>Browser: 302 Redirect /?token=JWT
    Browser->>User: Display success + store JWT

    Note over User,API: JWT Authentication Flow

    User->>Browser: Make API request
    Browser->>API: GET /api/hello<br/>Authorization: Bearer JWT

    API->>JWTService: Validate JWT
    JWTService->>API: Valid + Claims

    API->>Browser: Protected Resource Data
    Browser->>User: Display data
```

### OAuth2 Authorization Code Flow (Detailed)

```mermaid
sequenceDiagram
    participant Browser
    participant App as Spring Boot<br/>OAuth2 Client
    participant AuthServer as Google<br/>Authorization Server
    participant ResourceServer as Google<br/>Resource Server

    rect rgb(200, 220, 240)
        Note over Browser,AuthServer: Step 1: Authorization Request
        Browser->>App: GET /oauth2/authorization/google
        App->>Browser: 302 Redirect
        Note right of App: state=random123<br/>scope=openid profile email
        Browser->>AuthServer: GET /o/oauth2/v2/auth?<br/>response_type=code&<br/>client_id=XXX&<br/>redirect_uri=http://localhost:8080/login/oauth2/code/google&<br/>scope=openid+profile+email&<br/>state=random123
    end

    rect rgb(220, 240, 200)
        Note over Browser,AuthServer: Step 2: User Authentication & Consent
        AuthServer->>Browser: Login Page
        Browser->>AuthServer: POST credentials
        AuthServer->>Browser: Consent Screen
        Browser->>AuthServer: User grants consent
    end

    rect rgb(240, 220, 200)
        Note over Browser,AuthServer: Step 3: Authorization Code Grant
        AuthServer->>Browser: 302 Redirect<br/>http://localhost:8080/login/oauth2/code/google?<br/>code=4/0AX4XfWh...&state=random123
        Browser->>App: GET /login/oauth2/code/google?code=XXX&state=XXX
    end

    rect rgb(240, 200, 220)
        Note over App,AuthServer: Step 4: Token Exchange
        App->>App: Validate state parameter
        App->>AuthServer: POST /token<br/>grant_type=authorization_code<br/>code=4/0AX4XfWh...<br/>client_id=XXX<br/>client_secret=YYY<br/>redirect_uri=http://localhost:8080/login/oauth2/code/google
        AuthServer->>App: 200 OK<br/>{<br/> "access_token": "ya29.a0...",<br/> "id_token": "eyJhbGc...",<br/> "expires_in": 3599,<br/> "token_type": "Bearer",<br/> "scope": "openid profile email"<br/>}
    end

    rect rgb(200, 240, 220)
        Note over App,ResourceServer: Step 5: Fetch User Info
        App->>ResourceServer: GET /oauth2/v3/userinfo<br/>Authorization: Bearer ya29.a0...
        ResourceServer->>App: 200 OK<br/>{<br/> "email": "user@gmail.com",<br/> "name": "John Doe",<br/> "picture": "https://..."<br/>}
    end

    rect rgb(220, 200, 240)
        Note over App,Browser: Step 6: Generate JWT & Redirect
        App->>App: Generate JWT with user claims
        App->>Browser: 302 Redirect /?token=eyJhbGciOiJIUzI1NiIs...
        Browser->>Browser: Store JWT in localStorage
    end
```

### JWT Authentication Flow

```mermaid
sequenceDiagram
    participant Client
    participant JWTFilter as JWT Auth Filter
    participant JWTService as JWT Service
    participant SecurityContext as Security Context
    participant Controller

    Client->>JWTFilter: GET /api/hello<br/>Authorization: Bearer eyJhbGc...

    JWTFilter->>JWTFilter: Extract JWT from header

    alt JWT present
        JWTFilter->>JWTService: Validate token

        alt Valid token
            JWTService->>JWTService: Verify signature<br/>Check expiration
            JWTService->>JWTFilter: Valid + username

            JWTFilter->>JWTService: Extract username
            JWTService->>JWTFilter: user@example.com

            JWTFilter->>SecurityContext: Set Authentication<br/>(username, authorities)
            JWTFilter->>Controller: Continue to controller
            Controller->>Client: 200 OK + response data
        else Invalid/Expired token
            JWTService->>JWTFilter: Invalid
            JWTFilter->>Client: 401 Unauthorized
        end
    else No JWT
        JWTFilter->>Client: 401 Unauthorized
    end
```

### Security Filter Chain Architecture

```mermaid
graph TB
    Request[HTTP Request] --> Router{Request Path?}

    Router -->|/oauth2/**<br/>/login/oauth2/**| OAuth2Chain[OAuth2 Security Chain<br/>@Order 1]
    Router -->|/api/**| JWTChain[JWT Security Chain<br/>@Order 2]
    Router -->|/**, /static/**| DefaultChain[Default Security Chain<br/>@Order 3]

    OAuth2Chain --> OAuth2Login[OAuth2 Login Config]
    OAuth2Login --> OAuth2Success[OAuth2 Success Handler]
    OAuth2Success --> GenerateJWT[Generate JWT Token]
    GenerateJWT --> RedirectUser[Redirect with JWT]

    JWTChain --> JWTFilter[JWT Auth Filter]
    JWTFilter --> ValidateJWT{Validate JWT?}
    ValidateJWT -->|Valid| SetAuth[Set Authentication]
    ValidateJWT -->|Invalid| Reject401[401 Unauthorized]
    SetAuth --> Controller[Controller]

    DefaultChain --> PermitAll[Permit All]
    PermitAll --> StaticContent[Static Resources]

    style OAuth2Chain fill:#e1f5ff
    style JWTChain fill:#fff5e1
    style DefaultChain fill:#e1ffe1
    style GenerateJWT fill:#ffe1e1
    style ValidateJWT fill:#ffe1e1
```

### Component Interaction Diagram

```mermaid
graph LR
    subgraph Client Layer
        Browser[Browser/Client]
    end

    subgraph Spring Security Layer
        OAuth2Filter[OAuth2 Login Filter]
        JWTFilter[JWT Auth Filter]
        SecurityConfig[Security Configuration]
    end

    subgraph Service Layer
        OAuth2Service[OAuth2 Service]
        JWTService[JWT Service]
        OAuth2Handler[OAuth2 Success Handler]
    end

    subgraph Controller Layer
        OAuth2Controller[OAuth2 Controller]
        AuthController[Auth Controller]
        APIController[API Controllers]
    end

    subgraph External
        Google[Google OAuth2]
        GitHub[GitHub OAuth2]
    end

    Browser -->|1. OAuth2 Login| OAuth2Filter
    OAuth2Filter -->|2. Redirect| Google
    Google -->|3. Callback| OAuth2Filter
    OAuth2Filter -->|4. Process| OAuth2Handler
    OAuth2Handler -->|5. Get User Info| OAuth2Service
    OAuth2Service -->|6. Generate JWT| JWTService
    OAuth2Handler -->|7. Redirect| Browser

    Browser -->|8. API Request + JWT| JWTFilter
    JWTFilter -->|9. Validate| JWTService
    JWTFilter -->|10. Authorized| APIController
    APIController -->|11. Response| Browser

    Browser -->|Alt: JWT Login| AuthController
    AuthController -->|Generate| JWTService

    style Browser fill:#e1f5ff
    style Google fill:#4285f4,color:#fff
    style GitHub fill:#24292e,color:#fff
    style JWTService fill:#ffe1e1
    style OAuth2Service fill:#ffe1e1
```

### Data Flow: OAuth2 to JWT Conversion

```mermaid
flowchart TD
    Start([User Clicks<br/>Sign in with Google]) --> Redirect[Redirect to Google OAuth2]
    Redirect --> UserAuth[User Authenticates<br/>at Google]
    UserAuth --> Consent[User Grants<br/>Permissions]
    Consent --> Code[Google Returns<br/>Authorization Code]
    Code --> Exchange[App Exchanges Code<br/>for Access Token]
    Exchange --> GetUser[App Fetches User Info<br/>with Access Token]

    GetUser --> Extract[Extract User Data:<br/>- email<br/>- name<br/>- picture]
    Extract --> CreateClaims[Create JWT Claims:<br/>sub: email<br/>email: email<br/>name: name<br/>provider: google<br/>oauth2: true]
    CreateClaims --> SignJWT[Sign JWT with<br/>HMAC-SHA256<br/>secret key]
    SignJWT --> ReturnJWT[Return JWT to Client:<br/>eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...]
    ReturnJWT --> StoreClient[Client Stores JWT<br/>in localStorage]
    StoreClient --> UseAPI[Client Uses JWT for<br/>Subsequent API Calls]
    UseAPI --> End([Authenticated<br/>API Access])

    style Start fill:#e1f5ff
    style UserAuth fill:#4285f4,color:#fff
    style Consent fill:#4285f4,color:#fff
    style SignJWT fill:#ffe1e1
    style UseAPI fill:#e1ffe1
    style End fill:#e1ffe1
```

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                           CLIENT APPLICATION                         │
└────────────┬─────────────────────────────────────┬───────────────────┘
             │                                     │
             │ 1. Login Request                   │ 7. API Request
             │                                     │    (with JWT)
             ▼                                     ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         SPRING BOOT APPLICATION                      │
│                                                                      │
│ ┌──────────────────────┐     ┌──────────────────────────────────┐  │
│ │   OAuth2 Controller   │     │      JWT Auth Filter             │  │
│ └───────────┬──────────┘     └──────────┬───────────────────────┘  │
│             │                            │                          │
│             ▼                            ▼                          │
│ ┌──────────────────────┐     ┌──────────────────────────────────┐  │
│ │   OAuth2 Service     │     │       JWT Service                │  │
│ └───────────┬──────────┘     └──────────────────────────────────┘  │
│             │                                                       │
│             │ 2. Redirect to Provider                              │
└─────────────┼───────────────────────────────────────────────────────┘
              │
              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      OAUTH2 PROVIDER (Google/GitHub)                 │
│                                                                      │
│  3. User Authentication                                              │
│  4. Authorization Grant                                              │
│  5. Redirect back with code                                         │
└──────────────────────────────────────────────────────────────────────┘
```

## Detailed Authentication Flow

### OAuth2 Login Flow

```
User                    Spring App                 OAuth2 Provider
 │                           │                           │
 │  1. Click "Login with     │                           │
 │     Google/GitHub"        │                           │
 ├──────────────────────────>│                           │
 │                           │                           │
 │                           │  2. Redirect to Provider  │
 │<──────────────────────────┤                           │
 │                           │                           │
 │  3. Authenticate          │                           │
 ├───────────────────────────┼──────────────────────────>│
 │                           │                           │
 │                           │  4. Authorization Code    │
 │<──────────────────────────┼───────────────────────────┤
 │                           │                           │
 │  5. Redirect with code    │                           │
 ├──────────────────────────>│                           │
 │                           │                           │
 │                           │  6. Exchange code         │
 │                           │     for access token      │
 │                           ├──────────────────────────>│
 │                           │                           │
 │                           │  7. Return access token   │
 │                           │<──────────────────────────┤
 │                           │                           │
 │                           │  8. Fetch user info       │
 │                           ├──────────────────────────>│
 │                           │                           │
 │                           │  9. Return user data      │
 │                           │<──────────────────────────┤
 │                           │                           │
 │                           │ 10. Generate JWT          │
 │                           │     with user claims      │
 │                           │                           │
 │  11. Return JWT token     │                           │
 │<──────────────────────────┤                           │
 │                           │                           │
```

### JWT Authentication Flow

```
User                    Spring App                 Protected Resource
 │                           │                           │
 │  1. API Request with      │                           │
 │     Bearer Token          │                           │
 ├──────────────────────────>│                           │
 │                           │                           │
 │                           │ 2. JWT Filter validates  │
 │                           │    token                 │
 │                           │                           │
 │                           │ 3. Extract claims        │
 │                           │                           │
 │                           │ 4. Set Authentication    │
 │                           │    Context              │
 │                           │                           │
 │                           │ 5. Process request      │
 │                           ├──────────────────────────>│
 │                           │                           │
 │                           │ 6. Return data          │
 │                           │<──────────────────────────┤
 │                           │                           │
 │  7. API Response          │                           │
 │<──────────────────────────┤                           │
 │                           │                           │
```

## Component Responsibilities

### OAuth2 Components

1. **OAuth2Config**: Configures OAuth2 client registrations for providers
2. **OAuth2Properties**: Manages OAuth2 configuration properties
3. **OAuth2Service**: Handles OAuth2 authentication logic and JWT generation
4. **OAuth2SuccessHandler**: Processes successful OAuth2 authentication
5. **OAuth2Controller**: Exposes OAuth2-related endpoints
6. **OAuth2SecurityConfig**: Configures security for OAuth2 endpoints

### JWT Components (Existing)

1. **JwtService**: Generates and validates JWT tokens
2. **JwtAuthFilter**: Intercepts requests to validate JWT tokens
3. **JwtProperties**: JWT configuration properties
4. **SecurityConfig**: Main security configuration

## Key Endpoints

### OAuth2 Endpoints
- `GET /oauth2/authorization/{provider}` - Initiates OAuth2 login
- `GET /login/oauth2/code/{provider}` - OAuth2 callback endpoint
- `GET /api/oauth2/user` - Get OAuth2 user information
- `POST /api/oauth2/token` - Generate JWT from OAuth2 authentication
- `POST /api/oauth2/logout` - Revoke OAuth2 session

### JWT Protected Endpoints
- `GET /api/hello` - Sample protected endpoint
- Any endpoint under `/api/**` (except `/api/auth/**`)

## Security Configuration

The implementation uses two separate security filter chains:

1. **OAuth2 Security Chain** (Order 1)
   - Handles `/oauth2/**` and `/login/oauth2/**` paths
   - Manages OAuth2 authentication flow
   - Stateless session management

2. **JWT Security Chain** (Order 2)
   - Handles `/api/**` paths
   - Validates JWT tokens for API access
   - Stateless session management

## Configuration Example

```yaml
oauth2:
  providers:
    google:
      authorization-uri: https://accounts.google.com/o/oauth2/v2/auth
      token-uri: https://oauth2.googleapis.com/token
      user-info-uri: https://www.googleapis.com/oauth2/v3/userinfo
    github:
      authorization-uri: https://github.com/login/oauth/authorize
      token-uri: https://github.com/login/oauth/access_token
      user-info-uri: https://api.github.com/user

spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${OAUTH2_GOOGLE_CLIENT_ID}
            client-secret: ${OAUTH2_GOOGLE_CLIENT_SECRET}
          github:
            client-id: ${OAUTH2_GITHUB_CLIENT_ID}
            client-secret: ${OAUTH2_GITHUB_CLIENT_SECRET}
```

## Token Structure

### JWT Token Claims
```json
{
  "sub": "user@example.com",
  "email": "user@example.com",
  "name": "User Name",
  "provider": "google",
  "oauth2": true,
  "iat": 1234567890,
  "exp": 1234571490
}
```

## Usage Example

1. **OAuth2 Login**:
```bash
# Browser navigates to:
GET http://localhost:8080/oauth2/authorization/google
```

2. **Use JWT Token**:
```bash
curl -H "Authorization: Bearer <jwt-token>" \
     http://localhost:8080/api/hello
```