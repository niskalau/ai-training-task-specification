# Technical Specification

## 1. Overview

The **Dynamic Form Application** is a full-stack monolithic Spring Boot application that enables users to create, manage, and submit dynamic forms. The application follows a modern architecture with a React frontend packaged as a WebJar and served by a Spring Boot backend.

### Key Characteristics

- **Architecture Type**: Monolithic Spring Boot application
- **Deployment Model**: Single executable JAR containing both backend and frontend
- **Authentication**: OAuth 2.0 Resource Server with JWT tokens
- **Database**: PostgreSQL with JSON column support for flexible form storage

## 2. Technology Stack

### 2.1 Backend Stack

| Component | Technology |
|-----------|-----------|
| Language | Java 21 |
| Framework | Spring Boot |
| Build Tool | Maven |
| Database (Production) | PostgreSQL |
| Database (Dev/Test) | H2 |
| ORM | Spring Data JPA |
| Security | Spring Security OAuth2 |
| DTO Mapping | MapStruct |
| Code Generation | Lombok |
| Logging | SLF4J + Logback |

**Spring Boot Dependencies**:
- `spring-boot-starter-web` - REST API
- `spring-boot-starter-data-jpa` - Database access
- `spring-boot-starter-security` - Security framework
- `spring-boot-starter-oauth2-resource-server` - JWT validation
- `spring-boot-starter-validation` - Bean validation

**Testing Dependencies**:
- JUnit 5 - Test framework
- Mockito - Mocking
- Spring Boot Test - Integration testing
- Testcontainers 2.0.3 - Container-based testing
- H2 Database - In-memory testing
- Selenium 4.31.0 - UI testing

### 2.2 Frontend Stack

| Component | Technology |
|-----------|-----------|
| Language | TypeScript |
| Framework | React |
| Build Tool | Vite |
| Package Manager | npm |
| Node.js | Node.js |
| UI Framework | Bootstrap |
| UI Components | react-bootstrap |

**Core Libraries**:
- `react-router-dom` - Routing
- `react-hook-form` - Form management
- `zod` - Schema validation
- `@tanstack/react-query` - Server state management
- `react-oidc-context` - OAuth authentication
- `react-i18next` - Internationalization
- `i18next` - i18n core
- `lucide-react` - Icons

**Testing Libraries**:
- `vitest` - Test runner
- `@testing-library/react` - React testing utilities
- `@vitest/coverage-v8` - Code coverage

**Build Configuration**:
- `@vitejs/plugin-react` - React plugin for Vite
- `typescript-eslint` - TypeScript linting
- `prettier` - Code formatting

### 2.3 Build and Packaging

- **Frontend Build**: Maven Frontend Plugin
  - Downloads Node.js 24.13.0
  - Runs npm install and build
  - Packages as WebJar in `META-INF/resources/webjars/frontend/1.0.0/`
- **Backend Build**: Maven
  - Includes frontend WebJar as dependency
  - Creates executable JAR with embedded Tomcat
  - Final artifact: `backend-1.0.0.jar`

## 3. System Architecture

### 3.1 Architecture Pattern

**Monolithic Layered Architecture**:

```
┌─────────────────────────────────────────┐
│         Browser (User Interface)        │
└──────────────────┬──────────────────────┘
                   │ HTTP/REST
┌──────────────────▼──────────────────────┐
│         Spring Boot Application         │
│  ┌───────────────────────────────────┐  │
│  │   React Frontend (WebJar)         │  │
│  │   - Static files in /webjars/     │  │
│  └───────────────────────────────────┘  │
│  ┌───────────────────────────────────┐  │
│  │   REST API Layer                  │  │
│  │   - Controllers (/api/*)          │  │
│  └───────────┬───────────────────────┘  │
│  ┌───────────▼───────────────────────┐  │
│  │   Service Layer                   │  │
│  │   - Business Logic                │  │
│  └───────────┬───────────────────────┘  │
│  ┌───────────▼───────────────────────┐  │
│  │   Repository Layer                │  │
│  │   - Spring Data JPA               │  │
│  └───────────┬───────────────────────┘  │
└──────────────┼──────────────────────────┘
               │ JDBC
┌──────────────▼──────────────────────────┐
│      PostgreSQL Database                │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│   OAuth 2.0 Authorization Server        │
│   (External - localhost:9000)           │
└─────────────────────────────────────────┘
```

### 3.2 Module Structure

The project uses Maven multi-module structure:

