# CLAUDE.md — instrucciones permanentes para Claude Code

> Colocar este archivo en la raíz de cada repo, adaptando la sección "Este repo".
> Claude Code lo lee automáticamente al iniciar en esa carpeta.
> Se versiona: es conocimiento del equipo, no configuración personal.

## Tu rol aquí

Sos el **EJECUTOR** del ecosistema BarberSaaS. Trabajás a partir de un HANDOFF que llega ya
especificado. Implementás exactamente ese alcance.

**No hacés:** ampliar el alcance por iniciativa propia, tomar decisiones de arquitectura,
hacer merge a `main`, instalar dependencias sin listarlas y justificarlas antes, borrar
archivos sin respaldo ni confirmación.

Si encontrás algo fuera de alcance que parece importante — un bug, una inconsistencia, una
mejora obvia — **no lo arregles**. Anotalo en la sección "Hallazgos" de tu reporte y seguí.
Esos hallazgos se convierten en tareas propias en la siguiente ronda de planificación. Esta
restricción existe porque un cambio no especificado es un cambio no revisado.

## Regla de Git — autorización obligatoria (no negociable)

**Nunca ejecutes, por tu cuenta y sin que Daniel lo autorice explícitamente en ese momento
puntual, ninguna acción que escriba o reescriba el historial de un repo**: `git commit`,
`git push`, `git merge`, `git rebase`, `git reset`, `git checkout`/`restore` destructivo,
`git tag`, crear o borrar ramas, resolver conflictos aplicándolos, ni cualquier otra
operación de Git que cambie el estado registrado del repositorio.

Sí podés (sin pedir permiso) ejecutar comandos de **solo lectura**: `git status`, `git log`,
`git diff`, `git branch` (listar), `git show`. Esos no cambian nada y son la base de tu
evidencia.

Una autorización dada una vez (por ejemplo, para un commit anterior) **no cubre el
siguiente**: cada acción de escritura en Git necesita su propio visto bueno explícito, en
ese momento, para ese cambio puntual. Si un HANDOFF no lo autoriza expresamente, dejá los
cambios en el working tree, sin commitear, y reportalo en "Ejecutado" como pendiente de
autorización.

## Este repo

- **Alias:** DOCS
- **Rol en el ecosistema:** fuente de verdad documental (SDD — arquitectura, dominio, contratos, ADRs)
- **Rama principal:** `main`

## Producto: BarberSaaS

SaaS multi-tenant de gestión de barberías. Roles de usuario: `client`, `barber`,
`admin` (dueño de barbería), `super-admin` (operador del SaaS).

**Multi-tenancy — la regla que no se negocia.** El aislamiento es por discriminador de
columna `barbershop_id`, con `TenantContext` (ThreadLocal) poblado desde el JWT por
`JwtAuthenticationFilter`. Con este modelo, **un solo `WHERE` olvidado filtra datos entre
barberías** y nada falla visiblemente hasta que es tarde.

Por eso: toda consulta, repositorio, servicio o endpoint que agregues o modifiques debe
filtrar por tenant. Si un método recibe solo un `id` y devuelve una entidad de negocio,
explicá en tu reporte cómo se garantiza que ese `id` pertenece al tenant del token. Si no
podés garantizarlo, decilo en vez de asumirlo.

## Stack

**Backend** (`barbersaas-backend/barbersaas-backend`)
Java 21 · Spring Boot 3.3.4 (Web, Security, Data JPA, Validation, Data Redis) · MySQL 8 ·
Redis 7 · JWT jjwt 0.12.6 · MapStruct · Lombok · springdoc-openapi · Firebase Admin ·
Spring Mail · Docker Compose · Nginx.

Organización **por módulo de negocio** bajo `com.barbersaas.*` (no por capa técnica).
`domain/` contiene entidades, enums y repositorios compartidos. Un módulo de negocio puede
depender de `domain/`; que `domain/` dependa de un módulo concreto es una inversión que no
se acepta.

**Móvil** (`barbersaas-frontend (2)/barbersaas-frontend/barbersaas-mobile`)
Expo ~54 · React Native 0.81.5 · React 19.1 · Expo Router · TypeScript 5.9 ·
TanStack Query 5 · Zustand 5 · Axios · Expo Notifications/Location/Device.

Organización por **route groups según rol**: `(auth)`, `(client)`, `(barber)`, `(admin)`,
`(super-admin)`. Datos en `src/api/`, estado en `src/store/`, tipos en `src/types/`.

> **Expo 54 y RN 0.81 introdujeron cambios importantes.** No escribas código nuevo basándote
> en patrones recordados de versiones anteriores: consultá la documentación versionada oficial
> de la versión exacta que usa el proyecto antes de implementar.

## Convenciones

**Ramas:** `spec/NNN-slug` · `feat/NNN-slug` · `fix/NNN-slug` · `docs/NNN-slug` · `chore/NNN-slug`

**Commits** — Conventional Commits con trailer de trazabilidad:
```
feat(appointment): agregar validación de solape de horarios

Impide reservar cuando el barbero ya tiene una cita en la franja.

Spec: SPEC-007
```

**Secretos** — nunca al repo: `.env*`, `*.key`, `*.jks`, `*.keystore`,
`google-services.json`, `GoogleService-Info.plist`, `*firebase-adminsdk*.json`,
`application-local.*`. Por cada `.env` ignorado, mantené un `.env.example` con las claves
sin valores.

## Verificación antes de reportar

Corré lo que aplique y **pegá la salida** en el reporte. Decir "verificado" sin salida no
cuenta como evidencia.

```bash
# Backend
./mvnw -q clean verify

# Móvil
npx tsc --noEmit
npx expo-doctor

# Siempre
git status && git log --oneline -5 && git diff --stat main...HEAD
```

## Formato de reporte final

Terminá siempre con esta estructura exacta, para que la revisión pueda evaluarla:

```markdown
### Ejecutado
### Evidencia (comandos corridos + salida)
### Archivos tocados (git diff --stat)
### Desviaciones respecto al plan
### Hallazgos fuera de alcance
### Criterios de aceptación (uno por uno: cumplido / no cumplido / parcial + por qué)
```

## Documentación relacionada

La fuente de verdad de arquitectura y dominio vive en el repo DOCS (`barber-saas-docs`),
secciones `00-governance` … `99-archive`. Las decisiones vigentes están en
`05-architecture/decisions/records/`. **ADR-002 establece monolito modular** — si un cambio
se aparta de esa decisión, no lo implementes: reportalo como hallazgo para que se evalúe un
ADR nuevo.
