# Project Glossary — Luxury Barber

> **Instructions:** Define here all technical and business terms used in the project.
> This is the official dictionary — if there is ambiguity, this document wins.
> Add terms throughout the project, not just at the beginning.

---

## How to use this glossary

1. Before using a technical or business term in code, docs, or conversations: look it up here.
2. If it is not here: add it with its definition.
3. If there is disagreement about the definition: discuss it with the team and update this document.

---

## Domain Terms

| Term | Definition | Notes / Synonyms |
|---------|------------|------------------|
| **Appointment (or Booking)** | Assignment of a service to a customer, attended by a specific barber on a given date and time. | *Synonyms to AVOID: Turn, Event, Slot.* |
| **Appointment Status** | Phase of the lifecycle in which a booking currently is (e.g., Pending payment, Confirmed, Completed, Canceled). | |
| **Barber** | Internal user (employee) enabled to execute services and who only has access to manage their own daily schedule. | *Synonyms to AVOID: Stylist, Hairdresser.* |
| **Receptionist** | Internal user who manages appointments and physical payments at the front desk, without access to configuration or reports. | |
| **Administrator** | User with full access to manage the system: users, services, prices, inventory, and financial reports. | |
| **Service Catalog** | List of available options to book (e.g., Haircut, Beard, Classic shave), including price, duration, and trained barbers. | |
| **Luxury Barber** | Fictional barbershop conceived as a business case for the academic project. | |

---

## Technical Project Terms

| Term | Definition |
|---------|------------|
| **Microservice** | Independent service with a single responsibility, its own process, and its own database. In this project: Auth/User, Appointment, Payment, and Notification. |
| **Database per Service** | Pattern where each microservice exclusively manages its own database (MongoDB). External relationships are handled by referential IDs, never by direct joins. |
| **Domain Event** | Immutable fact that occurred in the business and that other services can observe (e.g., `AppointmentCreated`). Its name is always in the past tense. |
| **Event Choreography** | Implementation of the Saga pattern where each service reacts to events autonomously (via RabbitMQ) without a central coordinator. |
| **Bounded Context** | Boundary within which a particular domain model has consistent meaning (e.g., the "Payments" context vs. the "Appointments" context). |
| **Eventual Consistency** | Guarantee that, after a change (e.g., a successful payment), all services will update their databases to be consistent within a maximum of 5 seconds. |
| **Service Discovery** | Tool (Eureka) that allows dynamic registration of instances, avoiding hardcoding IPs in the source code. |
| **API Gateway** | Single HTTP entry point to the system that routes REST requests from clients to the corresponding microservices. |
| **Circuit Breaker** | Fault tolerance pattern (Resilience4j) that stops calls to a failing service (e.g., Notification Service) to prevent cascading blocks. |
| **Saga** | Sequence of distributed transactions across different microservices, with compensating actions in case of failure (e.g., reverting appointment status if payment fails). |
| **Dead Letter Queue (DLQ)** | Queue in RabbitMQ where messages/events that could not be processed after exceeding the retry limit (3 attempts) are sent. |
| **Idempotency** | Property guaranteeing that processing the same event multiple times (using its unique `eventId`) will not generate duplicated side effects, such as double charging. |

---

## Acronyms

| Acronym | Meaning |
|----------|-------------|
| **PDR** | Product/Project Design Requirements |
| **MVP** | Minimum Viable Product |
| **RBAC** | Role-Based Access Control |
| **SaaS** | Software as a Service |
| **AMQP** | Advanced Message Queuing Protocol (Protocol used by RabbitMQ) |
| **JWT** | JSON Web Token (Used for authentication) |
| **API** | Application Programming Interface |
| **CRUD** | Create, Read, Update, Delete |
| **DTO** | Data Transfer Object |
| **FR** | Functional Requirement (translated from RF) |
| **NFR** | Non-Functional Requirement (translated from RNF) |
| **CI/CD** | Continuous Integration / Continuous Delivery |
| **PR** | Pull Request |
