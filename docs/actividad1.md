> **Estudiante:** David Mauricio Pérez Cordero | **Documento:** xxxxxxxx2

# Actividad 1. Identificación y Priorización de Drivers Arquitectónicos
## Plataforma de Inscripción Académica (PIA) - Universidad de la Salle

---

## 1. Objetivos de Negocio

A partir del contexto institucional de la Universidad de la Salle (24.000 estudiantes, 1.100 docentes, 5 facultades y un sistema académico legado de más de 12 años), se identifican los siguientes objetivos estratégicos de negocio:

* **OB-01: Garantizar la continuidad operativa y la satisfacción estudiantil durante los períodos de matrícula.**
  Asegurar que la totalidad del cuerpo estudiantil (hasta 1.200 concurrentes nominales y hasta 1.800 en picos extraordinarios) realice su matrícula de asignaturas sin experimentar caídas de servicio, bloqueos prolongados o tiempos de espera frustrantes, protegiendo la reputación institucional y el cumplimiento del calendario académico.
* **OB-02: Eliminar al 100% las inconsistencias de cupos e inscripciones duplicadas.**
  Garantizar la exactitud matemática y regulatoria en la asignación de plazas académicas, erradicando los sobrecupos en grupos y las inscripciones repetidas causadas por reintentos de red, lo cual previene reprocesos administrativos, quejas formales y sobrecarga a los 1.100 docentes.
* **OB-03: Optimizar la mantenibilidad técnica y la capacidad de evolución del sistema.**
  Reducir el tiempo y esfuerzo técnico necesario para implementar y ajustar reglas académicas de matrícula entre períodos semestrales, permitiendo que el equipo de soporte (4 desarrolladores y 1 administrador) opere y evolucione la plataforma de manera autónoma y sostenible sin incurrir en costos de licencias comerciales.

---

## 2. Drivers de Atributos de Calidad

Los atributos de calidad condicionan directamente las decisiones estructurales del sistema bajo las condiciones de operación suministradas:

1. **QA-01: Rendimiento y Latencia (Performance):**
   * *Descripción:* Capacidad del sistema para responder a las consultas de asignaturas y grupos con tiempos de respuesta predecibles y mínimos bajo cargas extremas.
   * *Métrica:* El 95 % de las consultas de catálogo debe responder en un tiempo $\le$ 2,0 segundos (`C02`), soportando ráfagas iniciales de 1.200 estudiantes concurrentes (`C01`) y picos de hasta 1.800 usuarios concurrentes (+50% en Escenario B).
2. **QA-02: Disponibilidad y Continuidad Operativa (Availability & Fault Tolerance):**
   * *Descripción:* Resiliencia del sistema para mantenerse operativo frente a fallas de infraestructura o dependencias externas.
   * *Métrica:* Disponibilidad mensual $\ge$ 99,9 % durante los períodos de matrícula (`C03`), tolerando interrupciones totales del sistema legado de hasta 40 minutos (`C05` y Escenario A) sin suspender las operaciones esenciales. En caso de siniestro general, RTO $\le$ 30 minutos (`C06`) y RPO $\le$ 5 minutos (`C07`).
3. **QA-03: Integridad de Datos, Consistencia e Idempotencia (Data Integrity & Consistency):**
   * *Descripción:* Garantía de corrección transaccional en la mutación de plazas y registros de inscripción.
   * *Métrica:* Tolerancia cero a sobrecupos (`C04`: ningún grupo excederá su capacidad máxima autorizada) y tolerancia cero a inscripciones duplicadas, incluso ante ráfagas de hasta 5 reintentos automáticos idénticos por estudiante (Escenario B).
4. **QA-04: Seguridad, Privacidad y Trazabilidad (Security & Auditability):**
   * *Descripción:* Mecanismos de autenticación, control de acceso basado en roles (RBAC) y registro inalterable de auditoría.
   * *Métrica:* Cumplimiento total de `C09`, asegurando que el 100 % de los accesos a datos personales y académicos estén autenticados, y que cada intento de inscripción, modificación o cancelación quede registrado en una bitácora inmutable con marca de tiempo, usuario y estado resultante.
5. **QA-05: Modificabilidad y Mantenibilidad (Modifiability):**
   * *Descripción:* Facilidad para incorporar nuevas validaciones académicas y esquemas de matrícula sin afectar los componentes de integración ni la infraestructura base.
   * *Métrica:* El acoplamiento entre el motor de reglas académicas y los adaptadores de integración debe permitir incorporar una nueva regla de prerrequisitos o cupos en menos de 16 horas/hombre de desarrollo, verificable mediante pruebas automatizadas sin tocar el código del sistema legado.

---

## 3. Restricciones Arquitectónicas

