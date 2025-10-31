# Low-Level Design (LLD)

## Document Control

| Field | Details |
|-------|---------|
| **Version** | [e.g., 1.0.0] |
| **Date** | [Date] |
| **Status** | [Draft / Review / Approved] |
| **Author** | [Name, Role] |
| **Reviewers** | [Names] |

---

## 1. Module Overview

### 1.1 Purpose
[Explain what this LLD document covers]

**Example:** "This Low-Level Design document provides detailed technical specifications for implementing [Product Name]. It includes class structures, database schemas, API specifications, algorithms, and configuration details necessary for development."

### 1.2 Scope
This document covers:
- Detailed component/class design
- Database schema and relationships
- API endpoint specifications
- Business logic and algorithms
- Configuration and environment setup
- Error handling strategies
- Testing approaches

### 1.3 Prerequisites
Readers should be familiar with:
- [HLD document]
- [SRS document]
- [Technology stack - e.g., "Python, FastAPI, PostgreSQL, React"]

---

## 2. Detailed Component Breakdown

### 2.1 [Component 1 - e.g., "User Management Component"]

#### 2.1.1 Overview
[Brief description of what this component does]

#### 2.1.2 Structure

**Directory Layout:**
```
src/
├── modules/
│   ├── user/
│   │   ├── __init__.py
│   │   ├── controllers/
│   │   │   ├── user_controller.py
│   │   │   └── auth_controller.py
│   │   ├── services/
│   │   │   ├── user_service.py
│   │   │   └── auth_service.py
│   │   ├── repositories/
│   │   │   └── user_repository.py
│   │   ├── models/
│   │   │   ├── user_model.py
│   │   │   └── session_model.py
│   │   ├── schemas/
│   │   │   ├── user_schema.py
│   │   │   └── auth_schema.py
│   │   └── exceptions/
│   │       └── user_exceptions.py
```

#### 2.1.3 Controllers

**Class: UserController**
```python
Purpose: Handle HTTP requests for user operations
Location: src/modules/user/controllers/user_controller.py

class UserController:
    Methods:
    - register(request: RegisterRequest) -> UserResponse
    - get_user(user_id: str) -> UserResponse
    - update_user(user_id: str, request: UpdateUserRequest) -> UserResponse
    - delete_user(user_id: str) -> StatusResponse
    - list_users(filters: UserFilters, pagination: Pagination) -> UserListResponse
```

**Method Details: register()**
```
Input: RegisterRequest (email, password, full_name, terms_accepted)
Output: UserResponse (user_id, email, full_name, created_at)
Validation:
  - Email format validation
  - Password strength check (min 8 chars, 1 uppercase, 1 number)
  - Duplicate email check
  - Terms acceptance required
Process:
  1. Validate input data
  2. Call UserService.create_user()
  3. Handle exceptions
  4. Return formatted response
Error Handling:
  - DuplicateEmailException → 409 Conflict
  - ValidationException → 400 Bad Request
  - InternalException → 500 Internal Server Error
```

---

#### 2.1.4 Services (Business Logic)

**Class: UserService**
```
Purpose: Implement user management business logic
Location: src/modules/user/services/user_service.py

Dependencies:
- UserRepository (data access)
- EmailService (verification emails)
- CacheService (session caching)

Methods:
- create_user(data: CreateUserDTO) -> User
- get_user_by_id(user_id: str) -> User
- get_user_by_email(email: str) -> User
- update_user(user_id: str, data: UpdateUserDTO) -> User
- delete_user(user_id: str) -> bool
- verify_email(token: str) -> bool
```

**Method Details: create_user()**
```python
def create_user(data: CreateUserDTO) -> User:
    """
    Creates a new user account
    
    Args:
        data: User creation data (email, password, full_name)
    
    Returns:
        User: Created user object
        
    Raises:
        DuplicateEmailException: If email already exists
        ValidationException: If data validation fails
    
    Process:
        1. Validate email uniqueness
        2. Hash password using bcrypt (12 rounds)
        3. Generate verification token
        4. Create user record in database
        5. Queue verification email
        6. Invalidate relevant caches
        7. Return user object
    
    Transaction: Yes (atomic)
    Caching: Invalidate user list cache
    Events: Emit UserCreatedEvent
    """
```

---

#### 2.1.5 Repositories (Data Access)

**Class: UserRepository**
```
Purpose: Database operations for User entity
Location: src/modules/user/repositories/user_repository.py

Methods:
- create(user: User) -> User
- find_by_id(user_id: str) -> Optional[User]
- find_by_email(email: str) -> Optional[User]
- update(user_id: str, updates: dict) -> User
- delete(user_id: str) -> bool
- find_all(filters: dict, pagination: dict) -> List[User]
- exists_by_email(email: str) -> bool
```

**Query Examples:**
```sql
-- Find by email
SELECT * FROM users WHERE email = %s AND deleted_at IS NULL;

-- Create user
INSERT INTO users (id, email, password_hash, full_name, created_at, updated_at)
VALUES (%s, %s, %s, %s, NOW(), NOW())
RETURNING *;

-- Update user
UPDATE users 
SET full_name = %s, updated_at = NOW()
WHERE id = %s AND deleted_at IS NULL
RETURNING *;

-- Soft delete
UPDATE users SET deleted_at = NOW() WHERE id = %s;
```

---

#### 2.1.6 Models (Data Entities)