```
dynamic-form/                    (parent POM)
├── backend/                     (Spring Boot application)
│   ├── src/main/java/
│   │   └── com/example/backend/
│   │       ├── BackendApplication.java
│   │       ├── config/          (Configuration classes)
│   │       ├── controller/      (REST controllers)
│   │       ├── dto/             (Data Transfer Objects)
│   │       ├── entity/          (JPA entities)
│   │       ├── mapper/          (MapStruct mappers)
│   │       ├── repository/      (Spring Data repositories)
│   │       └── service/         (Business logic)
│   ├── src/main/resources/
│   │   ├── application.yaml
│   │   ├── application-dev.yaml
│   │   ├── application-test.yaml
│   │   ├── forms/               (Initial form definitions)
│   │   └── logback-spring.xml
│   └── pom.xml
└── frontend/                    (React application)
    ├── src/
    │   ├── components/          (React components)
    │   ├── context/             (React contexts)
    │   ├── pages/               (Route pages)
    │   ├── services/            (API clients)
    │   ├── types/               (TypeScript types)
    │   ├── App.tsx
    │   ├── main.tsx
    │   └── i18n.ts
    ├── public/
    │   └── locales/             (i18n translations)
    ├── package.json
    ├── vite.config.ts
    └── pom.xml                  (WebJar packaging)
```

### 3.3 Backend Architecture

#### Layer Responsibilities

**Controller Layer** (`*.controller.*`)
- Handles HTTP requests/responses
- Input validation (Bean Validation)
- Maps DTOs to domain objects
- Security annotations (`@PreAuthorize`)
- Exception handling via `@RestControllerAdvice`

**Service Layer** (`*.service.*`)
- Business logic implementation
- Transaction management
- Domain validation
- Throws standard Java exceptions:
  - `NoSuchElementException` for not-found scenarios
  - `IllegalArgumentException` for invalid input
  - `IllegalStateException` for invalid state

**Repository Layer** (`*.repository.*`)
- Data access through Spring Data JPA
- Extends `JpaRepository<Entity, ID>`
- Custom query methods using method naming conventions

**Entity Layer** (`*.entity.*`)
- JPA entities with Hibernate annotations
- Uses Lombok for boilerplate reduction
- JSON columns for flexible schema (form fields, submission data)

**DTO Layer** (`*.dto.*`)
- Data Transfer Objects for API contracts
- Bean Validation annotations
- Immutable using Lombok `@Value` where appropriate

**Mapper Layer** (`*.mapper.*`)
- MapStruct interfaces for entity-DTO conversion
- Component model: Spring
- Automatic dependency injection

#### Configuration Classes

**SecurityConfig** (`config/SecurityConfig.java`)
- Configures Spring Security filter chain
- OAuth2 Resource Server setup
- CORS configuration
- Public/authenticated endpoint rules
- JWT token validation

**DatabaseInitializer** (`config/DatabaseInitializer.java`)
- Implements `CommandLineRunner`
- Loads initial form definitions from JSON files
- Runs on application startup

### 3.4 Frontend Architecture

#### Component Structure

```
src/
├── components/
│   ├── common/              (Reusable components)
│   │   ├── DynamicForm.tsx
│   │   ├── ReadOnlyDynamicForm.tsx
│   │   └── field/           (Form field components)
│   ├── layout/
│   │   └── Navigation.tsx
│   └── Content.tsx          (Router and layout)
├── pages/                   (Route components)
│   ├── Forms.tsx
│   ├── FormSubmission.tsx
│   ├── FormSubmissions.tsx
│   ├── SubmissionDetail.tsx
│   └── EditForm.tsx
├── services/                (API layer)
│   ├── http.ts              (Base HTTP client)
│   └── formClient.ts        (Form API methods)
├── context/
│   └── oidcConfig.tsx       (OAuth configuration)
├── types/
│   └── Form.ts              (TypeScript interfaces)
├── App.tsx                  (Root component)
├── main.tsx                 (Entry point)
└── i18n.ts                  (i18n setup)
```

#### State Management

**Server State** (TanStack React Query):
- Caching and synchronization with backend
- Automatic refetching and cache invalidation
- Query keys: `['forms']`, `['form', formKey]`, `['form-submissions']`
- Optimistic updates

**Form State** (react-hook-form):
- Local form field state
- Field validation
- Form submission handling
- Error state management

**Authentication State** (react-oidc-context):
- OAuth2/OIDC authentication flow
- Token management
- User profile information

**Local UI State** (React useState):
- Component-level UI state
- Modal visibility
- Loading states

#### Routing

Routes configured in `Content.tsx` using react-router-dom v7:

