> **Estudiante:** David Mauricio Pérez Cordero | **Documento:** xxxxxxxx2

# ADR-001: Selección del Estilo Arquitectónico para la Plataforma PIA

* **Identificador:** ADR-001
* **Título:** Selección del Estilo Arquitectónico Base: Monolito Modular con Arquitectura Hexagonal
* **Fecha:** 14 de Octubre de 2026
* **Estado:** Aceptado

---

## 1. Contexto y Problema

La Universidad de la Salle requiere construir la nueva Plataforma de Inscripción Académica (PIA) para atender a 24.000 estudiantes y 1.100 docentes, superando los colapsos del sistema legado de más de 12 años. Durante los picos de matrícula, se conectan entre 1.200 (`C01`) y 1.800 usuarios concurrentes, exigiendo respuestas en menos de 2 segundos (`C02`) y cero sobrecupos (`C04`).

Sin embargo, el proyecto cuenta con dos restricciones críticas institucionales:
1. La universidad dispone exclusivamente de infraestructura Linux y PostgreSQL, sin presupuesto para servicios de nube de pago ni plataformas comerciales adicionales (`C08`).
2. El equipo de mantenimiento está restringido a cuatro (4) desarrolladores y un (1) administrador de infraestructura (`C10`).

El problema a resolver es: **¿Qué estilo arquitectónico permite maximizar la mantenibilidad, el rendimiento y la consistencia transaccional sin desbordar la capacidad operativa del equipo reducido ni infringir las restricciones de infraestructura?**

---

## 2. Drivers Arquitectónicos Relacionados

* **DRV-05 / C08:** Restricción tecnológica a servidores Linux y base de datos PostgreSQL estándar.
* **DRV-06 / C10:** Restricción de equipo (4 desarrolladores y 1 administrador).
* **DRV-01 / C04:** Integridad y consistencia estricta en cupos (0 sobrecupos).
* **DRV-10 / QA-05:** Modificabilidad para cambiar reglas de matrícula sin alterar integraciones.

---

## 3. Alternativas Evaluadas

### Alternativa A: Arquitectura de Microservicios Distribuidos
Descomponer la solución en múltiples microservicios independientes (Servicio de Catálogo, Servicio de Matrícula, Servicio de Cupos, Servicio de Auditoría y Servicio de Integración Legada), cada uno con su propio almacenamiento y comunicados mediante un broker de mensajería (Kafka/RabbitMQ) y un API Gateway.

### Alternativa B: Monolito Tradicional en Capas (Layered Monolith)
Construir una aplicación monolítica clásica organizada horizontalmente en capas (Presentación, Negocio, Acceso a Datos) sobre una única base de datos relacional compartida.

### Alternativa C: Monolito Modular con Arquitectura Hexagonal (Seleccionada)
Construir un único artefacto desplegable en Linux, estructurado verticalmente en módulos de dominio bien delimitados (Catálogo, Matrícula, Auditoría, Integración). Cada módulo aplica principios de Arquitectura Hexagonal (Puertos y Adaptadores), desacoplando las reglas de negocio de la infraestructura y aislando el acceso a la base de datos PostgreSQL mediante límites claros.

---

## 4. Criterios de Comparación y Evaluación

