# PDF RAG Chatbot

Production-ready Retrieval-Augmented Generation (RAG) system that transforms PDF documents into an intelligent, queryable knowledge base using vector similarity search and large language models.

## 🚀 Quick Start

```bash
# 1. Setup database
psql -U postgres -f setup-db.sql

# 2. Configure API key
export NVIDIA_API_KEY=nvapi-YOUR_KEY_HERE

# 3. Run application
./mvnw spring-boot:run

# 4. Upload a PDF
curl -X POST http://localhost:8080/api/documents -F "file=@document.pdf"

# 5. Query
curl -X POST http://localhost:8080/api/chat/query \
  -H "Content-Type: application/json" \
  -d '{"query": "What is this document about?"}'
```

## 📚 Documentation

### For New Users
- **[Zero-to-Hero Learning Path](./docs/ZERO_TO_HERO.md)** - Complete guide from basics to advanced

### For Experienced Engineers
- **[Principal-Level Guide](./docs/PRINCIPAL_GUIDE.md)** - Architecture insights and design decisions

### Reference Documentation
- [Project Overview](./docs/PROJECT_OVERVIEW.md)
- [API Reference](./docs/API_REFERENCE.md)
- [Architecture Improvements](./docs/ARCHITECTURE_IMPROVEMENTS.md)
- [ADR-001: Async Processing](./docs/ADR-001-async-processing-and-resilience.md)
- [Glossary](./docs/GLOSSARY.md)

## ✨ Key Features

- ✅ **Async Document Processing** - No HTTP timeouts
- ✅ **Circuit Breaker** - Graceful degradation when APIs fail
- ✅ **Intelligent Caching** - 60-80% cost reduction
- ✅ **Rate Limiting** - Protection against abuse
- ✅ **Security Headers** - Production-ready security
- ✅ **Health Checks** - Kubernetes-ready monitoring
- ✅ **Prometheus Metrics** - Full observability

## 🛠️ Technology Stack

- **Backend**: Spring Boot 3.5, Java 17
- **AI/ML**: NVIDIA NIM (Llama 3.1, NemoRetriever)
- **Database**: PostgreSQL + pgvector
- **Resilience**: Resilience4j (Circuit Breaker, Retry, Rate Limiter)
- **Caching**: Caffeine
- **Monitoring**: Micrometer + Prometheus

## 📋 Prerequisites

- Java 17+
- Maven 3.6+
- PostgreSQL 14+ with pgvector extension
- NVIDIA API key ([Get one here](https://build.nvidia.com/))

## 🔧 Setup

### 1. Database Setup
```bash
# Install PostgreSQL and pgvector
# Then create database
psql -U postgres -f setup-db.sql
```

### 2. Configuration
```properties
# src/main/resources/application.properties
nvidia.api.key=nvapi-YOUR_KEY_HERE
spring.datasource.username=postgres
spring.datasource.password=your_password
```

### 3. Build and Run
```bash
./mvnw clean install
./mvnw spring-boot:run
```

### 4. Verify
```bash
# Health check
curl http://localhost:8080/actuator/health

# Should return: {"status":"UP"}
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

```
Client → Upload Controller → Async Processing → PostgreSQL
                                    ↓
                            (Extract, Chunk, Embed)
                                    ↓
                            NVIDIA Embedding API
                                    
Client → Chat Controller → Retrieval Service → Vector Search
                              ↓
                        Circuit Breaker → NVIDIA Chat API
```

## 🔍 Monitoring

### Health Check
```bash
curl http://localhost:8080/actuator/health
```

### Metrics
```bash
# All metrics
curl http://localhost:8080/actuator/metrics

# Circuit breaker state
curl http://localhost:8080/actuator/metrics/resilience4j.circuitbreaker.state

# Cache statistics
curl http://localhost:8080/actuator/metrics/cache.gets
```

### Swagger UI
```
http://localhost:8080/swagger-ui/index.html
```

## ⚙️ Configuration

### Chunking
```properties
app.chunking.chunk-size=500      # Tokens per chunk
app.chunking.overlap=50          # Overlap between chunks
```

### Retrieval
```properties
app.retrieval.top-k=5                    # Number of results
app.retrieval.similarity-threshold=0.3   # Minimum similarity
```

### Circuit Breaker
```properties
resilience4j.circuitbreaker.instances.nvidia-api.failure-rate-threshold=50
resilience4j.circuitbreaker.instances.nvidia-api.wait-duration-in-open-state=30s
```

## 🧪 Testing

```bash
# Run all tests
./mvnw test

# Run specific test
./mvnw test -Dtest=ChatServiceImplTest
```

## 🚢 Deployment

### Docker (Coming Soon)
```bash
docker build -t pdf-rag-chatbot .
docker run -p 8080:8080 pdf-rag-chatbot
```

### Kubernetes
```yaml
# Health probes
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
```

## 📊 Performance

| Metric | Value |
|--------|-------|
| Upload Response | < 100ms |
| Cached Query | < 10ms |
| Uncached Query | 2-3s |
| Max File Size | 10MB |

## 🔒 Security

- ✅ Spring Security with security headers
- ✅ Restricted CORS (configurable)
- ✅ Rate limiting (100 req/min)
- ✅ Input sanitization
- ✅ CSRF protection

**Production Checklist**:
- [ ] Update CORS origins in `SecurityConfig.java`
- [ ] Externalize API keys to environment variables
- [ ] Enable HTTPS
- [ ] Add authentication (JWT/OAuth2)

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

## 📞 Support

- Documentation: [docs/](./demo/docs/)

---

**Built with ❤️ using Spring Boot and NVIDIA AI**