| Path | Component | Description |
|------|-----------|-------------|
| `/` | Forms | Home page - list of forms |
| `/forms` | Forms | Forms list |
| `/forms/:formKey` | FormSubmission | Submit new form |
| `/forms/:formKey/submissions/:id/edit` | FormSubmission | Edit submission |
| `/forms/submissions/:id` | SubmissionDetail | View submission |
| `/create-form` | EditForm | Create form definition |
| `/submissions` | FormSubmissions | All submissions |

## 4. Data Model

### 4.1 Database Schema

#### Form Entity

```sql
CREATE TABLE form (
    id BIGSERIAL PRIMARY KEY,
    form_key VARCHAR(255) NOT NULL UNIQUE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    fields JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Fields JSON Structure**:
```json
[
  {
    "name": "fieldName",
    "label": "Field Label",
    "type": "text|email|tel|number|date|textarea|select|radio|checkbox",
    "required": true,
    "placeholder": "Enter value...",
    "options": [
      {"value": "opt1", "label": "Option 1"},
      {"value": "opt2", "label": "Option 2"}
    ]
  }
]
```

#### FormData Entity

```sql
CREATE TABLE form_data (
    id BIGSERIAL PRIMARY KEY,
    form_key VARCHAR(255) NOT NULL,
    data JSONB NOT NULL,
    submitted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    submitted_by VARCHAR(255) NOT NULL
);
```

**Data JSON Structure**:
```json
{
  "fieldName1": "value1",
  "fieldName2": "value2",
  "selectField": "selectedOption",
  "checkboxField": true
}
```

### 4.2 Domain Model

#### Core Entities

**Form** (`com.example.backend.entity.Form`)
- Represents a form definition/template
- Contains field definitions as JSON
- Uniquely identified by `formKey`
- Tracks creation and update timestamps

**Field** (Embedded in Form JSON)
- Field metadata (name, label, type)
- Validation rules (required)
- UI hints (placeholder)
- Options for select/radio fields

**FormData** (`com.example.backend.entity.FormData`)
- Represents a form submission
- Links to form via `formKey` (soft reference)
- Stores submission data as JSON for flexibility
- Tracks submitter and timestamp

### 4.3 Field Types

Supported field types:

| Type | HTML Input | Validation | Options |
|------|-----------|------------|---------|
| text | text | String | No |
| email | email | Email format | No |
| tel | tel | Phone format | No |
| number | number | Numeric | No |
| date | date | Date format | No |
| textarea | textarea | String | No |
| select | select | One of options | Yes |
| radio | radio | One of options | Yes |
| checkbox | checkbox | Boolean | No |

## 5. API Specification

### 5.1 Base URL

- **Production**: `http://localhost:8080/api`
- **Development**: `http://localhost:8080/api` (backend) or proxy from `http://localhost:5173` (frontend dev server)

### 5.2 Authentication

All API requests (except `/api/forms` GET) require authentication via JWT token:

```
Authorization: Bearer <jwt_token>
```

Token obtained from OAuth 2.0 Authorization Server at `http://localhost:9000`.

### 5.3 Form Endpoints

#### Get Available Forms

```http
GET /api/forms
```

**Authentication**: Public

**Response**: `200 OK`
```json
[
  {
    "formKey": "contact",
    "title": "Contact Form"
  }
]
```

#### Get All Forms (Full Details)

```http
GET /api/forms/all
```

**Authentication**: Required

**Response**: `200 OK`
```json
[
  {
    "id": 1,
    "formKey": "contact",
    "title": "Contact Form",
    "description": "Contact us",
    "fields": [...]
  }
]
```

#### Get Form by Key

```http
GET /api/forms/{formKey}
```

**Authentication**: Required

**Response**: `200 OK` or `404 Not Found`

#### Create Form

```http
POST /api/forms
```

**Authentication**: Required (ROLE_ADMIN)

**Request Body**:
```json
{
  "formKey": "feedback",
  "title": "Feedback Form",
  "description": "Share your feedback",
  "fields": [
    {
      "name": "name",
      "label": "Name",
      "type": "text",
      "required": true,
      "placeholder": "Your name"
    }
  ]
}
```

**Response**: `201 Created` or `400 Bad Request`

#### Update Form

```http
PUT /api/forms/{formKey}
```

**Authentication**: Required (ROLE_ADMIN)

**Response**: `200 OK` or `404 Not Found`

#### Delete Form

```http
DELETE /api/forms/{formKey}
```

**Authentication**: Required (ROLE_ADMIN)

**Response**: `204 No Content` or `404 Not Found`

### 5.4 Form Data (Submission) Endpoints

#### Submit Form

```http
POST /api/form-data/{formKey}
```

**Authentication**: Required

**Request Body**:
```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "message": "Hello"
}
```

