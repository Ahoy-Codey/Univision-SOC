
---

## 📅 Week 4 — APIs & JSON Contracts

### Exercises

**1. Three API Routes for Vision Dashboard**

| Method | Route | Description | Request Body | Response |
|--------|-------|-------------|--------------|----------|
| POST | `/api/v1/analyze` | Submit image for analysis | `{ "image": "<base64>", "model": "yolo-v8" }` | `{ "job_id": "abc123", "status": "queued" }` |
| GET | `/api/v1/results/{job_id}` | Retrieve analysis results | — | `{ "job_id": "abc123", "status": "completed", "detections": [...] }` |
| GET | `/api/v1/health` | Check system status | — | `{ "status": "healthy", "models_loaded": ["yolo-v8"], "queue_depth": 3 }` |

**2. Example JSON for Image Analysis Result**

```json
{
    "job_id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "completed",
    "completed_at": "2026-06-25T17:41:00Z",
    "image_metadata": {
        "width": 1920,
        "height": 1080,
        "format": "jpeg"
    },
    "detections": [
        {
            "label": "person",
            "confidence": 0.947,
            "bounding_box": {
                "x": 340,
                "y": 120,
                "width": 180,
                "height": 420
            }
        },
        {
            "label": "car",
            "confidence": 0.823,
            "bounding_box": {
                "x": 800,
                "y": 600,
                "width": 400,
                "height": 250
            }
        }
    ],
    "processing_time_ms": 145,
    "model_version": "yolo-v8.0.2"
}
```

**3. Required vs. Optional Fields**

| Field | Required? | Reason |
|-------|-----------|--------|
| `job_id` | ✅ Required | Unique identifier for tracking |
| `status` | ✅ Required | Client needs to know if result is ready |
| `detections` | ❌ Optional | Empty if no objects detected |
| `completed_at` | ❌ Optional | Null if still processing |
| `image_metadata` | ❌ Optional | May be omitted for lightweight responses |
| `processing_time_ms` | ❌ Optional | Debug info, not critical for function |
| `model_version` | ❌ Optional | Useful for debugging, not for core flow |

**4. Testing with Postman/Browser**

```bash
# Test health endpoint (browser or curl)
curl https://your-api.com/api/v1/health

# Test analysis submission (Postman)
POST /api/v1/analyze
Content-Type: application/json

{
    "image": "data:image/jpeg;base64,/9j/4AAQ...",
    "confidence_threshold": 0.5
}

# Expected: 202 Accepted
# { "job_id": "...", "status": "queued", "estimated_wait_seconds": 3 }
```

**5. Why API Responses Should Be Predictable**

Predictable responses are a contract between frontend and backend. They enable:
- **Type safety**: TypeScript interfaces match actual JSON structure
- **Error handling**: Consistent error shape (`{ "error": "...", "code": "..." }`) lets UI show appropriate messages
- **Caching**: Same input → same structure → cacheable
- **Documentation**: OpenAPI specs generate from predictable patterns
- **Testing**: Mock servers can simulate real responses accurately
- **Versioning**: When contracts change, you can migrate gracefully

Unpredictable responses (sometimes string, sometimes number; missing fields appearing/disappearing) create defensive coding, bugs, and fragile integrations.

---

### Homework: Example Request and Response Bodies

**Request: Submit Image for Analysis**

```http
POST /api/v1/analyze HTTP/1.1
Host: vision-api.example.com
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

{
    "image": "data:image/jpeg;base64,/9j/4AAQSkZJRgABAQEASABIAAD...",
    "options": {
        "confidence_threshold": 0.75,
        "model": "yolo-v8",
        "return_cropped_objects": false
    }
}
```

**Success Response (202 Accepted)**

```http
HTTP/1.1 202 Accepted
Content-Type: application/json

{
    "job_id": "job_2v8x9k3m",
    "status": "queued",
    "estimated_wait_seconds": 5,
    "result_url": "/api/v1/results/job_2v8x9k3m"
}
```

**Success Response (200 OK — Results Ready)**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
    "job_id": "job_2v8x9k3m",
    "status": "completed",
    "completed_at": "2026-06-25T17:41:00Z",
    "detections": [
        {
            "label": "person",
            "confidence": 0.947,
            "bounding_box": {"x": 340, "y": 120, "width": 180, "height": 420}
        }
    ],
    "summary": {
        "total_objects": 1,
        "labels_found": ["person"],
        "processing_time_ms": 142
    }
}
```

**Error Response (400 Bad Request)**

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
    "error": {
        "code": "INVALID_IMAGE_FORMAT",
        "message": "Image must be JPEG or PNG format",
        "details": {
            "received_format": "gif",
            "supported_formats": ["jpeg", "png", "webp"]
        }
    }
}
```

**Error Response (404 Not Found)**

```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{
    "error": {
        "code": "JOB_NOT_FOUND",
        "message": "No analysis job found with ID 'job_invalid123'",
        "suggestion": "Check the job ID or submit a new analysis request"
    }
}
```
