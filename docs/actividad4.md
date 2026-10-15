> **Estudiante:** David Mauricio Pérez Cordero | **Documento:** xxxxxxxx2

# Actividad 4. Selección y Validación de Tácticas Arquitectónicas
## Plataforma de Inscripción Académica (PIA) - Universidad de la Salle

---

## 1. Introducción a las Tácticas Arquitectónicas

Las tácticas arquitectónicas son decisiones de diseño orientadas a controlar directamente las respuestas del sistema ante estímulos específicos, asegurando la satisfacción de los atributos de calidad priorizados en la **Actividad 1**. 

En este documento se detallan cuatro tácticas fundamentales que responden a los desafíos críticos del caso: consistencia absoluta de cupos (`C04`), absorción de ráfagas masivas (Escenario B), aislamiento y resiliencia ante caídas del sistema legado de 40 minutos (`C05`, Escenario A) y continuidad del servicio (`C06`, `C07`).

---

## 2. Táctica 1: Idempotencia Transaccional de Extremo a Extremo y Actualización Atómica Condicional

### 2.1. Atributo de Calidad Asociado
* **QA-03: Integridad de Datos, Consistencia e Idempotencia** (Satisfaciendo `C04` y el Escenario B).

### 2.2. Problema Específico a Resolver
Durante el proceso de matrícula, la impaciencia de los estudiantes ante tiempos de espera produce que envíen hasta 5 veces la misma solicitud de inscripción mediante clics repetidos o reintentos automáticos del navegador/scripts (Escenario B). Si cada petición se procesa independientemente, se generan cupos duplicados para el mismo alumno, condiciones de carrera (*race conditions*) y sobrecupos que violan la condición **C04** ("Ningún grupo académico puede registrar más estudiantes que su capacidad autorizada").

### 2.3. Componentes Involucrados
* **Frontend Web SPA:** Generador de llaves UUIDv4 por intento de acción.
* **IdempotencyMiddleware:** Interceptor de peticiones HTTP en el Backend.
* **GestorCuposAtomico:** Servicio de dominio responsable de la reserva atómica.
* **Base de Datos PostgreSQL:** Tabla de control de idempotencia y tabla `matricula.grupos_cupos`.

### 2.4. Funcionamiento Técnico
1. **Generación de la Llave de Idempotencia:** Cuando el estudiante hace clic en "Inscribir Asignatura", el frontend genera un encabezado HTTP `Idempotency-Key: e8b7a46d-3189-4e09-906a-54cfbc9b1234` único para esa intención específica.
2. **Interceptación y Validación:** El middleware intercepta la solicitud antes de entrar a la lógica de negocio y ejecuta una consulta atómica sobre la tabla `seguridad.peticiones_idempotencia`:
   ```sql
   INSERT INTO seguridad.peticiones_idempotencia (idempotency_key, usuario_id, endpoint, estado, creado_en)
   VALUES (:key, :usuarioId, '/api/v1/inscripciones', 'EN_PROCESO', NOW())
   ON CONFLICT (idempotency_key) DO NOTHING;
   ```
   * Si la fila ya existía y su estado es `COMPLETADO`, el middleware retorna inmediatamente la respuesta guardada en formato JSON (HTTP 201) en menos de 30 ms sin volver a procesar la lógica de negocio.
   * Si el estado es `EN_PROCESO`, se descartan los reintentos concurrentes retornando código HTTP 409 (Conflicto / Petición en curso).
3. **Reserva Condicional Atómica:** La primera petición procesa la reserva del cupo mediante una única sentencia SQL en PostgreSQL:
   ```sql
   UPDATE matricula.grupos_cupos
   SET cupos_ocupados = cupos_ocupados + 1
   WHERE id = :grupoId AND cupos_ocupados < capacidad_maxima;
   ```
   PostgreSQL adquiere un bloqueo exclusivo de fila (*RowExclusiveLock*) únicamente durante los microsegundos que tarda la ejecución del `UPDATE`.
   * Si `ROWCOUNT == 1`: El cupo fue adquirido con éxito.
   * Si `ROWCOUNT == 0`: El grupo se agotó exactamente antes de esta petición; se cancela la transacción y se informa "Cupo agotado".
4. **Cierre de Idempotencia:** Al finalizar la transacción exitosa, se actualiza el estado de la llave a `COMPLETADO` almacenando el cuerpo de la respuesta HTTP.

