> **Estudiante:** David Mauricio Pérez Cordero | **Documento:** xxxxxxxx2

# Actividad 5. Modelo de Gobernanza de Arquitectura y Controles Automáticos
## Plataforma de Inscripción Académica (PIA) - Universidad de la Salle

---

## 1. Enfoque de Gobernanza para Equipos Reducidos

El modelo de gobernanza arquitectónica para la Plataforma de Inscripción Académica (PIA) está rigurosamente adaptado a la capacidad del equipo institucional de la Universidad de la Salle, conformado por **cuatro (4) desarrolladores de software y un (1) administrador de infraestructura** (`C10`). 

En este contexto, un modelo burocrático tradicional con comités formales resulta inoperante y generaría cuellos de botella. La gobernanza propuesta se fundamenta en los principios de **Arquitectura Ágil, Gobernanza como Código (*Governance-as-Code*) y Verificación Automatizada en CI/CD**, garantizando que las decisiones y límites estructurales definidos en las Actividades 1, 2, 3 y 4 se mantengan incólumes a lo largo del ciclo de vida del software.

---

## 2. Políticas y Reglas Arquitectónicas Verificables

Se establecen tres reglas arquitectónicas inquebrantables, formuladas de manera formal y verificable mediante herramientas automatizadas:

```
+--------------------------------------------------------------------------------------------------------+
|                                    REGLAS ARQUITECTÓNICAS BÁSICAS                                      |
+--------------------------------------------------------------------------------------------------------+
|  RA-01: Aislamiento del Dominio y del Sistema Legado                                                   |
|  - El dominio de PIA NUNCA importa modelos del Sistema Legado.                                         |
|  - Toda comunicación debe pasar exclusivamente por el puerto Anti-Corruption Layer (ACL).              |
+--------------------------------------------------------------------------------------------------------+
|  RA-02: Fronteras Estrictas de Módulos y Persistencia                                                  |
|  - Prohibidos los JOINs SQL directos entre tablas de esquemas diferentes (ej. matricula vs auditoria). |
|  - La interacción inter-módulo se realiza mediante interfaces de servicio en memoria.                 |
+--------------------------------------------------------------------------------------------------------+
|  RA-03: Mutaciones Seguras, Idempotentes y Auditadas                                                    |
|  - Todo endpoint de escritura (POST/PUT/DELETE) exige la cabecera Idempotency-Key.                     |
|  - Todo cambio de estado debe registrar una traza inmutable en auditoria.trazabilidad en la misma Tx.  |
+--------------------------------------------------------------------------------------------------------+
```

### Regla RA-01: Aislamiento Estricto del Dominio y del Sistema Legado
* **Enunciado:** Ninguna clase, servicio o entidad de los módulos de Dominio o Aplicación puede importar dependencias, invocar directamente métodos o consumir modelos de datos originados en el Sistema Académico Legado. Toda comunicación externa debe canalizarse exclusivamente a través de la interfaz del puerto *Anti-Corruption Layer (ACL)* ubicado en el módulo de infraestructura.
* **Mecanismo de Verificación:** Regla estática ejecutada en CI/CD con **ArchUnit / import-linter**. La prueba falla automáticamente si detecta cualquier importación prohibida hacia paquetes o clientes del legado.

### Regla RA-02: Fronteras de Módulos y Prohibición de Consultas Cruzadas en Base de Datos
* **Enunciado:** Está terminantemente prohibido ejecutar sentencias SQL que realicen operaciones `JOIN` o mutaciones directas entre tablas pertenecientes a esquemas lógicos distintos (`matricula`, `replica_legado`, `auditoria`, `outbox`). Si el módulo de inscripciones requiere datos de auditoría o catálogo, debe solicitarlo a través de la interfaz pública del módulo correspondiente.
* **Mecanismo de Verificación:** Análisis estático de código SQL con linters y pruebas de integración sobre esquemas con permisos de usuario de base de datos diferenciados.

### Regla RA-03: Idempotencia Obligatoria y Trazabilidad Transaccional en Mutaciones
* **Enunciado:** Todo endpoint de la API que modifique estado (`POST`, `PUT`, `DELETE` en `/api/v1/inscripciones/**`) debe obligatoriamente:
  1. Validar la presencia del encabezado HTTP `Idempotency-Key`.
  2. Ejecutarse dentro de una transacción `@Transactional` que garantice la deducción atómica de cupo (`C04`), el registro en la bitácora `auditoria.trazabilidad` (`C09`) y la inserción del evento en `outbox.eventos_sincronizacion` (`C05`).
