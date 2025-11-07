# Introducción

Usaremos **Java** (en versión actual LTS) junto con **Maven** como herramienta de construcción y gestión de dependencias.
El IDE recomendado es **IntelliJ IDEA**, muy utilizado en el mercado laboral (de hecho, en 2025 alrededor del 84% de los desarrolladores Java usan IntelliJ IDEA como IDE principal).[^1]
Maven es el build tool más extendido en proyectos Java profesionales, valorado por su simplicidad y amplia comunidad.
La combinación de IntelliJ + Maven te ofrece un entorno robusto similar al que se usa en la industria

## Configuración del proyecto (Java + Maven en IntelliJ)

Antes de escribir código, crearemos un proyecto Java con Maven en IntelliJ IDEA.

1. **Crear nuevo proyecto Maven**: Abre IntelliJ IDEA y selecciona New Project. En la ventana, escoge Java y como *Build System* elige Maven.
   Dale un nombre al proyecto y selecciona la JDK instalada. IntelliJ generará la estructura estándar de un proyecto Maven (carpetas `src/main/java`, `src/test/java`, etc.) y un fichero `pom.xml` básico.
2. **Configurar `pom.xml`**: El `pom.xml` es el descriptor de Maven. Asegúrate de definir el `<groupId>`, `<artifactId>` y `<version>` del proyecto.
   Maven nos permitirá añadir fácilmente librerías externas conforme las necesitemos, manejando automáticamente las dependencias y el classpath. Empezaremos sin dependencias adicionales y las agregaremos en ejemplos posteriores.
3. *Estructura del código*: En `src/main/java` crea un paquete (por ejemplo `es.europea.ficheros`) y dentro una clase principal `Main.java` con un método `main`.
   Iremos implementando nuestros ejemplos en esta clase o en clases adicionales según convenga, usando el método main para ejecutar pruebas simples de cada funcionalidad.

Con todo esto ya tendrías un proyecto Maven configurado en IntelliJ, similar a un entorno profesional. A continuación, abordaremos diferentes ejemplos de manejo de ficheros, incrementando la complejidad gradualmente y aplicando buenas prácticas de programación en Java.

[^1]: JRebel, *Java Developer Productivity Report 2025*. <https://www.jrebel.com/resources/java-developer-productivity-report-2025>

## Ejemplo 1: Lectura secuencial de un fichero de texto

Comenzamos con una tarea fundamental: leer el contenido de un fichero de texto secuencialmente (línea por línea). Usaremos la forma de acceso secuencial, que es la más simple: recorrer el archivo desde el inicio hasta el final.
Supongamos que tenemos un fichero de texto llamado `"entrada.txt"` con varias líneas de texto. Queremos leerlo completo e imprimir su contenido por consola.

Crea un archivo de texto sencillo. Por ejemplo, en la carpeta del proyecto (o en `src/main/resources` para seguir buenas prácticas) guarda un fichero llamado entrada.txt con este contenido:

```text
Linea 1: Hola mundo
Linea 2: Aprendiendo Java
Linea 3: Acceso a datos en ficheros
```

A continuación, vamos a ver poco a poco un ejemplo de código en Java. En primer lugar, tenemos que poner el paquete.
Esto estructura el código en un namespace lógico. En producción, el nombre suele seguir el dominio invertido de la organización. Para este ejemplo, usaré el siguiente:

```java
package com.ejemplo.dam.ficheros;
```

Después tenemos todas las librerías que se van a importar para usar en nuestro proyecto, que son las siguientes:

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.BufferedReader;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
```

- `org.slf4j.*`: **SLF4J** es la API de logging de facto. Desacopla el código de la implementación concreta (Logback, Log4j2…). Permite cambiar el backend sin tocar el código.
- Imports de **NIO.2** (`Path`, `Files`) y clases de E/S: Actualmente, se prefiere NIO.2 sobre `java.io.File` (rutas inmutables, más utilidades, mejor manejo de errores…).
  `StandardCharsets.UTF_8` fija la codificación de caracteres (evitas problemas locales)

Después tenemos la declaración de la clase. Podemos usar comentarios con JavaDoc (qué hace la clase y cómo), útil para IDEs y generación de documentación.
`final` impide herencia accidental. Esta clase es utilitaria + ejecutable; no está pensada para extenderse.

```java
/**
 * Ejercicio 1: Lectura secuencial y segura de un fichero de texto (UTF-8) con NIO.2 y logging.
 */