### 2.5. Beneficios y Costos
* **Beneficios:**
  * Garantía matemática de cero sobrecupos (`C04`), sin importar la cantidad de solicitudes simultáneas.
  * Neutralización del 80% de la carga innecesaria producida por 5 reintentos repetidos.
  * Prevención de inscripciones duplicadas para el mismo estudiante.
* **Costos:**
  * Almacenamiento adicional en la tabla de idempotencia (requiere depuración periódica de registros > 48h).
  * Latencia de 1 a 2 ms adicionales por la consulta de la llave antes de procesar el negocio.

### 2.6. Mecanismo de Validación y Verificación Experimental
* **Herramienta:** Script de prueba de carga con **k6**.
* **Condición de Prueba:**
  * Se configuran 1.800 usuarios virtuales (VUs) concurrentes (Escenario B).
  * Se seleccionan 10 grupos académicos con capacidad limitada a exactamente 30 cupos cada uno (Total de 300 cupos disponibles).
  * Cada usuario virtual envía una ráfaga de 5 peticiones consecutivas con la misma `Idempotency-Key` en menos de 500 ms (total 9.000 solicitudes enviadas).
* **Criterios de Aceptación (Verificación Exitosa):**
  1. La consulta final en PostgreSQL debe arrojar exactamente 30 estudiantes inscritos por grupo (cero sobrecupos verificados: `cupos_ocupados <= 30`).
  2. El total de inscripciones creadas en `matricula.inscripciones` debe ser exactamente 300 registros únicos.
  3. Ningún estudiante debe poseer más de 1 inscripción para el mismo grupo.
  4. El 100% de las peticiones duplicadas (las 4 restantes de cada ráfaga) deben recibir HTTP 201 en un tiempo $\le$ 150 ms.

---

## 3. Táctica 2: Circuit Breaker, Transactional Outbox y Réplica Local de Prerrequisitos (Degradación Controlada)

### 3.1. Atributo de Calidad Asociado
* **QA-02: Disponibilidad y Tolerancia a Fallos** (Satisfaciendo `C03`, `C05` y el Escenario A).

### 3.2. Problema Específico a Resolver
El sistema académico legado de más de 12 años presenta interrupciones intempestivas de hasta 40 minutos (`C05`). En el **Escenario A**, esta caída ocurre exactamente al inicio de la jornada de matrícula. Si la nueva plataforma PIA consulta o notifica al legado síncronamente en cada inscripción, la caída externa agota el pool de conexiones HTTP y de base de datos de la PIA, tumbando la plataforma completa y violando la disponibilidad mensual del 99,9% (`C03`).

### 3.3. Componentes Involucrados
* **MotorReglasAcademicas:** Validador de prerrequisitos sobre base local.
* **Esquema PostgreSQL `replica_legado`:** Datos de mallas y kárdex pre-sincronizados a T-24h.
* **Tabla `outbox.eventos_sincronizacion`:** Cola transaccional persistente.
* **Worker de Sincronización:** Demonio en Linux con patrón Circuit Breaker.
* **Adaptador Anti-Corruption Layer (ACL):** Conector hacia el Sistema Legado.

### 3.4. Funcionamiento Técnico

```
[ Petición Alumno ] 
        |
        v
[ PIA AppService ] ---> Valida Prerrequisitos en [ replica_legado (Local PG) ]
        |
        +-------------> Descuenta Cupo Atómico [ matricula.grupos_cupos ]
        |
        +-------------> Registra [ matricula.inscripciones ] (Estado: INSCRITO_PENDIENTE_SYNC)
        |
        +-------------> Inserta Evento [ outbox.eventos_sincronizacion ]
        |
    (COMMIT ACID)
        |
        v
[ Worker Sincronizador ] --- (Lee Outbox via SKIP LOCKED)
        |
        v
 [ Circuit Breaker ] 
    |          |
 (CLOSED)    (OPEN: Legado caído 40 min)
    |          |
    |          +--> Pausa reintentos síncronos, activa Backoff Exponencial.
    v               La PIA sigue inscribiendo localmente sin fallar.
[ Envia a Legado ]
```

