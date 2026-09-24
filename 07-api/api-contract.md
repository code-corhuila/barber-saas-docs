# Contrato de API — BarberSaaS (Módulo de Citas y Barberías)

## 1. Decisiones Arquitectónicas Cerradas (ADR)
* **D-C1 (Paginación):** Se utilizará paginación basada en cursor (`?cursor=xyz&limit=10`) para listar reservas, evitando duplicación de datos cuando se agregan citas simultáneamente.
* **D-C2 (Formatos de Fecha/Hora):** Se implementa la norma ISO-8601 en formato UTC (`YYYY-MM-DDTHH:mm:ssZ`) para garantizar consistencia horaria entre el cliente y la plataforma.
* **D-C3 (Códigos de Estado HTTP):**
  * `200 OK`: Consulta exitosa con contenido.
  * `201 Created`: Cita o recurso creado exitosamente.
  * `400 Bad Request`: Error de validación en la estructura de la petición.
  * `401 Unauthorized`: Falta el token de autenticación JWT.
  * `404 Not Found`: El recurso solicitado (ej. barbero o cita) no existe.
  * `422 Unprocessable Entity`: Error en las reglas de negocio (ej. horario no disponible).
  * `500 Internal Server Error`: Error no controlado en el servidor.
* **D-C4 (Manejo de Rangos de Fecha):** Las búsquedas por rango de tiempo utilizarán los parámetros `start_date` y `end_date` en formato ISO-8601 UTC.
* **D-C5 (Idempotencia en Creación):** Peticiones de creación de reservas deben incluir el encabezado `X-Idempotency-Key` (UUIDv4) para prevenir duplicados por reintentos de red.
* **D-C6 (Formato de Identificadores):** Todos los identificadores únicos (`barber_id`, `client_id`, `appointment_id`) utilizarán el formato `UUIDv4`.

---

## 2. Estructura Estándar de Errores

Respuestas de error (`4xx` y `5xx`) responderán obligatoriamente bajo el esquema JSON unificado en una de sus tres formas:

### Forma 1: Error de Regla de Negocio (422 Unprocessable Entity)
```json
{
  "error": "BARBER_NOT_AVAILABLE",
  "message": "El barbero seleccionado no tiene disponibilidad en ese horario",
  "timestamp": "2026-09-24T08:30:00Z",
  "path": "/api/v1/appointments",
  "details": [
    {
      "field": "appointment_time",
      "message": "Horario ocupado por otra reserva"
    }
  ]
}
```
---

## 3. Ficha de Endpoint (E-01: Agendar Cita)

* **Endpoint:** `POST /api/v1/appointments`
* **Descripción:** Permite a un cliente reservar un turno con un barbero específico.
* **Headers:** 
  * `Content-Type: application/json`
  * `Authorization: Bearer <token_jwt>`
* **Cuerpo de la Petición (Request Body):**
```json
{
  "barbershop_id": "b10a28fb-5c12-4c2f-b201-9a7e80f4a812",
  "barber_id": "c8091a12-8802-4011-82ff-32b0a1d41bc1",
  "service_id": "f2901b81-9981-4200-a001-11b223344556",
  "appointment_date": "2026-09-25T15:00:00Z"
}
```
* **Respuesta Exitosa (`201 Created`):**
```json
{
  "id": "a90182bc-1122-3344-5566-778899aabbcc",
  "status": "CONFIRMED",
  "created_at": "2026-09-24T08:30:00Z"
}
```
---

## 4. Huecos Pendientes y Responsables (Gaps)

| ID Gap | Descripción del Hueco Pendiente | Módulo | Responsable |
| :--- | :--- | :--- | :--- |
| **GAP-01** | Definir reglas y tiempos límite para la cancelación de citas | Citas | *Por definir* |
| **GAP-02** | Especificar esquema de notificaciones de confirmación por WhatsApp/Email | Notificaciones | *Por definir* |
