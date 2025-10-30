## Ejercicio 3
Ahora, vamos a ver diferentes operaciones con directorios y archivos usando NIO.2.
Como de costumbre, al principio ponemos el paquete, que indica la organización del código por “módulos” lógicos.
Después, la API de logging SLF4J por un lado, y por otro la implementación (Logback, Log4j2), que puede ser cambiada sin tocar el código.

```java
package es.europea.ficheros;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
```

Luego importamos importamos la librería `IOException` para manejar excepciones de `NIO/IO`.
`BasicFileAttributes` se usa para metadatos portables (tamaño, timestamps, flags).
`java.time` es la API para manejar fechas (nos interesa la zona horaria y el formateador de fechas).

```java
import java.io.IOException;
import java.nio.file.*;
import java.nio.file.attribute.BasicFileAttributes;
import java.time.ZoneId;
import java.time.format.DateTimeFormatter;
```

Luego volvemos a tener la declaración de la clase. Incluimos JavaDoc (qué hace y cómo).
Después, como en ejercicios anteriores, declaramos un logger de SLF4J privado, estático e inmutable llamado log, obtenido con `LoggerFactory` para la categoría `Ex3DirOps.class`, y se usará para registrar mensajes de esa clase.
Incluimos también un `DateTimeFormatter` precompilado, definiendo un patrón claro: `"yyyy-MM-dd HH:mm:ss"`.
Estas variables deben ser únicas y compartidas por toda la clase (`static`) y su referencia no debe cambiar nunca (`final`), dejando claro que es constante y evitando errores.
También es una buena práctica definir un constructor privado (`private`) vacío que impide crear instancias (y, al ser la clase `final`, también la herencia), dejando claro que la clase solo ofrece utilidades estáticas

```java
/**
 * Ejercicio 3: Operaciones con directorios y archivos usando NIO.2 (listar, filtrar, metadatos).
 */
public final class Ex3DirOps {
    private static final Logger log = LoggerFactory.getLogger(Ex3DirOps.class);
    private static final DateTimeFormatter TS_FMT = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

    private Ex3DirOps() {}

    // TODO: implementar
}
```

Nuevamente, volvemos a tener la función `main`. Este entry point es recomendable que sea conciso delegando la lógica al método público que definiremos después.
Recordemos, si se pasa argumento → `Path.of(args[0])`. Si no → por defecto el directorio será `"datos"`. Es un patrón típico: sensible defaults + override por CLI.
Después, se llama al método público `listDirectory(dir)` con la funcionalidad implementada.
Con `log.error` se registran los placeholders y al final como último parámetro el stacktrace.
Luego finaliza el programa con un código de error (`System.exit(1)`).

```java
    public static void main(String[] args) {
        Path dir = args.length > 0 ? Path.of(args[0])
                : Path.of("datos");
        try {
            listDirectory(dir);
        } catch (IOException e) {
            log.error("Error listando {}: {}", dir, e.toString(), e);
            System.exit(1);
        }
    }
```

Después de establecer la parte de orquestación desde el `main`, ahora vamos a ver la lógica de negocio ya implementada:

```java
    /**
     * Lista el contenido de un directorio con tamaño, tipo y fecha de modificación, filtrando además por .txt.
     */
    public static void listDirectory(Path dir) throws IOException {
        if (!Files.exists(dir)) {
            log.warn("El directorio no existe: {}", dir.toAbsolutePath());
            return;
        }
        if (!Files.isDirectory(dir)) {
            log.warn("La ruta no es un directorio: {}", dir.toAbsolutePath());
            return;
        }

        log.info("Contenido de {}", dir.toAbsolutePath());

        // TODO: implementar
    }
```

Se usa `throws IOException` para ver propagar el error (una decisión de diseño). Esto facilita realizar pruebas unitarias (sin atrapar todo aquí).
En primer lugar, con `Files.exists(dir)` si el directorio no existe, loguea warning y sale.
Luego, con `Files.isDirectory(dir)` se ve si es un directorio, como no lo sea se lanza warning y retorno a la función `main`.
Ponemos un log para observar cuál es la ruta absoluta de la variable.

Ahora vamos con el punto clave:

```java
        try (DirectoryStream<Path> stream = Files.newDirectoryStream(dir)) {
            for (Path p : stream) {
                BasicFileAttributes attrs = Files.readAttributes(p, BasicFileAttributes.class);
                String type = attrs.isDirectory() ? "DIR " : (attrs.isRegularFile() ? "FILE" : "OTRO");
                long size = attrs.size();
                String mod = attrs.lastModifiedTime().toInstant().atZone(ZoneId.systemDefault()).format(TS_FMT);
                log.info("{}  {} bytes  {}  {}", type, size, mod, p.getFileName());
            }
        }
```

Usamos `try-with-resources` para gestionar automáticamente el cierre de recursos, como archivos y conexiones, sin necesidad de usar un bloque `finally`.
Declaramos `DirectoryStream` que itera sobre entradas del directorio sin cargarlas todas en memoria (útil para directorios grandes).
Ojo, no recorre recursivamente subdirectorios.
Usamos `Path` que es la representación moderna de una ruta de archivo/carpeta (no un String) para recorrer cada entrada (variable `p`, que es el `Path` de cada entrada, archivo o subcarpeta).

A continuación, cargamos los metadatos de cada entrada con `Files.readAttributes(...)`. es un API uniforme en Windows/Linux/macOS.

- Comprobamos a partir de los atributos si la entrada es un directorio (`"DIR"`) con `isDirectory()`. Si no lo es, comprobamos si es un archivo regular (`"FILE"`) con `isRegularFile()`. Si no es ninguna de las anteriores (`“OTRO”`), podría ser un symlink (enlace simbólico, como un puntero que apunta a otra ruta), socket (archivo especial para comunicación entre procesos), fifo (tubería para pasar datos de un proceso a otro en orden)…
- El método `attrs.size()` devuelve el tamaño en bytes del objeto de fichero consultado.
- Paso a paso, veamos como obtener el timestamp:
  - `lastModifiedTime()` nos da un `FileTime`.
  - `toInstant()` lo convierte a `Instant` (tiempo “UTC puro”).
  - `atZone(ZoneId.systemDefault())` aplica nuestra zona horaria local (la del SO) y nos devuelve un `ZonedDateTime`.
  - `format(TS_FMT)` imprime el `String` según el patrón que hemos especificado anteriormente (`"yyyy-MM-dd HH:mm:ss"`).
- Con toda esta información, usamos `log.info(...)` para mostrar por cada entrada: su tipo, tamaño, timestamp y nombre.

Después, tenemos un ejemplo para filtrar archivos con extensión `.txt`:

```java
        // Ejemplo de filtrado por extensión .txt
        try (DirectoryStream<Path> txts = Files.newDirectoryStream(dir, "*.txt")) {
            log.info("Archivos .txt en {}:", dir.getFileName());
            for (Path p : txts) {
                log.info(" - {}", p.getFileName());
            }
        }
```

Se hace un filtro usando el patrón *glob* (diferente a regex), en el que se usan comodines (como "*") para encontrar archivos. En nuestro ejemplo, con `"*.txt"` buscamos cualquier nombre de archivo que termine en *".txt"* (solo en este directorio, no es recursivo).
Escribimos un log con información sobre la ruta del archivo.
Análogamente a como hemos realizado anteriormente, iterameos sobre cada `Path` que hay en `DirectoryStream`.
Por último, con un log de información mostramos solo las entradas que contienen `"*.txt"` después de haber aplicado el glob.

Con esto completado, tenemos completado el ejercicio 3. Solo queda probar que funcione correctamente en IntelliJ.
