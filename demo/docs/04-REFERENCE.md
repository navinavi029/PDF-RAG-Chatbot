# Part 4: Quick Reference

## API Endpoints

### Document Management

```bash
# Upload PDF
POST /api/documents
Content-Type: multipart/form-data
Response: 202 Accepted

# Check status
GET /api/documents/{id}/status
Response: 200 OK

# List all documents
GET /api/documents
Response: 200 OK

# Get document details
GET /api/documents/{id}
Response: 200 OK

# Delete document
DELETE /api/documents/{id}
Response: 204 No Content
```

### Chat

```bash
# Query with RAG
POST /api/chat/query
Content-Type: application/json
Body: {"query": "your question"}
Response: 200 OK
```

### Monitoring

```bash
# Health check
GET /actuator/health

# Metrics
GET /actuator/metrics

# Prometheus
GET /actuator/prometheus
```

## Configuration Properties

### Application Settings
```properties
# Server
server.port=8080

# Database
spring.datasource.url=jdbc:postgresql://localhost:5432/pdf_rag_db
spring.datasource.username=postgres
spring.datasource.password=your_password

# NVIDIA API
nvidia.api.key=nvapi-your-key
nvidia.api.base-url=https://integrate.api.nvidia.com/v1
nvidia.embedding.model=llama-3.2-nemoretriever-300m-embed-v1
nvidia.chat.model=meta/llama-3.1-8b-instruct

# Chunking
app.chunking.chunk-size=500
app.chunking.overlap=50

# Retrieval
app.retrieval.top-k=5
app.retrieval.similarity-threshold=0.3
```

### Resilience4j
```properties
# Circuit Breaker
resilience4j.circuitbreaker.instances.nvidia-api.failure-rate-threshold=50
resilience4j.circuitbreaker.instances.nvidia-api.wait-duration-in-open-state=30s
resilience4j.circuitbreaker.instances.nvidia-api.sliding-window-size=10

# Retry
resilience4j.retry.instances.nvidia-api.max-attempts=3
resilience4j.retry.instances.nvidia-api.wait-duration=1s
resilience4j.retry.instances.nvidia-api.exponential-backoff-multiplier=2

# Rate Limiter
resilience4j.ratelimiter.instances.nvidia-api.limit-for-period=10
resilience4j.ratelimiter.instances.nvidia-api.limit-refresh-period=1s
```

### Cache
```properties
spring.cache.type=caffeine
spring.cache.cache-names=queryEmbeddings,chatResponses
spring.cache.caffeine.spec=maximumSize=1000,expireAfterWrite=3600s
```

## Database Schema

```sql
-- Documents
CREATE TABLE documents (
    id BIGSERIAL PRIMARY KEY,
    filename VARCHAR(255) NOT NULL,
    original_filename VARCHAR(255),
    file_size BIGINT,
    upload_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(20) NOT NULL,
    chunk_count INTEGER DEFAULT 0,
    error_message TEXT
);

-- Document Chunks
CREATE TABLE document_chunks (
    id BIGSERIAL PRIMARY KEY,
    document_id BIGINT NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    chunk_number INTEGER NOT NULL,
    content TEXT NOT NULL,
    token_count INTEGER,
    embedding vector(2048),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (document_id, chunk_number)
);

-- Indexes
CREATE INDEX idx_documents_status ON documents(status);
CREATE INDEX idx_chunks_document_id ON document_chunks(document_id);
CREATE INDEX idx_chunks_embedding ON document_chunks 
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
```

## Common Commands

### Maven
```bash
# Build
mvn clean install

# Run
mvn spring-boot:run

# Test
mvn test

# Package
mvn package
```

### Docker
```bash
# Build image
docker build -t pdf-rag-chatbot .

# Run container
docker run -p 8080:8080 \
  -e NVIDIA_API_KEY=your-key \
  -e DB_HOST=host.docker.internal \
  pdf-rag-chatbot

# Docker Compose
docker-compose up -d
```

### PostgreSQL
```bash
# Connect
psql -U postgres -d pdf_rag_db

# Install pgvector
CREATE EXTENSION vector;

# Check vector support
SELECT * FROM pg_extension WHERE extname = 'vector';

# View documents
SELECT id, filename, status, chunk_count FROM documents;

# View chunks
SELECT id, document_id, chunk_number, 
       substring(content, 1, 50) as preview 
FROM document_chunks LIMIT 10;
```

## Glossary

