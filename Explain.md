# LearnFlow — Complete Project Explanation & Interview Prep Guide

This document is designed to help you understand the **LearnFlow** project inside and out, so you can confidently talk about it in interviews as if you built it from scratch.

---

## 1. The "Elevator Pitch" (What is it?)

**LearnFlow** is a modern, scalable **Learning Management System (LMS)** built using a **Microservices Architecture**. It allows users to browse courses, purchase them via Stripe, consume video lectures (stored on Cloudinary), and track their learning progress. Because it's built with microservices, different parts of the application can scale independently and fail gracefully without taking down the entire system.

**Tech Stack:** Java, Spring Boot, PostgreSQL, Apache Kafka, gRPC, Spring Cloud Gateway, Netflix Eureka, Stripe, Cloudinary, Docker, Protobuf.

---

## 2. Core Architecture

The system is divided into several small, independent applications (microservices) rather than one giant codebase.

### 2.1 API Gateway (Spring Cloud Gateway)
This is the **front door**. The React frontend (or any client) doesn't talk to individual services. It talks to the Gateway on port `8080`. The Gateway routes the request to the correct underlying service. It also handles verifying the JWT (Security Token) so individual services don't have to duplicate that logic.

### 2.2 Service Discovery (Netflix Eureka)
Since microservices might scale up or change IP addresses, they need a way to find each other. When a service (like `user-service`) starts up, it **registers itself** with Eureka. When the API Gateway needs to route a request to `user-service`, it asks Eureka where it is.

### 2.3 The Microservices
| Service | Responsibility |
|---|---|
| `auth-service` | Handles login and generates JWT tokens |
| `user-service` | Manages user profiles, roles, and enrolled courses |
| `course-service` | Manages course metadata (titles, descriptions, prices, thumbnails) |
| `lecture-service` | Manages video lecture structures and content URLs |
| `payment-service` | Integrates with Stripe for course purchases |
| `progress-service` | Tracks how much of a course a student has completed |
| `cloudinary-service` | Dedicated service for handling heavy media uploads to Cloudinary |

---

## 3. Hybrid Communication Strategy

This project uses **three different communication methods**, each chosen based on the specific use case:

| Method | When to use | Example |
|---|---|---|
| **REST (HTTP/JSON)** | Client (React) → API Gateway | User login, course creation, fetching profiles |
| **Kafka (Async)** | Fire-and-forget events between services | Payment completed → enroll student, image upload |
| **gRPC (Sync, fast)** | Service → Service, when you need an **instant answer** | "Has this user paid for this course?" |

---

## 4. API Gateway Routing Logic

The API Gateway uses **Spring Cloud Gateway** to handle all request routing. Here is how each concept works:

### 4.1 Load Balancing & Dynamic Routing (`lb://`)
The destination URI uses the `lb://` prefix (e.g., `lb://USER-SERVICE`). This stands for **Load Balancer**. The Gateway asks the Eureka Discovery Server for the IP addresses of all running instances of `USER-SERVICE` and dynamically routes the request to one of them. If you scale up to 3 user-services, the Gateway automatically load-balances across all three.

### 4.2 Predicates (The "If" Condition)
Predicates dictate *which* route a request takes based on its URL path:
```properties
spring.cloud.gateway.routes[0].predicates[0]=Path=/auth/**
```
When a client sends a request to `http://localhost:8080/auth/login`, the Gateway evaluates predicates. Because the path matches `/auth/**`, it redirects to the `AUTH-SERVICE`.

### 4.3 Filters (Modifying the Request)
```properties
spring.cloud.gateway.routes[0].filters[0]=StripPrefix=1
```
If a request comes in as `/auth/login`, the `StripPrefix=1` filter removes the first segment (`/auth`). The Gateway forwards the request to the Auth Service as simply `/login`. This means microservices don't need to know about the gateway's routing paths.

### 4.4 Custom Security Filters (`JwtValidation`)
```properties
spring.cloud.gateway.routes[1].filters[1]=JwtValidation
```
This is a custom Java filter class. When a user tries to access `/api/users/profile`, this filter intercepts the request, looks for the JWT in the headers, and verifies the signature using the `jwt.secret`. If the token is invalid or expired, the filter immediately returns a `401 Unauthorized` error, and the request **never even reaches** the `USER-SERVICE`.