**Class: User**
```python
from datetime import datetime
from typing import Optional

class User:
    """
    User entity model
    
    Attributes:
        id (str): UUID primary key
        email (str): User email (unique)
        password_hash (str): Hashed password
        full_name (str): User's full name
        is_verified (bool): Email verification status
        is_active (bool): Account status
        created_at (datetime): Account creation timestamp
        updated_at (datetime): Last update timestamp
        deleted_at (Optional[datetime]): Soft delete timestamp
    """
    
    id: str
    email: str
    password_hash: str
    full_name: str
    is_verified: bool = False
    is_active: bool = True
    created_at: datetime
    updated_at: datetime
    deleted_at: Optional[datetime] = None
    
    # Relationships
    sessions: List['Session']
    profile: Optional['UserProfile']
```

---

### 2.2 [Component 2 - e.g., "Content Management Component"]

[Follow same structure as 2.1]

- 2.2.1 Overview
- 2.2.2 Structure
- 2.2.3 Controllers
- 2.2.4 Services
- 2.2.5 Repositories
- 2.2.6 Models

---

### 2.3 [Component 3 - e.g., "Notification Component"]

[Follow same structure]

---

### 2.4 Workers & Background Jobs

#### Worker: EmailWorker
```
Purpose: Process email sending jobs from queue
Queue: email_queue
Concurrency: 5 workers
Retry Policy: 3 attempts with exponential backoff (2^n seconds)
DLQ: email_dead_letter_queue

Process:
1. Consume message from queue
2. Parse email data
3. Call EmailService.send()
4. Log success/failure
5. ACK or NACK message based on result

Error Handling:
- Transient errors (network timeout): Retry
- Permanent errors (invalid email): Move to DLQ
- Rate limit errors: Delay and retry
```

#### Worker: DataAggregationWorker
```
Purpose: Aggregate data for analytics
Schedule: Cron - every hour at :00
Timeout: 10 minutes
Lock: Distributed lock to prevent concurrent runs

Process:
1. Acquire distributed lock
2. Fetch data from last hour
3. Perform aggregations (sum, avg, count)
4. Update aggregated_stats table
5. Invalidate analytics cache
6. Release lock
```

---

### 2.5 Agents & AI Components (if applicable)

#### Agent: [Agent Name - e.g., "CustomerSupportAgent"]
```
Purpose: Handle customer queries using AI
Framework: [e.g., CrewAI / LangGraph]
LLM: [e.g., GPT-4]

Tools Available:
- search_knowledge_base(): Query vector database
- get_user_info(): Fetch user data
- create_ticket(): Create support ticket

Workflow:
1. Receive user query
2. Classify intent
3. Search knowledge base
4. Generate response
5. If confident: Return response
6. If uncertain: Escalate to human agent
```

---

## 3. Class Diagrams

### 3.1 User Management Module

```mermaid
classDiagram
    class User {
        +String id
        +String email
        +String password_hash
        +String full_name
        +Boolean is_verified
        +Boolean is_active
        +DateTime created_at
        +DateTime updated_at
        +DateTime deleted_at
    }
    
    class Session {
        +String id
        +String user_id
        +String token
        +DateTime expires_at
        +DateTime created_at
    }
    
    class UserProfile {
        +String id
        +String user_id
        +String avatar_url
        +String bio
        +JSON preferences
    }
    
    class UserRepository {
        +create(user) User
        +find_by_id(id) User
        +find_by_email(email) User
        +update(id, data) User
        +delete(id) bool
    }
    
    class UserService {
        -UserRepository repo
        +create_user(data) User
        +get_user(id) User
        +update_user(id, data) User
        +verify_email(token) bool
    }
    
    class UserController {
        -UserService service
        +register(request) Response
        +login(request) Response
        +get_profile(user_id) Response
    }
    
    User "1" -- "many" Session : has
    User "1" -- "1" UserProfile : has
    UserRepository --> User : manages
    UserService --> UserRepository : uses
    UserController --> UserService : uses
```

### 3.2 [Another Module Class Diagram]

[Add more class diagrams as needed]

---

## 4. Sequence Diagrams

### 4.1 User Registration Flow

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant APIGateway
    participant AuthController
    participant UserService
    participant UserRepository
    participant Database
    participant EmailQueue
    participant EmailWorker
    participant EmailService
    
    User->>Frontend: Fill registration form
    Frontend->>Frontend: Client-side validation
    Frontend->>APIGateway: POST /api/v1/auth/register
    APIGateway->>AuthController: route request
    
    AuthController->>AuthController: Validate request schema
    AuthController->>UserService: create_user(data)
    
    UserService->>UserRepository: exists_by_email(email)
    UserRepository->>Database: SELECT * FROM users WHERE email=?
    Database-->>UserRepository: result
    UserRepository-->>UserService: false
    
    UserService->>UserService: hash_password(password)
    UserService->>UserRepository: create(user)
    UserRepository->>Database: INSERT INTO users
    Database-->>UserRepository: new user
    UserRepository-->>UserService: user
    
    UserService->>UserService: generate_verification_token()
    UserService->>EmailQueue: queue email job
    EmailQueue-->>UserService: ack
    
    UserService-->>AuthController: user
    AuthController-->>APIGateway: 201 Created
    APIGateway-->>Frontend: success response
    Frontend-->>User: Show success message
    
    EmailWorker->>EmailQueue: consume message
    EmailWorker->>EmailService: send_verification_email()
    EmailService-->>User: Email delivered