| Term | Definition |
|------|------------|
| **RAG** | Retrieval-Augmented Generation - combining search with AI |
| **Embedding** | Vector representation of text (2048 numbers) |
| **Vector** | Array of numbers representing meaning |
| **Chunk** | Small piece of text (500 tokens) |
| **Token** | Unit of text (~0.75 words) |
| **Cosine Similarity** | Measure of vector similarity (-1 to 1) |
| **Circuit Breaker** | Pattern to prevent cascading failures |
| **Async** | Non-blocking background processing |
| **Cache** | Temporary storage for fast retrieval |
| **TTL** | Time To Live - how long cache is valid |
| **Top-K** | Retrieve K most similar results |
| **IVFFlat** | Vector index type for similarity search |
| **pgvector** | PostgreSQL extension for vectors |
| **Spring Boot** | Java framework for web apps |
| **Resilience4j** | Library for resilience patterns |
| **Caffeine** | High-performance cache library |

## Error Codes

| Code | Meaning | Solution |
|------|---------|----------|
| 400 | Bad Request | Check request format |
| 404 | Not Found | Document doesn't exist |
| 413 | Payload Too Large | File > 10MB |
| 429 | Too Many Requests | Rate limit exceeded, slow down |
| 500 | Internal Server Error | Check logs |
| 503 | Service Unavailable | Circuit breaker open, API down |

## Monitoring Metrics

### Circuit Breaker
```bash
# State (0=CLOSED, 1=OPEN, 2=HALF_OPEN)
curl http://localhost:8080/actuator/metrics/resilience4j.circuitbreaker.state

# Failure rate
curl http://localhost:8080/actuator/metrics/resilience4j.circuitbreaker.failure.rate
```

### Cache
```bash
# Hit rate
curl http://localhost:8080/actuator/metrics/cache.gets

# Size
curl http://localhost:8080/actuator/metrics/cache.size
```

### Thread Pool
```bash
# Active threads
curl http://localhost:8080/actuator/metrics/executor.active

# Queue size
curl http://localhost:8080/actuator/metrics/executor.queue.size
```

### HTTP
```bash
# Request count
curl http://localhost:8080/actuator/metrics/http.server.requests

# Response time (p99)
curl http://localhost:8080/actuator/metrics/http.server.requests?tag=uri:/api/chat/query
```

## Troubleshooting

### Issue: Upload fails
```bash
# Check file size
ls -lh document.pdf

# Check file type
file document.pdf

# Check server logs
tail -f logs/spring-boot-application.log
```

### Issue: Query returns 503
```bash
# Check circuit breaker state
curl http://localhost:8080/actuator/circuitbreakers

# Check NVIDIA API
curl -H "Authorization: Bearer $NVIDIA_API_KEY" \
  https://integrate.api.nvidia.com/v1/models
```

### Issue: Slow responses
```bash
# Check cache hit rate
curl http://localhost:8080/actuator/metrics/cache.gets

# Check database connections
psql -U postgres -c "SELECT count(*) FROM pg_stat_activity;"

# Check thread pool
curl http://localhost:8080/actuator/metrics/executor.active
```

### Issue: High memory usage
```bash
# Check JVM memory
curl http://localhost:8080/actuator/metrics/jvm.memory.used

# Check cache size
curl http://localhost:8080/actuator/metrics/cache.size

# Adjust JVM heap
java -Xmx2g -Xms1g -jar app.jar
```

## Key Files

| File | Purpose |
|------|---------|
| `pom.xml` | Maven dependencies |
| `application.properties` | Configuration |
| `schema.sql` | Database schema |
| `DemoApplication.java` | Main entry point |
| `ChatServiceImpl.java` | RAG logic |
| `ResilientNvidiaChatClient.java` | Resilience wrapper |
| `SecurityConfig.java` | Security settings |
| `AsyncConfig.java` | Thread pool config |
| `CacheConfig.java` | Cache config |
| `ResilienceConfig.java` | Circuit breaker config |

## Useful Links

- [Spring Boot Docs](https://spring.io/projects/spring-boot)
- [Spring AI Docs](https://docs.spring.io/spring-ai/reference/)
- [Resilience4j Docs](https://resilience4j.readme.io/)
- [pgvector GitHub](https://github.com/pgvector/pgvector)
- [NVIDIA NIM](https://www.nvidia.com/en-us/ai-data-science/products/nim/)
- [Caffeine Cache](https://github.com/ben-manes/caffeine)

## Quick Start Checklist

- [ ] Install Java 17+
- [ ] Install PostgreSQL 14+
- [ ] Install pgvector extension
- [ ] Create database
- [ ] Get NVIDIA API key
- [ ] Update application.properties
- [ ] Run `mvn clean install`
- [ ] Run `mvn spring-boot:run`
- [ ] Test upload endpoint
- [ ] Test query endpoint
- [ ] Check health endpoint
- [ ] View metrics
