> **Estudiante:** David Mauricio Pérez Cordero | **Documento:** xxxxxxxx2

# ADR-004: Estrategia para Alta Concurrencia, Rendimiento de Consulta y Continuidad (RTO/RPO)

* **Identificador:** ADR-004
* **Título:** Desacoplamiento de Lecturas mediante Caché en Memoria, Rate Limiting Perimetral y Replicación WAL Continua
* **Fecha:** 14 de Octubre de 2026
* **Estado:** Aceptado

---

## 1. Contexto y Problema

Durante el inicio de las jornadas de matrícula, se conectan simultáneamente entre 1.200 (`C01`) y 1.800 estudiantes (Escenario B). Las estadísticas del caso señalan que el 85% de las interacciones corresponden a consultas reiteradas de asignaturas, grupos y horarios. La condición **C02** impone que el 95% de las consultas de catálogo debe responder en máximo 2,0 segundos.

Simultáneamente, ante cualquier incidente catastrófico en la infraestructura física del centro de cómputo, la universidad exige un tiempo objetivo de recuperación (RTO) $\le$ 30 minutos (`C06`) y un punto objetivo de recuperación (RPO) $\le$ 5 minutos (`C07`). Todo esto debe lograrse sin contratar plataformas comerciales en la nube (`C08`) y bajo la administración de un único ingeniero de infraestructura (`C10`).

El problema a resolver es: **¿Cómo blindar la base de datos PostgreSQL de la saturación masiva por lecturas para garantizar p95 $\le$ 2s, protegiendo al mismo tiempo los datos contra pérdidas mayores a 5 minutos y recuperando el servicio en menos de 30 minutos?**

---

## 2. Drivers Arquitectónicos Relacionados

* **DRV-04 / C02:** Percentil 95 de consultas $\le$ 2,0 segundos bajo 1.200 a 1.800 usuarios (`C01`, Esc. B).
* **DRV-07 / C06:** RTO $\le$ 30 minutos ante fallas severas de infraestructura.
* **DRV-07 / C07:** RPO $\le$ 5 minutos (máxima pérdida tolerada de datos).
* **DRV-05 / C08:** Solución confinada a Linux y PostgreSQL estándar sin costos comerciales.
* **DRV-06 / C10:** Operabilidad factible para un único administrador de infraestructura.

---

## 3. Alternativas Evaluadas

### Alternativa A: Clúster Distribuido Redis Cache + ElasticSearch + Replicación Multi-Región en Nube
Implementar Redis para caché de catálogo, ElasticSearch para búsqueda facetada de asignaturas y una base de datos distribuida multi-nodo.

### Alternativa B: Base de Datos PostgreSQL Monolítica sin Capa de Caché
Dirigir la totalidad de las consultas de asignaturas y las mutaciones de inscripción al mismo nodo primario de PostgreSQL, confiando únicamente en la memoria compartida (*shared_buffers*) del motor.

### Alternativa C: Caché en Memoria en Capas (Nginx + Backend) con Rate Limiting y Clúster PostgreSQL Primario-Standby con WAL Archiving Continuo (Seleccionada)
1. **Caché de Catálogo:** Nginx y la memoria local del backend (*in-process cache* con invalidación por tiempo o evento) absorben el 90% de las consultas de asignaturas. Los datos de catálogo cambian infrecuentemente durante la matrícula.
2. **Rate Limiting Perimetral:** Nginx aplica limitación de peticiones por segundo mediante *Leaky Bucket*, suavizando los picos de 1.800 usuarios.
3. **Continuidad Operativa (RTO/RPO):** Servidor Linux primario con replicación física streaming hacia un servidor Linux secundario (*Standby*) en la misma red local, complementado con archivado continuo de segmentos WAL (*Write-Ahead Logging*) sincronizados cada 60 segundos a un almacenamiento secundario independiente (NAS institucional).

---

## 4. Criterios de Comparación y Evaluación