1. **Pre-carga y Aislamiento de Lectura:** La PIA no consulta el legado para verificar prerrequisitos en tiempo real. Utiliza la réplica local en PostgreSQL (`replica_legado`), garantizando que la validación académica tome < 5 ms sin depender de la red institucional.
2. **Garantía Transaccional Outbox:** Cuando la inscripción es aprobada, se escribe el registro de matrícula y el evento de sincronización en la misma transacción de PostgreSQL. Si el legado está caído, el evento permanece en la tabla `outbox` con estado `PENDIENTE`.
3. **Mecanismo Circuit Breaker en el Worker:**
   * **Estado CLOSED:** El worker toma lotes de 100 eventos de la tabla Outbox y los envía al legado mediante su API.
   * **Transición a OPEN:** Si el sistema legado arroja 5 timeouts consecutivos o errores HTTP 500/503 (como en el inicio del Escenario A), el *Circuit Breaker* pasa a estado `OPEN`. Durante este estado:
     * Se suspenden temporalmente las llamadas hacia el legado por una ventana de 60 segundos.
     * No se interrumpe la inscripción de los estudiantes en la PIA.
     * La respuesta al estudiante indica: `Inscripción confirmada localmente; cupo reservado; sincronización central en cola`.
   * **Transición a HALF-OPEN y Reconciliación:** Cada 60 segundos, el worker envía una sonda (*canary request*). Al transcurrir los 40 minutos de la caída y responder el legado exitosamente, el estado pasa a `CLOSED` y el worker drena los eventos encolados en lotes con control de tasa (*throttling*) para no sobrecargar el sistema legado reanimado.

### 3.5. Beneficios y Costos
* **Beneficios:**
  * Cero interrupciones de servicio para los estudiantes durante la caída de 40 minutos del legado.
  * La cuota de indisponibilidad mensual de la PIA (`C03`) no se ve afectada en lo absoluto por fallas de terceros.
  * Garantía de entrega eventual exacta (*At-Least-Once Delivery*) sin pérdida de registros.
* **Costos:**
  * Almacenamiento en disco para eventos encolados en PostgreSQL durante la indisponibilidad.
  * Estado intermedio transitorio en las inscripciones que debe ser comunicado con claridad a los usuarios.

### 3.6. Mecanismo de Validación y Verificación Experimental
* **Herramienta:** Simulación de fallos de red con **Chaos Engineering** (utilizando iptables / Pumba en el entorno de pruebas Linux).
* **Condición de Prueba:**
  1. Se simula una carga constante de 1.200 usuarios virtuales realizando inscripciones a un ritmo de 50 inscripciones/segundo.
  2. En el minuto 2 de la prueba, se ejecuta un bloqueo total de red hacia la IP/puerto del Sistema Legado simulando la caída de 40 minutos del Escenario A:
     ```bash
     sudo iptables -A OUTPUT -p tcp -d 192.168.1.100 --dport 8080 -j DROP
     ```
  3. El bloqueo se mantiene durante 40 minutos ininterrumpidos y luego se remueve (`iptables -F`).
* **Criterios de Aceptación (Verificación Exitosa):**
  1. Durante los 40 minutos de corte de red, la tasa de éxito de inscripciones en la PIA debe ser $\ge$ 99,9% para estudiantes con prerrequisitos cumplidos.
  2. El tiempo de respuesta de inscripción para el estudiante debe mantenerse $\le$ 500 ms (sin sufrir los timeouts de la conexión caída).
  3. Al restaurarse la red, la tabla `outbox.eventos_sincronizacion` debe drenar el 100% de los eventos pendientes hacia el legado en menos de 10 minutos sin errores de consistencia.

---

## 4. Táctica 3: Caché Escalonada de Catálogo con Invalidación Selectiva y Rate Limiting Perimetral

### 4.1. Atributo de Calidad Asociado
* **QA-01: Rendimiento y Latencia** (Satisfaciendo `C01`, `C02` y Escenario B).

### 4.2. Problema Específico a Resolver
El 95% de las consultas de asignaturas y grupos debe responder en un tiempo inferior a 2 segundos (`C02`) frente a 1.200 (`C01`) y hasta 1.800 estudiantes concurrentes (Escenario B). La consulta de asignaturas involucra múltiples cruces relacionales (materias, docentes, horarios, aulas, cupos). Si cada una de las 1.800 solicitudes concurrentes impacta directamente al motor relacional PostgreSQL, se satura el pool de conexiones (*max_connections* agotado), disparando los tiempos de respuesta por encima de 6 segundos.

### 4.3. Componentes Involucrados
* **Proxy Inverso Nginx en Linux:** Caché perimetral de micro-duración y módulo `limit_req`.
* **API Backend PIA:** Caché de lectura en memoria de la aplicación (*In-Process Cache*).
* **Módulo de Catálogo:** Publicador de eventos de invalidación selectiva ante cambios de cupo.