### 4.5 Route Priority (Specific vs. General)
Routes are ordered so that specific, secure routes are evaluated before general wildcard routes:
- **Route 1 (Secure):** Specifically matches `/api/users/profile` → applies `JwtValidation` filter.
- **Route 2 (General):** Matches everything else `/api/users/**` → no JWT filter (allows public access to `/api/users/register`).

---

## 5. Authentication Flow

### Step 1: User Registration
- **Endpoint:** `POST http://localhost:8080/api/users/register`
- **Flow:** A new user submits their details (Name, Email, Password). The API Gateway routes this to the `user-service`. The `user-service` hashes the password securely and stores the new user record in `user-db`.

### Step 2: User Login
- **Endpoint:** `POST http://localhost:8080/auth/login`
- **Flow:** The user provides `email` and `password`. The API Gateway routes this to the `auth-service`. The `auth-service` verifies credentials and generates a signed **JWT (JSON Web Token)**. This token acts as a temporary "ID card" and is returned to the client.

### Step 3: Accessing Protected Routes
- **Endpoint:** `GET http://localhost:8080/api/users/profile`
- **Flow:** The client attaches the JWT (from login) in the `Authorization` header. The API Gateway intercepts and **validates the JWT** first. If invalid → blocked with `401`. If valid → forwarded to the `user-service`, which returns the profile data.

### Step 4: User Logout
- **Endpoint:** `POST http://localhost:8080/auth/logout`
- **Flow:** The `auth-service` invalidates the session by clearing the authentication cookie on the client side or blacklisting the active JWT.

---

## 6. Database-per-Service Pattern

Instead of one massive PostgreSQL database, **5 different services have their own isolated PostgreSQL databases**:

| Service | Database | Docker Port |
|---|---|---|
| `user-service` | `user-db` | 5000 |
| `course-service` | `course-db` | 5001 |
| `lecture-service` | `lecture-db` | 5002 |
| `progress-service` | `progress-db` | 5003 |
| `payment-service` | `payment-db` | 5004 |

**Why?**
- **Loose Coupling:** If the `course-db` goes down, users can still log in via the `user-service`.
- **Independent Scaling:** If `progress-service` gets heavy write traffic, we can scale its database without paying to scale the `user-db`.

---

## 7. Lecture & Progress Endpoints Flow

### 7.1 Creating Content (Instructor)
- **Endpoint:** `POST /api/lectures/{courseId}`
- An instructor adds a new lecture to an existing course. The `lecture-service` saves this lecture metadata into `lecture-db`, linked to the specified `courseId`.

### 7.2 Fetching Content (Student)
- **Endpoint:** `GET /api/lectures/{courseId}`
- When a student navigates to a course page, the frontend calls this to get the list of all available lectures for that course.

### 7.3 Marking a Lecture as Viewed
- **Endpoint:** `POST /api/progress/lecture/view/{courseId}/{lectureId}`
- When a student finishes watching a video, the frontend fires this endpoint. The `progress-service` records in `progress-db` that this user has viewed this specific lecture.

### 7.4 Getting Overall Course Progress
- **Endpoint:** `GET /api/progress/course/{courseId}`
- The `progress-service` checks how many lectures the user has marked as "viewed" for this course, compares it against the total, and returns the completion percentage (e.g., "50% done").

### End-to-End User Journey:
1. **Instructor** uses `POST /api/lectures` to build the course syllabus.
2. **Student** visits the course page → frontend calls `GET /api/lectures` to load videos and `GET /api/progress/course` to see which are already watched.
3. **Student** watches a video → frontend fires `POST /api/progress/lecture/view` to record it.
4. **UI Updates** → re-fetches `GET /api/progress/course` to update the progress bar from 50% to 60%.

---

## 8. File Upload Process (Cloudinary Service)

The `cloudinary-service` is a dedicated microservice that exclusively handles uploading files to Cloudinary's cloud servers. The project implements a **hybrid upload strategy**:

### 8.1 Video Upload (Synchronous via REST Controller)
Videos can be massive (500MB–1GB). They **cannot** be passed through Kafka (messages are designed to be < 1MB). So the frontend streams the video file **directly** to the `cloudinary-service` via a standard HTTP POST:

**File:** `cloudinary-service/.../controller/CloudinaryController.java`
```java
@PostMapping("/video-upload")
public ResponseEntity<?> uploadVideo(@RequestParam("file") MultipartFile file) {
    // 1. Checks if the user has the 'Instructor' role
    // 2. Passes the file to CloudinaryUtil.uploadVideo()
    // 3. Returns the secure Cloudinary URL back to the frontend
}
```

**File:** `cloudinary-service/.../util/CloudinaryUtil.java`
```java
public static Map uploadVideo(Cloudinary cloudinary, byte[] videoBytes) {
    Map<String, Object> options = ObjectUtils.asMap("resource_type", "video");
    return cloudinary.uploader().upload(videoBytes, options);
}
```

After receiving the `videoUrl` from Cloudinary, the frontend passes it as a JSON field when creating/editing a lecture in the `lecture-service`.

### 8.2 Image & Thumbnail Upload (Asynchronous via Kafka)
Images (profile pics, course thumbnails) are much smaller. The frontend converts them to a **Base64 string** (lightweight text), which *can* travel through Kafka.

**Flow for User Profile Photo:**
1. Frontend hits `user-service` to update the profile, passing the Base64 image string.
2. `user-service` publishes a Kafka event to `"user-photo-updated-topic"` containing the Base64 data.
3. `user-service` immediately responds `200 OK` (non-blocking).
4. `cloudinary-service` consumes the Kafka event, uploads the image to Cloudinary via `CloudinaryUtil.uploadImage()`.
5. `cloudinary-service` publishes a completion event back to Kafka with the new secure URL.
6. `user-service` receives the completion event and updates the `photoUrl` in `user-db`.

**Flow for Course Thumbnail:**
Same pattern, but using the `"course-edited-topic"` Kafka topic:
1. `course-service` publishes thumbnail data to Kafka.
2. `cloudinary-service` uploads it and fires back a `"course-thumbnail-uploaded-topic"` event.
3. `course-service` updates the `courseThumbnail` URL in `course-db`.

### 8.3 First-Time Thumbnail Upload (Two-Step Course Creation)
When a course is **first created**, no thumbnail is uploaded. The `CreateCourseRequest` only accepts a `courseTitle` and `category`. The course is saved as a basic shell.

The instructor is then redirected to a "Course Edit" page, where they upload the thumbnail, set the price, and add descriptions. When they click "Save", the `editCourse` endpoint is hit, which triggers the Kafka-based thumbnail upload flow described above.

**Why?** If the user abandons the form halfway through uploading massive assets, we haven't wasted API calls to Cloudinary uploading assets for a course that was never actually saved.

---

## 9. Payment Flow (Stripe Integration)

### Phase 1: Initiating the Payment
1. Student clicks "Buy Course" → frontend sends `POST /api/payment/{courseId}`.
2. The `payment-service` uses **gRPC** to call the `course-service` to get the course title and price (needed for the Stripe checkout page).
3. The `payment-service` uses the Stripe Java SDK to create a **Checkout Session** with the course details.
4. A `CoursePurchase` record is saved in `payment-db` with status `PENDING` and the unique Stripe Session ID.
5. The backend returns a Stripe-hosted URL to the frontend. The user is **redirected to Stripe's page** to enter their credit card.

### Phase 2: Processing the Payment (Webhook)
You **never** trust the frontend to confirm payment. Instead, Stripe's servers talk directly to your backend.

1. After successful payment, Stripe sends an HTTP POST to `StripeWebhookController` (`/webhook/stripe`).
2. The `StripeWebhookServiceImpl` **verifies the webhook signature** using the `STRIPE_WEBHOOK_SECRET` to ensure it's truly from Stripe and not a hacker.
3. If valid, it finds the `PENDING` purchase by Session ID and updates the status to `SUCCESS`.

```java
private void handleCheckoutSessionCompleted(Session session) {
    CoursePurchase purchase = coursePurchaseRepository.findByPaymentId(session.getId()).get();
    purchase.setStatus(Status.SUCCESS);
    coursePurchaseRepository.save(purchase);

    // Fire Kafka event to notify other services
    kafkaProducer.sendCoursePurchaseCompletedEvent(
            purchase.getCourseId().toString(),
            purchase.getUserId().toString()
    );
}
```