```

### 4.2 [Another Critical Flow - e.g., "Content Creation"]

[Add more sequence diagrams]

### 4.3 [Another Flow - e.g., "Payment Processing"]

[Add more diagrams]

---

## 5. Data Model Details

### 5.1 Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    USERS ||--o{ SESSIONS : has
    USERS ||--o| USER_PROFILES : has
    USERS ||--o{ CONTENT : creates
    USERS ||--o{ COMMENTS : writes
    CONTENT ||--o{ COMMENTS : has
    CONTENT }o--|| CATEGORIES : belongs_to
    USERS ||--o{ SUBSCRIPTIONS : has
    
    USERS {
        uuid id PK
        string email UK
        string password_hash
        string full_name
        boolean is_verified
        boolean is_active
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }
    
    SESSIONS {
        uuid id PK
        uuid user_id FK
        string token UK
        timestamp expires_at
        timestamp created_at
    }
    
    USER_PROFILES {
        uuid id PK
        uuid user_id FK
        string avatar_url
        text bio
        jsonb preferences
        timestamp updated_at
    }
    
    CONTENT {
        uuid id PK
        uuid user_id FK
        uuid category_id FK
        string title
        text body
        string status
        timestamp published_at
        timestamp created_at
        timestamp updated_at
    }
    
    COMMENTS {
        uuid id PK
        uuid user_id FK
        uuid content_id FK
        text body
        timestamp created_at
        timestamp updated_at
    }
    
    CATEGORIES {
        uuid id PK
        string name UK
        string slug UK
        text description
    }
    
    SUBSCRIPTIONS {
        uuid id PK
        uuid user_id FK
        string plan_type
        string status
        timestamp starts_at
        timestamp ends_at
        timestamp created_at
    }
```

### 5.2 Table Schemas

#### Table: users
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(255) NOT NULL,
    is_verified BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP NULL
);

-- Indexes
CREATE INDEX idx_users_email ON users(email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_created_at ON users(created_at);
CREATE INDEX idx_users_is_active ON users(is_active) WHERE is_active = TRUE;

-- Triggers
CREATE TRIGGER update_users_updated_at
BEFORE UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();
```

#### Table: sessions
```sql
CREATE TABLE sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token VARCHAR(512) NOT NULL UNIQUE,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_sessions_user_id ON sessions(user_id);
CREATE INDEX idx_sessions_token ON sessions(token);
CREATE INDEX idx_sessions_expires_at ON sessions(expires_at);
```

#### Table: [Another Table - e.g., "content"]
```sql
[Add schema definition]
```

---

### 5.3 Index Strategy

**Primary Indexes:**
- All primary keys (automatic B-tree indexes)
- Foreign keys for join optimization

**Query-Specific Indexes:**
```sql
-- For user login queries
CREATE INDEX idx_users_email_active ON users(email) 
WHERE is_active = TRUE AND deleted_at IS NULL;

-- For content listing
CREATE INDEX idx_content_published ON content(published_at DESC) 
WHERE status = 'published';

-- For search queries
CREATE INDEX idx_content_search ON content 
USING GIN(to_tsvector('english', title || ' ' || body));

-- Composite indexes
CREATE INDEX idx_content_user_status ON content(user_id, status, created_at);
```

**Index Maintenance:**
- Monitor index bloat monthly
- Rebuild fragmented indexes quarterly
- Review unused indexes quarterly

---

### 5.4 Redis Key Naming Strategy

**Pattern:** `{environment}:{service}:{resource}:{identifier}:{attribute}`

**Examples:**
```
# Session storage
prod:auth:session:user:abc123:token

# API rate limiting
prod:api:rate_limit:user:abc123:count

# Cached user data
prod:user:cache:user:abc123:profile

# Feature flags
prod:system:feature_flag:new_ui:enabled

# Queue counters
prod:worker:queue:email:pending_count
```

**TTL Strategy:**
- Sessions: 24 hours
- Rate limit counters: 60 seconds
- Cache: 5-60 minutes depending on data volatility
- Feature flags: No expiry (manual invalidation)

---

## 6. API Specifications

### 6.1 Authentication APIs

#### POST /api/v1/auth/register
**Description:** Register a new user account

**Request:**
```json
{
  "email": "user@example.com",
  "password": "SecurePass123",
  "full_name": "John Doe",
  "terms_accepted": true
}
```

**Validation Rules:**
- email: Required, valid email format, max 255 chars
- password: Required, min 8 chars, 1 uppercase, 1 number
- full_name: Required, min 2 chars, max 255 chars
- terms_accepted: Required, must be true

**Response: 201 Created**
```json
{
  "status": "success",
  "data": {
    "user_id": "123e4567-e89b-12d3-a456-426614174000",
    "email": "user@example.com",
    "full_name": "John Doe",
    "is_verified": false,
    "created_at": "2025-10-31T10:00:00Z"
  },
  "message": "Registration successful. Please check your email to verify your account."
}
```

**Error Responses:**
- `400 Bad Request` - Validation errors
- `409 Conflict` - Email already registered
- `500 Internal Server Error` - Server error

**Error Example:**
```json
{
  "status": "error",
  "error": {
    "code": "EMAIL_EXISTS",
    "message": "An account with this email already exists",
    "field": "email"
  }
}
```

**Rate Limit:** 5 requests per minute per IP

**Authentication:** Not required

---

#### POST /api/v1/auth/login
**Description:** Authenticate user and get access token

**Request:**
```json
{
  "email": "user@example.com",
  "password": "SecurePass123"
}
```

**Response: 200 OK**
```json
{
  "status": "success",
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "token_type": "Bearer",
    "expires_in": 3600,
    "user": {
      "user_id": "123e4567-e89b-12d3-a456-426614174000",
      "email": "user@example.com",
      "full_name": "John Doe"
    }
  }
}
```

**Error Responses:**
- `401 Unauthorized` - Invalid credentials
- `403 Forbidden` - Account not verified or inactive
- `429 Too Many Requests` - Rate limit exceeded

**Rate Limit:** 10 requests per minute per IP

---

### 6.2 User Management APIs

#### GET /api/v1/users/{user_id}
[Follow same structure]

#### PUT /api/v1/users/{user_id}
[Follow same structure]

#### DELETE /api/v1/users/{user_id}
[Follow same structure]

---

### 6.3 [Other API Group - e.g., "Content APIs"]

[Add more API specifications]

---

## 7. Business Logic & Algorithms

### 7.1 Password Hashing Algorithm

```python
import bcrypt

