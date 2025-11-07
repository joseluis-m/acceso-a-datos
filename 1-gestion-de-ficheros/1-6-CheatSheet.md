# Cheat Sheet
**Paquete:** `es.europea.ficheros` • **Build:** Maven • **Logging:** SLF4J + Logback • **Charset:** UTF-8

## POM (Maven)
**Objetivo:** compilar con JDK 25, logging desacoplado, JSON/CSV con Jackson, build reproducible.

- **Properties**
  - `maven.compiler.release = 25` → genera bytecode para Java 25.
  - `project.build.sourceEncoding = UTF-8` → evita problemas de acentos en compilación/recursos.
  - Versiones centralizadas: `slf4j.version`, `logback.version`.

- **Dependencias**
  - `org.slf4j:slf4j-api` (API) + `ch.qos.logback:logback-classic` (binding). **Regla:** un solo *binding* SLF4J en el classpath.
  - `com.fasterxml.jackson.core:jackson-databind` + `com.fasterxml.jackson.dataformat:jackson-dataformat-csv` para Ej. 5.

- **Plugins**
  - `maven-compiler-plugin` anclado (p. ej. 3.13.0) con `<release>25</release>`.

- **Logging (runtime)**
  - `src/main/resources/logback.xml` (opcional).

### Código `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>es.europea</groupId>
  <artifactId>acceso-datos-ficheros</artifactId>
  <version>1.0.0</version>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.release>25</maven.compiler.release>
    <slf4j.version>2.0.13</slf4j.version>
    <logback.version>1.5.20</logback.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
      <version>${slf4j.version}</version>
    </dependency>
    <dependency>
      <groupId>ch.qos.logback</groupId>
      <artifactId>logback-classic</artifactId>
      <version>${logback.version}</version>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>2.17.2</version>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.dataformat</groupId>
      <artifactId>jackson-dataformat-csv</artifactId>
      <version>2.17.2</version>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.13.0</version>
        <configuration>
          <release>${maven.compiler.release}</release>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

### Código `logback.xml`

```xml
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>
  <root level="INFO">
    <appender-ref ref="STDOUT"/>
  </root>
</configuration>
```

## 1-1-Ex1ReadText: Lectura secuencial segura (texto)
**Qué hace:** lee un fichero **línea a línea** (UTF-8) y lo vuelca a log.

- **APIs clave:** `Path`, `Files.newBufferedReader(path, UTF_8)`, `BufferedReader.readLine()`.
- **Contrato:** requiere que el fichero exista; si no, `IOException` con ruta absoluta.
- **Flujo:**
  1) Validar `Files.exists`.  
  2) Log informativo (ruta absoluta).  
  3) `try-with-resources` con `BufferedReader`.  
  4) Bucle `while ((line = br.readLine()) != null)`.  
  5) Contador y log resumen.
- **Logging:** usar placeholders `{}`; para ficheros grandes, considera mover las líneas a `DEBUG`.
- **Excepciones:** captura en `main`, log + *stacktrace* y `System.exit(1)`.
- **Rendimiento:** `BufferedReader` reduce syscalls; evita `readAllLines` en archivos grandes.
- **Pitfalls & fixes:**  
  - Asegura `public static void main(String[] args)` si ejecutamos con `java ...`.

> Nota: Aunque JDK 25 ya no lo exige, mantenemos `public` para asegurar compatibilidad con JDK anteriores y herramientas, y por claridad para todo el equipo.

## 1-2-Ex2WriteText: Escritura robusta (texto)

**Qué hace:** escribe líneas en UTF-8 creando carpetas si no existen y **truncando** si ya existían.

- **APIs clave:** `Files.createDirectories(parent)`,  
  `Files.newBufferedWriter(path, UTF_8, CREATE, TRUNCATE_EXISTING, WRITE)`,  
  `BufferedWriter.newLine()`.
- **Flujo:**
  1) `Path parent = file.getParent(); if (parent != null) createDirectories(parent);`  
  2) Log informativo (`lines.length` + ruta).  
  3) `try-with-resources` con `BufferedWriter` y opciones explícitas.  
  4) Escribir cada línea + `newLine()`.  
  5) Log de cierre.
- **Modos comunes:**  
  - Sobrescribir: `CREATE` + `TRUNCATE_EXISTING` + `WRITE`.  
  - Append: `CREATE` + `APPEND` + `WRITE`.  
  - No pisar: `CREATE_NEW` + `WRITE`.
- **Portabilidad:** `newLine()` usa separador del SO.

## 1-3-Ex3DirOps: Directorios, filtros y metadatos (NIO.2)

