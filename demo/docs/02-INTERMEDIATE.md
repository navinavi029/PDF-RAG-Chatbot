# Part 2: Intermediate Concepts

## Why Async Processing?

**Problem**: Processing a PDF takes 30-120 seconds. If the web server waits, it blocks and might timeout.

**Solution**: Async (asynchronous) processing
- Server accepts your upload immediately (< 100ms)
- Returns a document ID
- Processes in the background
- You check status by polling

```
You → Upload PDF → Server: "Got it! ID is 123" (immediate)
                    ↓
                Background: Processing... (30-120 seconds)
                    ↓
You → Check status → Server: "Still processing..."
You → Check status → Server: "Done!"
```

## Why Caching?

**Problem**: Same question asked twice = 2 expensive AI calls

**Solution**: Cache (remember) results
- First time: Generate embedding, call AI, store result (2-3 seconds)
- Second time: Return stored result (< 10ms)

**Cost Savings**: If 100 people ask the same question:
- Without cache: 100 AI calls = $$$
- With cache: 1 AI call + 99 instant responses = $

## Circuit Breaker Pattern

**Problem**: If NVIDIA API is down, our system keeps trying and failing, wasting resources.

**Solution**: Circuit breaker (like an electrical circuit breaker)

```
Normal State (CLOSED):
  Requests go through → If successful, continue
  
Failure State (OPEN):
  After 50% failures → Stop trying
  Return error immediately
  Wait 30 seconds
  
Testing State (HALF_OPEN):
  Try one request
  If success → Back to CLOSED
  If failure → Back to OPEN
```

**Benefit**: Fail fast, save resources, auto-recover

## Retry with Exponential Backoff

**Problem**: Sometimes API calls fail temporarily (network glitch)

**Solution**: Retry with increasing delays
- Try 1: Immediate
- Try 2: Wait 1 second
- Try 3: Wait 2 seconds
- Try 4: Wait 4 seconds

**Why increasing delays?** Gives the service time to recover without overwhelming it.

## Rate Limiting

**Problem**: Someone (or a bot) sends 1000 requests per second

**Solution**: Rate limiter
- Allow 100 requests per minute per IP
- Block additional requests
- Prevents abuse and protects resources

## Security Layers

### 1. CORS (Cross-Origin Resource Sharing)
Controls which websites can call your API
```
Before: Any website can call us (*)
After: Only specific domains (localhost:3000, yourdomain.com)
```

### 2. Security Headers
Protects against common web attacks:
- XSS (Cross-Site Scripting)
- Clickjacking
- MIME sniffing

### 3. Input Sanitization
Cleans user input to prevent injection attacks

## Project Structure

```
demo/
├── src/main/java/com/example/demo/
│   ├── controller/          # Handles web requests
│   │   ├── UploadController.java
│   │   └── ChatController.java
│   │
│   ├── service/             # Business logic
│   │   ├── PdfProcessingServiceImpl.java
│   │   ├── ChatServiceImpl.java
│   │   └── EmbeddingServiceImpl.java
│   │
│   ├── model/               # Data structures
│   │   ├── Document.java
│   │   └── DocumentChunk.java
│   │
│   ├── repository/          # Database access
│   │   ├── DocumentRepository.java
│   │   └── ChunkRepository.java
│   │
│   └── config/              # Configuration
│       ├── AsyncConfig.java
│       ├── CacheConfig.java
│       ├── ResilienceConfig.java
│       └── SecurityConfig.java
│
└── src/main/resources/
    ├── application.properties
    └── schema.sql
```

## The RAG Pipeline (Detailed)

### Step 1: Document Upload
```
POST /api/documents
  ↓
Validate (PDF? < 10MB?)
  ↓
Save to database (status: PENDING)
  ↓
Return 202 Accepted
  ↓
Trigger background processing
```

### Step 2: Background Processing
```
Extract text from PDF (Apache PDFBox)
  ↓
Split into chunks (500 tokens, 50 overlap)
  ↓
Generate embeddings (batch of 100)
  ↓
Store in database
  ↓
Update status to COMPLETED
```

### Step 3: Query Processing
```
POST /api/chat/query
  ↓
Generate query embedding (check cache first!)
  ↓
Vector search (find top 5 similar chunks)
  ↓
Build context from chunks
  ↓
Call AI with circuit breaker protection
  ↓
Return response + sources
```

## Database Schema

```sql
-- Documents table
documents
  - id
  - filename
  - status (PENDING, PROCESSING, COMPLETED, FAILED)
  - chunk_count

-- Chunks with embeddings
document_chunks
  - id
  - document_id
  - chunk_number
  - content (the actual text)
  - embedding (2048 numbers representing meaning)
```

## Configuration Tuning

### Chunk Size
```properties
app.chunking.chunk-size=500    # Tokens per chunk
app.chunking.overlap=50        # Overlap between chunks
```

Smaller chunks = more precise but more API calls
Larger chunks = less precise but fewer API calls

### Cache Duration
```properties
cache.ttl=3600    # 1 hour in seconds
```

Longer = more savings but stale data
Shorter = fresher data but more API calls

### Circuit Breaker
```properties
failure-rate-threshold=50      # Open after 50% failures
wait-duration-in-open-state=30s  # Wait before retry
```

## Monitoring

### Health Check
```bash
curl http://localhost:8080/actuator/health
```

Shows if system is healthy

### Metrics
```bash
curl http://localhost:8080/actuator/metrics
```

Shows performance data:
- Request counts
- Response times
- Cache hit rates
- Circuit breaker states

## Common Issues and Solutions

### Issue: Upload times out
**Solution**: Already solved with async processing

### Issue: Same query is slow every time
**Solution**: Check if caching is enabled

### Issue: Getting 503 errors
**Solution**: Circuit breaker is open, NVIDIA API might be down

### Issue: Getting 429 errors
**Solution**: Rate limit exceeded, slow down requests

## Next Steps

Ready for advanced topics? Move to **03-ADVANCED.md** to learn:
- Vector similarity search internals
- Performance optimization
- Scaling strategies
- Production deployment
