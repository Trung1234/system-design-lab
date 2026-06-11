# Travel AI Agent MVP: Architecture Choice

## 1. Problem clarification
We need to choose between two architectures for a travel AI agent MVP:

- **A:** Monolith with FastAPI + PostgreSQL
- **B:** Microservices + Kafka + Redis + PostgreSQL

The goal is to build a product that can validate the idea quickly, support basic travel planning workflows, and keep costs and operational complexity low.

## 2. Requirements
### Functional
- User signup/login
- Chat with the travel agent
- Search and suggest trips, hotels, flights, or activities
- Store user preferences and conversation history
- Generate itineraries

### Non-functional
- Fast delivery for MVP
- Simple operations
- Low infrastructure cost
- Easy debugging
- Room to scale later

## 3. High-level architecture
### Choose now: **A: Monolith + FastAPI + PostgreSQL**

**Why:**
- Faster to build
- Easier to deploy
- Easier to debug
- Fewer moving parts
- Good enough for MVP traffic

A clean monolith can still be organized by modules such as:
- auth
- chat
- itinerary
- search integration
- user profile
- bookings later

## 4. Data model
Use PostgreSQL as the main source of truth for:
- users
- conversations
- messages
- preferences
- itineraries
- search results cache if needed

Redis is optional later for:
- caching
- session storage
- rate limiting
- background job state

Kafka is not needed for the MVP unless there are many async workflows.

## 5. API design
Typical endpoints could be:
- `POST /auth/register`
- `POST /auth/login`
- `POST /chat/message`
- `GET /itineraries/{id}`
- `POST /itineraries/generate`
- `GET /users/me`

Keep APIs simple and synchronous first.

## 6. Scaling bottlenecks
### Monolith bottlenecks
- One app instance becomes CPU-bound
- Long AI calls may slow requests
- Database can become the first bottleneck

### Why this is acceptable now
For an MVP, this is usually fine because traffic is still low and the main goal is product validation.

### Future scaling path
- Add Redis for caching
- Add background workers for long-running tasks
- Split out services only when boundaries are clear

## 7. Reliability
### Monolith benefits
- Easier to monitor
- Fewer network failures
- Simpler rollback
- Easier incident debugging

### Risks
- A bug can affect the whole app
- Heavy AI requests may block request handling

### Mitigation
- Use timeouts
- Add retries for external APIs
- Offload long tasks to background jobs
- Add structured logging

## 8. Security
- Store secrets in environment variables
- Hash passwords securely
- Validate user input carefully
- Protect API endpoints with authentication
- Rate-limit chat and search requests
- Avoid logging sensitive travel or payment data

## 9. Cost
### Option A
- Lower cloud cost
- Fewer services to manage
- Smaller DevOps burden

### Option B
- Higher cost
- More infrastructure overhead
- More engineering time spent on operations

For an MVP, cost efficiency strongly favors A.

## 10. Interview-style feedback
### Final recommendation
**Choose A now.**

### Reason in one line
A monolith gives the fastest path to validating the travel AI agent idea without paying the complexity tax of microservices too early.

### When to move to B later
Move toward microservices only after:
- the product is validated
- traffic grows significantly
- async workflows become common
- multiple teams need independent deployment
- service boundaries are obvious

### Score
If I were grading this decision in a system design interview, I would give choosing **A** for the MVP a **9/10**.
