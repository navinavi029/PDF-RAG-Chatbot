# Part 1: The Basics

## What is This System?

A PDF RAG Chatbot that lets you upload PDF documents and ask questions about them. The system reads your PDFs, understands them, and answers your questions using AI.

## Core Concept: RAG (Retrieval-Augmented Generation)

Think of RAG like having a smart assistant with a perfect memory:

1. **Upload**: You give it documents to remember
2. **Ask**: You ask a question
3. **Retrieve**: It finds relevant parts from the documents
4. **Generate**: It uses AI to answer based on what it found

**Why RAG?** AI models don't know about your private documents. RAG solves this by:
- Finding relevant information from your documents
- Feeding that context to the AI
- Getting accurate, grounded responses

## The Simple Flow

```
You upload PDF → System breaks it into chunks → Converts to numbers (embeddings)
                                                ↓
You ask question ← AI generates answer ← System finds relevant chunks
```

## Key Technologies (Simple Explanation)

### Spring Boot
A Java framework that makes building web applications easy. Think of it as the foundation that handles all the web server stuff for you.

### PostgreSQL + pgvector
A database that can store both regular data (like filenames) and special "vector" data (numbers representing text meaning). This lets us find similar content quickly.

### NVIDIA NIM
Cloud service that provides two AI capabilities:
- **Embeddings**: Converts text to numbers (vectors)
- **Chat**: Generates human-like responses

## Basic Architecture

```
Client (You)
    ↓
Web Server (Spring Boot)
    ↓
Database (PostgreSQL) + AI Service (NVIDIA)
```

## What Happens When You Upload a PDF?

1. System accepts your file
2. Extracts text from PDF
3. Breaks text into small chunks (500 words each)
4. Converts each chunk to numbers (embeddings)
5. Stores everything in database

## What Happens When You Ask a Question?

1. System converts your question to numbers
2. Finds chunks with similar numbers (similar meaning)
3. Sends those chunks + your question to AI
4. AI generates an answer
5. You get the answer with source references

## Try It Yourself

### Upload a Document
```bash
curl -X POST http://localhost:8080/api/documents \
  -F "file=@mydocument.pdf"
```

Response:
```json
{
  "documentId": 1,
  "status": "PENDING"
}
```

### Check Status
```bash
curl http://localhost:8080/api/documents/1/status
```

### Ask a Question
```bash
curl -X POST http://localhost:8080/api/chat/query \
  -H "Content-Type: application/json" \
  -d '{"query": "What is this document about?"}'
```

## Key Terms (Beginner Level)

- **API**: A way for programs to talk to each other
- **Endpoint**: A specific URL that does something (like `/api/documents`)
- **Chunk**: A small piece of text from your document
- **Embedding**: Numbers that represent the meaning of text
- **Vector**: An array of numbers (like [0.23, -0.45, 0.67, ...])

## Next Steps

Once you understand these basics, move to **02-INTERMEDIATE.md** to learn about:
- How the system handles failures
- Why responses are fast
- How security works
