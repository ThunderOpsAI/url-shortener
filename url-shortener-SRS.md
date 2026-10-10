# Software Requirements Specification
## URL Shortener API

---

## 1. Project Overview

**Project Name:** URL Shortener API  
**Repository Name:** `backend-url-shortener`  
**Purpose:** Demonstrate backend logic beyond basic CRUD, introduce redirect handling and database design  
**Target Users:** Recruiters assessing backend development capabilities  
**Estimated Development Time:** 3-4 hours

---

## 2. Functional Requirements

### 2.1 Core Features

- **FR1:** Accept long URL via POST request and return shortened version
- **FR2:** Generate unique 6-character short code automatically
- **FR3:** Store URL mapping in persistent database
- **FR4:** Redirect short code to original URL via GET request
- **FR5:** Return 404 error for non-existent short codes
- **FR6:** Validate URLs before shortening (proper format, reachable protocol)
- **FR7:** Prevent duplicate short codes through collision detection
- **FR8:** Track click count for each shortened URL (bonus feature)

### 2.2 User Flow

1. User sends POST request with long URL
2. System validates URL format
3. System generates unique short code
4. System stores mapping in database
5. System returns short URL to user
6. User shares short URL
7. When accessed, system redirects to original URL
8. Click count increments (optional)

---

## 3. Technical Requirements

### 3.1 Technology Stack

- **Runtime:** Node.js 18+
- **Framework:** Express.js 4.x
- **Database:** SQLite3 (file-based, simple deployment)
- **Validation:** express-validator or custom regex
- **ID Generation:** nanoid library or custom base62 encoder
- **Testing:** Jest + Supertest (optional for MVP)

### 3.2 Database Schema

```sql
CREATE TABLE urls (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  short_code TEXT UNIQUE NOT NULL,
  original_url TEXT NOT NULL,
  clicks INTEGER DEFAULT 0,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_short_code ON urls(short_code);
```

**Rationale:**
- `short_code` indexed for fast lookups during redirects
- `clicks` counter for basic analytics
- `created_at` for potential expiration features later

### 3.3 API Endpoints

#### POST `/api/shorten`

**Description:** Create a new shortened URL

**Request Body:**
```json
{
  "url": "https://example.com/very/long/url/with/parameters?id=123&ref=abc"
}
```

**Success Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "shortCode": "a3Xk9m",
    "shortUrl": "http://localhost:3000/a3Xk9m",
    "originalUrl": "https://example.com/very/long/url/with/parameters?id=123&ref=abc",
    "createdAt": "2026-01-16T10:30:00Z"
  }
}
```

**Error Response (400 Bad Request):**
```json
{
  "success": false,
  "error": "Invalid URL format. Must start with http:// or https://"
}
```

---

#### GET `/:shortCode`

**Description:** Redirect to original URL

**Success Response:** 302 Redirect to original URL  
**Failure Response:** 404 Not Found with HTML error page

**Example:**
```
GET /a3Xk9m
→ 302 Redirect to https://example.com/very/long/url...
```

---

#### GET `/api/stats/:shortCode` (Bonus)

**Description:** Get statistics for a shortened URL

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "shortCode": "a3Xk9m",
    "originalUrl": "https://example.com/very/long/url...",
    "clicks": 42,
    "createdAt": "2026-01-15T10:30:00Z"
  }
}
```

**Error Response (404 Not Found):**
```json
{
  "success": false,
  "error": "Short code not found"
}
```

---

### 3.4 Short Code Generation Strategy

**Algorithm Options:**

| Method | Pros | Cons | Selected |
|--------|------|------|----------|
| Sequential (base62) | Deterministic, compact | Predictable | ❌ |
| Random (nanoid) | Unpredictable, secure | Small collision chance | ✅ |
| Hash-based | Deterministic for same URL | Longer codes | ❌ |

**Implementation:**
```javascript
const { nanoid } = require('nanoid');

function generateShortCode() {
  return nanoid(6); // 6-character alphanumeric
}
```

**Character Set:** `a-zA-Z0-9` (62 characters)  
**Length:** 6 characters  
**Combinations:** 62^6 = 56,800,235,584 possible codes  
**Collision Handling:** Retry up to 3 times if code already exists

---

### 3.5 URL Validation Rules