1. **RES-01: Infraestructura y Presupuesto Tecnológico (`C08`):**
   La solución debe ejecutarse estrictamente sobre servidores Linux y motor de base de datos relacional PostgreSQL. No se autoriza la adquisición ni contratación de servicios de nube comerciales adicionales (como AWS Aurora, DynamoDB, Redis Enterprise administrado o brokers comerciales).
2. **RES-02: Capacidad y Tamaño del Equipo Institucional (`C10`):**
   El soporte y mantenimiento a largo plazo será ejecutado exclusivamente por un equipo conformado por cuatro (4) desarrolladores y un (1) administrador de infraestructura. Las decisiones arquitectónicas no deben requerir especialistas en tecnologías exóticas ni una carga operativa inmanejable.
3. **RES-03: Dependencia del Sistema Académico Legado (`C05` y Condición Adicional):**
   El sistema legado (con más de 12 años en operación) continuará siendo la única fuente oficial del historial académico consolidado y de las mallas curriculares. La PIA es la autoridad temporal de las inscripciones del período en curso y debe sincronizarlas con el legado de forma no intrusiva y confiable.

---

## 4. Matriz de Priorización de Drivers

| ID | Tipo de Driver | Descripción Corta | Origen | Prioridad (Negocio / Técnica) | Medida de Aceptación Cuantificable | Justificación Arquitectónica |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **DRV-01** | Atributo de Calidad | Consistencia absoluta de cupos y cero sobrecupos | `C04` | **Alta / Alta** | Sobrecupos = 0 en el 100% de los grupos asignados | Violación de cupo acarrea problemas legales y de acreditación académica. |
| **DRV-02** | Atributo de Calidad | Idempotencia ante reintentos automáticos | Escenario B, `C04` | **Alta / Alta** | 0 inscripciones duplicadas ante 5 reintentos repetidos | Evita la saturación del sistema y bloqueos de cupos fantasmas. |
| **DRV-03** | Atributo de Calidad | Aislamiento y resiliencia ante caídas del legado | `C05`, Escenario A, `C03` | **Alta / Alta** | Continuidad del 100% de consultas e inscripciones durante 40 min de caída del legado | El legado cae con frecuencia; la PIA no puede propagar el fallo a los alumnos. |
| **DRV-04** | Atributo de Calidad | Rendimiento en consulta masiva de materias | `C01`, `C02`, Escenario B | **Alta / Media** | p95 $\le$ 2,0s con 1.200 a 1.800 estudiantes concurrentes | La consulta representa más del 80% del tráfico durante el proceso. |
| **DRV-05** | Restricción | Stack limitado a Linux y PostgreSQL | `C08` | **Alta / Alta** | 100% de la arquitectura funcional en Linux + PostgreSQL estándar | Mandato financiero e institucional ineludible. |
| **DRV-06** | Restricción | Equipo limitado (4 devs + 1 admin) | `C10` | **Alta / Media** | Complejidad ciclomática y operativa dominable por el equipo local | Descarta arquitecturas de microservicios hiperfragmentadas. |
| **DRV-07** | Atributo de Calidad | Continuidad RTO $\le$ 30m y RPO $\le$ 5m | `C06`, `C07` | **Media / Alta** | Recuperación ante desastre en $\le$ 30 min con pérdida de datos $\le$ 5 min de WAL | Mitiga incidentes graves en los servidores del campus. |
| **DRV-08** | Atributo de Calidad | Trazabilidad inmutable de operaciones | `C09` | **Media / Media** | 100% de transacciones y fallos con bitácora de auditoría no repudiable | Exigido para responder a reclamaciones estudiantiles y auditorías. |
| **DRV-09** | Atributo de Calidad | Disponibilidad mensual global $\ge$ 99,9% | `C03` | **Media / Media** | Tiempo fuera de servicio mensual $\le$ 43,8 minutos | Requiere redundancia activa y graceful degradation. |
| **DRV-10** | Atributo de Calidad | Modificabilidad de reglas de negocio | Contexto | **Media / Baja** | Adición de regla en $\le$ 16 horas/hombre sin tocar integración | Facilita la adaptación a nuevas resoluciones académicas. |

---

## 5. Escenarios de Calidad Verificables

Se formulan 5 escenarios formales bajo la estructura estándar de SEI (Fuente, Estímulo, Artefacto, Entorno, Respuesta y Medida Cuantificable):

### Escenario 1: Consulta masiva de catálogo bajo pico de concurrencia (QA-01 / Rendimiento)
* **Fuente del Estímulo:** 1.800 estudiantes concurrentes al inicio del período de matrícula.
* **Estímulo:** Envío masivo y concurrente de peticiones de consulta de catálogo de asignaturas, grupos y horarios.
* **Artefacto:** Módulo de Catálogo y Consulta de Asignaturas de la PIA.
* **Entorno / Contexto:** Horas pico de apertura de matrícula (concurrencia base de 1.200 alumnos incrementada en un 50% según Escenario B).
* **Respuesta Esperada:** El sistema sirve la información desde una capa de caché en memoria de solo lectura sin bloquear la base de datos transaccional.
* **Medida Cuantificable:** **Percentil 95 (p95) $\le$ 1,8 segundos**, con una tasa de éxito HTTP 200 $\ge$ 99,95 % y consumo de CPU en la BD inferior al 60 %.

