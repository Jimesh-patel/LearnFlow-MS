# LearnFlow - Microservices Learning Management System

LearnFlow is a comprehensive, scalable e-learning platform built using a modern microservices architecture. It leverages Spring Boot, PostgreSQL, Apache Kafka, and Spring Cloud to deliver a robust, highly decoupled, and event-driven backend.

## 🚀 Architecture & Technologies

* **Microservices Framework:** Java & Spring Boot
* **Service Discovery:** Eureka Discovery Server
* **API Gateway:** Spring Cloud Gateway (Centralized routing and load balancing)
* **Databases:** PostgreSQL (Database-per-Service pattern)
* **Message Broker / Event Streaming:** Apache Kafka (KRaft mode)
* **Security:** JWT Authentication
* **External Integrations:** 
  * Stripe (Payments & Subscriptions)
  * Cloudinary (Media Storage & Video Management)
* **Containerization:** Docker & Docker Compose

## 🏗️ Services Overview

The system is decomposed into several independent business capabilities:

1. **API Gateway (`api-gateway`)**: The single entry point for all client requests. It handles intelligent routing and JWT security validation.
2. **Discovery Server (`discovery-server`)**: Eureka server for dynamic service registration and discovery, allowing services to find each other without hardcoded IPs.
3. **Auth Service (`auth-service`)**: Handles user authentication, registration, and JWT token generation.
4. **User Service (`user-service`)**: Manages user profiles, roles (Admin, Instructor, Student), and accounts.
5. **Course Service (`course-service`)**: Manages course creation, metadata, updates, and cataloging.
6. **Lecture Service (`lecture-service`)**: Handles individual lecture content, structures, and sequencing within courses.
7. **Cloudinary Service (`cloudinary-service`)**: Asynchronously processes and manages media uploads (images, videos) via Kafka events.
8. **Progress Service (`progress-service`)**: Tracks user learning progress, course enrollment, and completion statuses.
9. **Payment Service (`payment-service`)**: Processes financial transactions, manages pricing, and handles asynchronous Stripe webhooks.

## ⚙️ Getting Started

### Prerequisites
* Docker and Docker Compose
* Java 17+ (if running locally without Docker)
* Maven (if building from source)

### Setup & Installation

1. **Environment Variables Configuration**
   Copy the provided `.env.example` file to a new file named `.env` in the root directory. Fill in your secure credentials (such as Stripe API keys, Cloudinary credentials, a secure JWT secret, and database passwords):
   ```bash
   cp .env.example .env
   ```

2. **Start the Infrastructure**
   The project is fully containerized. You can spin up all microservices, PostgreSQL instances, Eureka, and Kafka using Docker Compose:
   ```bash
   docker-compose up -d
   ```

3. **Accessing the System**
   - **API Gateway (Main Entry Point):** `http://localhost:8080`
   - **Eureka Service Dashboard:** `http://localhost:8761`

## 📡 Event-Driven Communication (Kafka)

LearnFlow heavily utilizes **Apache Kafka** for asynchronous, decoupled inter-service communication. This ensures high availability and responsiveness. For example:
- Media uploads are offloaded. When a user uploads a video, an event is published to Kafka, which is then asynchronously consumed by the `cloudinary-service`.
- Post-payment processing and course unlocking are handled via Kafka events emitted by the `payment-service` after successful Stripe webhook verification.

## 💾 Database-per-Service Pattern

To ensure true microservices loose coupling and data autonomy, LearnFlow implements the Database-per-Service pattern. Each service has its own dedicated PostgreSQL container:
* `user-db` (Port: 5000)
* `course-db` (Port: 5001)
* `lecture-db` (Port: 5002)
* `progress-db` (Port: 5003)
* `payment-db` (Port: 5004)