**Response**: `201 Created`
```json
{
  "id": 1,
  "formKey": "contact",
  "data": {...},
  "submittedAt": "2024-01-15T10:30:00",
  "submittedBy": "john.doe"
}
```

#### Update Submission

```http
PUT /api/form-data/submission/{id}
```

**Authentication**: Required

**Response**: `200 OK` or `404 Not Found`

#### Get All Submissions

```http
GET /api/form-data
```

**Authentication**: Required

**Response**: `200 OK`

#### Get Submissions by Form Key

```http
GET /api/form-data/{formKey}
```

**Authentication**: Required

**Response**: `200 OK`

#### Get Submission by ID

```http
GET /api/form-data/submission/{id}
```

**Authentication**: Required

**Response**: `200 OK` or `404 Not Found`

#### Delete Submission

```http
DELETE /api/form-data/submission/{id}
```

**Authentication**: Required (ROLE_ADMIN)

**Response**: `204 No Content` or `404 Not Found`

## 6. Security

### 6.1 Authentication Flow

1. User accesses application
2. Frontend redirects to OAuth 2.0 Authorization Server (`http://localhost:9000`)
3. User authenticates with credentials
4. Authorization server redirects back with authorization code
5. Frontend exchanges code for JWT access token
6. Frontend includes token in `Authorization` header for API calls
7. Backend validates JWT signature and claims

### 6.2 Authorization Rules

**Public Endpoints**:
- `GET /api/forms` - List available forms
- `/`, `/index.html`, `/webjars/**` - Frontend static files
- `/h2-console/**` - H2 console (dev only)

**Authenticated Endpoints**:
- All other `/api/**` endpoints require valid JWT

**Admin Endpoints** (require `ROLE_ADMIN`):
- `POST /api/forms` - Create form
- `PUT /api/forms/{key}` - Update form
- `DELETE /api/forms/{key}` - Delete form
- `DELETE /api/form-data/submission/{id}` - Delete submission

### 6.3 CORS Configuration

Allowed origins:
- `http://localhost:8080` - Production mode
- `http://localhost:5173` - Frontend dev server
- `http://localhost:9000` - Authorization server

Allowed methods: `GET`, `POST`, `PUT`, `DELETE`, `OPTIONS`

Credentials: Allowed

### 6.4 JWT Token Structure

```json
{
  "sub": "username",
  "iss": "http://localhost:9000",
  "aud": "frontend-client",
  "exp": 1234567890,
  "iat": 1234567890,
  "scope": "openid profile"
}
```

## 7. Configuration

### 7.1 Logging Configuration

**logback-spring.xml**:
- Console appender with pattern
- Rolling file appender (optional)
- Level: INFO (default), DEBUG (dev profile)
- Specific loggers:
  - `com.example.backend` - DEBUG
  - `org.springframework.web` - DEBUG (dev only)
  - `org.hibernate.SQL` - DEBUG (dev only)

## 8. Build and Deployment

### 8.1 Development Setup

**Prerequisites**:
- Java 21+
- Maven 3.9+
- Node.js 24.13.0+ (for local frontend development)
- PostgreSQL 14+ (for production profile)
- OAuth 2.0 Authorization Server running on port 9000

**Backend Development**:
```bash
cd backend
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

**Frontend Development**:
```bash
cd frontend
npm install
npm run dev
```

**Full Build**:
```bash
mvn clean install
```

### 8.2 Production Build

**Build Process**:
1. Frontend Maven plugin downloads Node.js
2. Runs `npm run ci` (install, build, test)
3. Packages frontend as WebJar
4. Backend includes WebJar as dependency
5. Creates executable JAR with embedded Tomcat

**Output**: `backend/target/backend-1.0.0.jar`

### 8.3 Production Deployment

**Run Application**:
```bash
java -jar backend/target/backend-1.0.0.jar
```

**Environment Variables**:
```bash
SPRING_DATASOURCE_URL=jdbc:postgresql://db-host:5432/dbname
SPRING_DATASOURCE_USERNAME=dbuser
SPRING_DATASOURCE_PASSWORD=dbpass
SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI=https://auth-server
SERVER_PORT=8080
```

**Database Setup**:
```sql
CREATE DATABASE template;
-- Tables auto-created by Hibernate (ddl-auto: update)
```

### 8.4 Testing

**Backend Tests**:
```bash
cd backend
mvn test              # Unit tests
mvn verify            # Integration tests
```

**Frontend Tests**:
```bash
cd frontend
npm test              # Unit tests
npm run test:coverage # With coverage
```

## 9. Internationalization (i18n)

### 9.1 Backend

Currently, backend does not implement i18n. All error messages and responses are in English.

**Future Enhancement**: Use `MessageSource` with resource bundles for multi-language support.

### 9.2 Frontend

**Framework**: react-i18next with i18next

**Configuration** (`src/i18n.ts`):
- Backend: HTTP backend loading from `/locales/{lng}/translation.json`
- Language detection: Browser language
- Fallback language: English (en)
- Interpolation: Enabled

**Translation Files**:
- `frontend/public/locales/en/translation.json` - English translations
- Additional languages can be added in `public/locales/{lng}/translation.json`

**Usage**:
```typescript
import { useTranslation } from 'react-i18next';