def hash_password(password: str) -> str:
    """
    Hash password using bcrypt with 12 rounds
    
    Args:
        password: Plain text password
        
    Returns:
        Hashed password string
        
    Security:
        - Uses bcrypt with 12 rounds (2^12 iterations)
        - Salt is automatically generated
        - Resistant to rainbow table attacks
    """
    salt = bcrypt.gensalt(rounds=12)
    hashed = bcrypt.hashpw(password.encode('utf-8'), salt)
    return hashed.decode('utf-8')

def verify_password(password: str, password_hash: str) -> bool:
    """
    Verify password against stored hash
    
    Args:
        password: Plain text password to verify
        password_hash: Stored hashed password
        
    Returns:
        True if password matches, False otherwise
    """
    return bcrypt.checkpw(
        password.encode('utf-8'),
        password_hash.encode('utf-8')
    )
```

---

### 7.2 JWT Token Generation

```python
import jwt
from datetime import datetime, timedelta
from typing import Dict

def generate_access_token(user_id: str, email: str) -> str:
    """
    Generate JWT access token
    
    Args:
        user_id: User's UUID
        email: User's email
        
    Returns:
        JWT token string
        
    Configuration:
        - Algorithm: HS256
        - Expiration: 1 hour
        - Issuer: Your App Name
    """
    payload = {
        'sub': user_id,  # Subject (user ID)
        'email': email,
        'iat': datetime.utcnow(),  # Issued at
        'exp': datetime.utcnow() + timedelta(hours=1),  # Expiration
        'iss': 'YourAppName',  # Issuer
        'type': 'access'
    }
    
    token = jwt.encode(
        payload,
        settings.JWT_SECRET_KEY,
        algorithm='HS256'
    )
    
    return token
```

---

### 7.3 [Another Algorithm - e.g., "Recommendation Engine"]

[Add algorithm details]

---

### 7.4 State Machine: [Example - "Content Workflow"]

**States:**
- `draft`: Initial state
- `pending_review`: Submitted for review
- `approved`: Review approved
- `published`: Live and visible
- `archived`: No longer active

**Transitions:**
```
draft -> pending_review: submit_for_review()
pending_review -> approved: approve()
pending_review -> draft: reject()
approved -> published: publish()
published -> archived: archive()
archived -> draft: restore()
```

**Implementation:**
```python
from enum import Enum

class ContentStatus(Enum):
    DRAFT = "draft"
    PENDING_REVIEW = "pending_review"
    APPROVED = "approved"
    PUBLISHED = "published"
    ARCHIVED = "archived"

ALLOWED_TRANSITIONS = {
    ContentStatus.DRAFT: [ContentStatus.PENDING_REVIEW],
    ContentStatus.PENDING_REVIEW: [ContentStatus.APPROVED, ContentStatus.DRAFT],
    ContentStatus.APPROVED: [ContentStatus.PUBLISHED],
    ContentStatus.PUBLISHED: [ContentStatus.ARCHIVED],
    ContentStatus.ARCHIVED: [ContentStatus.DRAFT]
}

def can_transition(current: ContentStatus, target: ContentStatus) -> bool:
    """Check if state transition is allowed"""
    return target in ALLOWED_TRANSITIONS.get(current, [])
```

---

## 8. Configurations

### 8.1 Environment Variables

| Variable | Description | Example | Default | Required |
|----------|-------------|---------|---------|----------|
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://user:pass@host:5432/db` | - | Yes |
| `REDIS_URL` | Redis connection string | `redis://localhost:6379/0` | - | Yes |
| `JWT_SECRET_KEY` | Secret key for JWT signing | `your-secret-key-here` | - | Yes |
| `JWT_EXPIRATION_HOURS` | JWT token validity period | `24` | `1` | No |
| `SMTP_HOST` | Email server host | `smtp.gmail.com` | - | Yes |
| `SMTP_PORT` | Email server port | `587` | `587` | No |
| `AWS_ACCESS_KEY_ID` | AWS credentials | `AKIAIOSFODNN7EXAMPLE` | - | Yes |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key | `wJalrXUtnFEMI/K7MDENG...` | - | Yes |
| `AWS_S3_BUCKET` | S3 bucket name | `my-app-uploads` | - | Yes |
| `MAX_UPLOAD_SIZE_MB` | Max file upload size | `100` | `10` | No |
| `RATE_LIMIT_PER_MINUTE` | API rate limit | `60` | `100` | No |
| `LOG_LEVEL` | Logging level | `INFO` | `INFO` | No |
| `ENABLE_DEBUG_MODE` | Debug mode flag | `false` | `false` | No |

---

### 8.2 Vault Secrets

**Path:** `/secret/production/app-name/`

| Secret Key | Description | Rotation Policy |
|------------|-------------|-----------------|
| `database_password` | Database master password | Quarterly |
| `jwt_secret` | JWT signing key | Annually |
| `api_keys/stripe` | Stripe API key | As needed |
| `api_keys/sendgrid` | SendGrid API key | As needed |
| `encryption_key` | Data encryption key | Annually |

**Access Pattern:**
```python
import hvac

def get_secret(key: str) -> str:
    client = hvac.Client(url=settings.VAULT_URL)
    client.token = settings.VAULT_TOKEN
    secret = client.secrets.kv.v2.read_secret_version(
        path=f"production/app-name/{key}"
    )
    return secret['data']['data']['value']
```

---

### 8.3 Feature Flags

