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

### 2.1 [Component Name - e.g., "User Management Component"]

#### 2.1.1 Overview
[Brief description of what this component does]

#### 2.1.2 Structure

**Directory Layout:**
```
backend/
├── Dockerfile
├── app/
│   ├── api.py
│   ├── {project_name}/
│   │   ├── __init__.py
│   │   ├── database/
│   │   │   ├── __init__.py
│   │   │   ├── postgres/
│   │   │   │   ├── __init__.py
│   │   │   │   ├── post_config.py
│   │   │   │   └── post_utils.py
│   │   │   └── redis/
│   │   │       ├── __init__.py
│   │   │       └── redis_config.py
│   │   ├── handler/
│   │   │   ├── __init__.py
│   │   │   └── exception.py
│   │   ├── logger/
│   │   │   ├── __init__.py
│   │   │   └── logging_tool.py
│   │   ├── {library_name}/
│   │   │   ├── __init__.py
│   │   │   └── services/
│   │   │       ├── __init__.py
│   │   │       ├── {service_name}/
│   │   │       │   ├── __init__.py
│   │   │       │   └── {service_name}_process.py
│   │   │       └── common/
│   │   │           ├── __init__.py
│   │   │           └── utils.py
│   │   └── task_queue/
│   │       ├── __init__.py
│   │       └── celery_q/
│   │           ├── __init__.py
│   │           ├── celery_config.py
│   │           └── tasks.py
│   ├── requirements.txt
│   └── start.sh
└── docker-compose.yml
```

#### 2.1.3 Controllers

**Class: UserController**
```
Purpose: Handle HTTP requests for user operations
Location: backend/app/api.py or backend/app/{project_name}/{library_name}/services/{service_name}/{service_name}_process.py

class UserController:
    Methods:
    - register(request: RegisterRequest) -> UserResponse
    - get_user(user_id: str) -> UserResponse
    - update_user(user_id: str, request: UpdateUserRequest) -> UserResponse
    - delete_user(user_id: str) -> StatusResponse
```

**Method Details: register()**
```
Input: RegisterRequest (email, password, full_name, terms_accepted)
Output: UserResponse (user_id, email, full_name, created_at)
Validation:
  - Email format validation
  - Password strength check (min 8 chars, 1 uppercase, 1 number)
  - Duplicate email check
Process:
  1. Validate input data
  2. Call UserService.create_user()
  3. Handle exceptions
  4. Return formatted response
Error Handling:
  - DuplicateEmailException → 409 Conflict
  - ValidationException → 400 Bad Request
```

---

#### 2.1.4 Services (Business Logic)

**Class: UserService**
```
Purpose: Implement user management business logic
Location: backend/app/{project_name}/{library_name}/services/{service_name}/{service_name}_process.py

Dependencies:
- Database utilities (post_utils.py, redis_config.py)
- Logger (logging_tool.py)
- Exception handler (handler/exception.py)

Methods:
- create_user(data: CreateUserDTO) -> User
- get_user_by_id(user_id: str) -> User
- update_user(user_id: str, data: UpdateUserDTO) -> User
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
        6. Return user object
    
    Transaction: Yes (atomic)
    """
```

---

#### 2.1.5 Repositories (Data Access)

**Class: UserRepository**
```
Purpose: Database operations for User entity
Location: backend/app/{project_name}/database/postgres/post_utils.py

Methods:
- create(user: User) -> User
- find_by_id(user_id: str) -> Optional[User]
- find_by_email(email: str) -> Optional[User]
- update(user_id: str, updates: dict) -> User
- delete(user_id: str) -> bool
```

**Query Examples:**
```sql
-- Find by email
SELECT * FROM users WHERE email = %s AND deleted_at IS NULL;

-- Create user
INSERT INTO users (id, email, password_hash, full_name, created_at, updated_at)
VALUES (%s, %s, %s, %s, NOW(), NOW())
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
```

---

### 2.2 Workers & Background Jobs

#### Worker: EmailWorker
```
Purpose: Process email sending jobs from queue
Location: backend/app/{project_name}/task_queue/celery_q/tasks.py
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
```

---

### 2.3 Agents & AI Components (if applicable)

#### Agent: [Agent Name - e.g., "CustomerSupportAgent"]
```
Purpose: Handle customer queries using AI
Location: backend/app/{project_name}/{library_name}/services/{service_name}/{service_name}_process.py
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
    }
    
    class UserController {
        -UserService service
        +register(request) Response
        +login(request) Response
        +get_profile(user_id) Response
    }
    
    UserRepository --> User : manages
    UserService --> UserRepository : uses
    UserController --> UserService : uses
```

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
    
    User->>Frontend: Fill registration form
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
    
    UserService->>EmailQueue: queue email job
    EmailQueue-->>UserService: ack
    
    UserService-->>AuthController: user
    AuthController-->>APIGateway: 201 Created
    APIGateway-->>Frontend: success response
    Frontend-->>User: Show success message