function Component() {
  const { t } = useTranslation();
  return <h1>{t('forms.title')}</h1>;
}
```

**Translation Keys**:
- `navigation.*` - Navigation labels
- `forms.*` - Forms list page
- `submissions.*` - Submissions list page
- `submissionDetail.*` - Submission detail page
- `form.*` - Form submission page
- `common.*` - Common labels (submit, cancel, etc.)

## 10. Performance Considerations

### 10.1 Backend

**Database**:
- Hibernate second-level cache (can be enabled)
- Connection pooling via HikariCP (Spring Boot default)
- JSON column indexing for frequent queries

**API**:
- Stateless authentication (no session overhead)
- DTO projection to avoid over-fetching

### 10.2 Frontend

**Bundle Optimization**:
- Vite for fast builds and optimized bundles
- Code splitting by route (lazy loading)
- Tree shaking for unused code elimination

**Runtime Performance**:
- React Query caching reduces redundant API calls
- React.memo for expensive components
- useCallback/useMemo for optimization
- Virtualization for long lists (can be added)

**Network**:
- Gzip compression (via Spring Boot)
- HTTP/2 support (via embedded Tomcat)
- Static asset caching headers

## 11. Extensibility

### 11.1 Adding New Field Types

**Backend**:
1. Update `Field` class with new type constant
2. No code changes needed (JSON storage is flexible)

**Frontend**:
1. Add type to `FormField` interface in `types/Form.ts`
2. Create new field component in `components/common/field/`
3. Update `DynamicForm.tsx` to render new field type
4. Update `ReadOnlyDynamicForm.tsx` for display
5. Add validation in form schema

### 11.2 Adding New Languages

**Frontend**:
1. Create translation file: `frontend/public/locales/{lng}/translation.json`
2. Copy structure from `en/translation.json`
3. Translate all values
4. i18next will auto-detect and load

### 11.3 Custom Validation Rules

**Backend**:
- Add Bean Validation custom annotations
- Implement `ConstraintValidator`
- Apply to DTOs

**Frontend**:
- Extend Zod schema with custom validators
- Apply in form schema generation

## 12. Monitoring and Observability

### 12.1 Logging

- SLF4J with Logback
- Structured logging with MDC for request correlation
- Log levels configurable per package
- Rolling file appenders for production

### 12.2 Metrics (Future Enhancement)

Spring Boot Actuator endpoints:
- `/actuator/health` - Health checks
- `/actuator/metrics` - Application metrics
- `/actuator/info` - Application info

### 12.3 Distributed Tracing (Future Enhancement)

- Spring Cloud Sleuth for trace/span IDs
- Export to Zipkin/Jaeger

## 13. Known Limitations

1. **Form-FormData Relationship**: Soft reference via `formKey` string, not foreign key constraint
3. **No File Upload**: Field types limited to text-based inputs
4. **Single Language Backend**: Error messages not internationalized
5. **No Audit Trail**: No history of form/submission changes
6. **Authorization Server**: External dependency, not included in this project
7. **No Validation on Submission**: Backend doesn't validate submission data against form definition

## 14. Future Enhancements

1. **Field Type Extensions**: File upload, rich text editor, multi-select
2. **Conditional Logic**: Show/hide fields based on other field values
3. **Form Versioning**: Track form definition versions
4. **Advanced Validation**: Cross-field validation, regex patterns
5. **Export/Import**: Export submissions to CSV/Excel
6. **Analytics**: Form completion rates, field insights
7. **Workflows**: Multi-step forms, approval flows
8. **Notifications**: Email notifications on submission
9. **API Rate Limiting**: Prevent abuse
10. **Real-time Collaboration**: Multiple users editing forms simultaneously

## 15. References

- [Spring Boot Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [React Documentation](https://react.dev/)
- [TypeScript Documentation](https://www.typescriptlang.org/docs/)
- [PostgreSQL JSON Support](https://www.postgresql.org/docs/current/datatype-json.html)
- [OAuth 2.0 RFC](https://oauth.net/2/)
- [OpenAPI Specification](https://swagger.io/specification/)