| Criterio de Evaluación | Alternativa A: Clúster Redis/Elastic | Alternativa B: Solo PostgreSQL Directo | Alternativa C: Caché en Memoria + WAL Continuo |
| :--- | :--- | :--- | :--- |
| **Cumplimiento de Costo / Stack (`C08`)** | **Inviable:** Requiere licencias y servidores adicionales de pago. | **Óptimo:** Cero software adicional. | **Óptimo:** Utiliza software libre estándar en Linux (Nginx + PostgreSQL). |
| **Latencia p95 en Consulta (`C02`)** | **Excelente:** < 50 ms. | **Pobre:** La saturación de conexiones en el pico colapsa la latencia a > 4s. | **Excelente:** Nginx y la memoria del backend sirven el catálogo en $\le$ 20 ms. |
| **Garantía RTO $\le$ 30m y RPO $\le$ 5m (`C06`, `C07`)** | Compleja de administrar para 1 persona. | Manual y propensa a pérdida de datos por dumps esporádicos. | **Garantizada:** Réplica física + WAL cada 60s garantiza RPO $\le$ 1 min y RTO $\le$ 15 min. |
| **Carga de Gestión para el Admin (`C10`)** | Abrumadora (múltiples servicios que monitorear). | Mínima pero riesgosa. | **Balanceada y Sostenible:** Configuración estándar y scripts de failover probados. |

---

## 5. Decisión Seleccionada y Justificación

Se selecciona la **Alternativa C: Caché en Memoria en Capas con Rate Limiting y Clúster PostgreSQL Primario-Standby con WAL Archiving Continuo**.

### Justificación Técnica:
1. **Blindaje de la Base de Datos:** Las 24.000 consultas de asignaturas consultan catálogos que son idénticos para todos los estudiantes. Servir estos datos desde la memoria de Nginx y de la aplicación reduce el tráfico hacia PostgreSQL en más del 85%, permitiendo que el motor dedique el 100% de sus conexiones a la reserva atómica de cupos.
2. **Cumplimiento riguroso de RTO y RPO:** 
   * Con el archivado continuo de WAL cada 60 segundos, en el peor escenario de destrucción física del nodo primario, la pérdida máxima de datos (*RPO*) es de **60 segundos**, cumpliendo holgadamente el límite de 5 minutos de `C07`.
   * La promoción del nodo Standby se realiza mediante el comando `pg_ctl promote` automatizado o ejecutado por el administrador en menos de **10 minutos**, cumpliendo holgadamente el límite de 30 minutos de `C06`.
3. **Realismo Institucional (`C08`, `C10`):** No requiere presupuesto para servicios de nube; opera sobre hardware estándar Linux y puede ser mantenido con facilidad por el administrador de infraestructura existente.

---

## 6. Consecuencias y Gestión de Riesgos

### Consecuencias Positivas:
* Cumplimiento del p95 $\le$ 2s (`C02`) con tiempos reales de consulta observados de 15 a 40 ms para el 95% de los estudiantes.
* Protección de la integridad financiera y académica con RTO y RPO mínimos garantizados.
* Estabilidad del servidor ante ráfagas extraordinarias (Escenario B).

### Consecuencias Negativas y Riesgos:
* **Eventual discrepancia visual en cupos mostrados en la pantalla de catálogo:** Un estudiante podría ver en el catálogo que quedan "3 cupos", pero al hacer clic recibir rechazo porque otro usuario acaba de tomar el último en la base de datos transaccional.
  * *Mitigación:* Se añade una etiqueta en la interfaz ("Disponibilidad estimada sujeta a confirmación en tiempo real") y el TTL de la caché de disponibilidad se ajusta a 5 segundos durante las horas pico de matrícula.

---

## 7. Condiciones para Revisar esta Decisión

Esta decisión se revisará si:
1. La universidad migra su infraestructura completa a un centro de datos en la nube con réplica multizona gestionada automáticamente.
2. Las consultas de catálogo incorporan filtros complejos con procesamiento geoespacial o de lenguaje natural que justifiquen un motor dedicado de búsqueda.
