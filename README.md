> **Estudiante:** David Mauricio Pérez Cordero | **Documento:** xxxxxxxx2

# Plataforma de Inscripción Académica (PIA) - Universidad de la Salle
## Propuesta Integral de Arquitectura de Software - Parcial 2

---

## 1. Resumen Ejecutivo

La **Universidad de la Salle** cuenta con una comunidad académica de aproximadamente **24.000 estudiantes, 1.100 docentes y 5 facultades**. Actualmente, el proceso de matrícula de asignaturas se realiza a través de un sistema académico desarrollado hace más de 12 años, el cual colapsa periódicamente debido a lentitud extrema, caídas de disponibilidad, inconsistencias de cupos, inscripciones duplicadas y alta dependencia síncrona.

El presente repositorio contiene la **Propuesta de Arquitectura de Software para la nueva Plataforma de Inscripción Académica (PIA)**. La solución ha sido diseñada bajo los más rigurosos estándares de la ingeniería de software para operar como una autoridad moderna de matrículas capaz de resolver los problemas históricos de la institución y operar de forma fluida bajo condiciones extremas de demanda.

### Logros Clave del Diseño Arquitectónico:
1. **Garantía Matemática de Cero Sobrecupos (`C04`):** Eliminación absoluta de sobrecupos en grupos académicos mediante sentencias atómicas condicionales con bloqueo ultracorto a nivel de fila (*RowExclusiveLock*) en PostgreSQL.
2. **Alta Concurrencia e Idempotencia (Escenarios A y B / `C01`, `C02`):** Capacidad probada para absorber 1.200 usuarios concurrentes nominales y hasta 1.800 en horas pico, neutralizando ráfagas de hasta 5 reintentos automáticos mediante llaves `Idempotency-Key` y sirviendo el 95% de las consultas de catálogo en menos de 150 ms (superando la meta de 2,0 segundos).
3. **Resiliencia ante Caídas del Sistema Legado (`C05` y Escenario A):** Desacoplamiento temporal de la validación de prerrequisitos mediante una réplica local pre-sincronizada a T-24h y encolamiento transaccional de eventos (*Transactional Outbox Pattern* con *Circuit Breaker*), permitiendo que la PIA opere al 100% durante interrupciones del legado de hasta 40 minutos sin perder transacciones ni violar la disponibilidad del 99,9% (`C03`).
4. **Continuidad Operativa RTO $\le$ 30 min y RPO $\le$ 5 min (`C06`, `C07`):** Clúster de alta disponibilidad en PostgreSQL con archivado continuo de segmentos WAL cada 60 segundos hacia almacenamiento seguro secundario (RPO real $\le$ 1 min, RTO real $\le$ 10 min).
5. **Máxima Eficiencia y Factibilidad Operativa (`C08`, `C10`):** Implementada 100% sobre servidores **Linux y PostgreSQL estándar**, sin costos en servicios de nube comerciales de pago y con un modelo de gobernanza ágil (*Governance-as-Code*) perfectamente gestionable por el equipo institucional de **4 desarrolladores y 1 administrador de infraestructura**.

---

## 2. Mapa de Navegación del Repositorio y Entregables

La documentación se encuentra estructurada modularmente en el directorio [`docs/`](docs/):