| Flag Name | Description | Default | Impact |
|-----------|-------------|---------|---------|
| `enable_new_ui` | Enable redesigned UI | `false` | Frontend |
| `enable_advanced_search` | Enable Elasticsearch search | `false` | Backend |
| `enable_video_upload` | Allow video uploads | `false` | Storage |
| `maintenance_mode` | Activate maintenance mode | `false` | All |
| `rate_limit_strict` | Stricter rate limiting | `true` | API |

**Implementation:**
```python
from typing import Optional

class FeatureFlags:
    @staticmethod
    def is_enabled(flag_name: str, user_id: Optional[str] = None) -> bool:
        """
        Check if feature flag is enabled
        
        Args:
            flag_name: Name of the feature flag
            user_id: Optional user ID for per-user flags
            
        Returns:
            True if enabled, False otherwise
        """
        # Check Redis for flag value
        key = f"feature_flag:{flag_name}"
        if user_id:
            # Check user-specific override
            user_key = f"feature_flag:{flag_name}:user:{user_id}"
            user_value = redis.get(user_key)
            if user_value is not None:
                return user_value == "true"
        
        # Check global flag
        value = redis.get(key)
        return value == "true" if value else False
```

---

### 8.4 Message Queue Configuration

#### RabbitMQ Queues

| Queue Name | Purpose | Durability | TTL | DLQ |
|------------|---------|-----------|-----|-----|
| `email_queue` | Email sending | Durable | 1 hour | `email_dlq` |
| `notification_queue` | Push notifications | Durable | 30 min | `notification_dlq` |
| `data_processing_queue` | Heavy data tasks | Durable | 6 hours | `processing_dlq` |
| `priority_queue` | High-priority tasks | Durable | 5 min | `priority_dlq` |

**Exchange Configuration:**
```
Type: topic
Name: app_exchange
Bindings:
  - email_queue: routing key "email.*"
  - notification_queue: routing key "notification.*"
  - data_processing_queue: routing key "processing.*"
```

---

## 9. Error Handling Strategy

### 9.1 Error Code Schema

**Format:** `{MODULE}_{TYPE}_{NUMBER}`

**Examples:**
- `AUTH_INVALID_CREDENTIALS_001`
- `USER_NOT_FOUND_002`
- `PAYMENT_FAILED_003`

### 9.2 Exception Hierarchy

```python
class AppException(Exception):
    """Base exception for all application errors"""
    def __init__(self, message: str, code: str, status_code: int = 500):
        self.message = message
        self.code = code
        self.status_code = status_code
        super().__init__(self.message)

class ValidationException(AppException):
    """Input validation errors"""
    def __init__(self, message: str, field: str = None):
        super().__init__(message, "VALIDATION_ERROR", 400)
        self.field = field

class NotFoundException(AppException):
    """Resource not found"""
    def __init__(self, resource: str, identifier: str):
        message = f"{resource} with id {identifier} not found"
        super().__init__(message, "NOT_FOUND", 404)

class UnauthorizedException(AppException):
    """Authentication failed"""
    def __init__(self, message: str = "Invalid credentials"):
        super().__init__(message, "UNAUTHORIZED", 401)

class ForbiddenException(AppException):
    """Permission denied"""
    def __init__(self, message: str = "Access denied"):
        super().__init__(message, "FORBIDDEN", 403)
```

### 9.3 Error Response Format

```json
{
  "status": "error",
  "error": {
    "code": "USER_NOT_FOUND_002",
    "message": "User with id abc123 not found",
    "field": "user_id",
    "timestamp": "2025-10-31T10:00:00Z",
    "trace_id": "abc-123-def-456"
  }
}
```

### 9.4 Retry Logic

**Exponential Backoff Strategy:**
```python
def exponential_backoff(attempt: int, base_delay: int = 1) -> int:
    """
    Calculate delay for exponential backoff
    
    Args:
        attempt: Current attempt number (0-indexed)
        base_delay: Base delay in seconds
        
    Returns:
        Delay in seconds
        
    Formula: delay = base_delay * (2 ^ attempt)
    Example: 1s, 2s, 4s, 8s, 16s...
    """
    return base_delay * (2 ** attempt)

def retry_with_backoff(func, max_attempts: int = 3):
    """
    Retry function with exponential backoff
    
    Transient errors (network, timeout): Retry
    Permanent errors (validation, not found): Don't retry
    """
    for attempt in range(max_attempts):
        try:
            return func()
        except TransientException as e:
            if attempt == max_attempts - 1:
                raise
            delay = exponential_backoff(attempt)
            time.sleep(delay)
        except PermanentException:
            raise  # Don't retry
```

---

## 10. Observability

### 10.1 Metrics

#### Application Metrics
```python
from prometheus_client import Counter, Histogram, Gauge

# API Metrics
api_requests_total = Counter(
    'api_requests_total',
    'Total API requests',
    ['method', 'endpoint', 'status']
)

api_request_duration = Histogram(
    'api_request_duration_seconds',
    'API request duration',
    ['method', 'endpoint']
)

# Business Metrics
users_registered_total = Counter(
    'users_registered_total',
    'Total users registered'
)

active_users = Gauge(
    'active_users',
    'Number of currently active users'
)

# Database Metrics
db_query_duration = Histogram(
    'db_query_duration_seconds',
    'Database query duration',
    ['query_type']
)

db_connection_pool_size = Gauge(
    'db_connection_pool_size',
    'Database connection pool size'
)
```

#### Custom Metrics Collection
```python
@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    start_time = time.time()
    
    response = await call_next(request)
    
    duration = time.time() - start_time
    
    api_requests_total.labels(
        method=request.method,
        endpoint=request.url.path,
        status=response.status_code
    ).inc()
    
    api_request_duration.labels(
        method=request.method,
        endpoint=request.url.path
    ).observe(duration)
    
    return response
```