| Criterio de Evaluación | Alternativa A: Microservicios | Alternativa B: Monolito en Capas | Alternativa C: Monolito Modular Hexagonal |
| :--- | :--- | :--- | :--- |
| **Carga Operativa para el Equipo (`C10`)** | **Muy Alta (Negativa):** Requiere coordinar múltiples pipelines CI/CD, service discovery, observabilidad distribuida y orquestación compleja. | **Baja (Positiva):** Despliegue simple, un solo repositorio y monitoreo básico. | **Baja-Media (Óptima):** Despliegue unificado en un solo binario Linux; gestión operativa accesible para 1 admin. |
| **Consistencia Transaccional de Cupos (`C04`)** | **Baja/Compleja (Riesgo):** Exige transacciones distribuidas (Saga / 2PC) o consistencia eventual, propenso a sobrecupos bajo 1.800 concurrentes. | **Alta (Positiva):** Permite transacciones ACID en PostgreSQL, pero propensa a contención de tablas si no hay modularidad. | **Máxima (Excelente):** Transacciones ACID locales ultrarrápidas dentro del módulo de matrícula en PostgreSQL. |
| **Cumplimiento de Costo / Stack (`C08`)** | **Pobre:** Requiere múltiples servidores, brokers distribuidos y consumo elevado de memoria RAM. | **Excelente:** Se adapta a un único servidor Linux y PostgreSQL. | **Excelente:** Optimiza el uso de CPU y memoria en un único servidor Linux con PostgreSQL. |
| **Aislamiento y Mantenibilidad (`QA-05`)** | **Alta:** Servicios físicamente aislados. | **Pobre:** Tendencia histórica al espagueti y acoplamiento de capas ("Big Ball of Mud"). | **Alta:** Aislamiento estricto por paquetes e interfaces; previene fugas de dependencias hacia el legado. |

---

## 5. Decisión Seleccionada y Justificación

Se selecciona la **Alternativa C: Monolito Modular con Arquitectura Hexagonal**.

### Justificación Técnica:
1. **Alineación con la capacidad del equipo (`C10`):** Con solo 4 desarrolladores y 1 administrador, mantener un clúster de microservicios provocaría fatiga operativa, ralentizando el ciclo de desarrollo. El Monolito Modular permite una experiencia de desarrollo cohesionada en un solo repositorio (*monorepo*), simplificando el depurado local, la ejecución de pruebas y el despliegue mediante un único servicio `systemd` en Linux.
2. **Garantía matemática de cero sobrecupos (`C04`):** El módulo de inscripciones puede ejecutar transacciones ACID locales de menos de 2 milisegundos en PostgreSQL, evitando la complejidad y latencia de transacciones distribuidas entre redes.
3. **Aislamiento del modelo legado:** La Arquitectura Hexagonal asegura que el sistema legado sea tratado como un adaptador secundario tras un puerto de salida (*Anti-Corruption Layer*), impidiendo que el diseño de la nueva plataforma se contamine con los vicios del sistema anterior.

---

## 6. Consecuencias y Gestión de Riesgos

### Consecuencias Positivas:
* Costo operativo cero en licencias de orquestación o brokers comerciales (`C08`).
* Latencia de comunicación inter-módulo prácticamente nula (llamadas en memoria, eliminando *network overhead*).
* Facilidad de pruebas unitarias y de integración sobre el dominio académico sin dependencias externas.

### Consecuencias Negativas y Riesgos:
* **Riesgo de degradación hacia un monolito acoplado:** Si los desarrolladores rompen las fronteras de los módulos haciendo llamadas directas o consultas SQL cruzadas.
  * *Mitigación:* Se implementan pruebas de arquitectura automatizadas en CI/CD mediante herramientas como `ArchUnit` o `import-linter`, que bloquean cualquier Pull Request que viole el aislamiento de los módulos (ver Actividad 5).
* **Escalado vertical como primer recurso:** Escalar la aplicación implica escalar todo el binario en el servidor Linux.
  * *Mitigación:* Nginx balancea la carga localmente entre 2 a 4 instancias del binario en la misma máquina si se requiere aprovechar múltiples núcleos de CPU.

---

## 7. Condiciones para Revisar esta Decisión

Esta decisión deberá someterse a revisión si:
1. El equipo de desarrollo e infraestructura se incrementa a más de 12 ingenieros con equipos dedicados por facultad.
2. La universidad aprueba presupuesto formal de nube pública para servicios administrados.
3. El módulo de consulta de asignaturas experimenta una demanda 10 veces superior que obligue a desacoplarlo físicamente en un clúster independiente de lectura.