public final class Ex1ReadText {
    // TODO: implementar
}
```

Dentro de la clase, ponemos el logger y el constructor:

```java
private static final Logger log = LoggerFactory.getLogger(Ex1ReadText.class);

private Ex1ReadText() {}
```

- `private static final Logger log = LoggerFactory.getLogger(...)`: Patrón estándar.
  `static final`: Un único logger por clase, inmutable.
  No se suele usar `System.out.println` porque no tiene niveles, ni formato, ni destinos configurables, ni metadata (clase/hilo/fecha), y es caro y ruidoso en producción.
- Constructor `private` en clase utilitaria: impide crear instancias (y heredar) y deja claro que solo se usa mediante métodos estáticos.

Ahora vendría la función `main` como punto de ejecución:

```java
public static void main(String[] args) {
    Path input = args.length > 0 ? Path.of(args[0]) : Path.of("entrada.txt");
    try {
        readUtf8Lines(input);
    } catch (IOException e) {
        log.error("Fallo leyendo {}: {}", input, e.getMessage(), e);
        System.exit(1);
    }
}
```

Podemos declarar la resolución del Path de entrada. Si pasas un argumento, `Path.of(args[0])`. Si no, por defecto `entrada.txt`.
Luego se llama al método `readUtf8Lines(input)` para mantener un `main` conciso (se orquesta y delega la lógica a un método probado/reutilizable).
En producción nunca se imprime con `System.err`, se usa el logger. Se pasa la excepción como último parámetro para stack trace.
`System.exit(1)` finaliza con código de error (1) para que el sistema operativo o la plataforma de integración continua marquen el paso como fallido.

> Nota: Integración continua es la práctica de integrar código con frecuencia y automatizar compilación y pruebas en cada cambio para detectar fallos temprano y mantener el proyecto siempre listo (por ejemplo, GitHub Actions o Jenkins).

Ahora vamos a ver cómo funciona el método de lectura:

```java
/**
 * Lee un fichero de texto en UTF-8 y vuelca su contenido a LOG (INFO), línea a línea.
 */
public static void readUtf8Lines(Path file) throws IOException {
    if (!Files.exists(file)) {
        throw new IOException("No existe el fichero: " + file.toAbsolutePath());
    }
    log.info("Leyendo fichero: {}", file.toAbsolutePath());
    // TODO: implementar
}
```

En primer lugar, cabe destacar que `throws IOException` indica en la firma del método que puede lanzar una excepción checked (`IOException`), obligando a quien lo invoque a capturarla o propagarla.
Después, hay una precondición explícita donde si el fichero no existe, se lanza la excepción `IOException`. Controlar rápida y claramente estos fallos ayuda a depurar el código.
Con el método `toAbsolutePath()` en el mensaje se evitan ambigüedades de ruta.
Luego añadimos un log informativo con la ruta absoluta antes de abrir el archivo. Esto es importante por motivos de trazabilidad (sabremos qué archivo se intentó leer).

```java
try (BufferedReader br = Files.newBufferedReader(file, StandardCharsets.UTF_8)) {
    String line;
    long count = 0;
    while ((line = br.readLine()) != null) {
        log.info("{}", line);
        count++;
    }
    log.info("Total líneas leídas: {}", count);
}
```

Después entra en juego la sentencia *try-with-resources*, que garantiza el cierre del stream incluso si hay una excepción (sin `finally`).
`BufferedReader` con `UTF-8` explícito lee del SO en bloques grandes y atiende lecturas pequeñas desde un buffer interno (menos syscalls y latencia).
Al fijar la codificación, se garantiza el mismo comportamiento en cualquier máquina o contenedor, evitando mojibake.
Finalmente, comienza la lectura secuencial línea a línea con `readLine()`. Se invoca el método `info()` del logger usando placeholders `{}` (mejor que concatenar).
Si el archivo fuese muy grande, se puede usar `DEBUG` para no llenar todo de logs. Luego se muestra el número de líneas leídas (puede ser útil en logs) y termina el bloque *try-with-resources*, el método y se cierra la clase.

> Nota: El código se queda así (con `log.info`, `log.debug`, `log.error`). Lo que cambia es la configuración en un archivo llamado `logback.xml` (por ejemplo, poniendo `<root level="DEBUG">` para ver los mensajes `DEBUG`).

### `pom.xml`

A continuación, vamos a ver cómo funciona el archivo de configuración central de Maven. De momento, estará adaptado para que el ejercicio que acabamos de realizar pueda funcionar correctamente (lectura de ficheros con logging de SLF4J).
El `pom.xml` (Project Object Model) es el archivo principal de Maven que describe un proyecto Java: define sus coordenadas (groupId, artifactId, version), dependencias, plugins, y la configuración de build (perfiles, repositorios, módulos).
Básicamente, le dice a Maven qué necesita el proyecto y cómo compilarlo, testearlo y empaquetarlo.

Comenzamos con la cabecera del POM:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
```

