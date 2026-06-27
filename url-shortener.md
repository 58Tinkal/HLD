# URL Shortener - High Level Design (HLD)

## Architecture

![URL Shortener Architecture](images/URLshortener.png)

---

# 1. Requirements

## Functional Requirements

- Create a short URL from a long URL.
- Optionally support custom aliases.
- Optionally support expiration time.
- Redirect users from the short URL to the original URL.

---

## Non-Functional Requirements

- Low latency for redirects (~200ms).
- Scale to **100M Daily Active Users** and **1B URLs**.
- Ensure **unique short URL generation** (strong consistency).
- Provide **high availability** for redirects with **eventual consistency**.

---

# 2. Back of the Envelope Estimation

Assumptions:

- 100M Daily Active Users
- 1B stored URLs
- Read : Write = **10 : 1**

Estimated traffic:

- Writes ≈ **1,000 RPS**
- Reads ≈ **10,000 RPS**

---

# 3. Core Entities

### URL

- Original URL
- Short URL
- Custom Alias
- Expiration Time
- Creation Time
- User ID

### User

- User ID
- User Information

---

# 4. API Design

## Create Short URL

```http
POST /urls
```

Request

```json
{
  "originalUrl": "https://example.com/very/long/url",
  "alias": "chatgpt",
  "expirationTime": "2026-12-31T23:59:59Z"
}
```

Response

```json
{
  "shortUrl": "https://short.ly/aB12Cd"
}
```

---

## Redirect

```http
GET /{shortUrl}
```

Response

- **302 Redirect** → Original URL
- **404 Not Found** → URL expired or does not exist

---

# 5. Database Schema

## URL Table

| Column | Description |
|---------|-------------|
| shortUrl (PK) | Generated short URL |
| customAlias | Optional custom alias |
| originalUrl | Long URL |
| expirationTime | Expiration timestamp |
| creationTime | Creation timestamp |
| userId | Owner |

---

## User Table

| Column | Description |
|---------|-------------|
| userId | Primary Key |
| ... | User Details |

---

# 6. High Level Design

```
                Client
                   │
         POST / GET Requests
                   │
             API Gateway
             /          \
            /            \
     Write Service    Read Service
            │             │
            │             │
      Global Counter   Redis Cache
            │             │
            └──────┬──────┘
                   │
             Primary Database
```

### Components

### API Gateway

- Authentication
- Routing
- Rate limiting
- Load balancing

### Write Service

Responsible for

- Creating short URLs
- Supporting custom aliases
- Handling expiration
- Storing mappings
- Generating unique IDs

### Read Service

Responsible for

- Looking up original URLs
- Returning HTTP redirects
- Reading from Redis cache first

### Redis

- Read-through cache
- LRU eviction policy

```
Key   : Short URL
Value : Original URL
```

### Global Counter

Responsible for generating globally unique IDs.

```
Counter
↓

Base62 Encoding
↓

Short URL
```

---

# 7. Redirect Strategy

### 302 Redirect (Recommended)

- Temporary redirect
- Browsers do not permanently cache
- Allows analytics on every request
- Enables URL updates

### 301 Redirect

- Permanent redirect
- Browser caches aggressively
- Faster after first request
- Less flexibility

For URL shortening services, **302 Redirect** is generally preferred.

---

# 8. Short URL Generation

## Option 1 — Prefix

Use a prefix of the original URL.

❌ Not unique.

---

## Option 2 — Random String

Generate a random Base62 string.

Example:

```
aZ91Bc
```

For 6 characters:

```
62⁶ ≈ 56 Billion possibilities
```

Issues:

- Birthday paradox
- Collisions
- Requires database lookup before insertion

---

## Option 3 — Hash

```
MD5(LongURL)
      ↓
Base62
      ↓
Take first 6 characters
```

Issues:

- Collisions still possible
- Database lookup required

---

## Option 4 — Global Counter (Recommended)

```
Counter
↓

123456789

↓

Base62 Encoding

↓

aB91Zx
```

Advantages

- No collisions
- No read-before-write
- Fast generation

Disadvantages

- Predictable sequence

Mitigations

- Rate limiting
- Do not shorten sensitive/private URLs
- Obfuscate IDs using a bijective encoder like **Sqids**

---

# 9. Caching Strategy

Redis is used as a read-through cache.

```
Read Request

↓

Redis

↓

Cache Hit
↓

Return URL

OR

Cache Miss

↓

Database

↓

Update Redis

↓

Return URL
```

Benefits

- Low latency
- Reduced database load
- Frequently accessed URLs remain in cache using LRU

---

# 10. Scalability

Read traffic is significantly higher than write traffic.

Approximate ratio:

```
Reads : Writes = 10 : 1
```

Therefore,

- Separate Read and Write services
- Scale Read Service independently
- Scale Write Service independently
- Multiple application servers behind load balancers
- Dedicated Counter Service

Horizontal scaling is preferred over vertical scaling.

---

# 11. Storage Estimation

Approximate storage per URL

| Field | Size |
|--------|------|
| Short URL | 8 Bytes |
| Long URL | 100 Bytes |
| Creation Time | 8 Bytes |
| Custom Alias | 100 Bytes |
| Expiration | 8 Bytes |
| Metadata | ~250 Bytes |

Approximate record size:

```
≈ 500 Bytes
```

For **1 Billion URLs**

```
500 Bytes × 1B
≈ 500 GB
```

---

# 12. Consistency vs Availability

## Write Path

Requirements:

- Generate unique short URLs
- Prevent duplicate IDs

Uses:

- Strong consistency
- Global Counter
- Primary database

Consistency is prioritized over availability.

---

## Read Path

Requirements:

- Extremely low latency
- High availability

Uses:

- Redis cache
- Read-through caching
- Eventual consistency

Availability is prioritized over strict consistency.

---

# 13. Final Architecture

```
                  Client
                     │
              API Gateway
               /        \
              /          \
     Read Service     Write Service
           │                │
           │                │
        Redis Cache    Global Counter
           │                │
           └────────┬───────┘
                    │
             Primary Database
```

---

# Key Design Decisions

- ✅ Global Counter + Base62 for unique ID generation
- ✅ Redis Read-Through Cache
- ✅ Separate Read and Write Services
- ✅ API Gateway
- ✅ Horizontal Scaling
- ✅ 302 Redirect
- ✅ Strong Consistency for Writes
- ✅ Eventual Consistency for Reads
- ✅ High Availability on Redirects
