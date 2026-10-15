> **Estudiante:** David Mauricio Pérez Cordero | **Documento:** xxxxxxxx2

# ADR-003: Estrategia de Integración y Tolerancia a Fallas del Sistema Legado

* **Identificador:** ADR-003
* **Título:** Estrategia de Integración Asíncrona Resiliente: Pre-carga de Kárdex, Patrón Transactional Outbox y Circuit Breaker
* **Fecha:** 14 de Octubre de 2026
* **Estado:** Aceptado

---

## 1. Contexto y Problema

El sistema académico legado (desarrollado hace más de 12 años) presenta serias deficiencias estructurales pero permanece como la única fuente oficial autorizada de antecedentes académicos y prerrequisitos. La condición **C05** estipula que este sistema puede sufrir caídas e interrupciones de hasta 40 minutos ininterrumpidos. 

Bajo el **Escenario A (Dígito PAR)**, el sistema legado colapsa exactamente al comenzar el período de matrícula durante 40 minutos. Si la PIA depende síncronamente del sistema legado para validar prerrequisitos o para confirmar inscripciones, la PIA quedaría totalmente paralizada, violando la disponibilidad requerida del 99,9% (`C03`), que tolera un máximo acumulado de 43,8 minutos de indisponibilidad en todo el mes.

El problema a resolver es: **¿Cómo desacoplar la operación de la PIA del sistema legado para garantizar la continuidad ininterrumpida de las matrículas durante sus caídas sin permitir inscripciones académicamente inválidas ni registros huérfanos?**

---

## 2. Drivers Arquitectónicos Relacionados

* **DRV-03 / C05 / Escenario A:** Resiliencia y continuidad operativa ante caídas de 40 minutos del sistema legado.
* **DRV-09 / C03:** Disponibilidad mensual del sistema $\ge$ 99,9%.
* **DRV-05 / C08:** Solución implementable en Linux y PostgreSQL sin herramientas comerciales de pago.
* **RES-03:** El sistema legado es la fuente de verdad del historial y prerrequisitos.

---

## 3. Alternativas Evaluadas

### Alternativa A: Integración Síncrona Directa (RPC / REST síncrono en cada inscripción)
Cada vez que un estudiante intenta inscribir una materia, la PIA realiza una llamada HTTP síncrona al legado para verificar prerrequisitos y enviar el registro de inscripción.

### Alternativa B: Replicación Bidireccional Asíncrona Continua por Disparadores (Triggers de BD)
Conectar ambas bases de datos directamente mediante enlaces de base de datos (`dblink`/`fdw`) y sincronizar mediante disparadores a nivel de motor.

### Alternativa C: Pre-Carga de Lectura + Patrón Transactional Outbox + Circuit Breaker + ACL (Seleccionada)
1. **Pre-carga de Lectura (T-24h):** 24 horas antes de la matrícula, se extrae un volcado consolidado de los prerrequisitos, asignaturas aprobadas y mallas de los 24.000 estudiantes hacia el esquema `replica_legado` en PostgreSQL.
2. **Validación Autónoma:** La validación de prerrequisitos se ejecuta localmente contra esta réplica en PostgreSQL, sin invocar al legado durante la matrícula.
3. **Transactional Outbox:** Cada inscripción confirmada en la PIA inserta un evento de dominio en la tabla `outbox.eventos_sincronizacion` dentro de la misma transacción ACID del cupo.
4. **Worker Desacoplado con Circuit Breaker:** Un proceso demonio en segundo plano lee la tabla Outbox y despacha los registros al legado. Si el legado cae, el *Circuit Breaker* se abre evitando llamadas inútiles; las inscripciones se marcan como `INSCRITO_PENDIENTE_SINCRONIZACION_OFICIAL` y se sincronizan automáticamente en cuanto el legado revive.

---

## 4. Criterios de Comparación y Evaluación