La primera línea es la declaración XML, con su versión y codificación del archivo. En la etiqueta `project`, declaramos lo siguiente:
- `xmlns` es el namespace por defecto, evita choques de nombres diciendo que todos los elementos (project, dependencies, etc.) pertenecen al vocabulario POM 4.0.0 de Maven.
- `xmlns:xsi` define el prefijo `xsi` para usar atributos estándar de validación de XML Schema.
- `xsi:schemaLocation` indica dódne validar: la primera parte (namespace) identifica el vocabulario y la segunda (URL) apunta al fichero XSD que valida la estructura.
- La etiqueta `modelVersion` identifica la versión del modelo POM (no de la app).

Después hay que poner las coordenadas del artefacto:

```xml
  <groupId>es.europea</groupId>
  <artifactId>acceso-datos-ficheros</artifactId>
  <version>1.0.0</version>
```

Aquí, indicamos: con `groupId`, el identificador de la organización/proyecto (dominio invertido recomendado);
con `artifactId`, el nombre concreto del módulo (jar final), el cual debe ser estable y legible;
y con `version`, la versión semántica del artefacto (en nuestro ejemplo, con poner 1.0.0 es suficiente).

> Nota: En Maven, un artefacto es el archivo publicable que produce o usa un proyecto, identificado de forma única por sus coordenadas `groupId:artifactId:version`. Ejemplos: JAR (librería o ejecutable), WAR, POM, BOM.

Ahora veremos algunas propiedades clave para el ejercicio que hemos realizado anteriormente:

```xml
  <properties>                                                               
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.release>25</maven.compiler.release>
    <slf4j.version>2.0.13</slf4j.version>
    <logback.version>1.5.6</logback.version>
  </properties>
```

- `<project.build.sourceEncoding>` asegura que todas las fuentes y recursos se compilan y empaquetan como UTF-8. Evita mojibake y problemas de tildes o caracteres como la 'ñ' entre máquinas.
- `<maven.compiler.release>` establece Java 25 como release level (mejor que source/target porque no solo fija sintaxis y bytecode, sino que además bloquea el uso de APIs de JDK que no existen en la versión objetivo). Genera bytecode compatible con el runtime 25 y usa el JDK Platform Module System apropiado.
- `<slf4j.version>` y `<logback.version>` establece las versiones de logging. Se evita mismatch entre API (SLF4J) e implementación (binding Logback).

Ahora tenemos que introducir algunas dependencias mínimas para nuestro ejercicio. `slf4j-api` es la API de logging que importamos en el código (`Logger`, `LoggerFactory`). Cabe destacar que `${slf4j.version}` es un placeholder de Maven que se sustituye en tiempo de build por la versión que hemos definido antes en `<properties>` (en nuestro caso, la 2.0.13). Esto es importante porque se declara la versión una sola vez, y luego se reutiliza en todo el proyecto evitando desajustes entre módulos.

