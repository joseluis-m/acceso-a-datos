## Ejemplo 2: Escritura en un fichero de texto

Después de haber realizado el ejercicio anterior, ahora vamos a ver cómo escribir de forma robusta en un fichero de texto. Crearemos directorios y configuraremos algunas opciones explícitas que veremos a continuación.

Nos creamos una nueva clase para este ejercicio, a modo de ejemplo la llamaremos `Ex2WriteText`

Inicialmente, teníamos que ubicar la clase en el namespace del módulo (es importante mantener cierta coherencia con la estructuura de carpetas que tengamos). En nuestro ejemplo:

```java
package es.europea.ficheros;
```

Después, volvemos a importar SLF4J como API de logging desacoplada (igual que en el ejemplo anterior). Esto permite cambiar la implementación (Logback) sin tocar el código.

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
```

Luego tenemos que importar algunas bibliotecas de E/S:

```java
import java.io.BufferedWriter;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardOpenOption;
```

`BufferedWriter` lo usaremos para escritura bufferizada (menos syscalls, mejor rendimiento).
`IOException` para poder manejar errores de E/S.
`StandardCharsets.UTF_8` es la codificación explícita (evitando sorpresas por ejecutar el código entre diferentes SO o contenedores).
`Files` y `Path` vienen de NIO.2, la API recomendada actualmente.
`StandardOpenOption` nos servirá para especificar el modo de apertura explícito (CREATE/TRUNCATE/WRITE/APPEND…).

Con todo esto ya en nuestro proyecto, podemos comenzar a codificar. Es recomendable usar JavaDoc de forma concisa y resaltando lo más importante. Usamos `final` ya que esta clase no se puede heredar, es simplemente utilitaria y diseñada para ser ejecutada.

```java
/**
 * Ejercicio 2: Escritura robusta en fichero de texto (UTF-8) con creación
 * de directorios y opciones explícitas (CREATE + TRUNCATE_EXISTING).
 */
public final class Ex2WriteText {
  \\ TODO: implementar
}
```

Nuevamente, añadimos un Logger inmutable por clase. También suele ser buena práctica poner el constructor a `private` para impedir instanciación accidental.

```java
    private static final Logger log = LoggerFactory.getLogger(Ex2WriteText.class);

    private Ex2WriteText() {}
```

Después, nos encontramos la función `main`, desde donde se orquestará todo ya que es nuestro punto de entrada por la CLI. Su contenido debe mantenerse conciso delegar la lógica a otros métodos como buena práctica.

```java
    public static void main(String[] args) {
        Path output = args.length > 0 ? Path.of(args[0]) : Path.of("out/salida.txt");
        String[] lines = {
                "Hola",
                "Estamos en Acceso a Datos",
                "Esto es un ejemplo de escritura segura con codificación UTF-8"
        };
        try {
            writeUtf8Lines(output, lines);
        } catch (IOException e) {
            log.error("Fallo escribiendo {}: {}", output, e.toString(), e);
            System.exit(1);
        }
    }
```

La ruta de salida por defecto nos la indicará el usuario como argumento por la CLI (`args[0]`) o, si no ha indicado nada, por default podemos decir que sea `"out/salida.txt"`, por ejemplo. Hacer esto suele ser un patrón típico (sensible defaults + posibilidad de override).
Desde IntelliJ, es posible establecer los parámetros por la GUI: *Run → Edit Configurations… → Application → Program arguments* y ahí se puede poner el parámetro, por ejemplo `out/copia.txt`.

Después definimos en un array de `String` datos simulados a escribir (en producción, podrían llegar desde dominio o servicios). Se invoca la lógica y capturamos posibles `IOException`. Recordemos que el `log` usa placeholders `{}` y stacktrace como último parámetro (SLF4J), es decir, si el último argumento es una excepción `e` y ya no quedan `{}` por usar, SLF4J la trata de forma especial e imprime el stacktrace completo. A esto le sigue un `System.exit(1)` con código de error útil en scripts e integración continua.

Ahora veamos el método de escritura robusta que tenemos:

```java
    public static void writeUtf8Lines(Path file, String[] lines) throws IOException {
        Path parent = file.getParent();
        if (parent != null) {
            Files.createDirectories(parent);
        }
        log.info("Escribiendo {} líneas en {}", lines.length,
                file.toAbsolutePath());
        // TODO: implementar
    }
```

En primer lugar, establecemos que este método sea público, reutilizable y establecemos que al llamarlo se pueda lanzar la excepción `IOException`.
Después tenemos la preparación de los directorios.
`file.getParent()` puede ser `null` si el path no tiene carpeta (p.ej., `"salida.txt"`). En este caso, `Files.createDirectories(...)` es idempotente, es decir, crea el árbol si falta (no falla si ya existe), permitiendo que al ejecutar múltiples veces el programa se obtenga el mismo resultado, independientemente de que sea la primera vez que ejecutamos el programa. En nuestro ejemplo, `file.getParent()` convirtiéndolo a String contendría `"out"`, ya que `file` es igual a `"out/salida.txt"`. Desués ponemos un log indicando cuántas líneas estamos escribiendo y dónde.

```java
        try (BufferedWriter bw = Files.newBufferedWriter(
                file,
                StandardCharsets.UTF_8,
                StandardOpenOption.CREATE,
                StandardOpenOption.TRUNCATE_EXISTING,
                StandardOpenOption.WRITE)) {
            for (String l : lines) {
                bw.write(l);
                bw.newLine();
            }
        }
        log.info("Escritura completada.");
```

Finalmente, tendríamos un bloque try-with-resources con varias opciones claras:
- `Files.newBufferedWriter(file, StandardCharsets.UTF_8, ...)`: devuelve un `BufferedWriter` con codificación fija.
- `StandardOpenOption.CREATE`: crea el fichero si no existe.
- `StandardOpenOption.TRUNCATE_EXISTING`: si existe, lo vacía (truncar un fichero es dejar su tamaño en 0 bytes).
- `StandardOpenOption.WRITE`: modo escritura (explícito y autoexplicativo).

> Nota: Si quisiéramos solo añadir al final, cambiaríamos `TRUNCATE_EXISTING` por `APPEND`. Si quisiéramos que el programa fallse si ya existe el archivo, usaríamos `CREATE_NEW` en vez de `CREATE`.

Una vez que tenemos nuestro `BufferedWriter` configurado, pasamos al bucle de escritura.
Con el método `bw.write(l)` escribimos cada línea `l` del array `String[] lines` sin separador.
Con `bw.newLine()` añadimos el salto según el SO (Windows `\r\n`, Unix `\n)`.
Este patrón evita mezclar separadores “a mano” y mantiene portabilidad.
Cabe destacar el cierre automático del writer (sin añadir `finally` y dentro `bw.close()`). Al salir del `try` (incluso si salta una excepción dentro), Java llama automáticamente a `bw.close()`. Por tanto, con try-with-resources conseguimos el cierre automático con un `flush` garantizado aunque falle algo durante la escritura. Solo queda poner un Log de éxito.

Con todo esto, ya podemos acceder a datos de un fichero y escribir de forma robusta en ellos, viendo algunas de las clases más importantes que se usan actualmente y explorando algunas de las opciones y patrones más típicos de programación.
