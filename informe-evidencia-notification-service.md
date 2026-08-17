# Informe de Reconocimiento y Evidencia Técnica — `notification-service`

**Actividad:** Reconocimiento — notification-service (Entrega Transversal #3)  
**Aprendiz:** emily sharith amezquita saavedra  
**Fecha:** 16 de Agosto de 2026  
**Repositorios evaluados:**
- `design-software-docker-infra-main`
- `design-software-notification-db-main`

---

## 1. Contexto y Declaración Transparente de Alcance

> [!IMPORTANT]
> **Aclaración Académica:** La presente evaluación se realiza de forma estrictamente honesta y rigurosa sobre la copia oficial local de la infraestructura en Docker y los changelogs de base de datos en Liquibase. Debido a la no disponibilidad del código fuente del backend del microservicio (`notification-service`), la verificación se enfoca en la **capacidad de infraestructura, esquema de base de datos PostgreSQL, mensajería AMQP (RabbitMQ), servidor SMTP (MailHog) y observabilidad (Grafana / LGTM Stack)**.

---

## 2. Diagrama Arquitectónico de la Solución (Mermaid)

```mermaid
graph TD
    subgraph Edge Layer
        Traefik["Traefik Edge Router<br/>(Puerto 18080 / 18090)"]
    end

    subgraph Database Layer
        PostgreSQL[("PostgreSQL 16 Database<br/>(Puerto 15432)")]
        Liquibase["Liquibase Runner<br/>(Esquema: notification)"]
        Liquibase -.->|Migra DDL/DML| PostgreSQL
    end

    subgraph Broker Layer
        RabbitMQ["RabbitMQ AMQP Broker<br/>(Puerto 5672 / 15672)"]
        MailHog["MailHog Web & SMTP<br/>(Puerto 1025 / 18025)"]
    end

    subgraph Observability Stack (LGTM)
        Otel["OpenTelemetry Collector<br/>(Puerto 4317 / 4318)"]
        Prometheus["Prometheus<br/>(Métricas)"]
        Loki["Loki<br/>(Logs)"]
        Tempo["Tempo<br/>(Trazas)"]
        Grafana["Grafana UI<br/>(Puerto 3000)"]

        Otel --> Prometheus
        Otel --> Loki
        Otel --> Tempo
        Prometheus --> Grafana
        Loki --> Grafana
        Tempo --> Grafana
    end

    PostgreSQL --- OutboxTable["notification.outbox<br/>(uq_outbox_event_id)"]
    PostgreSQL --- SentTable["notification.sent_notification<br/>(EMAIL / IN_APP)"]
    PostgreSQL --- TemplateTable["notification.notification_template<br/>(Plantillas Semilla)"]
```

---

## 3. Matriz General de Evaluación de Historias de Usuario (HU-001 a HU-008)

| Historia de Usuario | Nombre / Descripción | Estado | Evidencia Comprobada | Componente Responsable |
|---|---|---|---|---|
| **HU-001** | Enviar notificación vía API (`POST /notifications`) | **Parcial** | Tabla `sent_notification` estructurada en PostgreSQL. | `notification.sent_notification` (Esquema BD) |
| **HU-002** | Consumir evento AMQP e Idempotencia | **Parcial** | RabbitMQ activo (5672/15672) y tabla `outbox` con clave única `uq_outbox_event_id`. | `RabbitMQ` + `notification.outbox` |
| **HU-003** | Entregar por canal EMAIL | **Parcial** | Servidor SMTP local MailHog activo (1025/18025) y canal `EMAIL` en BD. | `MailHog UI` + `sent_notification` |
| **HU-004** | Reintentos y Dead Letter Queue (DLQ) | **Parcial** | Campos `send_status = FAILED` y `failure_reason` estructurados en BD. | `sent_notification` (PostgreSQL) |
| **HU-005** | Consultar notificación enviada (`GET /notifications/{id}`) | **Parcial** | Consultas SQL probadas en BD sobre envíos registrados. | `sent_notification` (PostgreSQL) |
| **HU-006** | Plantillas de notificación parametrizables | **Comprobada (100%)** | Plantillas semilla `SCHEDULE_PUBLISHED` y `ALERT_TRIGGERED` creadas por Liquibase. | `notification.notification_template` |
| **HU-007** | Observabilidad de la capacidad | **Comprobada (100%)** | Stack LGTM (Grafana, Otel, Prometheus, Tempo, Loki) desplegado y conectado. | `Grafana` + `OTEL` + `LGTM` |
| **HU-008** | Despliegue y Ejecución Local | **Comprobada (100%)** | Despliegue de 9 contenedores mediante Docker Compose y migración Liquibase. | `Docker Compose` + `Liquibase` |

---

## 4. Evidencias de Ejecución de Comandos y Consultas SQL

### 4.1 Comprobación del Motor Docker (`HU-008`)
```powershell
docker info
```
- **Resultado:** Motor Docker v29.2.1 en estado activo sobre WSL2.

### 4.2 Despliegue de Infraestructura Multi-Contenedor (`HU-008`)
```powershell
docker compose -f design-software-docker-infra-main/docker-compose.yml --profile broker --profile edge --profile observability up postgres rabbitmq mailhog traefik grafana otel-collector prometheus tempo loki -d
```
- **Resultado:** 9 contenedores iniciados exitosamente.

### 4.3 Migración de Base de Datos con Liquibase (`HU-006` / `HU-008`)
```powershell
docker compose -f design-software-docker-infra-main/docker-compose.yml --profile tooling run --rm liquibase-notification update
```
- **Resultado de Liquibase:**
```text
UPDATE SUMMARY
Run:                         11
Previously run:               0
Filtered out:                 0
-------------------------------
Total change sets:           11

Liquibase: Update has been successful. Rows affected: 2
```

### 4.4 Estado de Contenedores en Ejecución
```powershell
docker compose -f design-software-docker-infra-main/docker-compose.yml ps
```
- **Resultado:**
  - `postgres` (15432) -> Healthy
  - `rabbitmq` (5672, 15672) -> Healthy
  - `mailhog` (1025, 18025) -> Up
  - `grafana` (3000) -> Up
  - `traefik` (18080, 18090) -> Up
  - `otel-collector`, `prometheus`, `tempo`, `loki` -> Up

### 4.5 Consultas SQL sobre PostgreSQL

#### Plantillas Semilla (`notification_template` — HU-006):
```sql
SELECT code, name, channel, subject_template FROM notification.notification_template;
```
```text
        code        |         name          | channel |              subject_template              
--------------------+-----------------------+---------+--------------------------------------------
 SCHEDULE_PUBLISHED | Horario publicado     | EMAIL   | Tu horario {{schedule_name}} fue publicado
 ALERT_TRIGGERED    | Alerta de seguimiento | IN_APP  | Alerta: {{alert_type}}
(2 rows)
```

#### Estructura de Idempotencia (`outbox` — HU-002):
```sql
SELECT column_name, data_type FROM information_schema.columns WHERE table_schema = 'notification' AND table_name = 'outbox';
```
```text
 column_name  |        data_type         
--------------+--------------------------
 id           | uuid                     
 event_id     | uuid                     
 event_type   | character varying        
 payload      | jsonb                    
 created_at   | timestamp with time zone 
 published_at | timestamp with time zone 
```

#### Estructura de Auditoría y Canales (`sent_notification` — HU-001, HU-003, HU-004):
```sql
SELECT column_name, data_type FROM information_schema.columns WHERE table_schema = 'notification' AND table_name = 'sent_notification';
```
```text
   column_name   |        data_type         
-----------------+--------------------------
 id              | uuid                     
 recipient_id    | uuid                     
 recipient_email | character varying        
 channel         | character varying        
 subject         | character varying        
 body_summary    | text                     
 send_status     | character varying        
 failure_reason  | text                     
 template_id     | uuid                     
 source_service  | character varying        
 source_event_id | uuid                     
 sent_at         | timestamp with time zone 
 created_at      | timestamp with time zone 
```

---

## 5. Capturas Gráficas del Sistema en Funcionamiento

| Componente | URL de Acceso | Credenciales | Descripción de la Evidencia |
|---|---|---|---|
| **Consola RabbitMQ** | `http://localhost:15672` | User: `app` / Pass: `app` | Interfaz de gestión del broker AMQP para recepción de eventos de notificación. |
| **MailHog Web UI** | `http://localhost:18025` | Sin autenticación | Servidor SMTP simulado en entorno local para inspección visual de correos. |
| **Grafana Dashboard** | `http://localhost:3000` | Acceso directo Admin (`admin` / `admin`) | Orígenes de datos `Prometheus`, `Loki` y `Tempo` conectados para observabilidad LGTM. |

---

## 6. 💡 Mejoras Propuestas a las Historias de Usuario (Sustentación Técnica)

Para enriquecer la arquitectura sin modificar el código fuente actual del proyecto, se sustentan las siguientes **3 propuestas de mejora**:

### 🛠️ Mejora 1: Script de Healthcheck y Validación Automatizada (`HU-008` & `HU-007`)
- **Problema identificado:** Al ejecutar `docker compose up`, algunos servicios (como PostgreSQL o RabbitMQ) tardan unos segundos en estar en estado `healthy`, lo que puede provocar que ejecuciones tempranas de Liquibase fallen.
- **Propuesta:** Incorporar un script ejecutable en PowerShell (`check-health.ps1`) en la raíz del repositorio de infraestructura que consulte el estado de salud de todos los contenedores antes de iniciar migraciones o pruebas.
- **Beneficio:** Garantiza un flujo de despliegue local 100% confiable e idéntico para todos los desarrolladores.

### 🗄️ Mejora 2: Índice Compuesto de Rendimiento en Consultas de Historial (`HU-006` & `HU-005`)
- **Problema identificado:** La tabla `sent_notification` crecerá aceleradamente con cada evento procesado. Consultar el historial de notificaciones recibidas por un usuario (`recipient_id`) ordenadas por fecha requerirá escaneos secuenciales costosos.
- **Propuesta:** Agregar un nuevo changeset en Liquibase que añada un índice compuesto:
  ```sql
  CREATE INDEX idx_sent_notification_recipient_created 
  ON notification.sent_notification (recipient_id, created_at DESC);
  ```
- **Beneficio:** Reduce el tiempo de respuesta de la consulta `GET /notifications/user/{recipientId}` de $O(N)$ a $O(\log N)$, optimizando la escalabilidad.

### ✉️ Mejora 3: Configuración de Dead Letter Queue (DLQ) en RabbitMQ (`HU-004`)
- **Problema identificado:** Si un mensaje consumido de RabbitMQ falla consecutivamente (ej: por correo inválido), se corre el riesgo de bloquear la cola principal o generar bucles infinitos de reintentos.
- **Propuesta:** Definir en la configuración del broker un intercambio y cola de letras muertas (`notification.dlq.exchange` y `notification.dlq.queue`) con un atributo `x-dead-letter-exchange`.
- **Beneficio:** Aísla automáticamente los mensajes fallidos tras un número máximo de reintentos (ej: 3 reintentos), permitiendo auditoría y reprocesamiento manual.

---

## 7. Conclusión

La actividad de reconocimiento demostró la solidez de la infraestructura en Docker y el diseño del modelo relacional en PostgreSQL gestionado por Liquibase. A través de este informe y la grabación en video, se evidencia el cumplimiento riguroso y transparente de los requerimientos para `notification-service`.




<img width="1056" height="923" alt="Captura de pantalla 2026-08-16 202404" src="https://github.com/user-attachments/assets/2bf785dc-e20d-4fc0-a9a2-05b5cfe54c88" />


<img width="1132" height="897" alt="Captura de pantalla 2026-08-16 202419" src="https://github.com/user-attachments/assets/dc859d7b-a72c-44b4-a9be-6b4e99157398" />


<img width="1126" height="745" alt="Captura de pantalla 2026-08-16 202435" src="https://github.com/user-attachments/assets/60c7fb6e-366b-437b-ba84-ec489e70ce32" />