| Criterio de Evaluación | Alternativa A: Síncrona Directa | Alternativa B: Triggers Directos | Alternativa C: Pre-Carga + Outbox + Circuit Breaker |
| :--- | :--- | :--- | :--- |
| **Resiliencia ante Caída de 40 min (`C05`, Esc. A)** | **Nula (Catastrófica):** La PIA colapsa por completo al agotarse los hilos esperando respuesta del legado. | **Pobre:** Los triggers fallan o bloquean las transacciones en la PIA si el nodo remoto está caído. | **Total (Excelente):** La PIA continúa operando al 100% sin degradar la experiencia del estudiante. |
| **Aislamiento de Modelos (ACL)** | **Pobre:** El modelo legado contamina la API moderna. | **Nulo:** Acoplamiento directo a nivel de esquemas de tablas físicas de hace 12 años. | **Excelente:** La capa anticorrupción (ACL) traduce los modelos y protege el dominio de la PIA. |
| **Impacto en Rendimiento (`C02`)** | **Crítico:** Añade 800-2500 ms de latencia por cada validación de red externa. | **Medio:** Latencia adicional de red en cada commit de base de datos. | **Mínimo:** Lectura y escritura en PostgreSQL local en $\le$ 5 ms. |
| **Mantenibilidad por el Equipo (`C10`)** | Media. | Pésima (mantenimiento de disparadores y permisos de red complejos). | Alta (código en la aplicación estándar y tabla relacional clara). |

---

## 5. Decisión Seleccionada y Justificación

Se selecciona la **Alternativa C: Pre-Carga de Lectura + Patrón Transactional Outbox + Circuit Breaker + Anti-Corruption Layer (ACL)**.

### Justificación Técnica:
1. **Solución definitiva al Escenario A:** Durante los 40 minutos de apagón del legado, los estudiantes pueden inscribir sus asignaturas sin percatarse de la contingencia externa. El cupo queda reservado matemáticamente en la PIA, la validación académica se realiza contra los datos del kárdex consolidados previamente y el evento queda en cola Outbox.
2. **Cumplimiento de la Disponibilidad del 99,9% (`C03`):** Al remover el sistema legado del camino crítico (*critical path*) de la inscripción, una falla externa de 40 minutos no consume ni un solo segundo de la cuota de error mensual de la PIA.
3. **Cero pérdida de datos (Garantía At-Least-Once con Idempotencia):** La inserción del evento en la tabla Outbox ocurre dentro de la misma transacción ACID del cupo. No existe la posibilidad de que un estudiante tenga un cupo asignado sin que exista su correspondiente evento de sincronización pendiente.
4. **Protección mediante Capa Anticorrupción (ACL):** El adaptador ACL encapsula los formatos anticuados y peculiaridades del sistema legado, evitando que afecten el diseño limpio de la PIA.

---

## 6. Consecuencias y Gestión de Riesgos

### Consecuencias Positivas:
* Tolerancia absoluta a caídas del legado de 40 minutos (o superiores).
* Sincronización ordenada y controlada hacia el legado mediante lotes (*batching*) que no saturan su infraestructura envejecida.
* Transparencia para el estudiante, informando claramente el estado transaccional de su solicitud.

### Consecuencias Negativas y Riesgos:
* **Riesgo de inconsistencia temporal si el kárdex cambia en las 24h previas:** Si un docente altera una nota en el legado horas antes de la matrícula, la réplica local podría estar desactualizada.
  * *Mitigación:* Se establece como política institucional que el kárdex del semestre anterior queda congelado administrativamente 48 horas antes de la jornada de matrícula.
* **Crecimiento de la tabla Outbox durante caídas prolongadas:**
  * *Mitigación:* El Worker procesa los eventos mediante `SELECT ... FOR UPDATE SKIP LOCKED` en bloques de 100 registros con control de flujo al reactivarse el servicio.

---

## 7. Condiciones para Revisar esta Decisión

Esta decisión se revisará si:
1. La universidad sustituye completamente el sistema legado por un ERP académico moderno con soporte de mensajería asíncrona de alto rendimiento.
2. El sistema legado implementa APIs con SLA garantizado de 99,99% y tiempos de respuesta inferiores a 100 ms.
