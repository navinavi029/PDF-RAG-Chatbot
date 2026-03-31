# Part 3: Advanced Topics

## Vector Similarity Search Deep Dive

### What Are Embeddings?

Text converted to 2048-dimensional vectors (arrays of numbers):
```
"machine learning" → [0.23, -0.45, 0.67, ..., 0.12]  (2048 numbers)
"artificial intelligence" → [0.25, -0.43, 0.69, ..., 0.14]
```

Similar concepts have similar vectors.

### Cosine Similarity

Measures how similar two vectors are (-1 to 1):
```
1.0  = Identical
0.5  = Somewhat similar
0.0  = Unrelated
-1.0 = Opposite
```

Formula: `similarity = (A · B) / (||A|| × ||B||)`

### Vector Indexing: IVFFlat vs HNSW

**IVFFlat** (Inverted File with Flat Compression):
- Divides vectors into clusters
- Searches only relevant clusters
- Supports up to 2048 dimensions ✓
- Used in this project

**HNSW** (Hierarchical Navigable Small World):
- Graph-based search
- Faster but limited to 2000 dimensions ✗
- Not used (our embeddings are 2048-dim)

### The Vector Search Query

```sql
SELECT 
  content,
  embedding <=> query_vector AS distance
FROM document_chunks
ORDER BY embedding <=> query_vector
LIMIT 5;
```

`<=>` is cosine distance operator (1 - cosine similarity)

## Resilience Patterns in Detail

### Circuit Breaker State Machine

```
┌─────────┐
│ CLOSED  │ ◄─── Normal operation
└────┬────┘
     │ 50% failures
     ▼
┌─────────┐
│  OPEN   │ ◄─── Failing fast
└────┬────┘
     │ 30 seconds
     ▼
┌──────────┐
│HALF_OPEN │ ◄─── Testing recovery
└────┬─────┘
     │
     ├─ Success → CLOSED
     └─ Failure → OPEN
```

**Configuration**:
```properties
resilience4j.circuitbreaker.instances.nvidia-api:
  failure-rate-threshold: 50
  wait-duration-in-open-state: 30s
  sliding-window-size: 10
  minimum-number-of-calls: 5
```

### Retry Strategy

```java
@Retry(name = "nvidia-api")
public String callAPI() {
    // Retries: 3 attempts
    // Delays: 1s, 2s, 4s (exponential)
    // Jitter: Random variation to prevent thundering herd
}
```

**Thundering Herd**: When many clients retry at the same time, overwhelming the recovering service. Jitter adds randomness to prevent this.

### Rate Limiter (Token Bucket)

```
Bucket capacity: 100 tokens
Refill rate: 100 tokens/minute

Request arrives:
  - Token available? → Process request, consume token
  - No token? → Reject with 429 Too Many Requests
```

## Caching Strategy

### Multi-Level Cache

```
Request → L1: Query Embeddings Cache
            ↓ miss
          L2: Chat Responses Cache
            ↓ miss
          Generate fresh response
            ↓
          Store in both caches
```

### Cache Configuration

```java
@Cacheable(value = "queryEmbeddings", key = "#query")
public float[] generateQueryEmbedding(String query) {
    // Only called on cache miss
    return nvidiaClient.generateEmbedding(query);
}
```

**Caffeine Cache Settings**:
- Maximum size: 1000 entries
- TTL: 1 hour
- Eviction: LRU (Least Recently Used)

### Cache Metrics

```bash
# Cache hit rate
cache.gets.hit / cache.gets.total

# Target: > 60% hit rate
# Actual: 60-80% in production
```

## Async Processing Architecture

### Thread Pool Configuration

```java
@Bean(name = "documentProcessingExecutor")
public Executor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(5);      // Always alive
    executor.setMaxPoolSize(10);      // Max concurrent
    executor.setQueueCapacity(100);   // Waiting queue
    executor.setThreadNamePrefix("doc-processor-");
    return executor;
}
```