### Phase 3: Granting Course Access (Kafka)
The `payment-service` doesn't directly give the user access. It fires a `CoursePurchaseCompletedEvent` to Kafka:
- **`user-service`** listens and adds the `courseId` to the student's `enrolledCourseIds`.
- **`progress-service`** listens and initializes a blank progress tracker (0% complete).

### Stripe Keys (Where they come from)
| Key | Source | Purpose |
|---|---|---|
| `STRIPE_SECRET_KEY` | Stripe Dashboard → Developers → API Keys (starts with `sk_test_...`) | Authenticates all Stripe API calls. Set **globally** at startup in `StripeConfig.java` via `Stripe.apiKey = stripeSecretKey` |
| `STRIPE_WEBHOOK_SECRET` | Stripe Dashboard → Developers → Webhooks → Signing Secret (starts with `whsec_...`) | Verifies incoming webhook signatures in `StripeWebhookServiceImpl` |

Both keys are injected via environment variables in the `.env` file, which Docker Compose passes into the containers.

---

## 10. gRPC (Inter-Service Synchronous Communication)

### What is gRPC?
gRPC is a **high-performance Remote Procedure Call framework** by Google. Instead of sending text-based JSON over HTTP (like REST), gRPC sends **binary data** using **Protocol Buffers (Protobuf)**. This makes it ~10x faster than REST for internal service-to-service calls.

### Why use gRPC instead of REST or Kafka?
- **Can't use Kafka:** Kafka is asynchronous — you fire a message and never get a response back. But sometimes you need an answer **right now**.
- **Why not REST?** gRPC is significantly faster because it uses binary serialization and HTTP/2 multiplexing, which matters for high-frequency internal calls.

### Where gRPC is used in the project:

| Caller (Client) | Server | gRPC Method | Why? |
|---|---|---|---|
| `lecture-service` | `payment-service` | `checkUserCoursePurchase` | Check if student paid before showing video URL |
| `payment-service` | `course-service` | `getCourseDetailsById` | Get course title & price to create Stripe checkout |
| `progress-service` | `payment-service` | `checkUserCoursePurchase` | Verify purchase before tracking progress |
| `progress-service` | `course-service` | `getCourseDetailsById` | Get course metadata for progress calculations |

### How it works (Example):

**Client Side** (`lecture-service` calling `payment-service`):
```java
CheckUserCoursePurchaseResponse response = paymentStub.checkUserCoursePurchase(
        CheckUserCoursePurchaseRequest.newBuilder()
                .setUserId(currentUserId.toString())
                .setCourseId(courseId.toString())
                .build()
);
boolean hasPurchased = response.getHasPurchased();
```

**Server Side** (`payment-service` handling the call):
```java
@GrpcService
public class GrpcPaymentService extends PaymentServiceGrpc.PaymentServiceImplBase {
    public void checkUserCoursePurchase(CheckUserCoursePurchaseRequest request,
                                        StreamObserver<CheckUserCoursePurchaseResponse> responseObserver) {
        boolean exists = coursePurchaseRepository.existsByUserIdAndCourseIdAndStatus(
                userId, courseId, Status.SUCCESS
        );
        responseObserver.onNext(CheckUserCoursePurchaseResponse.newBuilder()
                .setHasPurchased(exists).build());
        responseObserver.onCompleted();
    }
}
```

---

## 11. Kafka (Asynchronous Event-Driven Communication)

### How it works (The Story):
1. A student buys a course.
2. Stripe processes the payment and sends a webhook to the `payment-service`.
3. The `payment-service` doesn't directly tell the `progress-service` to unlock the course via a REST call — if `progress-service` is temporarily down, the data is lost.
4. Instead, `payment-service` publishes a message to **Kafka**: `"User X paid for Course Y"`.
5. The `payment-service` is now done and responds quickly.
6. The `progress-service` (subscribed to Kafka) sees the message and unlocks Course Y for User X.

**Why is this good?** It provides **Eventual Consistency** and **Fault Tolerance**. If the `progress-service` is down, the message stays safely in Kafka until it comes back online.