```xml
  <dependencies>
    <!-- API de logging (SLF4J) -->
    <dependency>                                                             
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
      <version>${slf4j.version}</version>
    </dependency>
```

Sin embargo, sin un “binding” en el classpath, SLF4J no sabe dónde escribir (consola, fichero, etc.).
Por tanto, es necesaria la implementación binding de SLF4J, que será el backend real que sí escribe los logs.
Algunas opciones típicas son Logback, SLF4J Simple o Log4j2. Nosotros vamos a usar `logback-classic` para proveer el backend real.
El binding busca un archivo `logback.xml` en classpath y, si no está (como en nuestro ejercicio), aplica una configuración por defecto (normalmente salida a consola). El `logback.xml` solo hace falta si quieres controlar nivel, patrón, destino, etc.

En nuestro ejercicio, la clase `Ex1ReadText` usa `LoggerFactory.getLogger(...)` que viene `slf4j-api`.
Que se imprima algo depende de `logback-classic` + el `logback.xml` (nivel, patrón, destino).
Recordemos que `<groupID>` y `<artifactID>` son parte de la identidad del artefacto (biblioteca, app, plugin, etc.).
Junto con `<version>`, identifican unívocamente un JAR (o WAR, POM, etc.): `groupId:artifactId:version`.

> Nota: Un *JAR* (Java ARchive) es un archivo ZIP con extensión `.jar` que empaqueta clases compiladas (`.class`), recursos (propiedades, imágenes, XML, etc.) y un `META-INF/MANIFEST.MF` con metadatos.

```xml
    <!-- Implementación de logging (Logback: binding de SLF4J) -->
    <dependency>
      <groupId>ch.qos.logback</groupId>
      <artifactId>logback-classic</artifactId>
      <version>${logback.version}</version>
    </dependency>
  </dependencies>
```

Por último, solo nos queda poner el build y el plugin de compilación:

```xml
  <build>
    <plugins>
      <!-- Compilación moderna: usa 'release' para target multiplataforma -->
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

En `<build>` configuramos el proceso de construcción del proyecto mediante `<plugins>` (contenedor que agrupa todos los `<plugin>`).
`maven-compiler-plugin` se encarga de compilar el código apuntando a Java 25 (`<release>25</release>`, ya que `${maven.compiler.release}` es un placeholder que se sustituye en tiempo de build). Por tanto, exige compilar con un JDK 25 (javac 25), generando un `.class` de Java 25 (si lo ejecutas con 17/21, fallará con *“Unsupported class file major version 69”*).

**Importante**: Cada vez que se modifique el pom.xml, hay que recargar los proyectos de Maven. En IntelliJ, haz click en *“Reload All Maven Projects”*.

> Nota: Al darle a *Run* en IntelliJ, el IDE lee el `pom.xml`, resuelve y descarga los artefactos (los JARs de dependencias como `slf4j-api` y `logback-classic`) al repo local y los pone en el classpath; luego invoca `javac` del JDK 25 (debes tenerlo como Project SDK para que cuadre con `<release>25</release>`) y compila los `.java` a bytecode `.class` en `target/classes` (formato de clase de Java 25), tras lo cual la JVM ejecuta el `main` de `es.europea.ficheros.Ex1ReadText` cargando las clases y las de los artefactos del classpath. Durante la ejecución, SLF4J envía los logs a Logback, que imprime en consola con su configuración por defecto (aunque no exista `logback.xml`).

En conclusión, con este ejercicio hemos montado un entorno de trabajo realista (Java LTS + Maven + IntelliJ), configurado un pom.xml mínimo y coherente (UTF-8, maven-compiler-plugin con release, SLF4J + Logback) y construido una utilidad que lee un fichero línea a línea con NIO.2, usando try-with-resources, comprobando precondiciones y registrando trazabilidad con el logger.