```javascript
const urlValidation = {
  protocol: /^https?:\/\//i,
  maxLength: 2048,
  blockedPatterns: [
    /localhost/i,      // Prevent self-reference loops
    /127\.0\.0\.1/,    // Prevent local network access
    /192\.168\./,      // Prevent private IP ranges
    /10\./             // Prevent private IP ranges
  ]
};
```

**Validation Steps:**
1. Check URL is not empty
2. Check URL starts with `http://` or `https://`
3. Check URL length ≤ 2048 characters
4. Check URL is not localhost or private IP
5. Optionally: Verify domain resolves (DNS check)

---

## 4. Non-Functional Requirements

### 4.1 Performance

- **Redirect Response Time:** < 100ms (database indexed on short_code)
- **Shorten API Response Time:** < 200ms
- **Concurrent Requests:** Handle 100 simultaneous requests (sufficient for portfolio demo)
- **Database Connection:** Use connection pooling to prevent bottlenecks

### 4.2 Reliability

- **Atomicity:** Database transactions prevent duplicate short codes
- **Error Recovery:** Graceful degradation if database unavailable
- **Data Persistence:** SQLite file backed up regularly (future consideration)

### 4.3 Security

**Input Sanitization:**
- Escape special characters to prevent XSS
- Use parameterized queries to prevent SQL injection
- Validate URLs to prevent SSRF attacks

**Rate Limiting:**
```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 10 // 10 requests per minute per IP
});

app.use('/api/shorten', limiter);
```

**CORS Configuration:**
```javascript
const cors = require('cors');

app.use(cors({
  origin: process.env.ALLOWED_ORIGINS || '*',
  methods: ['GET', 'POST']
}));
```

### 4.4 Maintainability

- **Code Structure:** Separate routes, controllers, and database logic
- **Environment Variables:** Configuration via `.env` file
- **Logging:** Request logging with morgan or winston
- **Error Handling:** Centralized error handler middleware

---

## 5. Project Structure

```
backend-url-shortener/
├── src/
│   ├── config/
│   │   └── database.js       # SQLite connection setup
│   ├── controllers/
│   │   └── urlController.js  # Business logic
│   ├── routes/
│   │   └── urlRoutes.js      # Route definitions
│   ├── middleware/
│   │   ├── errorHandler.js   # Centralized error handling
│   │   └── validator.js      # URL validation logic
│   ├── utils/
│   │   └── shortCodeGenerator.js
│   └── app.js                # Express app setup
├── database/
│   └── urls.db               # SQLite database file (gitignored)
├── tests/
│   └── url.test.js           # API tests (optional)
├── .env.example              # Environment variable template
├── .gitignore
├── package.json
└── README.md
```

---

## 6. Environment Configuration

**`.env` File:**
```bash
PORT=3000
NODE_ENV=development
DATABASE_PATH=./database/urls.db
BASE_URL=http://localhost:3000
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX_REQUESTS=10
```

---

## 7. Error Handling

### 7.1 HTTP Status Codes

| Status Code | Scenario |
|-------------|----------|
| 200 OK | Statistics retrieved successfully |
| 201 Created | URL shortened successfully |
| 302 Found | Redirect to original URL |
| 400 Bad Request | Invalid URL format or missing parameters |
| 404 Not Found | Short code doesn't exist |
| 429 Too Many Requests | Rate limit exceeded |
| 500 Internal Server Error | Database or server error |

### 7.2 Error Response Format

```json
{
  "success": false,
  "error": "Human-readable error message",
  "code": "ERROR_CODE_CONSTANT",
  "timestamp": "2026-01-16T10:30:00Z"
}
```

---

## 8. Database Operations

### 8.1 Create Short URL

```javascript
async function createShortUrl(originalUrl) {
  const shortCode = generateShortCode();
  
  const query = `
    INSERT INTO urls (short_code, original_url)
    VALUES (?, ?)
  `;
  
  await db.run(query, [shortCode, originalUrl]);
  return shortCode;
}
```

### 8.2 Retrieve Original URL

```javascript
async function getOriginalUrl(shortCode) {
  const query = `
    UPDATE urls 
    SET clicks = clicks + 1 
    WHERE short_code = ?
    RETURNING original_url
  `;
  
  const result = await db.get(query, [shortCode]);
  return result?.original_url || null;
}
```

---

## 9. Out of Scope (Future Enhancements)

