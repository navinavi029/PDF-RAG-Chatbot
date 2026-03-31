# PDF RAG Chatbot

Production-ready Retrieval-Augmented Generation (RAG) system that transforms PDF documents into an intelligent, queryable knowledge base using vector similarity search and large language models.

## 🚀 Quick Start

### Windows Users
```bash
# 1. Setup database
psql -U postgres -f demo/setup-db.sql

# 2. Configure API key (PowerShell)
$env:NVIDIA_API_KEY="nvapi-YOUR_KEY_HERE"

# Or use the batch file
START_APP.bat

# 3. Run application
cd demo
mvnw.cmd spring-boot:run

# 4. Upload a PDF
curl -X POST http://localhost:8080/api/documents -F "file=@document.pdf"

# 5. Query
curl -X POST http://localhost:8080/api/chat/query -H "Content-Type: application/json" -d "{\"query\": \"What is this document about?\"}"
```

### Linux/Mac Users
```bash
# 1. Setup database
psql -U postgres -f demo/setup-db.sql

# 2. Configure API key
export NVIDIA_API_KEY=nvapi-YOUR_KEY_HERE

# 3. Run application
cd demo
./mvnw spring-boot:run

# 4. Upload a PDF
curl -X POST http://localhost:8080/api/documents -F "file=@document.pdf"

# 5. Query
curl -X POST http://localhost:8080/api/chat/query \
  -H "Content-Type: application/json" \
  -d '{"query": "What is this document about?"}'
```

## 📚 Documentation

### Learning Path (Progressive)

Learn the system progressively from basics to advanced:

1. **[Part 1: Basics](./demo/docs/01-BASICS.md)** - Start here! Understand what RAG is and how the system works
2. **[Part 2: Intermediate](./demo/docs/02-INTERMEDIATE.md)** - Learn about async processing, caching, and resilience patterns
3. **[Part 3: Advanced](./demo/docs/03-ADVANCED.md)** - Deep dive into vector search, performance optimization, and scaling
4. **[Part 4: Reference](./demo/docs/04-REFERENCE.md)** - Quick reference for APIs, configuration, and troubleshooting

### Project Structure

**[Project Structure Guide](./demo/docs/PROJECT_STRUCTURE.md)** - Complete reference explaining every folder and file in the project, including:
- Directory organization and purpose
- Java package breakdown (config, controller, service, repository, etc.)
- Configuration files and their roles
- Architectural patterns and naming conventions
- Quick navigation guide for common tasks

## ✨ Key Features

- ✅ **Async Document Processing** - No HTTP timeouts
- ✅ **Circuit Breaker** - Graceful degradation when APIs fail
- ✅ **Intelligent Caching** - 60-80% cost reduction
- ✅ **Rate Limiting** - Protection against abuse
- ✅ **Security Headers** - Production-ready security
- ✅ **Health Checks** - Kubernetes-ready monitoring
- ✅ **Prometheus Metrics** - Full observability

## 🛠️ Technology Stack

- **Backend**: Spring Boot 3.5.12, Java 17
- **AI/ML**: NVIDIA NIM (Llama 3.1-8B-Instruct, NemoRetriever-300M)
- **Database**: PostgreSQL 14+ with pgvector extension
- **Resilience**: Resilience4j 2.2.0 (Circuit Breaker, Retry, Rate Limiter)
- **Caching**: Caffeine (in-memory, 60-80% hit rate)
- **Rate Limiting**: Bucket4j 8.10.1
- **Monitoring**: Micrometer + Prometheus + Spring Boot Actuator
- **Security**: Spring Security with CORS, CSP, XSS protection
- **API Documentation**: SpringDoc OpenAPI 2.7.0 (Swagger UI)

## 📋 Prerequisites