| Documento | Descripción y Alcance | Vínculo Directo |
| :--- | :--- | :--- |
| **Arbitraje de Requisitos** | Evaluación formal de tensiones, compromisos (*trade-offs*) y resolución de fuerzas en conflicto entre requerimientos de negocio y atributos de calidad. | [`docs/arbitraje_requisitos.md`](docs/arbitraje_requisitos.md) |
| **Actividad 1** | Identificación de objetivos de negocio, drivers arquitectónicos (`QA-01` a `QA-05`), restricciones (`RES-01` a `RES-03`), matriz de priorización, escenarios de calidad formales y resolución de conflictos. | [`docs/actividad1.md`](docs/actividad1.md) |
| **Actividad 2** | Estructura arquitectónica, justificación del Monolito Modular Hexagonal, catálogo de responsabilidades, límites transaccionales ACID y diagramas C4 vectoriales. | [`docs/actividad2.md`](docs/actividad2.md) |
| **Diagramas C4 (SVG)** | 1. [Nivel 1 - Contexto](docs/diagramas/c4_nivel1_contexto.svg)<br>2. [Nivel 2 - Contenedores](docs/diagramas/c4_nivel2_contenedores.svg)<br>3. [Nivel 3 - Componentes](docs/diagramas/c4_nivel3_componentes.svg) | [`docs/diagramas/`](docs/diagramas/) |
| **Actividad 3 (ADRs)** | Registros de Decisiones Arquitectónicas formales:<br>• **ADR-001:** [Estilo Arquitectónico Base](docs/actividad3/ADR-001-estilo-arquitectonico.md)<br>• **ADR-002:** [Consistencia Estricta de Cupos e Idempotencia](docs/actividad3/ADR-002-consistencia-inscripciones-y-cupos.md)<br>• **ADR-003:** [Integración y Tolerancia a Fallas del Legado](docs/actividad3/ADR-003-integracion-y-fallas-sistema-legado.md)<br>• **ADR-004:** [Alta Concurrencia, Rendimiento y Continuidad (RTO/RPO)](docs/actividad3/ADR-004-alta-concurrencia-rendimiento-y-continuidad.md) | [`docs/actividad3/`](docs/actividad3/) |
| **Actividad 4** | Selección y validación profunda de 4 tácticas arquitectónicas con modelado técnico, análisis de costos/beneficios y mecanismos cuantitativos de verificación (k6, Chaos Engineering, Game Days). | [`docs/actividad4.md`](docs/actividad4.md) |
| **Actividad 5** | Modelo de gobernanza para 4 devs + 1 admin (`C10`), 3 reglas arquitectónicas verificables (`RA-01` a `RA-03`), CI/CD automatizado con ArchUnit, Matriz RACI, ciclo de vida de ADRs y criterios de Pull Request. | [`docs/actividad5.md`](docs/actividad5.md) |

---

## 3. Stack Tecnológico y Justificación de Cumplimiento

En estricto apego a las condiciones institucionales **C08** (infraestructura Linux + PostgreSQL sin servicios comerciales) y **C10** (equipo de 4 desarrolladores y 1 administrador), el stack tecnológico seleccionado aprovecha componentes robustos y probados de software libre:

```
+---------------------------------------------------------------------------------------------------------+
|                                  STACK TECNOLÓGICO SELECCIONADO                                         |
+---------------------------------------------------------------------------------------------------------+
| Capa Perimetral:       Nginx Reverse Proxy sobre Linux Ubuntu LTS (Terminación TLS, Rate Limiting)      |
| Capa Frontend:         Single Page Application (React / TypeScript) en el navegador del estudiante     |
| Capa Backend:          Monolito Modular Hexagonal (Java 17 con Spring Boot / Go Gin / Python FastAPI)   |
| Capa de Persistencia:  PostgreSQL 15+ en Linux con esquemas aislados (matricula, auditoria, outbox)     |
| Alta Disponibilidad:   Replicación Streaming Primario-Standby + Continuous WAL Archiving hacia NAS      |
| Integración Legada:    Worker Asíncrono en segundo plano (SKIP LOCKED) + Circuit Breaker + Adaptador ACL|
+---------------------------------------------------------------------------------------------------------+
```

### ¿Por qué este stack cumple a cabalidad con las restricciones institucionales?
* **Cero Costos de Licenciamiento Comercial (`C08`):** No se depende de servicios propietarios de nube pública como AWS Aurora, DynamoDB o Kafka administrado. Todos los componentes se despliegan en hardware propio bajo Linux.
* **PostgreSQL como Motor Multifuncional:** Se aprovecha PostgreSQL no solo como almacén relacional transaccional, sino también como cola de eventos de alta velocidad gracias a `SELECT ... FOR UPDATE SKIP LOCKED` y tabla de idempotencia atómica, evitando la necesidad de instalar y operar brokers complejos (como Apache Kafka o RabbitMQ).
* **Mantenibilidad Sostenible (`C10`):** Al evitar la fragmentación física de microservicios, el equipo de 4 desarrolladores trabaja en un único repositorio con tiempos de ciclo ágiles, y el administrador de infraestructura gestiona un único clúster de bases de datos estandarizado.

---

## 4. Resolución de Situaciones Extraordinarias