* **Mecanismo de Verificación:** Prueba de integración automatizada que dispara peticiones sin encabezado esperando `HTTP 400 Bad Request` y suites de concurrencia que verifican que ante 2 peticiones paralelas con la misma llave sólo se inserte 1 registro en la base de datos.

---

## 3. Estructura de Roles y Matriz de Responsabilidades (RACI)

Dado el equipo de 5 integrantes (`C10`), se distribuyen las responsabilidades arquitectónicas de forma clara y pragmática:

* **Líder Técnico / Arquitecto (1 Desarrollador Senior del equipo):** Custodio de la coherencia arquitectónica, moderador de ADRs y aprobador final de cambios mayores.
* **Desarrolladores de Software (3 Desarrolladores):** Implementación de historias de usuario, elaboración de pruebas unitarias/arquitectura y propuesta de mejoras técnicas.
* **Administrador de Infraestructura (1 Administrador de Sistemas Linux y PostgreSQL):** Responsable de la configuración de servidores Linux, tuning de PostgreSQL, políticas de backup WAL (RTO/RPO), Nginx y monitoreo de producción.

### Matriz RACI

| Actividad / Decisión Arquitectónica | Líder Técnico (Arquitecto) | Desarrolladores (3) | Administrador de Infraestructura |
| :--- | :---: | :---: | :---: |
| **Proponer un nuevo ADR o modificar uno existente** | A / R | R | C |
| **Aprobar cambios estructurales o patrones de diseño** | **A** | C | C |
| **Modificar configuración de PostgreSQL (WAL, shared_buffers)** | C | I | **A / R** |
| **Configuración de Nginx (Rate Limiting, Reverse Proxy)** | C | I | **A / R** |
| **Aprobar Pull Requests cotidianos de código** | R | **A / R** | I |
| **Aprobar Pull Requests con impacto arquitectónico mayor** | **A** | R | C |
| **Simulacros de Recuperación ante Desastres (RTO/RPO)** | C | I | **A / R** |
| **Autorizar una excepción temporal de deuda técnica** | **A** | C | C |

*Convenciones: **R** = Responsable de ejecutar, **A** = Aprobador final (Accountable), **C** = Consultado, **I** = Informado.*

---

## 4. Procedimiento para el Ciclo de Vida de los ADR (Decisiones Arquitectónicas)

Cualquier cambio que impacte las condiciones `C01` a `C10`, los límites entre módulos o la selección tecnológica debe documentarse formalmente como un **Architectural Decision Record (ADR)**.

```
+------------+       Pull Request        +--------------+      Revisión Técnica     +------------+
|  PROPUESTO | ------------------------> | EN REVISIÓN  | ------------------------> |  ACEPTADO  |
+------------+   (Rama feat/adr-xxx)     +--------------+   (Aprobación Líder Tech) +------------+
                                                |                                          |
                                                | Discrepancias insalvables                | Reemplazado
                                                v                                          v
                                         +--------------+                           +--------------+
                                         |  RECHAZADO   |                           | OBSOLETO /   |
                                         +--------------+                           | REEMPLAZADO  |
                                                                                    +--------------+
```

### Pasos del Procedimiento:
1. **Identificación de la Necesidad:** Cualquier miembro del equipo identifica una fuerza o conflicto arquitectónico no cubierto.
2. **Redacción del Borrador:** Se crea una rama `docs/adr-nuevo-tema` y se redacta el archivo en `docs/actividad3/ADR-XXX-titulo.md` utilizando la plantilla oficial (Contexto, Drivers, Alternativas, Decisión, Consecuencias y Criterios de Revisión). Estado inicial: `Propuesto`.
3. **Revisión por Pares (Peer Review):** Se abre un Pull Request en el repositorio. Los demás desarrolladores y el administrador de infraestructura comentan y formulan objeciones durante un plazo máximo de 48 horas.
4. **Decisión y Aprobación:** El Líder Técnico convoca una reunión de sincronización técnica de 30 minutos. Se adopta la decisión por consenso o decisión del Líder Técnico. El estado del ADR se actualiza a `Aceptado`.
5. **Merge e Implementación:** El PR se fusiona a la rama principal. Ningún código asociado a la decisión puede fusionarse antes de que el ADR sea aceptado.
6. **Deprecación / Sustitución:** Si las condiciones del entorno cambian (ej. presupuesto para nube o cambio del sistema legado), el ADR original no se borra; se marca como `Reemplazado por ADR-YYY`, manteniendo la memoria histórica del proyecto.