### Escenario 2: Caída total del sistema legado durante la matrícula (QA-02 / Resiliencia - Escenario Extraordinario A)
* **Fuente del Estímulo:** Sistema Académico Legado (>12 años).
* **Estímulo:** Pérdida total de conectividad y cese de respuestas (HTTP 500 / Timeout) durante 40 minutos ininterrumpidos.
* **Artefacto:** Adaptador de Integración (Anti-Corruption Layer) y Módulo de Inscripción de la PIA.
* **Entorno / Contexto:** Apertura de ventana de matrícula en plena concurrencia estudiantil.
* **Respuesta Esperada:** Se activa el *Circuit Breaker*. La PIA atiende las solicitudes validando prerrequisitos contra la réplica local pre-sincronizada, emite la reserva atómica de cupo, registra la inscripción con estado `INSCRITO_PENDIENTE_SINCRONIZACION_OFICIAL` y encola la sincronización en una tabla Outbox transaccional para posterior despacho.
* **Medida Cuantificable:** **0 interrupciones percibidas por el usuario**, disponibilidad local mantenida al 100% durante los 40 minutos de corte, y **0 transacciones perdidas**. Al restaurarse el legado, el 100% de los eventos encolados se despachan sin duplicidad.

### Escenario 3: Ráfagas de reintentos concurrentes por cupos limitados (QA-03 / Consistencia - Escenario Extraordinario B)
* **Fuente del Estímulo:** Múltiples estudiantes intentando inscribir los últimos 2 cupos de un grupo popular.
* **Estímulo:** Un estudiante genera hasta 5 peticiones idénticas automáticas/manuales en un lapso de 1 segundo compitiendo con otros 50 estudiantes.
* **Artefacto:** Módulo de Control de Cupos y Gestor Transaccional de Inscripciones.
* **Entorno / Contexto:** Alta saturación y contención sobre el mismo registro de grupo.
* **Respuesta Esperada:** El filtro de idempotencia detecta la llave (`Idempotency-Key`); la primera petición ejecuta un `UPDATE` condicional atómico; las 4 peticiones redundantes reciben el resultado de la primera sin descontar cupos adicionales; y a partir del agotamiento de la plaza, los demás estudiantes reciben respuesta inmediata de cupo agotado.
* **Medida Cuantificable:** **Sobrecupos = 0**, inscripciones duplicadas en base de datos = 0, y tiempo de respuesta para peticiones duplicadas $\le$ 300 ms.

### Escenario 4: Trazabilidad y auditoría ante reclamación académica (QA-04 / Seguridad)
* **Fuente del Estímulo:** Auditor o Director de Programa Académico.
* **Estímulo:** Solicitud de verificación forense sobre la cancelación o asignación del cupo de un estudiante específico.
* **Artefacto:** Módulo de Auditoría y Bitácora Transaccional Inmutable.
* **Entorno / Contexto:** Operación ordinaria o post-matrícula.
* **Respuesta Esperada:** El sistema extrae el historial completo de eventos asociados al estudiante (IP de origen, ID de usuario autenticado, timestamp en milisegundos, grupo solicitado, estado anterior y estado posterior).
* **Medida Cuantificable:** Registro inmutable del 100 % de las operaciones con una latencia de consulta de auditoría $\le$ 3 segundos y garantía criptográfica de no repudio.

### Escenario 5: Falla catastrófica de nodo de base de datos (QA-02 / Continuidad RTO y RPO)
* **Fuente del Estímulo:** Falla de hardware / corrupción de sistema operativo en el servidor principal de base de datos.
* **Estímulo:** Caída imprevista del proceso PostgreSQL durante la jornada de matrícula.
* **Artefacto:** Arquitectura de Datos PostgreSQL (Cluster Primary-Standby en Linux).
* **Entorno / Contexto:** Horario de tráfico regular de inscripciones.
* **Respuesta Esperada:** El administrador o el mecanismo de failover promueve el nodo Standby que recibe réplica continua de WAL (Write-Ahead Logging).
* **Medida Cuantificable:** **RTO real $\le$ 15 minutos** (cumpliendo sobradamente el límite de 30 min de `C06`) y **RPO $\le$ 1 minuto** (cumpliendo sobradamente el límite de 5 min de `C07`).

---

## 6. Identificación de Conflictos entre Drivers y Propuesta de Resolución

