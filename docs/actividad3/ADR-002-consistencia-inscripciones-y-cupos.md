> **Estudiante:** David Mauricio Pérez Cordero | **Documento:** xxxxxxxx2

# ADR-002: Estrategia para Mantener la Consistencia de Inscripciones y Cupos

* **Identificador:** ADR-002
* **Título:** Estrategia de Consistencia Estricta de Cupos mediante Actualización Atómica Condicional e Idempotencia
* **Fecha:** 14 de Octubre de 2026
* **Estado:** Aceptado

---

## 1. Contexto y Problema

Uno de los mayores defectos del sistema académico legado es la generación de cupos sobrepasados (*sobrecupos*) e inscripciones duplicadas provocadas por la desesperación de los estudiantes al reenviar peticiones durante la matrícula. La condición **C04** exige de forma terminante: *"Ningún grupo académico puede registrar más estudiantes que su capacidad autorizada"*. 

Adicionalmente, bajo el **Escenario B**, la concurrencia puede ascender a 1.800 estudiantes simultáneos (`C01` + 50%) y algunos usuarios generan hasta 5 peticiones automáticas/manuales consecutivas para la misma asignatura en fracciones de segundo.

El problema a resolver es: **¿Cómo garantizar matemáticamente cero sobrecupos y cero duplicados bajo altísima contención sin bloquear la base de datos ni violar el tiempo de respuesta de 2 segundos (`C02`)?**

---

## 2. Drivers Arquitectónicos Relacionados

* **DRV-01 / C04:** Restricción absoluta de cero sobrecupos por grupo académico.
* **DRV-02 / Escenario B:** Erradicación de inscripciones duplicadas ante hasta 5 reintentos repetidos.
* **DRV-04 / C02:** Tiempos de respuesta p95 $\le$ 2 segundos.
* **DRV-05 / C08:** Ejecución en PostgreSQL sin herramientas propietarias externas.

---

## 3. Alternativas Evaluadas

### Alternativa A: Bloqueo Pesimista Prolongado (`SELECT ... FOR UPDATE`)
Al procesar la inscripción, se ejecuta un `SELECT ... FOR UPDATE` sobre la fila del grupo académico, bloqueando el registro de manera exclusiva durante toda la verificación de prerrequisitos, cruces de horario e inserción de la matrícula, liberando el bloqueo al ejecutar el `COMMIT`.

### Alternativa B: Cola Asíncrona de Reserva con Consistencia Eventual
Las peticiones de inscripción entran a una cola en memoria; se responde inmediatamente un código HTTP 202 (Aceptado) al usuario, y un proceso por lotes intenta asignar los cupos en segundo plano, notificando posteriormente el resultado.

### Alternativa C: Actualización Atómica Condicional con Idempotencia Transaccional (Seleccionada)
1. Desacoplar las validaciones de negocio previas (prerrequisitos y cruces se validan contra réplicas de solo lectura sin bloquear).
2. La reserva del cupo se ejecuta en una única instrucción SQL atómica respaldada por el motor MVCC de PostgreSQL:
   ```sql
   UPDATE matricula.grupos_cupos
   SET cupos_ocupados = cupos_ocupados + 1
   WHERE id = :grupoId AND cupos_ocupados < capacidad_maxima;
   ```
3. Control estricto de reintentos mediante encabezado HTTP `Idempotency-Key` respaldado por una tabla de idempotencia y una restricción relacional `UNIQUE(estudiante_id, asignatura_id, periodo_academico)`.

---

## 4. Criterios de Comparación y Evaluación