---

## 5. Controles Automáticos en el Repositorio (CI/CD Pipeline)

Para garantizar el cumplimiento de las reglas sin depender de inspecciones manuales falibles, se implementa un pipeline de integración continua en el repositorio (`.github/workflows/ci.yml` o GitLab CI equivalente) con dos controles automáticos mandatorios:

```yaml
name: Pipeline de Integración Continua y Gobernanza Arquitectónica

on:
  push:
    branches: [ main, dev_* ]
  pull_request:
    branches: [ main, dev_* ]

jobs:
  governance-and-quality:
    name: Verificación de Reglas Arquitectónicas y Pruebas
    runs-on: ubuntu-latest
    
    services:
      postgres-test:
        image: postgres:15-alpine
        env:
          POSTGRES_DB: pia_test
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Descargar Código Fuente
        uses: actions/checkout@v3

      - name: Configurar Entorno de Ejecución (Java 17 / Go / Python)
        uses: actions/setup-java@v3
        with:
          distribution: 'temurin'
          java-version: '17'
          cache: 'maven'

      # ======================================================================
      # CONTROL AUTOMÁTICO 1: Verificación de Reglas Arquitectónicas (ArchUnit)
      # ======================================================================
      - name: Control 1 - Ejecutar Pruebas Estáticas de Arquitectura (ArchUnit)
        run: |
          echo "Verificando RA-01 (Aislamiento de Legado) y RA-02 (Fronteras Modulares)..."
          mvn test -Dtest=*ArchitectureTest
        # Este paso falla el build si alguna clase del dominio importa el paquete del legado
        # o si existen dependencias cíclicas o violaciones a la arquitectura hexagonal.

      # ======================================================================
      # CONTROL AUTOMÁTICO 2: Pruebas Unitarias, de Integración y Concurrencia
      # ======================================================================
      - name: Control 2 - Ejecución de Pruebas Unitarias y de Integración
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/pia_test
          SPRING_DATASOURCE_USERNAME: test_user
          SPRING_DATASOURCE_PASSWORD: test_password
        run: |
          echo "Ejecutando pruebas de integración, idempotencia y validación de cupos..."
          mvn test -Dtest=!*ArchitectureTest
        # Valida RA-03: pruebas con 50 hilos concurrentes intentando sobrepasar cupos,
        # verificando que cupos_ocupados nunca exceda la capacidad máxima (C04).
```

### Ejemplo de Prueba Arquitectónica Automatizada (`ArchitectureTest.java`):

```java
@AnalyzeClasses(packages = "co.edu.lasalle.pia")
public class ArchitectureTest {

    // Verificación de RA-01: El dominio no puede depender del sistema legado
    @ArchTest
    public static final ArchRule dominio_no_debe_acoplarse_al_legado =
        noClasses().that().resideInAPackage("co.edu.lasalle.pia.domain..")
            .should().dependOnClassesThat().resideInAPackage("co.edu.lasalle.pia.infrastructure.legacy..");

    // Verificación de RA-02: Los módulos de aplicación no deben tener dependencias cíclicas
    @ArchTest
    public static final ArchRule modulos_libres_de_ciclos =
        slices().matching("co.edu.lasalle.pia.(*)..")
            .should().beFreeOfCycles();
            
    // Verificación de Hexagonal: La capa de dominio es pura y agnóstica de frameworks
    @ArchTest
    public static final ArchRule dominio_independiente_de_spring =
        noClasses().that().resideInAPackage("co.edu.lasalle.pia.domain..")
            .should().dependOnClassesThat().resideInAPackage("org.springframework..");
}
```

---

## 6. Procedimiento para el Manejo de Excepciones y Deuda Técnica

En situaciones de emergencia operativa previa a una jornada de matrícula, puede requerirse una solución temporal que viole una regla arquitectónica (ej. una consulta directa temporal para un reporte no planificado).