---

### 10.2 Logging

#### Log Levels
- **DEBUG:** Detailed diagnostic information
- **INFO:** General informational messages
- **WARNING:** Warning messages for potentially harmful situations
- **ERROR:** Error messages for serious problems
- **CRITICAL:** Critical messages for very serious errors

#### Log Format (JSON)
```json
{
  "timestamp": "2025-10-31T10:00:00.123Z",
  "level": "INFO",
  "logger": "user_service",
  "message": "User created successfully",
  "trace_id": "abc-123-def-456",
  "span_id": "xyz-789",
  "user_id": "123e4567-e89b-12d3-a456-426614174000",
  "email": "u***@example.com",
  "action": "user_registration",
  "duration_ms": 245,
  "context": {
    "ip_address": "192.168.1.1",
    "user_agent": "Mozilla/5.0..."
  }
}
```

#### Logging Implementation
```python
import logging
import json
from datetime import datetime

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            'timestamp': datetime.utcnow().isoformat() + 'Z',
            'level': record.levelname,
            'logger': record.name,
            'message': record.getMessage(),
            'trace_id': getattr(record, 'trace_id', None),
            'user_id': getattr(record, 'user_id', None)
        }
        
        if record.exc_info:
            log_data['exception'] = self.formatException(record.exc_info)
        
        return json.dumps(log_data)

# Usage
logger = logging.getLogger('user_service')
logger.info(
    "User created successfully",
    extra={'user_id': user.id, 'trace_id': trace_id}
)
```

#### PII Redaction
```python
def redact_email(email: str) -> str:
    """Redact email for logging"""
    if '@' in email:
        local, domain = email.split('@')
        return f"{local[0]}***@{domain}"
    return "***"

def redact_sensitive_data(data: dict) -> dict:
    """Redact sensitive fields"""
    sensitive_fields = ['password', 'credit_card', 'ssn', 'token']
    redacted = data.copy()
    
    for field in sensitive_fields:
        if field in redacted:
            redacted[field] = '***REDACTED***'
    
    if 'email' in redacted:
        redacted['email'] = redact_email(redacted['email'])
    
    return redacted
```

---

### 10.3 Distributed Tracing

#### Span Naming Conventions
- HTTP requests: `HTTP {METHOD} {path}`
- Database queries: `DB {operation} {table}`
- External API calls: `EXT {service} {operation}`
- Background jobs: `JOB {job_name}`

**Example:**
```
HTTP POST /api/v1/users
  ├─ DB INSERT users
  ├─ EXT sendgrid send_email
  └─ CACHE SET user:123:profile
```

#### Implementation
```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

async def create_user(data: CreateUserDTO):
    with tracer.start_as_current_span("create_user") as span:
        span.set_attribute("user.email", data.email)
        
        # Database operation
        with tracer.start_as_current_span("DB INSERT users"):
            user = await user_repository.create(data)
        
        # External API call
        with tracer.start_as_current_span("EXT sendgrid send_email"):
            await email_service.send_verification(user.email)
        
        span.set_attribute("user.id", user.id)
        return user
```

#### Required Trace Attributes
- `trace.id`: Unique trace identifier
- `span.id`: Unique span identifier
- `service.name`: Service name
- `http.method`: HTTP method
- `http.status_code`: HTTP status
- `db.system`: Database type
- `db.operation`: SQL operation type

---

## 11. Testing Strategy

### 11.1 Unit Tests

**Coverage Target:** 80% minimum

**Structure:**
```
tests/
├── unit/
│   ├── services/
│   │   ├── test_user_service.py
│   │   └── test_auth_service.py
│   ├── repositories/
│   │   └── test_user_repository.py
│   └── utils/
│       └── test_validators.py
```

**Example Test:**
```python
import pytest
from unittest.mock import Mock, patch
from src.services.user_service import UserService
from src.exceptions import DuplicateEmailException

class TestUserService:
    @pytest.fixture
    def user_repository_mock(self):
        return Mock()
    
    @pytest.fixture
    def email_service_mock(self):
        return Mock()
    
    @pytest.fixture
    def user_service(self, user_repository_mock, email_service_mock):
        return UserService(user_repository_mock, email_service_mock)
    
    def test_create_user_success(self, user_service, user_repository_mock):
        # Arrange
        user_data = {
            'email': 'test@example.com',
            'password': 'SecurePass123',
            'full_name': 'Test User'
        }
        user_repository_mock.exists_by_email.return_value = False
        user_repository_mock.create.return_value = Mock(id='123')
        
        # Act
        result = user_service.create_user(user_data)
        
        # Assert
        assert result.id == '123'
        user_repository_mock.create.assert_called_once()
    
    def test_create_user_duplicate_email(self, user_service, user_repository_mock):
        # Arrange
        user_data = {'email': 'test@example.com', 'password': 'Pass123'}
        user_repository_mock.exists_by_email.return_value = True
        
        # Act & Assert
        with pytest.raises(DuplicateEmailException):
            user_service.create_user(user_data)
```

---

### 11.2 Integration Tests

**Purpose:** Test component interactions

**Example:**
```python
import pytest
from fastapi.testclient import TestClient
from src.main import app

class TestUserAPIIntegration:
    @pytest.fixture
    def client(self):
        return TestClient(app)
    
    @pytest.fixture
    def test_db(self):
        # Setup test database
        setup_test_db()
        yield
        # Teardown
        teardown_test_db()
    
    def test_user_registration_flow(self, client, test_db):
        # Register user
        response = client.post('/api/v1/auth/register', json={
            'email': 'test@example.com',
            'password': 'SecurePass123',
            'full_name': 'Test User',
            'terms_accepted': True
        })
        
        assert response.status_code == 201
        data = response.json()
        assert data['data']['email'] == 'test@example.com'
        
        # Verify database record
        user = get_user_from_db(data['data']['user_id'])
        assert user is not None
        assert user.is_verified == False
```