### 4.4. Funcionamiento Técnico
1. **Rate Limiting Perimetral (Leaky Bucket en Nginx):**
   En la configuración de Nginx (`/etc/nginx/nginx.conf`), se implementa una zona de memoria compartida para limitar peticiones por dirección IP y token JWT:
   ```nginx
   limit_req_zone $binary_remote_addr zone=catalogo_limit:10m rate=10r/s;
   limit_req_zone $http_authorization zone=user_write_limit:10m rate=2r/s;

   server {
       location /api/v1/asignaturas {
           limit_req zone=catalogo_limit burst=20 nodelay;
           proxy_cache catalogo_cache;
           proxy_cache_valid 200 3s; # Micro-caché de 3 segundos para absorber ráfagas masivas
           proxy_pass http://backend_cluster;
       }
   }
   ```
2. **Caché de Catálogo en Memoria de la Aplicación:**
   * La estructura de asignaturas, grupos y horarios (que no cambia durante la matrícula) se carga en memoria RAM del proceso backend en una estructura de lectura altamente concurrente (*ConcurrentHashMap* o equivalente).
   * La disponibilidad de cupos aproximada se refresca cada 2 segundos.
   * Cuando se produce una mutación real de cupo, se actualiza la base de datos PostgreSQL y se invalida selectivamente la clave del grupo específico en la caché local.

### 4.5. Beneficios y Costos
* **Beneficios:**
  * Más del 85% del tráfico de consulta se atiende directamente en memoria RAM o desde Nginx sin tocar PostgreSQL.
  * Tiempos de respuesta para consulta de asignaturas inferiores a 20 ms (muy por debajo del límite de 2,0 s de `C02`).
  * Conservación del 90% de la capacidad de CPU de la base de datos para procesar exclusivamente transacciones de inscripción.
* **Costos:**
  * Consumo de memoria RAM en el servidor Linux (aproximadamente 250 MB para todo el catálogo de las 5 facultades).
  * Posible discrepancia de hasta 2 segundos en el número de cupos visualizados en la pantalla de búsqueda previa al clic de inscripción.

### 4.6. Mecanismo de Validación y Verificación Experimental
* **Herramienta:** Prueba de rendimiento distribuida con **Locust / k6**.
* **Condición de Prueba:**
  * 1.800 usuarios virtuales ejecutando un patrón de navegación realista: 80% consultas de catálogo con filtros de facultad/docente y 20% solicitudes de inscripción.
  * Carga sostenida durante 30 minutos continuos.
* **Criterios de Aceptación (Verificación Exitosa):**
  1. **p95 de tiempos de respuesta en consultas de asignaturas $\le$ 150 ms** (superando con amplitud la meta de 2,0 s).
  2. Percentil 99 (p99) $\le$ 800 ms.
  3. Utilización de CPU en el servidor de base de datos PostgreSQL $\le$ 45%.
  4. Tasa de errores HTTP 5xx igual a 0,00%.

---

## 5. Táctica 4: Continuidad Operativa mediante Continuous WAL Archiving y Réplica Standby Física

### 5.1. Atributo de Calidad Asociado
* **QA-02: Disponibilidad y Continuidad de Negocio** (Satisfaciendo `C06`: RTO $\le$ 30 min y `C07`: RPO $\le$ 5 min).

### 5.2. Problema Específico a Resolver
Si el servidor principal de base de datos sufre una avería catastrófica de hardware o corrupción crítica durante el período de matrícula, la universidad no puede permitirse perder más de 5 minutos de inscripciones realizadas (`C07`), ni tardar más de 30 minutos en restaurar la operación (`C06`). Dado que no se cuenta con servicios comerciales en la nube (`C08`), la solución debe ser completamente autónoma, implementable en Linux y operable por un único administrador (`C10`).

### 5.3. Componentes Involucrados
* **Nodo PostgreSQL Primario (Linux):** Generador de registros de transacción WAL (*Write-Ahead Logs*).
* **Nodo PostgreSQL Standby (Linux):** Servidor secundario en réplica física *streaming*.
* **Almacenamiento NFS/NAS Secundario:** Destino seguro de segmentos WAL archivados cada 60 segundos.
* **Script de Promoción y Failover (`pg_promote.sh`):** Herramienta automatizada de conmutación.

### 5.4. Funcionamiento Técnico
1. **Configuración de Replicación Física Streaming:**
   El nodo primario transmite sus cambios continuamente al nodo secundario mediante el protocolo nativo de replicación de PostgreSQL a través de una red local dedicada.