Para evitar que la excepción se convierta en degradación permanente del sistema:
1. **Formulario de Registro de Deuda Técnica:** Toda excepción debe registrarse como un *Issue* en GitHub etiquetado obligatoriamente como `technical-debt` y `architecture-exception`.
2. **Información Mínima Exigida:**
   * Regla arquitectónica vulnerada (ej. `RA-02`).
   * Justificación operativa y riesgo evaluado.
   * Plan de remediación y código temporal etiquetado con comentario `@TechnicalDebt(ticket = "PIA-405", deadline = "2026-11-30")`.
3. **Plazo de Expiración:** Ninguna excepción puede tener una vigencia superior a **45 días calendario** o debe resolverse antes del siguiente período semestral de matrícula.
4. **Presupuesto de Deuda Técnica:** En cada iteración o sprint de desarrollo del equipo de 4 ingenieros, se destina formalmente el **20% de la capacidad de desarrollo (horas/hombre)** exclusivamente a la amortización y refactorización de tickets de deuda técnica.

---

## 7. Estrategia Docs-as-Code para Diagramas y Documentación

Para garantizar que la documentación de arquitectura y los diagramas C4 no queden obsoletos frente a la evolución del código:
* **Almacenamiento Centralizado:** Todos los documentos se gestionan en la carpeta `/docs/` del mismo repositorio Git de la solución en formato Markdown (`.md`).
* **Diagramas Vectoriales Versionables:** Los diagramas C4 se mantienen como archivos vectoriales `.svg` en `docs/diagramas/`. Cada vez que se añade un contenedor o se modifica un flujo, el diagrama debe actualizarse en el mismo Pull Request que introduce el cambio en el código.
* **Linter de Documentación:** Se valida en CI que las referencias a archivos e imágenes en los Markdown no tengan enlaces rotos.

---

## 8. Criterios de Aceptación y Rechazo de un Pull Request (PR Gate Checklist)

Para que una solicitud de integración (*Pull Request*) sea aceptada e incorporada a las ramas protegidas (`main` o `dev_*`), debe cumplir estrictamente con los siguientes criterios objetivos:

### Lista de Verificación Obligatoria (PR Checklist):

- [ ] **1. Controles CI Verificados en Verde:**
  - El pipeline automático ha compilado y ejecutado exitosamente los tests de arquitectura (`ArchUnit`) sin violaciones a `RA-01` ni `RA-02`.
  - El 100% de las pruebas unitarias y de integración pasan satisfactoriamente.
- [ ] **2. Pruebas de Concurrencia e Idempotencia (`C04` / Escenario B):**
  - Si el cambio modifica lógica de cupos o inscripciones, incluye pruebas automatizadas con concurrencia simulada verificando que la capacidad máxima jamás sea vulnerada y que las peticiones duplicadas retornen idempotentemente.
- [ ] **3. Cumplimiento de Límites Transaccionales y Auditoría (`C09`):**
  - Toda nueva operación de escritura sobre matrícula incluye su correspondiente evento en `auditoria.trazabilidad` y en `outbox.eventos_sincronizacion` dentro del mismo bloque transaccional.
- [ ] **4. Impacto en Rendimiento (`C02`):**
  - Las consultas añadidas no introducen escaneos secuenciales masivos de tablas (*Sequential Scans* en PostgreSQL); se han validado índices apropiados mediante `EXPLAIN ANALYZE`.
- [ ] **5. Documentación y Diagramas al Día (Docs-as-Code):**
  - Si el cambio altera interfaces de servicio, modelos de dominio o componentes, se actualizó la documentación en `/docs/` y el diagrama correspondiente.
- [ ] **6. Aprobaciones Humanas Obligatorias:**
  - Aprobación de mínimo un (1) desarrollador par.
  - Aprobación explícita del Líder Técnico / Arquitecto en caso de tocar contratos de interfaces, esquemas de BD o configuraciones de Nginx/PostgreSQL.

### Motivos Inmediatos de Rechazo Automático:
* Fallo en el paso de verificación de reglas de arquitectura de CI.
* Llamadas directas al Sistema Legado evadiendo el adaptador ACL.
* Consultas SQL con `JOIN` entre esquemas de diferentes módulos.
* Omisión de la cabecera `Idempotency-Key` en endpoints de mutación de matrícula.