### Conflicto 1: Dependencia de Prerrequisitos en el Legado vs. Disponibilidad $\ge$ 99,9% (`C03`) y Caída de 40 min (`C05` / Escenario A)
* **Naturaleza del Conflicto:** El sistema legado es la fuente oficial de antecedentes académicos y prerrequisitos. Si la PIA consulta el sistema legado sincrónicamente en cada intento de inscripción, cualquier indisponibilidad de hasta 40 minutos en el legado paraliza completamente la PIA, reduciendo la disponibilidad mensual a menos del 90% (violando `C03` que sólo permite 43,8 minutos de caída en 30 días).
* **Propuesta de Resolución:**
  1. *Desacoplamiento temporal mediante Pre-Carga de Lectura:* 24 horas antes del inicio de la matrícula, la PIA ejecuta un proceso de extracción y sincronización masiva de las mallas, prerrequisitos e historial académico consolidado de los 24.000 estudiantes, almacenándolo en un esquema local de solo lectura en PostgreSQL (`replica_legado`).
  2. *Validación Local Inmediata:* El motor de reglas valida prerrequisitos exclusivamente contra este esquema local en memoria/disco rápido, permitiendo responder en milisegundos sin tocar el legado.
  3. *Patrón Transactional Outbox:* La confirmación oficial ante el legado se desacopla mediante eventos encolados en la misma transacción de la inscripción. Si el legado cae 40 minutos, las inscripciones continúan de manera ininterrumpida; una vez restaurado, el Worker drena los eventos pendientes respetando el orden cronológico.

### Conflicto 2: Consistencia Estricta de Cupos (`C04`: Cero Sobrecupos) vs. Rendimiento p95 $\le$ 2s (`C02`) bajo 1.800 Usuarios Concurrentes (Escenario B)
* **Naturaleza del Conflicto:** Para asegurar 0 sobrecupos, la intuición tradicional sugiere bloquear la fila o la tabla del grupo (`SELECT FOR UPDATE`), obligando a que todas las solicitudes concurrentes se formen en una fila secuencial. Si 200 estudiantes compiten por el mismo grupo popular, los tiempos de espera acumulados en el pool de conexiones de la base de datos degradan la latencia a más de 10 segundos, saturando los recursos de CPU y memoria.
* **Propuesta de Resolución:**
  1. *Actualización Atómica Condicional sin Bloqueo Prolongado:* La deducción del cupo se ejecuta mediante una única sentencia SQL atómica:
     ```sql
     UPDATE grupos
     SET cupos_ocupados = cupos_ocupados + 1
     WHERE id = :grupoId AND cupos_ocupados < capacidad_maxima;
     ```
     PostgreSQL maneja esta instrucción con un bloqueo exclusivo de fila que dura menos de 1 milisegundo. Si las filas afectadas son 0, se rechaza la solicitud instantáneamente por falta de cupo sin generar colas.
  2. *Caché Local para Consultas de Disponibilidad:* Las pantallas de visualización leen la disponibilidad desde una réplica o caché en memoria con invalidación dirigida, absorbiendo el 85% de la carga de lectura y reservando la base de datos principal exclusivamente para las mutaciones atómicas.
  3. *Idempotencia de Entrada:* Mediante un encabezado `Idempotency-Key` único por intento, las peticiones duplicadas de un mismo estudiante no ingresan a la contención de base de datos; reciben directamente la respuesta en caché de la primera transacción.

### Conflicto 3: Alta Resiliencia y Modularidad vs. Restricción de Stack (`C08`) y Equipo Reducido (`C10`)
* **Naturaleza del Conflicto:** Implementar alta disponibilidad, tolerancia a fallos y desacoplamiento de componentes tradicionalmente se aborda mediante microservicios, clústeres de mensajería (Kafka/RabbitMQ) y múltiples bases de datos no relacionales. Sin embargo, esto requiere licencias o infraestructura compleja (`C08`) y desborda la capacidad operativa de 4 desarrolladores y 1 administrador (`C10`).
* **Propuesta de Resolución:**
  1. *Estilo de Monolito Modular con Arquitectura Hexagonal:* Todo el sistema se construye en un único repositorio y se despliega como un proceso unificado en Linux, pero organizado internamente en módulos estrictamente desacoplados por paquetes e interfaces Java/Go/Python.
  2. *Base de Datos PostgreSQL como Plataforma Multifunción:* PostgreSQL no sólo almacena los datos relacionales de la matrícula, sino que su soporte nativo para JSONB, tablas no registradas (*UNLOGGED tables*) y mecanismos como `LISTEN/NOTIFY` o colas transaccionales con `SELECT ... FOR UPDATE SKIP LOCKED` sustituyen la necesidad de un broker de mensajería externo (como RabbitMQ o Kafka), reduciendo la superficie operativa a un único motor de alta confiabilidad que el administrador domina al 100%.