```

---

## 5. Data Model Details

### 5.1 Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    USERS ||--o{ SESSIONS : has
    USERS ||--o| USER_PROFILES : has
    
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

-- Triggers
CREATE TRIGGER update_users_updated_at
BEFORE UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();
```

### 5.3 Index Strategy

**Primary Indexes:**
- All primary keys (automatic B-tree indexes)
- Foreign keys for join optimization

**Query-Specific Indexes:**
```sql
-- For user login queries
CREATE INDEX idx_users_email_active ON users(email) 
WHERE is_active = TRUE AND deleted_at IS NULL;

-- Composite indexes
CREATE INDEX idx_users_created_active ON users(created_at, is_active);
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
```

**TTL Strategy:**
- Sessions: 24 hours
- Rate limit counters: 60 seconds
- Cache: 5-60 minutes depending on data volatility

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

### 6.2 [Other API Groups]

[Add more API specifications following the same structure]

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

### 7.2 [Other Algorithms - e.g., "JWT Token Generation", "Recommendation Engine"]

[Add algorithm details]

---

## 8. Configurations

### 8.1 Environment Variables

| Variable | Description | Example | Default | Required |
|----------|-------------|---------|---------|----------|
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://user:pass@host:5432/db` | - | Yes |
| `REDIS_URL` | Redis connection string | `redis://localhost:6379/0` | - | Yes |
| `JWT_SECRET_KEY` | Secret key for JWT signing | `your-secret-key-here` | - | Yes |
| `JWT_EXPIRATION_HOURS` | JWT token validity period | `24` | `1` | No |
| `LOG_LEVEL` | Logging level | `INFO` | `INFO` | No |

---

### 8.2 Feature Flags

| Flag Name | Description | Default | Impact |
|-----------|-------------|---------|---------|
| `enable_new_ui` | Enable redesigned UI | `false` | Frontend |
| `maintenance_mode` | Activate maintenance mode | `false` | All |

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
        value = redis.get(key)
        return value == "true" if value else False
```

---

### 8.3 Message Queue Configuration

#### RabbitMQ Queues

| Queue Name | Purpose | Durability | TTL | DLQ |
|------------|---------|-----------|-----|-----|
| `email_queue` | Email sending | Durable | 1 hour | `email_dlq` |
| `notification_queue` | Push notifications | Durable | 30 min | `notification_dlq` |

---

## 9. Error Handling Strategy

### 9.1 Error Code Schema

**Format:** `{MODULE}_{TYPE}_{NUMBER}`

**Examples:**
- `AUTH_INVALID_CREDENTIALS_001`
- `USER_NOT_FOUND_002`

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
  "user_id": "123e4567-e89b-12d3-a456-426614174000"
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
```

---

### 10.3 Distributed Tracing

#### Span Naming Conventions
- HTTP requests: `HTTP {METHOD} {path}`
- Database queries: `DB {operation} {table}`
- External API calls: `EXT {service} {operation}`

**Example:**
```
HTTP POST /api/v1/users
  ├─ DB INSERT users
  └─ CACHE SET user:123:profile
```

---

## 11. Testing Strategy

### 11.1 Unit Tests

**Coverage Target:** 80% minimum

**Example Test:**
```python
import pytest
from unittest.mock import Mock
from backend.app.{project_name}.{library_name}.services.{service_name}.{service_name}_process import UserService
from backend.app.{project_name}.handler.exception import DuplicateEmailException

class TestUserService:
    @pytest.fixture
    def user_repository_mock(self):
        return Mock()
    
    @pytest.fixture
    def user_service(self, user_repository_mock):
        return UserService(user_repository_mock)
    
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
```

---

### 11.2 Integration Tests

**Purpose:** Test component interactions

**Example:**
```python
import pytest
from fastapi.testclient import TestClient
from backend.app.api import app

class TestUserAPIIntegration:
    @pytest.fixture
    def client(self):
        return TestClient(app)
    
    def test_user_registration_flow(self, client):
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
```

---

### 11.3 Performance Tests

**Tools:** Locust, Apache JMeter, K6

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

### 12.2 Failure Modes

#### Database Connection Pool Exhausted
**Symptoms:** Requests timeout, "connection pool exhausted" errors

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

## 13. Deployment Details

### 13.1 Docker Configuration

**Dockerfile:**
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY backend/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY backend/ .

# Create non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

EXPOSE 8000

CMD ["uvicorn", "app.api:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

### 13.2 Health Checks

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

# Apply migration
alembic upgrade head

# Rollback if needed
alembic downgrade -1
```

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

### 15.3 CORS Configuration

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

## Appendix A: Development Setup Guide

[Quick start guide for new developers]

1. Clone repository
2. Install dependencies
3. Setup local database
4. Run migrations
5. Start development server

---

## Appendix B: Troubleshooting Guide

| Issue | Possible Cause | Solution |
|-------|----------------|----------|
| Database connection fails | Wrong credentials | Check DATABASE_URL |
| Redis connection timeout | Redis not running | Start Redis service |
| Import errors | Missing dependencies | Run `pip install -r requirements.txt` |