- Java 17+
- Maven 3.6+
- PostgreSQL 14+ with pgvector extension
- NVIDIA API key ([Get one here](https://build.nvidia.com/))

## 🔧 Setup

### 1. Database Setup
```bash
# Install PostgreSQL 14+ and pgvector extension
# Windows: Download from https://www.postgresql.org/download/windows/
# Linux: sudo apt-get install postgresql postgresql-contrib
# Mac: brew install postgresql

# Create database and enable pgvector
psql -U postgres -f demo/setup-db.sql
```

### 2. Configuration

See `API_KEY_QUICK_REFERENCE.txt` for detailed setup instructions.

**Option A: Environment Variables (Recommended)**
```bash
# Windows (PowerShell)
$env:NVIDIA_API_KEY="nvapi-YOUR_KEY_HERE"

# Linux/Mac
export NVIDIA_API_KEY=nvapi-YOUR_KEY_HERE
```

**Option B: Application Properties**
```properties
# demo/src/main/resources/application.properties
nvidia.api.key=nvapi-YOUR_KEY_HERE
spring.datasource.url=jdbc:postgresql://localhost:5432/pdf_rag_db
spring.datasource.username=postgres
spring.datasource.password=your_password
```

**Option C: Use Batch File (Windows)**
```bash
# Edit CREATE_CONFIG_MANUALLY.bat with your API key
CREATE_CONFIG_MANUALLY.bat

# Then run
START_APP.bat
```

### 3. Build and Run
```bash
cd demo

# Windows
mvnw.cmd clean install
mvnw.cmd spring-boot:run

# Linux/Mac
./mvnw clean install
./mvnw spring-boot:run
```

### 4. Verify
```bash
# Health check
curl http://localhost:8080/actuator/health
# Should return: {"status":"UP"}

# Swagger UI
# Open browser: http://localhost:8080/swagger-ui/index.html
```

## 📖 Usage Examples

### Upload Document
```bash
curl -X POST http://localhost:8080/api/documents \
  -F "file=@research_paper.pdf"
```

Response:
```json
{
  "documentId": 1,
  "status": "PENDING",
  "filename": "research_paper_20260331_143022.pdf"
}
```

### Check Status
```bash
curl http://localhost:8080/api/documents/1/status
```

### Query
```bash
curl -X POST http://localhost:8080/api/chat/query \
  -H "Content-Type: application/json" \
  -d '{"query": "What are the main findings?"}'
```

Response:
```json
{
  "response": "The main findings include...",
  "sources": [
    {
      "documentId": 1,
      "filename": "research_paper.pdf",
      "chunkNumber": 5,
      "similarity": 0.87
    }
  ]
}
```

## 🏗️ Architecture

### High-Level Flow
```
┌─────────┐
│ Client  │
└────┬────┘
     │
     ├─── Upload PDF ──────────────────────────────────────┐
     │                                                      │
     │    ┌──────────────────┐                            │
     │    │ Upload Controller│                            │
     │    └────────┬─────────┘                            │
     │             │                                       │
     │             ▼                                       │
     │    ┌──────────────────┐      ┌──────────────┐    │
     │    │ Async Processing │─────▶│ PostgreSQL   │    │
     │    │  (Background)    │      │ + pgvector   │    │
     │    └────────┬─────────┘      └──────────────┘    │
     │             │                                       │
     │             ├─ Extract Text (PDFBox)              │
     │             ├─ Chunk (500 tokens, 50 overlap)     │
     │             └─ Generate Embeddings (2048-dim)     │
     │                        │                            │
     │                        ▼                            │
     │               ┌─────────────────┐                  │
     │               │ NVIDIA Embedding│                  │
     │               │ API (NemoRetriever)                │
     │               └─────────────────┘                  │
     │                                                      │
     └─── Query ───────────────────────────────────────────┘
          │
          ▼
     ┌──────────────┐
     │Chat Controller│
     └──────┬───────┘
            │
            ▼
     ┌──────────────────┐      ┌──────────────┐
     │ Retrieval Service│─────▶│Vector Search │
     │  (with Cache)    │      │ (Cosine Sim) │
     └──────┬───────────┘      └──────────────┘
            │
            ▼
     ┌──────────────────┐
     │ Circuit Breaker  │
     │   + Retry        │
     └──────┬───────────┘
            │
            ▼
     ┌──────────────────┐
     │ NVIDIA Chat API  │
     │ (Llama 3.1-8B)   │
     └──────────────────┘
```

### Component Details

**Controllers**
- `UploadController`: Handles PDF uploads, returns 202 Accepted immediately
- `ChatController`: Processes queries with RAG pipeline

**Services**
- `PdfProcessingService`: Async PDF extraction and chunking
- `EmbeddingService`: Generates 2048-dim vectors (cached)
- `RetrievalService`: Vector similarity search (top-K)
- `ChatService`: RAG orchestration with context building
- `ResilientNvidiaChatClient`: Wraps API calls with resilience patterns

**Resilience Patterns**
- Circuit Breaker: Fails fast when API is down (50% threshold, 30s wait)
- Retry: 3 attempts with exponential backoff (1s, 2s, 4s)
- Rate Limiter: 100 requests/minute per IP
- Cache: 60-80% hit rate, 1-hour TTL

## 🔍 Monitoring & Observability

### Health Checks
```bash
# Overall health
curl http://localhost:8080/actuator/health

# Liveness probe (Kubernetes)
curl http://localhost:8080/actuator/health/liveness

# Readiness probe (Kubernetes)
curl http://localhost:8080/actuator/health/readiness
```

### Metrics
```bash
# All available metrics
curl http://localhost:8080/actuator/metrics

# Circuit breaker state (0=CLOSED, 1=OPEN, 2=HALF_OPEN)
curl http://localhost:8080/actuator/metrics/resilience4j.circuitbreaker.state

# Cache hit rate
curl http://localhost:8080/actuator/metrics/cache.gets

# Thread pool utilization
curl http://localhost:8080/actuator/metrics/executor.active

# HTTP request metrics
curl http://localhost:8080/actuator/metrics/http.server.requests

# Prometheus format (for Grafana)
curl http://localhost:8080/actuator/prometheus
```

### API Documentation
```
# Swagger UI (Interactive API testing)
http://localhost:8080/swagger-ui/index.html

# OpenAPI JSON spec
http://localhost:8080/v3/api-docs
```

### Key Metrics to Monitor

| Metric | Target | Description |
|--------|--------|-------------|
| Cache Hit Rate | > 60% | Percentage of cached responses |
| Circuit Breaker State | CLOSED | API health indicator |
| Response Time (p99) | < 3s | 99th percentile latency |
| Thread Pool Usage | < 80% | Async processing capacity |
| Error Rate | < 1% | Failed requests percentage |

## ⚙️ Configuration

### Document Processing
```properties
# Chunking strategy
app.chunking.chunk-size=500      # Tokens per chunk (optimal: 300-700)
app.chunking.overlap=50          # Overlap between chunks (prevents context loss)

# File upload limits
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=10MB
```

### Retrieval & Search
```properties
# Vector search parameters
app.retrieval.top-k=5                    # Number of similar chunks to retrieve
app.retrieval.similarity-threshold=0.3   # Minimum cosine similarity (0-1)

# Vector index configuration (in schema.sql)
# IVFFlat index with 100 lists (optimal for ~10K chunks)
```

### Resilience Patterns
```properties
# Circuit Breaker
resilience4j.circuitbreaker.instances.nvidia-api.failure-rate-threshold=50
resilience4j.circuitbreaker.instances.nvidia-api.wait-duration-in-open-state=30s
resilience4j.circuitbreaker.instances.nvidia-api.sliding-window-size=10

# Retry with exponential backoff
resilience4j.retry.instances.nvidia-api.max-attempts=3
resilience4j.retry.instances.nvidia-api.wait-duration=1s
resilience4j.retry.instances.nvidia-api.exponential-backoff-multiplier=2

# Rate Limiting
resilience4j.ratelimiter.instances.nvidia-api.limit-for-period=10
resilience4j.ratelimiter.instances.nvidia-api.limit-refresh-period=1s
```

### Caching
```properties
# Caffeine cache configuration
spring.cache.type=caffeine
spring.cache.cache-names=queryEmbeddings,chatResponses
spring.cache.caffeine.spec=maximumSize=1000,expireAfterWrite=3600s

# Cache strategy:
# - queryEmbeddings: Caches query vectors (saves embedding API calls)
# - chatResponses: Caches full responses (saves chat API calls)
```

### Thread Pool (Async Processing)
```properties
# Configured in AsyncConfig.java
# Core pool size: 5 threads
# Max pool size: 10 threads
# Queue capacity: 100 tasks
```

### Environment-Specific Profiles
```bash
# Development
spring.profiles.active=dev

# Production
spring.profiles.active=prod

# Local testing
spring.profiles.active=local
```

## 🧪 Testing

### Run Tests
```bash
cd demo

# Windows
mvnw.cmd test

# Linux/Mac
./mvnw test

# Run specific test class
./mvnw test -Dtest=ChatServiceImplTest

# Run with coverage
./mvnw test jacoco:report
```

### Testing Checklist

See `TESTING_CHECKLIST.md` for comprehensive testing scenarios including:
- Document upload (valid/invalid files)
- Async processing verification
- Query with/without context
- Cache behavior
- Circuit breaker states
- Rate limiting
- Error handling
- Security headers

### Manual API Testing

Use the provided Swagger UI for interactive testing:
```
http://localhost:8080/swagger-ui/index.html
```

Or use curl commands from the documentation.

## 🚢 Deployment

### Environment Variables (Production)
```bash
# Required
export NVIDIA_API_KEY=nvapi-your-production-key
export DB_HOST=prod-db.example.com
export DB_USERNAME=app_user
export DB_PASSWORD=secure_password

# Optional
export ALLOWED_ORIGINS=https://app.example.com,https://www.example.com
export CACHE_TTL=3600
export CACHE_MAX_SIZE=5000
export SPRING_PROFILES_ACTIVE=prod
```

### Docker (Coming Soon)
```bash
# Build
docker build -t pdf-rag-chatbot:1.0.0 .

# Run
docker run -p 8080:8080 \
  -e NVIDIA_API_KEY=your-key \
  -e DB_HOST=host.docker.internal \
  -e SPRING_PROFILES_ACTIVE=prod \
  pdf-rag-chatbot:1.0.0
```

### Kubernetes
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pdf-rag-chatbot
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: pdf-rag-chatbot:1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        env:
        - name: NVIDIA_API_KEY
          valueFrom:
            secretKeyRef:
              name: nvidia-api-secret
              key: api-key
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

### Production Checklist
- [ ] Update CORS origins in `SecurityConfig.java`
- [ ] Externalize all secrets to environment variables
- [ ] Enable HTTPS/TLS
- [ ] Configure proper database connection pooling
- [ ] Set up monitoring (Prometheus + Grafana)
- [ ] Configure log aggregation (ELK/Splunk)
- [ ] Implement authentication (JWT/OAuth2)
- [ ] Set up automated backups
- [ ] Configure rate limiting per user/API key
- [ ] Review and tune JVM settings (-Xmx, -Xms)
- [ ] Set up alerting for circuit breaker state changes

## 📊 Performance

| Metric | Value |
|--------|-------|
| Upload Response | < 100ms |
| Cached Query | < 10ms |
| Uncached Query | 2-3s |
| Max File Size | 10MB |

## 🔒 Security

### Implemented Security Features

- ✅ **Spring Security** with comprehensive security headers
- ✅ **CORS Protection** - Configurable allowed origins (default: localhost only)
- ✅ **Rate Limiting** - 100 requests/minute per IP (Bucket4j)
- ✅ **Input Sanitization** - Prevents injection attacks
- ✅ **CSRF Protection** - Disabled for stateless API (can be enabled)
- ✅ **XSS Protection** - Content Security Policy headers
- ✅ **Clickjacking Protection** - X-Frame-Options: DENY
- ✅ **File Upload Validation** - Type and size restrictions
- ✅ **SQL Injection Prevention** - JPA/Hibernate parameterized queries

### Security Headers Applied
```
Content-Security-Policy: default-src 'self'
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

### Authentication (Not Implemented)

For production, add authentication:
- JWT tokens with Spring Security
- OAuth2/OIDC integration
- API key authentication
- Role-based access control (RBAC)

### Security Best Practices

1. **Never commit secrets** - Use environment variables
2. **Update dependencies** regularly for security patches
3. **Enable HTTPS** in production
4. **Implement rate limiting** per user/API key
5. **Monitor for suspicious activity** via logs
6. **Regular security audits** and penetration testing

## 🤝 Contributing

1. Fork the repository
2. Create feature branch (`git checkout -b feature/amazing-feature`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Open Pull Request

## 📝 License

This project is licensed under the MIT License.

## 🙏 Acknowledgments

- [Spring Boot](https://spring.io/projects/spring-boot)
- [NVIDIA NIM](https://www.nvidia.com/en-us/ai-data-science/products/nim/)
- [pgvector](https://github.com/pgvector/pgvector)
- [Resilience4j](https://resilience4j.readme.io/)

## 🐛 Troubleshooting

### Common Issues

**Issue: Upload fails with 413 Payload Too Large**
- Solution: File exceeds 10MB limit. Reduce file size or increase limit in `application.properties`

**Issue: Query returns 503 Service Unavailable**
- Solution: Circuit breaker is open. Check NVIDIA API status and verify API key
- Check: `curl http://localhost:8080/actuator/circuitbreakers`

**Issue: Slow query responses**
- Check cache hit rate: `curl http://localhost:8080/actuator/metrics/cache.gets`
- Verify database indexes are created
- Monitor thread pool: `curl http://localhost:8080/actuator/metrics/executor.active`

**Issue: Database connection errors**
- Verify PostgreSQL is running
- Check connection string in `application.properties`
- Ensure pgvector extension is installed: `SELECT * FROM pg_extension WHERE extname = 'vector';`

**Issue: High memory usage**
- Reduce cache size in configuration
- Tune JVM heap: `java -Xmx2g -Xms1g -jar app.jar`
- Check for memory leaks in logs

### Getting Help

1. Check the comprehensive documentation in `demo/docs/`
2. Review `TESTING_CHECKLIST.md` for validation scenarios
3. Check application logs in `logs/` directory
4. Use Swagger UI for API testing: `http://localhost:8080/swagger-ui/index.html`
5. Monitor metrics via Actuator endpoints

## 📞 Support & Resources

### Documentation
- [Part 1: Basics](./demo/docs/01-BASICS.md) - Start here! Understand RAG fundamentals
- [Part 2: Intermediate](./demo/docs/02-INTERMEDIATE.md) - Async, caching, resilience patterns
- [Part 3: Advanced](./demo/docs/03-ADVANCED.md) - Vector search, optimization, scaling
- [Part 4: Reference](./demo/docs/04-REFERENCE.md) - Quick reference, APIs, troubleshooting

### Quick Reference Files
- `API_KEY_QUICK_REFERENCE.txt` - API key setup guide
- `TESTING_CHECKLIST.md` - Comprehensive testing scenarios
- `CREATE_CONFIG_MANUALLY.bat` - Windows configuration helper
- `START_APP.bat` - Windows startup script

### External Resources
- [Spring Boot Documentation](https://spring.io/projects/spring-boot)
- [Spring AI Documentation](https://docs.spring.io/spring-ai/reference/)
- [NVIDIA NIM Platform](https://www.nvidia.com/en-us/ai-data-science/products/nim/)
- [pgvector GitHub](https://github.com/pgvector/pgvector)
- [Resilience4j Documentation](https://resilience4j.readme.io/)

## 🗺️ Roadmap

### Current Version (v1.0)
- ✅ PDF upload and processing
- ✅ Vector similarity search
- ✅ RAG-based chat
- ✅ Async processing
- ✅ Circuit breaker and resilience
- ✅ Caching and rate limiting
- ✅ Monitoring and metrics

### Planned Features (v2.0)
- [ ] Docker containerization
- [ ] Multi-format support (DOCX, TXT, HTML)
- [ ] User authentication (JWT/OAuth2)
- [ ] Multi-tenancy support
- [ ] Advanced search filters
- [ ] Conversation history
- [ ] Streaming responses
- [ ] Redis cache for horizontal scaling
- [ ] Kubernetes Helm charts

### Future Enhancements
- [ ] Fine-tuned embeddings
- [ ] Hybrid search (vector + keyword)
- [ ] Document versioning
- [ ] Collaborative annotations
- [ ] GraphQL API
- [ ] WebSocket support for real-time updates

---

**Built with ❤️ using Spring Boot 3.5, NVIDIA AI, and PostgreSQL**

*Last Updated: March 31, 2026*