**Qué hace:** lista un directorio, muestra tipo/tamaño/última modificación y filtra `*.txt`.

- **APIs clave:**  
  `Files.exists/isDirectory`, `DirectoryStream<Path>`,  
  `Files.readAttributes(p, BasicFileAttributes.class)`,  
  `DateTimeFormatter`, `ZoneId`.
- **Flujo:**  
  1) Guardas: existe y es directorio (si no, `log.warn` y `return`).  
  2) `DirectoryStream` (lazy) para no cargar todo en memoria.  
  3) Atributos: `isDirectory`, `isRegularFile`, `size`, `lastModifiedTime`.  
  4) Formato de fecha: `Instant → atZone(ZoneId.systemDefault()) → formatter`.  
  5) Segundo `DirectoryStream` con *glob* `*.txt`.

## 1-4-Ex4RandomAccess: Acceso aleatorio (binario, registros fijos)

**Qué hace:** gestiona un fichero binario de **enteros (4 bytes)** contiguos con lectura/escritura por índice.

- **APIs clave:** `RandomAccessFile("r"/"rw")`, `seek(long)`, `readInt()`, `writeInt(int)`, `setLength(0)`.
- **Contrato del fichero:** registros de **tamaño fijo** `INT_BYTES = Integer.BYTES (4)`, **Big-Endian**.
- **Flujo:**  
  - **init:** truncar (`setLength(0)`) + escribir `0..count-1`.  
  - **read:** `pos = index * INT_BYTES` (usar `long`) → rango válido → `seek` → `readInt`.  
  - **write:** misma `pos`.

## 1-5-Ex5CsvToJson: Conversión CSV → JSON (Jackson)

**Qué hace:** lee un CSV con **cabecera** y lo convierte en un **array JSON** de objetos tipados.

- **Modelo:** `record Persona(String name, int edad, String ciudad)`  
  con `@JsonProperty` para mapear nombres de columna. En nuestro caso, CSV esperado: `name,age,city`.
- **APIs clave:**  
  `CsvMapper`, `CsvSchema.emptySchema().withHeader()`,  
  `readerFor(Persona.class).with(schema).readValues(Reader)`,  
  `ObjectMapper.writerWithDefaultPrettyPrinter().writeValue(File, data)`.
- **Flujo:**  
  1) Validar existencia del CSV (`NoSuchFileException`).  
  2) Crear directorio del JSON si procede.  
  3) `CsvMapper` + `CsvSchema.withHeader` ⇒ mapea cabeceras a campos.  
  4) `MappingIterator<Persona>` ⇒ `readAll()` a `List<Persona>`.  
  5) `ObjectMapper` ⇒ escribir JSON **pretty** (UTF-8 por defecto).
- **Nullables:** si una columna puede venir vacía, usa `Integer` en lugar de `int` y valida antes.
- **Streaming (opcional para CSV grandes):** procesar fila a fila en vez de `readAll()`.

## Reglas transversales de buenas prácticas

- **main ejecutable:** usa `public static void main(String[] args)` si vas a ejecutar con `java ...`.
- **UTF-8 explícito** en lectura/escritura y en `pom.xml`.
- **try-with-resources** en toda operación IO (cierre garantizado).
- **Logs** con SLF4J (placeholders `{}` y excepción como **último** argumento para *stacktrace*).
- **Mensajes útiles:** rutas absolutas en errores y logs de intención (qué, dónde, cuántos).
- **Evita números mágicos:** usa constantes (p. ej., `INT_BYTES`).
- **Validaciones** claras (existencia, tipos, rangos) y **fail-fast** con mensajes precisos.
- **Portabilidad:** separadores de línea (`newLine()`), glob vs regex, *locale* en formatos (`DateTimeFormatter`).
- **Calidad (opcionales):** `maven-enforcer-plugin` (JDK/Maven mínimos), `spotless` (formato), `spotbugs`/`errorprone` (análisis estático).

## Smoke tests rápidos (manual)

- **Ex1**  
  `entrada.txt` con 3 líneas → run → INFO de 3 líneas + total.
- **Ex2**  
  run sin args → crea `out/salida.txt`; re-run → truncado y reescrito.
- **Ex3**  
  directorio con `archivo1.txt`, `archivo2.log` → run `.` o `datos` → listado + bloque `*.txt`.
- **Ex4**  
  run → log `9` y luego `99` en índice 9; prueba `readIntAtIndex(..., 100)` → `IOException`.
- **Ex5**  
  `datos/datos.csv`:
  ```text
  name,age,city
  Alice,25,Madrid
  Bob,30,Barcelona
  ```
  run → `out/salida.json` con 2 objetos “pretty”.
