# Módulo Profesional: Acceso a Datos (2.º DAM)

Repositorio de ejercicios y material de apoyo para el **ciclo formativo de Grado Superior en Desarrollo de Aplicaciones Multiplataforma**.
Todo el código sigue **buenas prácticas actuales de mercado** y unifica estilo, herramientas y procesos.

## Stack y convenciones comunes
- **Java JDK 25**, **Maven 3.9+**, **IntelliJ IDEA** (SDK del proyecto apuntando a JDK 25).
- E/S moderna con **NIO.2** (`Path`/`Files`) y **UTF-8** explícito.
- Gestión de recursos con **try-with-resources**, validaciones *upfront* y mensajes de error trazables.
- **Logging profesional** con **SLF4J** (`2.0.13`) + **Logback** (`1.5.20`) — *un solo binding en classpath*.
- Diseño testeable: lógica en métodos públicos; el `main` debe ser `public static void main(String[] args)`.
- Evitar “números mágicos”: usar **constantes semánticas** (p. ej., `Integer.BYTES`).
- Documentar **contratos**: formato, *endianness*, rangos válidos, opciones de apertura, rutas de E/S.

## Estructura mínima
- `pom.xml`
- `src/main/java/...`  — código por ejercicios (Ex1..ExN) y utilidades compartidas  
- `src/main/resources/` — `logback.xml` y otros recursos  
- `src/test/java/...`  — *opcional*, pruebas JUnit 5

## POM (claves)
- `project.build.sourceEncoding = UTF-8`
- `maven.compiler.release = 25`
- Dependencias base: **`slf4j-api`**, **`logback-classic`**
- Según necesidad: **`jackson-databind`**, **`jackson-dataformat-csv`**
- *Opcional (tests)*: **`junit-jupiter`** + **`maven-surefire-plugin`**

## Logging
- Fichero: `src/main/resources/logback.xml`
- Nivel por defecto **INFO**; elevar a **DEBUG** para trazas detalladas en lectura/escritura.

## Construcción y ejecución
- Compilar y resolver dependencias de *runtime*: `mvn -DskipTests package dependency:copy-dependencies -DincludeScope=runtime`
- Ejecutar (Linux/macOS): `java -cp "target/classes:target/dependency/*" <paquete.ClaseMain> [args...]`
- Ejecutar (Windows PowerShell): `java -cp "target\classes;target\dependency\*" <paquete.ClaseMain> [args...]`

## Buenas prácticas transversales
- Validar existencia, permisos y tipos (directorio/archivo) **antes** de operar.
- Separar **intención** (logs de inicio/fin) del **detalle** (logs por elemento/línea).
- Usar patrones de fecha/hora explícitos y, si aplica, `Locale` fijo para consistencia.
- Incluir *hardening* cuando proceda: escritura atómica, `NOFOLLOW_LINKS`, sincronización a disco.
- Mantener README y comentarios **breves, precisos y actualizados**.