**Sizing Guidelines**:
- Core pool: Number of CPU cores
- Max pool: 2x CPU cores
- Queue: Expected burst size

### Async Method

```java
@Async("documentProcessingExecutor")
public CompletableFuture<Void> processDocument(Long documentId) {
    // Runs in background thread
    // Doesn't block HTTP thread
    return CompletableFuture.completedFuture(null);
}
```

## Performance Optimization

### Batch Processing

Instead of:
```java
for (String chunk : chunks) {
    embedding = generateEmbedding(chunk);  // 100 API calls
}
```

Do:
```java
List<float[]> embeddings = generateEmbeddings(chunks);  // 1 API call
```

**Savings**: 100x fewer API calls

### Database Indexing

```sql
-- Vector index for fast similarity search
CREATE INDEX idx_chunks_embedding 
ON document_chunks 
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

**lists parameter**: Number of clusters
- Too few: Slow search
- Too many: Slow index build
- Rule of thumb: sqrt(total_rows)

### Connection Pooling

```properties
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
```

## Security Deep Dive

### Spring Security Filter Chain

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) {
    return http
        .cors(cors -> cors.configurationSource(corsConfig()))
        .csrf(csrf -> csrf.disable())  // For API
        .headers(headers -> headers
            .contentSecurityPolicy("default-src 'self'")
            .frameOptions().deny()
            .xssProtection().enable())
        .build();
}
```

### CORS Configuration

```java
configuration.setAllowedOrigins(Arrays.asList(
    "http://localhost:3000",      // Development
    "https://yourdomain.com"      // Production
));
configuration.setAllowedMethods(Arrays.asList("GET", "POST", "DELETE"));
configuration.setAllowCredentials(true);
```

### Input Sanitization

```java
private String sanitizeInput(String input) {
    return input
        .replaceAll("[<>\"']", "")  // Remove HTML/SQL chars
        .trim()
        .substring(0, Math.min(input.length(), 1000));  // Limit length
}
```

## Monitoring and Observability

### Key Metrics to Track

```properties
# Circuit Breaker
resilience4j.circuitbreaker.state
  - CLOSED: Normal ✓
  - OPEN: Failing ✗
  - HALF_OPEN: Testing

# Cache Performance
cache.gets.hit / cache.gets.total
  - Target: > 60%

# Thread Pool
executor.active / executor.pool.size
  - Target: < 80%

# API Latency
http.server.requests.p99
  - Target: < 3s
```

### Prometheus Integration

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'spring-boot'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['localhost:8080']
```

### Grafana Dashboard

Key panels:
1. Request rate (requests/second)
2. Response time (p50, p95, p99)
3. Error rate (%)
4. Circuit breaker state
5. Cache hit rate
6. Thread pool utilization

## Scaling Strategies

### Current State (Phase 1)
- Single instance
- In-memory cache (Caffeine)
- Thread pool for async
- PostgreSQL with pgvector

**Capacity**: ~100 concurrent users

### Phase 2: Horizontal Scaling
```
Load Balancer
    ↓
┌────────┬────────┬────────┐
│ App 1  │ App 2  │ App 3  │
└────┬───┴────┬───┴────┬───┘
     └────────┼────────┘
              ↓
    ┌─────────────────┐
    │ Redis (Cache)   │
    │ RabbitMQ (Queue)│
    │ PostgreSQL (DB) │
    └─────────────────┘
```

**Changes needed**:
- Migrate to Redis for shared cache
- Add message queue for document processing
- Database read replicas
- Session management

### Phase 3: Microservices
```
API Gateway
    ↓
┌──────────┬──────────┬──────────┐
│ Upload   │ Query    │ Admin    │
│ Service  │ Service  │ Service  │
└────┬─────┴────┬─────┴────┬─────┘
     └──────────┼──────────┘
                ↓
    ┌───────────────────────┐
    │ Shared Infrastructure │
    │ - Redis               │
    │ - Kafka               │
    │ - Vector DB (Pinecone)│
    └───────────────────────┘