The following features are intentionally excluded from MVP:

- ❌ Custom short codes (user-specified aliases)
- ❌ Expiration dates for URLs
- ❌ User accounts and authentication
- ❌ Analytics dashboard UI
- ❌ QR code generation
- ❌ Link preview metadata (Open Graph)
- ❌ Bulk URL shortening
- ❌ API authentication/API keys
- ❌ Geographic analytics
- ❌ A/B testing different target URLs

---

## 10. Testing Strategy

### 10.1 Manual Testing Checklist

- [ ] POST `/api/shorten` with valid URL returns 201
- [ ] POST `/api/shorten` with invalid URL returns 400
- [ ] POST `/api/shorten` with empty body returns 400
- [ ] GET `/:shortCode` redirects to correct URL
- [ ] GET `/:shortCode` increments click counter
- [ ] GET `/:invalidCode` returns 404
- [ ] Rate limiter blocks after 10 requests/minute
- [ ] Same URL can be shortened multiple times (generates different codes)

### 10.2 Test Cases (Optional Unit Tests)

```javascript
describe('URL Shortener API', () => {
  test('should shorten valid URL', async () => {
    const response = await request(app)
      .post('/api/shorten')
      .send({ url: 'https://example.com' })
      .expect(201);
    
    expect(response.body.success).toBe(true);
    expect(response.body.data.shortCode).toHaveLength(6);
  });
  
  test('should redirect short code to original URL', async () => {
    const response = await request(app)
      .get('/a3Xk9m')
      .expect(302);
    
    expect(response.headers.location).toBe('https://example.com');
  });
});
```

---

## 11. Deployment Considerations

**Recommended Platforms:**
- **Railway:** Easy SQLite persistence
- **Render:** Free tier with automatic deploys
- **Fly.io:** Excellent for global edge deployment

**Deployment Checklist:**
- [ ] Environment variables configured
- [ ] Database directory writable
- [ ] Rate limiting enabled
- [ ] CORS configured for production
- [ ] Health check endpoint added (`GET /health`)
- [ ] Logging configured for production

---

## 12. Success Criteria

The project is considered successful when:

✅ Short codes generate reliably without collisions  
✅ Redirects work instantly (< 100ms)  
✅ Error handling is comprehensive and clear  
✅ Code demonstrates proper database interaction patterns  
✅ API is easily testable with Postman/curl  
✅ Rate limiting prevents abuse  
✅ No SQL injection vulnerabilities  
✅ README provides clear setup instructions

---

## 13. README Requirements

The project README must include:

1. **Project Description:** URL shortener with redirect tracking
2. **Tech Stack:** Node.js, Express, SQLite
3. **API Documentation:**
   - Endpoint descriptions
   - Request/response examples
   - Error codes
4. **Setup Instructions:**
   - Install dependencies
   - Configure environment
   - Initialize database
   - Run locally
5. **Testing Guide:** How to test with curl/Postman
6. **Deployment Guide:** How to deploy to Railway/Render
7. **Architecture Decisions:** Why SQLite, why nanoid, etc.

---

## 14. Portfolio Value

**Signal This Sends:**
> "I understand backend logic beyond simple CRUD operations, including URL redirection, database design, and API security."

**Key Talking Points for Interviews:**
- How you prevent short code collisions
- Why you chose your ID generation strategy
- How you handle database transactions
- Your approach to input validation and security
- How you would scale this to millions of URLs
- Trade-offs between sequential and random IDs

---

## 15. Example API Usage

### Using cURL

**Shorten a URL:**
```bash
curl -X POST http://localhost:3000/api/shorten \
  -H "Content-Type: application/json" \
  -d '{"url": "https://github.com/yourusername/portfolio"}'
```

**Access shortened URL:**
```bash
curl -L http://localhost:3000/a3Xk9m
# Follows redirect to original URL
```

**Get statistics:**
```bash
curl http://localhost:3000/api/stats/a3Xk9m
```

### Using JavaScript Fetch

```javascript
// Shorten URL
const response = await fetch('http://localhost:3000/api/shorten', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ url: 'https://example.com' })
});

const data = await response.json();
console.log(data.data.shortUrl); // http://localhost:3000/a3Xk9m
```

---

**Document Version:** 1.0  
**Last Updated:** January 2026  
**Status:** Ready for Implementation