---

### 11.3 Performance Tests

**Tools:** Locust, Apache JMeter, K6

**Example Locust Test:**
```python
from locust import HttpUser, task, between

class UserBehavior(HttpUser):
    wait_time = between(1, 3)
    
    def on_start(self):
        # Login once
        response = self.client.post('/api/v1/auth/login', json={
            'email': 'test@example.com',
            'password': 'SecurePass123'
        })
        self.token = response.json()['data']['access_token']
        self.headers = {'Authorization': f'Bearer {self.token}'}
    
    @task(3)
    def get_profile(self):
        self.client.get(
            '/api/v1/users/me',
            headers=self.headers,
            name="Get Profile"
        )
    
    @task(1)
    def update_profile(self):
        self.client.put(
            '/api/v1/users/me',
            json={'full_name': 'Updated Name'},
            headers=self.headers,
            name="Update Profile"
        )
```

**Performance Criteria:**
- API response time P95 < 200ms
- Database query time P99 < 100ms
- System can handle 1000 concurrent users
- Error rate < 0.1%

---

## 12. Edge Cases & Failure Modes

### 12.1 Edge Cases

#### Case 1: Concurrent User Creation with Same Email
**Scenario:** Two requests to create user with same email arrive simultaneously

**Handling:**
- Database unique constraint on email column
- Catch IntegrityError in repository layer
- Return DuplicateEmailException
- Transaction rollback ensures no partial data

**Code:**
```python
try:
    user = await db.execute(insert_query)
    await db.commit()
except IntegrityError as e:
    await db.rollback()
    if 'email' in str(e):
        raise DuplicateEmailException()
    raise
```

---

#### Case 2: Token Expiration During Request
**Scenario:** JWT token expires while request is being processed

**Handling:**
- Validate token at start of request
- Return 401 if expired
- Client should refresh token and retry
- Grace period of 5 minutes for token refresh

---

#### Case 3: Large File Upload Timeout
**Scenario:** User uploads 100MB file, connection times out

**Handling:**
- Implement chunked upload
- Store upload progress in Redis
- Support resume from last successful chunk
- Cleanup incomplete uploads after 24 hours

---

### 12.2 Failure Modes

#### Database Connection Pool Exhausted
**Symptoms:** Requests timeout, "connection pool exhausted" errors

**Causes:**
- Too many concurrent requests
- Long-running queries blocking connections
- Connection leaks (not properly closed)

**Mitigation:**
- Increase pool size (with monitoring)
- Implement connection timeout
- Query optimization
- Circuit breaker for database calls

**Recovery:**
- Restart application pods
- Kill long-running queries
- Scale horizontally

---

#### External API Rate Limit Hit
**Symptoms:** 429 errors from third-party API

**Mitigation:**
- Implement exponential backoff
- Queue requests locally
- Cache responses when possible
- Implement circuit breaker

**Code:**
```python
class RateLimitedClient:
    def __init__(self):
        self.circuit_breaker = CircuitBreaker()
    
    async def call_api(self, *args):
        if self.circuit_breaker.is_open():
            raise ServiceUnavailableException()
        
        try:
            response = await self._make_request(*args)
            self.circuit_breaker.record_success()
            return response
        except RateLimitException:
            self.circuit_breaker.record_failure()
            # Implement backoff
            await asyncio.sleep(60)
            raise
```

---

#### Memory Leak in Background Worker
**Symptoms:** Worker memory usage grows over time, eventual OOM kill

**Detection:**
- Monitor memory metrics
- Alert when memory > 80% for sustained period

**Mitigation:**
- Implement worker restart after N jobs processed
- Use memory profiling tools
- Fix leak in code
- Limit job batch size

---

## 13. Deployment Details

### 13.1 Docker Configuration

**Dockerfile:**
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**docker-compose.yml (Local Development):**
```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/appdb
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - db
      - redis
    volumes:
      - ./src:/app/src
  
  db:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: appdb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
  
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
  
  worker:
    build: .
    command: celery -A tasks worker --loglevel=info
    depends_on:
      - redis
      - db

volumes:
  postgres_data:
```

---

### 13.2 Kubernetes Manifests

**Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
  labels:
    app: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: your-registry/api:latest
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
```

**Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
  type: LoadBalancer
```

**HorizontalPodAutoscaler:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-deployment
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

### 13.3 Health Checks

**Liveness Probe (Is the app alive?):**
```python
@app.get("/health")
async def health_check():
    """
    Basic health check - returns 200 if app is running
    """
    return {"status": "healthy"}
```

**Readiness Probe (Is the app ready to serve traffic?):**
```python
@app.get("/ready")
async def readiness_check():
    """
    Checks if app is ready to serve requests
    - Database connection is working
    - Redis connection is working
    - Required services are available
    """
    checks = {}
    
    # Check database
    try:
        await db.execute("SELECT 1")
        checks["database"] = "healthy"
    except Exception as e:
        checks["database"] = f"unhealthy: {str(e)}"
    
    # Check Redis
    try:
        await redis.ping()
        checks["redis"] = "healthy"
    except Exception as e:
        checks["redis"] = f"unhealthy: {str(e)}"
    
    # Check if all services are healthy
    all_healthy = all(v == "healthy" for v in checks.values())
    
    status_code = 200 if all_healthy else 503
    return JSONResponse(
        status_code=status_code,
        content={"status": "ready" if all_healthy else "not_ready", "checks": checks}
    )
```

---

## 14. Versioning & Migration Notes

### 14.1 API Versioning Strategy