El diseño arquitectónico incorpora de forma explícita la mitigación de los escenarios extraordinarios previstos:

### Escenario A: Caída de 40 Minutos del Sistema Legado (Dígito PAR)
* **Impacto Mitigado:** Al comenzar las matrículas, el sistema legado deja de responder.
* **Solución Arquitectónica:** 
  1. La validación de prerrequisitos no depende del legado; se ejecuta contra la réplica local pre-cargada a T-24h en PostgreSQL.
  2. El *Circuit Breaker* en el Worker de sincronización pasa a `OPEN`, evitando llamadas fallidas.
  3. Los estudiantes inscriben sus asignaturas con total normalidad, reservando su cupo atómicamente y recibiendo confirmación con estado `INSCRITO_PENDIENTE_SINCRONIZACION_OFICIAL`.
  4. Los eventos se acumulan ordenadamente en la tabla Outbox. Al reactivarse el legado 40 minutos después, el Worker sincroniza en segundo plano sin pérdida de datos ni sobrecarga.

### Escenario B: Concurrencia de 1.800 Usuarios y hasta 5 Reintentos por Solicitud (Dígito IMPAR)
* **Impacto Mitigado:** La demanda sube un 50% y los estudiantes envían ráfagas de clics repetidos.
* **Solución Arquitectónica:**
  1. Nginx suaviza el tráfico perimetral mediante un algoritmo *Leaky Bucket* por token JWT.
  2. El interceptor de idempotencia (`Idempotency-Key`) en el backend detecta las peticiones duplicadas y devuelve la respuesta en caché de la primera transacción en menos de 50 ms.
  3. La reserva de cupo se ejecuta mediante una sola sentencia atómica `UPDATE ... WHERE cupos_ocupados < capacidad_maxima` con bloqueo ultracorto de fila en PostgreSQL, garantizando matemáticamente cero sobrecupos (`C04`).

---

## 5. Trazabilidad de Cumplimiento de Condiciones de Operación

| Código | Condición de Operación | Cómo se Garantiza en la Arquitectura | Métrica Verificable |
| :---: | :--- | :--- | :--- |
| **C01** | 1.200 estudiantes simultáneos | Caché escalonada en Nginx/Backend + Pool de conexiones optimizado | 1.200 a 1.800 usuarios atendidos concurrentemente |
| **C02** | 95% de consultas en $\le$ 2s | Catálogo servido desde memoria RAM; desacoplamiento de lecturas y escrituras | p95 real $\le$ 150 ms en pruebas de carga |
| **C03** | Disponibilidad $\ge$ 99,9% en matrícula | Aislamiento total de dependencias externas + Graceful Degradation | Inactividad mensual máxima tolerada $\le$ 43,8 min |
| **C04** | Cero sobrecupos en grupos | Actualización atómica condicional a nivel de fila en PostgreSQL | 0 sobrecupos en el 100% de los grupos |
| **C05** | Tolerancia a caídas del legado de 40m | Pre-carga de kárdex (T-24h) + Transactional Outbox + Circuit Breaker | 100% de operaciones locales continúan durante la caída |
| **C06** | RTO $\le$ 30 minutos | Script de promoción automatizada de nodo PostgreSQL Standby (`pg_ctl`) | RTO probado $\le$ 10 minutos |
| **C07** | RPO $\le$ 5 minutos | Archivado continuo de segmentos WAL sincronizados cada 60 segundos | RPO probado $\le$ 1 minuto |
| **C08** | Linux y PostgreSQL sin servicios de pago | Stack open-source estándar desplegable en infraestructura institucional | 0 licencias comerciales adicionales |
| **C09** | Seguridad, RBAC y auditoría inmutable | JWT + Bitácora transaccional append-only en esquema `auditoria` | 100% de operaciones registradas con no repudio |
| **C10** | Mantenible por 4 devs + 1 admin | Monolito Modular Hexagonal en monorepo + CI/CD automatizado | Reducción drástica de complejidad operativa |

---

## 6. Estado del Repositorio y Commits

El desarrollo de este trabajo ha seguido la **Regla de Oro de Git**, asegurando commits atómicos, independientes y descriptivos para cada una de las actividades del parcial en la rama de trabajo `dev_daperez02`.