| Criterio de Evaluación | Alternativa A: Bloqueo Pesimista | Alternativa B: Consistencia Eventual | Alternativa C: Actualización Atómica Condicional |
| :--- | :--- | :--- | :--- |
| **Garantía de Cero Sobrecupos (`C04`)** | **Alta:** Bloqueo exclusivo previene escrituras simultáneas. | **Pobre / Riesgosa:** Si la cola recibe más solicitudes que cupos reales, el descarte diferido causa frustración e inconsistencias. | **Matemática y Absoluta:** La condición del `WHERE` es evaluada por PostgreSQL atómicamente a nivel de registro. |
| **Tiempo de Respuesta bajo Concurrencia (`C02`)** | **Muy Malo:** Provoca *lock contention* masiva; colas de espera en la BD superan los 8 segundos ante 1.800 usuarios. | **Excelente:** Retorno inmediato (< 100 ms), pero traslada la incertidumbre al usuario. | **Excelente:** El bloqueo a nivel de fila dura menos de 1 ms, permitiendo cientos de reservas por segundo. |
| **Protección contra 5 Reintentos (Escenario B)** | **Inadecuada:** Los 5 reintentos forman cola y pueden terminar insertando filas duplicadas si no hay lock compuesto. | **Media:** Requiere deduplicación compleja en la cola de mensajes. | **Óptima:** Las 4 peticiones duplicadas retornan en $\le$ 50 ms la respuesta ya calculada sin tocar el cupo. |
| **Simplicidad de Implementación (`C10`)** | Media. | Compleja (manejo de estados intermedios y reintentos). | Alta (SQL estándar de PostgreSQL nativo). |

---

## 5. Decisión Seleccionada y Justificación

Se selecciona la **Alternativa C: Actualización Atómica Condicional con Idempotencia Transaccional**.

### Justificación Técnica:
1. **Eliminación de la contención de bloqueos (*Lock-Free Read / Ultra-short Write*):** A diferencia de la Alternativa A, donde la fila del grupo permanece bloqueada durante toda la validación académica (10-30 ms), en la Alternativa C las validaciones de prerrequisitos y cruces se efectúan en memoria contra esquemas de lectura. El `UPDATE` atómico condicional dura fracciones de milisegundo. Si el grupo ya no tiene cupo, `rows affected` retorna 0 y se aborta la transacción limpiamente.
2. **Mitigación total del Escenario B:** Al combinar la llave de idempotencia con la restricción `UNIQUE` en la tabla de inscripciones, se neutraliza por completo la ráfaga de 5 reintentos automáticos de los estudiantes, logrando que el 80% del tráfico duplicado se atienda instantáneamente sin recargar la base de datos.
3. **Cumplimiento estricto de PostgreSQL (`C08`):** No requiere extensiones propietarias ni software adicional; se fundamenta en las garantías ACID estándares del motor relacional.

---

## 6. Consecuencias y Gestión de Riesgos

### Consecuencias Positivas:
* Garantía de cero sobrecupos (`C04`) demostrable matemáticamente a nivel de motor de base de datos.
* Tiempos de respuesta constantes $\le$ 300 ms en la operación de inscripción, cumpliendo con creces el p95 $\le$ 2s (`C02`).
* Cero registros duplicados en el historial de matrícula.

### Consecuencias Negativas y Riesgos:
* **Falsos rechazos por carrera de último cupo:** Cuando queda exactamente 1 cupo y 50 estudiantes lo solicitan simultáneamente, 1 gana y 49 reciben rechazo por cupo agotado.
  * *Evaluación:* Este no es un defecto, sino el comportamiento matemáticamente correcto y exigido por la universidad.
* **Sobrecarga de la tabla de idempotencia:**
  * *Mitigación:* Se implementa un trabajo programado (*cron / background task*) que depura registros de idempotencia con más de 48 horas de antigüedad.

---

## 7. Condiciones para Revisar esta Decisión

Esta decisión se revisará si:
1. La universidad adopta un modelo de matrícula por subasta de créditos o listas de espera dinámicas que no requieran confirmación de cupo en tiempo real.
2. Se introduce la necesidad de reservar cupos distribuidos a través de múltiples campus geográficos independientes con bases de datos heterogéneas.