### Kafka Topics used in the project:
| Topic | Producer | Consumer | Purpose |
|---|---|---|---|
| `user-photo-updated-topic` | `user-service` | `cloudinary-service` | Upload user profile photo |
| `user-photo-upload-completed-topic` | `cloudinary-service` | `user-service` | Update photo URL in user-db |
| `course-edited-topic` | `course-service` | `cloudinary-service` | Upload course thumbnail |
| `course-thumbnail-uploaded-topic` | `cloudinary-service` | `course-service` | Update thumbnail URL in course-db |
| `course-purchase-completed-topic` | `payment-service` | `user-service`, `progress-service` | Enroll student & initialize progress |

---

## 12. Common Interview Questions & Answers

**Q: Why did you choose Microservices instead of a Monolith?**
> *"I wanted to build a system that could scale specific domains independently. For an LMS, the 'progress tracking' or 'video streaming' services get way more traffic than the 'payment' service. Microservices let us allocate resources exactly where needed. It also forced me to learn distributed system patterns like API Gateways and Event-Driven architecture."*

**Q: How did you handle communication between your microservices?**
> *"I used a hybrid approach. For client-facing APIs, I used REST via the API Gateway. For internal synchronous queries—like checking if a user has purchased a course—I used gRPC because it's ~10x faster than REST with binary Protobuf serialization. For state changes like payments and media uploads, I used asynchronous Kafka events to prevent tight coupling."*

**Q: How do you handle data consistency since every service has its own database?**
> *"I relied on Eventual Consistency. We don't have distributed transactions (like Two-Phase Commit) because they are slow and lock databases. Instead, if a transaction spans multiple services, we use Kafka events. If a step fails, we would implement a compensation transaction (Saga Pattern) to undo the previous steps."*

**Q: How does Authentication work in your architecture?**
> *"We use JWT (JSON Web Tokens). When a user logs in via the auth-service, it validates credentials and returns a signed JWT. For subsequent requests, the client passes this JWT in the Authorization header. The API Gateway intercepts the request, validates the JWT signature, and if valid, routes the request to the underlying microservice. This means the microservices themselves don't need to implement auth logic."*

**Q: Why did you use gRPC instead of just REST for internal calls?**
> *"REST sends text-based JSON over HTTP/1.1, which involves parsing overhead. gRPC uses binary Protobuf serialization over HTTP/2 with multiplexing, making it significantly faster. For high-frequency internal calls—like checking purchase status on every page load—this performance difference matters. Also, gRPC gives us strongly-typed contracts via .proto files, catching integration errors at compile time rather than runtime."*

**Q: How does the payment system work?**
> *"I used Stripe Checkout for PCI compliance — we never handle credit card data directly. When a student clicks 'Buy', I create a Stripe Checkout Session and redirect them to Stripe's hosted page. After payment, Stripe sends a webhook to our backend. I verify the webhook signature cryptographically to prevent spoofing, update the purchase status to SUCCESS, and fire a Kafka event so the user-service can enroll the student and the progress-service can initialize their tracker."*

**Q: Why did you separate the Cloudinary service?**
> *"Uploading media is an I/O-heavy, slow operation. If I handled 1GB video uploads inside the lecture-service, it would block threads and make the entire service unresponsive for students trying to load course pages. By isolating media processing in a dedicated service and using Kafka for lightweight assets (images/thumbnails), the core services remain fast and available even during heavy upload activity."*

**Q: What was the hardest part of building this?**
> *"The hardest part was getting Docker Compose networking right with 12+ containers (databases, Kafka, Eureka, services) all communicating correctly. Also, ensuring that Stripe webhooks were processed idempotently — so if Stripe sends the webhook twice, we don't process the payment twice — required careful design with unique constraints on the CoursePurchase table."*

**Q: How do you handle a situation where a service is temporarily down?**
> *"For async operations, Kafka acts as a buffer. If the progress-service is down when a payment completes, the message stays in the Kafka topic until the service recovers. For sync gRPC calls, we wrap them in try-catch blocks with graceful fallbacks — for example, if the payment-service gRPC call fails when loading lectures, we default to hasPurchased = false so the student sees preview-only content instead of a crash."*
