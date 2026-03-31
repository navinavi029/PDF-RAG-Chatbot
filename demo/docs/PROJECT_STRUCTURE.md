# Project Structure Guide

Complete reference for understanding the organization and purpose of every folder and file in the PDF RAG Chatbot project.

## Root Directory

```
Testing RAG - Enhanced/
├── .claude/                    # Claude AI configuration
├── .git/                       # Git version control
├── demo/                       # Main Spring Boot application
├── .gitignore                  # Git ignore patterns
├── API_KEY_QUICK_REFERENCE.txt # Quick guide for API key setup
├── CREATE_CONFIG_MANUALLY.bat  # Windows batch script for config
├── README.md                   # Main project documentation
├── RUN_FROM_POWERSHELL.ps1    # PowerShell startup script
└── START_APP.bat              # Windows batch startup script
```

### Root Files Explained

| File | Purpose |
|------|---------|
| `API_KEY_QUICK_REFERENCE.txt` | Step-by-step guide for obtaining and configuring NVIDIA API key |
| `CREATE_CONFIG_MANUALLY.bat` | Windows helper to create application.properties with API key |
| `README.md` | Main documentation with quick start, features, and architecture |
| `RUN_FROM_POWERSHELL.ps1` | PowerShell script to run the application on Windows |
| `START_APP.bat` | Simple batch file to start the application on Windows |
| `.gitignore` | Specifies files/folders Git should ignore (logs, builds, etc.) |

---

## Demo Directory (Main Application)

```
demo/
├── .mvn/                       # Maven wrapper configuration
├── docs/                       # Comprehensive documentation
├── src/                        # Source code
├── target/                     # Compiled output (generated)
├── .gitattributes             # Git line ending configuration
├── .gitignore                 # Demo-specific Git ignore patterns
├── mvnw                       # Maven wrapper script (Linux/Mac)
├── mvnw.cmd                   # Maven wrapper script (Windows)
├── pom.xml                    # Maven project configuration
├── setup-db.sql               # Database initialization script
└── TESTING_CHECKLIST.md       # Comprehensive testing scenarios
```

### Demo Root Files

| File | Purpose |
|------|---------|
| `pom.xml` | Maven configuration: dependencies, plugins, build settings |
| `setup-db.sql` | Creates PostgreSQL database, enables pgvector, creates tables |
| `TESTING_CHECKLIST.md` | Manual testing scenarios for all features |
| `mvnw` / `mvnw.cmd` | Maven wrapper - runs Maven without system installation |

---

## Documentation (demo/docs/)

```
demo/docs/
├── 01-BASICS.md              # Part 1: RAG fundamentals for beginners
├── 02-INTERMEDIATE.md        # Part 2: Async, caching, resilience patterns
├── 03-ADVANCED.md            # Part 3: Vector search, optimization, scaling
└── 04-REFERENCE.md           # Part 4: Quick reference, APIs, troubleshooting
```

### Documentation Files

| File | Audience | Content |
|------|----------|---------|
| `01-BASICS.md` | Beginners | What is RAG, basic concepts, simple examples |
| `02-INTERMEDIATE.md` | Developers | Async processing, caching, circuit breakers, security |
| `03-ADVANCED.md` | Advanced | Vector similarity, performance tuning, production deployment |
| `04-REFERENCE.md` | All | Quick reference: APIs, configs, commands, troubleshooting |

---

## Source Code (demo/src/main/)

```
demo/src/main/
├── java/com/example/demo/     # Java source code
└── resources/                 # Configuration files
```

---

## Java Source Code Structure

```
demo/src/main/java/com/example/demo/
├── config/                    # Configuration classes
├── controller/                # REST API endpoints
├── dto/                       # Data Transfer Objects
├── exception/                 # Custom exceptions and handlers
├── model/                     # JPA entities (database models)
├── repository/                # Database access layer
├── service/                   # Business logic
└── DemoApplication.java       # Main application entry point
```

### Package Breakdown

#### 1. Config Package (`config/`)

Configuration classes that set up various aspects of the application.

| File | Purpose |
|------|---------|
| `AppConfig.java` | General application configuration beans |
| `AsyncConfig.java` | Thread pool configuration for async processing |
| `CacheConfig.java` | Caffeine cache configuration (query embeddings, responses) |
| `NvidiaConfig.java` | NVIDIA API client configuration and properties |
| `ResilienceConfig.java` | Circuit breaker, retry, rate limiter configuration |
| `SecurityConfig.java` | Spring Security, CORS, security headers |
| `VectorType.java` | Custom PostgreSQL vector type handler |
| `WebConfig.java` | Web MVC configuration, CORS settings |

#### 2. Controller Package (`controller/`)

REST API endpoints that handle HTTP requests.

| File | Endpoints | Purpose |
|------|-----------|---------|
| `ChatController.java` | `/api/chat/*` | Query processing with RAG |
| `UploadController.java` | `/api/documents/*` | PDF upload, status check, document management |

**ChatController Endpoints:**
- `POST /api/chat/query` - Submit a question and get AI-generated answer