2. **Archivado Continuo de Segmentos WAL (Garantía RPO $\le$ 5 min):**
   En el archivo `postgresql.conf` del nodo primario se establece un tiempo máximo de retención de WAL sin archivar de 60 segundos:
   ```ini
   wal_level = replica
   archive_mode = on
   archive_command = 'test ! -f /mnt/nas_backup/wal_archive/%f && cp %p /mnt/nas_backup/wal_archive/%f'
   archive_timeout = 60s  # Fuerza el volcado de un segmento cada 60 segundos aunque no esté lleno
   ```
   Esto asegura que, incluso si el nodo primario se destruye físicamente de forma repentina, el 100% de las transacciones confirmadas hasta hace máximo 60 segundos se encuentran ya en el almacenamiento seguro NAS (RPO real $\le$ 1 minuto, cumpliendo holgadamente el RPO $\le$ 5 min de `C07`).
3. **Procedimiento de Conmutación ante Desastre (Garantía RTO $\le$ 30 min):**
   Ante la caída definitiva del nodo primario:
   * El administrador de infraestructura ejecuta el script de promoción en el nodo Standby:
     ```bash
     pg_ctl promote -D /var/lib/postgresql/data
     ```
   * El nodo Standby aplica los últimos segmentos WAL archivados y se convierte en el nuevo nodo Primario de lectura y escritura en menos de 90 segundos.
   * Se actualiza la IP virtual o el DNS interno del clúster (`db.lasalle.local`) para apuntar al nuevo primario.
   * La aplicación PIA reconecta automáticamente sus pools de conexiones (HikariCP/pgx) sin requerir reconfiguración de código.

### 5.5. Beneficios y Costos
* **Beneficios:**
  * Cumplimiento estricto y demostrable de `C06` y `C07` sin costos de licenciamiento de software comercial (`C08`).
  * Procedimiento de failover determinístico y ejecutable en menos de 5 minutos por el administrador de infraestructura (`C10`).
* **Costos:**
  * Requerimiento de un segundo servidor Linux (hardware local de la universidad) y almacenamiento NAS para el archivado.
  * Tráfico de red adicional en la LAN interna para la replicación streaming.

### 5.6. Mecanismo de Validación y Verificación Experimental
* **Herramienta:** Simulacro programado de caída de hardware (*Game Day*).
* **Condición de Prueba:**
  1. Durante una prueba de carga activa con 500 inscripciones en curso, se ejecuta un apagado forzado del nodo primario de base de datos (`kill -9 $(pgrep postgres)` o desconexión de energía).
  2. Se toma la marca de tiempo exacta del incidente ($T_0$).
  3. El administrador de infraestructura recibe la alerta de caída e inicia el procedimiento de failover hacia el nodo Standby.
* **Criterios de Aceptación (Verificación Exitosa):**
  1. **RTO Real Observado $\le$ 8 minutos:** El servicio PIA debe estar recibiendo y procesando nuevas inscripciones en el nodo Standby promovido en menos de 8 minutos desde $T_0$ (muy por debajo del límite de 30 minutos de `C06`).
  2. **RPO Real Observado $\le$ 60 segundos:** Al comparar las inscripciones confirmadas en la bitácora de auditoría antes del corte versus las registradas en la base promovida, la diferencia de pérdida de datos no debe exceder los últimos 60 segundos de transacciones (muy por debajo del límite de 5 minutos de `C07`).

---

## 6. Cuadro Resumen de Tácticas y su Cobertura de Drivers

| Táctica Arquitectónica | Driver / Condición | Métrica Objetivo | Mecanismo de Validación |
| :--- | :--- | :--- | :--- |
| **1. Idempotency-Key + UPDATE Atómico Condicional** | `C04`, Escenario B | Cero sobrecupos y 0 duplicados ante 5 reintentos | Prueba k6 con 1.800 VUs disparando 5 solicitudes por intento sobre 300 cupos |
| **2. Circuit Breaker + Outbox + Réplica Local** | `C03`, `C05`, Escenario A | Continuidad total durante 40 min de caída del legado | Inyección de fallas iptables cortando tráfico al legado durante 40 min continuos |
| **3. Caché Escalonada + Rate Limiting en Nginx** | `C01`, `C02`, Escenario B | p95 $\le$ 2s con 1.800 estudiantes concurrentes | Carga sostenida Locust/k6 con 1.800 usuarios navegando catálogo e inscribiendo |
| **4. WAL Archiving Continuo + Réplica Standby** | `C06`, `C07`, `C08` | RTO $\le$ 30 min y RPO $\le$ 5 min en Linux puro | Simulacro Game Day con apagado forzado de BD midiendo tiempo de recuperación real |
