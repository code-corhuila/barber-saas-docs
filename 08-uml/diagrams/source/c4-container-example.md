# Example: C4 Container-Level Diagram

> **Framework example.** BarberSaaS's real container diagram is in
> `05-architecture/overview.md` §3 (8 domain services behind the api-gateway, one database per
> domain).

> This file shows how to document the system with a C4 Level 2 (Container) diagram.
> The diagram uses Mermaid, which renders natively on GitHub, GitLab and most modern editors
> (VS Code with an extension, Obsidian, etc.).
>
> **Instruction:** Copy this structure, replace the example services (api-gateway,
> auth-service) with your project's real services, and move it to `diagrams/source/c4-container.md`.

---

## Example system: [X] management platform

```mermaid
C4Container
    title Container Diagram — [System Name]

    Person(user, "User / Client", "Accesses the system from the browser or mobile app")
    Person(admin, "Administrator", "Manages the system's configuration and users")

    System_Boundary(system, "[System Name]") {

        Container(frontend, "Web Frontend", "React / Vue / Angular", "User interface. SPA served as static files.")

        Container(gateway, "API Gateway", "[Your framework]", "Single entry point. Authenticates JWTs, routes to the right service, applies rate limiting.")

        Container(auth, "Auth Service", "[Your stack]", "Handles authentication (login/register/refresh). Owner of the User and Role entities.")

        Container(service_a, "[Service A]", "[Your stack]", "[Main responsibility of service A. Owner of [Entity A].]")

        Container(service_b, "[Service B]", "[Your stack]", "[Main responsibility of service B. Owner of [Entity B].]")

        ContainerDb(db_auth, "Auth DB", "PostgreSQL", "Users, roles, refresh tokens.")
        ContainerDb(db_a, "[Service A] DB", "[Chosen engine]", "Domain data of [Service A].")
        ContainerDb(db_b, "[Service B] DB", "[Chosen engine]", "Domain data of [Service B].")
        ContainerDb(redis, "Redis", "Redis", "Token blacklist, rate limiting, shared cache.")
        ContainerDb(broker, "Message Broker", "Kafka / RabbitMQ", "Domain events between services.")
    }

    System_Ext(ext_email, "Email Service", "Sendgrid / SES / SMTP")
    System_Ext(ext_payment, "Payment gateway", "[If it applies to the project]")

    %% User → System relations
    Rel(user, frontend, "Uses", "HTTPS")
    Rel(admin, frontend, "Administers", "HTTPS")
    Rel(frontend, gateway, "API calls", "HTTPS / REST")

    %% Gateway → Services
    Rel(gateway, auth, "Verifies tokens / routes", "Internal HTTP")
    Rel(gateway, service_a, "Routes requests", "Internal HTTP")
    Rel(gateway, service_b, "Routes requests", "Internal HTTP")

    %% Services → DB (each service has its own DB)
    Rel(auth, db_auth, "Reads / writes", "SQL")
    Rel(auth, redis, "Token blacklist", "Redis protocol")
    Rel(gateway, redis, "Rate limiting", "Redis protocol")
    Rel(service_a, db_a, "Reads / writes", "SQL / NoSQL")
    Rel(service_b, db_b, "Reads / writes", "SQL / NoSQL")

    %% Asynchronous communication
    Rel(auth, broker, "Publishes events (user.registered)", "Async")
    Rel(service_a, broker, "Publishes / consumes events", "Async")
    Rel(service_b, broker, "Consumes [Service A] events", "Async")

    %% External systems
    Rel(service_b, ext_email, "Sends notifications", "HTTPS / SMTP")
    Rel(service_a, ext_payment, "Processes payments", "HTTPS")
```

---

## How to use this diagram

1. **Render on GitHub:** The diagram renders automatically on push. No extra tool is needed.

2. **Render in VS Code:** Install the "Markdown Preview Mermaid Support" extension or use the official Mermaid extension.

3. **Export as an image:** Use the [Mermaid CLI](https://github.com/mermaid-js/mermaid-cli):
```bash
mmdc -i c4-container-example.md -o ../exports/c4-container.svg
```

4. **Update the diagram:** The diagram lives in Git next to the code. When the architecture changes (new service, new DB engine), update the diagram in the same PR.

---

## Conventions for this project

| Element | Color / Style | When to use it |
|---------|---------------|----------------|
| `Container` | Blue | Microservice or deployable application |
| `ContainerDb` | Blue cylinder | Database, cache, broker |
| `Person` | Person icon | Human actor |
| `System_Ext` | Gray | Third-party system you do not control |

---

## Correlations

- Service catalog → `09-microservices/service-catalog.md`
- Details of each service → `09-microservices/services/`
- Index of every diagram → `08-uml/diagram-index.md`