**UploadController Endpoints:**
- `POST /api/documents` - Upload PDF (returns 202 Accepted)
- `GET /api/documents/{id}/status` - Check processing status
- `GET /api/documents` - List all documents
- `GET /api/documents/{id}` - Get document details
- `DELETE /api/documents/{id}` - Delete document and chunks

#### 3. DTO Package (`dto/`)

Data Transfer Objects for API requests and responses.

| File | Purpose |
|------|---------|
| `ChatRequest.java` | Request body for chat queries |
| `ChatResponse.java` | Response with AI answer and source references |
| `DocumentMetadataDto.java` | Document information (id, filename, status) |
| `DocumentUploadResponse.java` | Response after PDF upload |
| `ErrorResponse.java` | Standardized error response format |
| `ProcessingStatusDto.java` | Document processing status details |
| `RetrievedChunk.java` | Retrieved document chunk with similarity score |
| `SourceReference.java` | Source citation in chat response |

#### 4. Exception Package (`exception/`)

Custom exceptions and global error handling.

| File | Purpose |
|------|---------|
| `FileSizeExceededException.java` | Thrown when file > 10MB |
| `FileUploadException.java` | General file upload errors |
| `GlobalExceptionHandler.java` | Centralized exception handling for all controllers |
| `ResourceNotFoundException.java` | Thrown when document not found |
| `UnsupportedFileTypeException.java` | Thrown for non-PDF files |

#### 5. Model Package (`model/`)

JPA entities representing database tables.

| File | Database Table | Purpose |
|------|----------------|---------|
| `Document.java` | `documents` | Stores PDF metadata (filename, status, chunk count) |
| `DocumentChunk.java` | `document_chunks` | Stores text chunks with 2048-dim embeddings |
| `ProcessingStatus.java` | (enum) | Status values: PENDING, PROCESSING, COMPLETED, FAILED |

**Document Fields:**
- `id`, `filename`, `originalFilename`, `fileSize`
- `uploadTimestamp`, `status`, `chunkCount`, `errorMessage`

**DocumentChunk Fields:**
- `id`, `documentId`, `chunkNumber`, `content`
- `tokenCount`, `embedding` (vector[2048]), `createdAt`

#### 6. Repository Package (`repository/`)

Spring Data JPA repositories for database access.

| File | Purpose |
|------|---------|
| `ChunkRepository.java` | CRUD operations on document_chunks table |
| `DocumentRepository.java` | CRUD operations on documents table |

**Key Methods:**
- `ChunkRepository.findSimilarChunks()` - Vector similarity search using cosine distance
- `DocumentRepository.findByStatus()` - Find documents by processing status

#### 7. Service Package (`service/`)

Business logic layer with interfaces and implementations.

| File | Type | Purpose |
|------|------|---------|
| `ChatService.java` | Interface | Chat service contract |
| `ChatServiceImpl.java` | Implementation | RAG orchestration: retrieve + generate |
| `DocumentChunker.java` | Interface | Text chunking contract |
| `DocumentChunkerImpl.java` | Implementation | Splits text into 500-token chunks with 50-token overlap |
| `EmbeddingService.java` | Interface | Embedding generation contract |
| `EmbeddingServiceImpl.java` | Implementation | Generates 2048-dim vectors (with caching) |
| `NvidiaChatClient.java` | Client | Direct NVIDIA chat API client |
| `NvidiaEmbeddingClient.java` | Client | Direct NVIDIA embedding API client |
| `PdfProcessingService.java` | Interface | PDF processing contract |
| `PdfProcessingServiceImpl.java` | Implementation | Async PDF extraction, chunking, embedding |
| `ResilientNvidiaChatClient.java` | Wrapper | Adds circuit breaker, retry, rate limiting |
| `RetrievalService.java` | Interface | Vector search contract |
| `RetrievalServiceImpl.java` | Implementation | Finds similar chunks using cosine similarity |

**Service Layer Architecture:**
```
Controller
    ↓
Service Interface
    ↓
Service Implementation
    ↓
Repository / External API
```

---

## Resources (demo/src/main/resources/)

```
demo/src/main/resources/
├── static/                    # Static web resources (empty)
├── templates/                 # Thymeleaf templates (empty)
├── application.properties     # Main configuration
├── application-dev.properties # Development profile
├── application-local.properties # Local testing profile
├── application-prod.properties # Production profile
└── schema.sql                 # Database schema (auto-executed)
```

### Resource Files

| File | Purpose |
|------|---------|
| `application.properties` | Main config: database, NVIDIA API, chunking, retrieval |
| `application-dev.properties` | Development overrides (verbose logging, etc.) |
| `application-local.properties` | Local testing overrides |
| `application-prod.properties` | Production overrides (optimized settings) |
| `schema.sql` | Creates tables, indexes, and vector extension |

**Configuration Hierarchy:**
```
application.properties (base)
    ↓
application-{profile}.properties (overrides)
```

Activate with: `spring.profiles.active=dev`

---

## Target Directory (demo/target/)

**Generated during build - DO NOT EDIT**

```
demo/target/
├── classes/                   # Compiled .class files
├── generated-sources/         # Auto-generated code
├── maven-status/              # Build metadata
├── test-classes/              # Compiled test files
└── demo-0.0.1-SNAPSHOT.jar   # Executable JAR file
```