```

## Production Deployment

### Environment Variables

```bash
# Database
export DB_HOST=prod-db.example.com
export DB_USERNAME=app_user
export DB_PASSWORD=secure_password

# NVIDIA API
export NVIDIA_API_KEY=nvapi-prod-key

# CORS
export ALLOWED_ORIGINS=https://app.example.com,https://www.example.com

# Cache
export CACHE_TTL=3600
export CACHE_MAX_SIZE=5000
```

### Kubernetes Deployment

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
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
```

### Database Migration

```sql
-- Add new column
ALTER TABLE documents ADD COLUMN processed_at TIMESTAMP;

-- Create index
CREATE INDEX CONCURRENTLY idx_documents_status 
ON documents(status) WHERE status = 'PENDING';

-- Update existing data
UPDATE documents SET processed_at = upload_timestamp 
WHERE status = 'COMPLETED';
```

## Troubleshooting

### Circuit Breaker Stuck Open

**Symptoms**: All requests return 503

**Diagnosis**:
```bash
curl http://localhost:8080/actuator/circuitbreakers
```

**Solutions**:
1. Check NVIDIA API status
2. Verify API key is valid
3. Check network connectivity
4. Review circuit breaker config

### High Memory Usage

**Diagnosis**:
```bash
curl http://localhost:8080/actuator/metrics/jvm.memory.used
```

**Solutions**:
1. Reduce cache size
2. Tune JVM heap: `-Xmx2g -Xms1g`
3. Check for memory leaks
4. Review thread pool size

### Slow Queries

**Diagnosis**:
```sql
-- Enable query logging
SET log_min_duration_statement = 1000;  -- Log queries > 1s

-- Check slow queries
SELECT * FROM pg_stat_statements 
ORDER BY mean_exec_time DESC 
LIMIT 10;
```

**Solutions**:
1. Add missing indexes
2. Optimize vector search parameters
3. Increase database resources
4. Consider read replicas

## Code Quality

### Testing Strategy

```java
// Unit test
@Test
void shouldGenerateEmbedding() {
    String text = "test";
    float[] embedding = embeddingService.generateEmbedding(text);
    assertNotNull(embedding);
    assertEquals(2048, embedding.length);
}

// Integration test
@SpringBootTest
@Test
void shouldProcessDocumentEndToEnd() {
    // Upload → Process → Query
}

// Performance test
@Test
void shouldHandleConcurrentRequests() {
    // Simulate 100 concurrent users
}
```

### Code Review Checklist

- [ ] Error handling for all external calls
- [ ] Input validation and sanitization
- [ ] Proper logging (no sensitive data)
- [ ] Cache keys are unique
- [ ] Async methods return CompletableFuture
- [ ] Circuit breaker configured
- [ ] Tests cover edge cases
- [ ] Documentation updated

## References

### Key Files
- `ChatServiceImpl.java` - RAG orchestration
- `ResilientNvidiaChatClient.java` - Resilience patterns
- `RetrievalServiceImpl.java` - Vector search
- `AsyncConfig.java` - Thread pool config
- `ResilienceConfig.java` - Circuit breaker config

### External Resources
- [Resilience4j Docs](https://resilience4j.readme.io/)
- [pgvector GitHub](https://github.com/pgvector/pgvector)
- [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)
- [Caffeine Cache](https://github.com/ben-manes/caffeine)

## Next Steps

You've completed the learning path! You now understand:
- ✓ Basic RAG concepts
- ✓ Async processing and resilience
- ✓ Vector similarity search
- ✓ Performance optimization
- ✓ Production deployment

**Suggested next actions**:
1. Set up local development environment
2. Run the application and test APIs
3. Review the actual code files
4. Make a small feature contribution
5. Deploy to staging environment
