# Broker Create API

The Broker Create API is the backend service endpoint used by the Article Summarizer Android application to submit articles for summarization. The broker acts as an intermediary that receives article content or URLs, processes the summarization request, and returns a structured summary.

---

## Base URL

```
https://api.articlesummarizer.example.com/v1
```

---

## Authentication

All requests must include a valid API key in the `Authorization` header.

```
Authorization: Bearer <your_api_key>
```

---

## Endpoint

### Create a Summarization Request

```
POST /broker/create
```

Creates a new article summarization job. The broker accepts either a raw article body or a publicly accessible URL pointing to the article.

---

### Request Headers

| Header          | Required | Description                              |
|-----------------|----------|------------------------------------------|
| `Authorization` | Yes      | Bearer token for authentication          |
| `Content-Type`  | Yes      | Must be `application/json`               |
| `Accept`        | No       | Response format, defaults to `application/json` |

---

### Request Body

```json
{
  "source_type": "url",
  "source": "https://example.com/article",
  "options": {
    "max_length": 200,
    "language": "en",
    "format": "paragraph"
  }
}
```

#### Fields

| Field                 | Type     | Required | Description                                                                 |
|-----------------------|----------|----------|-----------------------------------------------------------------------------|
| `source_type`         | `string` | Yes      | Type of input. Accepted values: `url`, `text`                               |
| `source`              | `string` | Yes      | The article URL (when `source_type` is `url`) or raw article text (when `source_type` is `text`) |
| `options`             | `object` | No       | Optional configuration for the summarization output                         |
| `options.max_length`  | `integer`| No       | Maximum word count of the returned summary. Default: `150`                  |
| `options.language`    | `string` | No       | BCP 47 language code of the source article. Default: `en`                   |
| `options.format`      | `string` | No       | Output format of the summary. Accepted values: `paragraph`, `bullets`. Default: `paragraph` |

---

### Response

#### Success — `201 Created`

```json
{
  "job_id": "a3f2c1d9-4e5b-48a7-9c0e-1f2b3d4e5678",
  "status": "queued",
  "created_at": "2026-04-06T10:27:11Z",
  "source_type": "url",
  "source": "https://example.com/article",
  "summary": null,
  "options": {
    "max_length": 200,
    "language": "en",
    "format": "paragraph"
  }
}
```

#### Response Fields

| Field        | Type     | Description                                                                 |
|--------------|----------|-----------------------------------------------------------------------------|
| `job_id`     | `string` | Unique identifier for the summarization job (UUID v4)                       |
| `status`     | `string` | Current status of the job. Values: `queued`, `processing`, `completed`, `failed` |
| `created_at` | `string` | ISO 8601 timestamp of when the job was created                              |
| `source_type`| `string` | The input type that was submitted                                           |
| `source`     | `string` | The submitted URL or truncated preview of the submitted text                |
| `summary`    | `string` | The generated summary. `null` while the job is still in progress            |
| `options`    | `object` | The options applied to this summarization job                               |

---

## Error Responses

| HTTP Status | Error Code              | Description                                           |
|-------------|-------------------------|-------------------------------------------------------|
| `400`       | `INVALID_SOURCE_TYPE`   | `source_type` must be `url` or `text`                 |
| `400`       | `MISSING_SOURCE`        | The `source` field is required                        |
| `400`       | `INVALID_URL`           | The provided URL is not a valid, publicly accessible URL |
| `401`       | `UNAUTHORIZED`          | Missing or invalid API key                            |
| `403`       | `FORBIDDEN`             | API key does not have permission for this operation   |
| `413`       | `CONTENT_TOO_LARGE`     | Submitted text exceeds the maximum allowed size (50 KB) |
| `429`       | `RATE_LIMIT_EXCEEDED`   | Too many requests. See rate limiting section          |
| `500`       | `INTERNAL_SERVER_ERROR` | An unexpected error occurred on the server            |

#### Error Response Body

```json
{
  "error": {
    "code": "INVALID_URL",
    "message": "The provided URL could not be reached or does not contain readable article content."
  }
}
```

---

## Rate Limiting

The API enforces a rate limit of **60 requests per minute** per API key. When the limit is exceeded, the server responds with `429 Too Many Requests` and includes the following headers:

| Header                  | Description                                              |
|-------------------------|----------------------------------------------------------|
| `X-RateLimit-Limit`     | Maximum number of requests allowed per minute            |
| `X-RateLimit-Remaining` | Number of requests remaining in the current window       |
| `X-RateLimit-Reset`     | Unix timestamp when the rate limit window resets         |

---

## Examples

### Submit an Article URL

```bash
curl -X POST https://api.articlesummarizer.example.com/v1/broker/create \
  -H "Authorization: Bearer <your_api_key>" \
  -H "Content-Type: application/json" \
  -d '{
    "source_type": "url",
    "source": "https://example.com/article",
    "options": {
      "max_length": 100,
      "format": "bullets"
    }
  }'
```

### Submit Raw Article Text

```bash
curl -X POST https://api.articlesummarizer.example.com/v1/broker/create \
  -H "Authorization: Bearer <your_api_key>" \
  -H "Content-Type: application/json" \
  -d '{
    "source_type": "text",
    "source": "Artificial intelligence (AI) is intelligence demonstrated by machines...",
    "options": {
      "max_length": 50,
      "language": "en",
      "format": "paragraph"
    }
  }'
```

---

## Notes

- Summarization is performed asynchronously. A `job_id` is returned immediately, and the summary is available once the job `status` reaches `completed`.
- Poll the job status endpoint (`GET /broker/{job_id}`) to retrieve the completed summary.
- The maximum supported article size for `text` source type is **50 KB**.
- For `url` source type, the broker fetches and extracts the article content automatically. The URL must be publicly accessible without authentication.