This directory is created by Maven during `mvn compile` or `mvn package`.

---

## Key Architectural Patterns

### 1. Layered Architecture
```
Controller → Service → Repository → Database
         ↓
        DTO
```

### 2. Dependency Injection
All components use Spring's `@Autowired` or constructor injection.

### 3. Async Processing
`@Async` methods run in separate thread pool (configured in `AsyncConfig`).

### 4. Resilience Patterns
- Circuit Breaker: Fails fast when API is down
- Retry: Exponential backoff for transient failures
- Rate Limiter: Prevents API abuse

### 5. Caching Strategy
- L1: Query embeddings (saves embedding API calls)
- L2: Chat responses (saves chat API calls)

---

## File Naming Conventions

### Java Classes
- **Controllers**: `*Controller.java` (e.g., `ChatController`)
- **Services**: `*Service.java` (interface), `*ServiceImpl.java` (implementation)
- **Repositories**: `*Repository.java` (Spring Data JPA)
- **DTOs**: `*Dto.java` or `*Request.java` / `*Response.java`
- **Models**: Plain names (e.g., `Document.java`, `DocumentChunk.java`)
- **Configs**: `*Config.java` (e.g., `AsyncConfig`)
- **Exceptions**: `*Exception.java` (e.g., `FileUploadException`)

### Configuration Files
- **Properties**: `application*.properties`
- **SQL**: `*.sql` (e.g., `schema.sql`, `setup-db.sql`)
- **Documentation**: `*.md` (Markdown)

---

## How to Navigate the Codebase

### Starting Points

1. **Understanding the flow**: Start with `DemoApplication.java` (main entry point)
2. **API endpoints**: Check `ChatController.java` and `UploadController.java`
3. **Business logic**: Read `ChatServiceImpl.java` for RAG orchestration
4. **Database schema**: Review `schema.sql` and model classes
5. **Configuration**: Check `application.properties` and config classes

### Common Tasks

| Task | Files to Check |
|------|----------------|
| Add new API endpoint | Create controller, service, DTO |
| Change chunking strategy | `DocumentChunkerImpl.java` |
| Modify vector search | `RetrievalServiceImpl.java` |
| Adjust caching | `CacheConfig.java`, service implementations |
| Update security | `SecurityConfig.java` |
| Change resilience settings | `ResilienceConfig.java`, `application.properties` |

---

## Dependencies (from pom.xml)

### Core Framework
- **Spring Boot 3.5.12** - Web framework
- **Spring Data JPA** - Database access
- **Spring Security** - Security layer
- **Spring Cache** - Caching abstraction

### AI/ML
- **Spring AI 1.1.3** - AI integration framework
- **NVIDIA NIM** - LLM and embedding APIs

### Resilience
- **Resilience4j 2.2.0** - Circuit breaker, retry, rate limiter
- **Caffeine** - High-performance cache
- **Bucket4j 8.10.1** - Rate limiting

### Database
- **PostgreSQL** - Relational database
- **pgvector** - Vector similarity search

### Monitoring
- **Micrometer** - Metrics collection
- **Prometheus** - Metrics export
- **Spring Boot Actuator** - Health checks, metrics endpoints

### Documentation
- **SpringDoc OpenAPI 2.7.0** - Swagger UI

---

## Quick Reference

### Find a Feature

| Feature | Primary Files |
|---------|---------------|
| PDF Upload | `UploadController`, `PdfProcessingServiceImpl` |
| Text Chunking | `DocumentChunkerImpl` |
| Embedding Generation | `EmbeddingServiceImpl`, `NvidiaEmbeddingClient` |
| Vector Search | `RetrievalServiceImpl`, `ChunkRepository` |
| Chat/RAG | `ChatServiceImpl`, `ResilientNvidiaChatClient` |
| Async Processing | `AsyncConfig`, `@Async` methods |
| Caching | `CacheConfig`, `@Cacheable` annotations |
| Circuit Breaker | `ResilienceConfig`, `ResilientNvidiaChatClient` |
| Security | `SecurityConfig`, `WebConfig` |
| Error Handling | `GlobalExceptionHandler`, exception classes |

### Configuration Locations

| Setting | File |
|---------|------|
| Database connection | `application.properties` |
| NVIDIA API key | `application.properties` or environment variable |
| Chunking parameters | `application.properties` |
| Retrieval settings | `application.properties` |
| Circuit breaker | `application.properties` (resilience4j section) |
| Cache settings | `application.properties` (spring.cache section) |
| Security/CORS | `SecurityConfig.java` |
| Thread pool | `AsyncConfig.java` |

---

## Next Steps

1. **For Beginners**: Start with `01-BASICS.md` to understand RAG concepts
2. **For Developers**: Read `02-INTERMEDIATE.md` for implementation details
3. **For Advanced Users**: Study `03-ADVANCED.md` for optimization and scaling
4. **For Quick Reference**: Use `04-REFERENCE.md` for APIs and troubleshooting

---

**Last Updated**: March 31, 2026