**Strategy:** URL-based versioning

**Format:** `/api/v{major}/resource`

**Example:**
- `/api/v1/users`
- `/api/v2/users`

**Version Support Policy:**
- Support current version (v2) and one previous (v1)
- Deprecation notice: 6 months before sunset
- Breaking changes require major version bump

---

### 14.2 Database Migration Strategy

**Tool:** Alembic (for SQLAlchemy)

**Migration Workflow:**
```bash
# Create new migration
alembic revision --autogenerate -m "Add user_profile table"

# Review generated migration
# Edit if needed

# Apply migration
alembic upgrade head

# Rollback if needed
alembic downgrade -1
```

**Migration Example:**
```python
"""Add user_profile table

Revision ID: abc123
Create Date: 2025-10-31 10:00:00
"""

from alembic import op
import sqlalchemy as sa

def upgrade():
    op.create_table(
        'user_profiles',
        sa.Column('id', sa.UUID(), nullable=False),
        sa.Column('user_id', sa.UUID(), nullable=False),
        sa.Column('avatar_url', sa.String(255)),
        sa.Column('bio', sa.Text()),
        sa.Column('preferences', sa.JSON()),
        sa.Column('created_at', sa.TIMESTAMP(), nullable=False),
        sa.Column('updated_at', sa.TIMESTAMP(), nullable=False),
        sa.PrimaryKeyConstraint('id'),
        sa.ForeignKeyConstraint(['user_id'], ['users.id'], ondelete='CASCADE'),
        sa.UniqueConstraint('user_id')
    )
    
    op.create_index('idx_user_profiles_user_id', 'user_profiles', ['user_id'])

def downgrade():
    op.drop_index('idx_user_profiles_user_id')
    op.drop_table('user_profiles')
```

---

### 14.3 Data Migration Guidelines

**For Large Tables:**
1. Add new column with default value
2. Backfill data in batches (off-peak hours)
3. Verify data integrity
4. Update application code
5. Deploy new version
6. Drop old column (after verification period)

**Example Backfill Script:**
```python
async def backfill_user_profiles(batch_size=1000):
    """
    Backfill user_profiles table from users table
    """
    offset = 0
    
    while True:
        # Fetch batch of users
        users = await db.fetch_all(
            "SELECT id, email FROM users OFFSET :offset LIMIT :limit",
            {"offset": offset, "limit": batch_size}
        )
        
        if not users:
            break
        
        # Create profiles
        for user in users:
            await db.execute(
                "INSERT INTO user_profiles (id, user_id, created_at, updated_at) "
                "VALUES (gen_random_uuid(), :user_id, NOW(), NOW()) "
                "ON CONFLICT (user_id) DO NOTHING",
                {"user_id": user['id']}
            )
        
        offset += batch_size
        print(f"Processed {offset} users")
        
        # Sleep to avoid overloading database
        await asyncio.sleep(1)
```

---

### 14.4 Breaking Changes Log

| Version | Date | Change | Migration Path |
|---------|------|--------|----------------|
| v2.0.0 | 2025-11-01 | Changed user ID from integer to UUID | Run migration script, update client code |
| v2.1.0 | 2025-12-01 | Removed `status` field from User model | Use `is_active` field instead |

---

## 15. Security Implementation Details

### 15.1 Input Validation

```python
from pydantic import BaseModel, EmailStr, constr, validator

class RegisterRequest(BaseModel):
    email: EmailStr
    password: constr(min_length=8, max_length=128)
    full_name: constr(min_length=2, max_length=255)
    terms_accepted: bool
    
    @validator('password')
    def password_strength(cls, v):
        if not any(c.isupper() for c in v):
            raise ValueError('Password must contain at least one uppercase letter')
        if not any(c.isdigit() for c in v):
            raise ValueError('Password must contain at least one digit')
        if not any(c in '!@#$%^&*()_+-=' for c in v):
            raise ValueError('Password must contain at least one special character')
        return v
    
    @validator('terms_accepted')
    def terms_must_be_accepted(cls, v):
        if not v:
            raise ValueError('Terms must be accepted')
        return v
```

---

### 15.2 SQL Injection Prevention

**Always use parameterized queries:**
```python
# GOOD - Parameterized query
user = await db.fetch_one(
    "SELECT * FROM users WHERE email = :email",
    {"email": email}
)

# BAD - String concatenation (SQL injection risk)
user = await db.fetch_one(
    f"SELECT * FROM users WHERE email = '{email}'"
)
```

---

### 15.3 XSS Prevention

**Escape user input in responses:**
```python
import html

def sanitize_html(text: str) -> str:
    """Escape HTML special characters"""
    return html.escape(text)

# Usage
user_input = "<script>alert('XSS')</script>"
safe_output = sanitize_html(user_input)
# Result: &lt;script&gt;alert(&#x27;XSS&#x27;)&lt;/script&gt;
```

---

### 15.4 CORS Configuration

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://yourdomain.com",
        "https://app.yourdomain.com"
    ],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
    max_age=3600
)
```

---

## Appendix A: Common Code Patterns

### Pattern 1: Repository Pattern
[Include examples]

### Pattern 2: Service Layer Pattern
[Include examples]

### Pattern 3: Dependency Injection
[Include examples]

---

## Appendix B: Development Setup Guide

[Quick start guide for new developers]

1. Clone repository
2. Install dependencies
3. Setup local database
4. Run migrations
5. Start development server

---

## Appendix C: Troubleshooting Guide

| Issue | Possible Cause | Solution |
|-------|----------------|----------|
| Database connection fails | Wrong credentials | Check DATABASE_URL |
| Redis connection timeout | Redis not running | Start Redis service |
| Import errors | Missing dependencies | Run `pip install -r requirements.txt` |