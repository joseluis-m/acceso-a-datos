## Ejercicio 4

En este ejercicio, a diferencia del acceso secuencial, vamos a ver el acceso aleatorio.
Este tipo de acceso nos permite saltar a posiciones concretas, ideal para registros de tamaño fijo, índices, cabeceras, etc.
Para nuestro ejemplo, creamos un fichero de registros fijos de 4 bytes (enteros contiguos) para ver el funcionamiento de la clase `RandomAccessFile`.
Esta clase es muy importante porque nos permite leer/escribir en cualquier posición del fichero mediante un puntero interno.

Como siempre, definimos el paquete y los imports que vamos a usar.
Debemos tener el `pom.xml` bien configurado para que nos funcione correctamente la API de logging SLF4J y su backend (en nuesro caso, Logback).

```java
package es.europea.ficheros;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;
import java.io.RandomAccessFile;
import java.nio.file.Files;
import java.nio.file.Path;
```

De momento, lo que viene es análogo a los ejercicios anteriores, ponemos un comentario de clase con JavaDoc e impedimos que la clase sea extensible con `final`.
Establecemos un Logger por clase (estático e inmutable), y una constante `INT_BYTES` para mejorar la claridad y reducir bugs si queremos cambiar el tipo de dato.
Ponemos un constructor vacío en `private` para que la clase no sea instanciable (es un patrón para clases utiliarias).
Nuevamente, el método `main` será nuestro entry point donde orquestamos todo, llamando a los métodos públicos que iremos viendo uno por uno después.
Luego, o bien el usuario pasa por argumento una ruta o, si no pasa nada, by default (es un ejemplo) será `"bin/enteros.bin"`.
Si ocurre algún error, se captura y registra con `log.error(..., e)` (muestra stacktrace) y terminamos con un `System.exit(1)`.

```java
public final class Ex4RandomAccess {

    private static final Logger log = LoggerFactory.getLogger(Ex4RandomAccess.class);
    private static final int INT_BYTES = Integer.BYTES;

    private Ex4RandomAccess() {}

    public static void main(String[] args) {
        Path path = args.length > 0 ? Path.of(args[0])
                : Path.of("bin/enteros.bin");
        try {
            // TODO: implementar
        } catch (IOException e) {
            log.error("Error acceso aleatorio en {}: {}", path, e.toString(), e);
            System.exit(1);
        }
    }

    // TODO: métodos a implementar
}
```

Dentro del bloque `try` en el método `main`, tenemos lo siguiente:

```java
            Files.createDirectories(path.getParent());
            initializeSequentialInts(path, 20);
            int tenth = readIntAtIndex(path, 9);
            log.info("Valor original en índice 9: {}", tenth);
            writeIntAtIndex(path, 9, 77);
            int tenthAfter = readIntAtIndex(path, 9);
            log.info("Valor actualizado en índice 9: {}", tenthAfter);
```

- Recordemos que `Files.createDirectories(path.getParent())` crea el directorio padre si no existe.
  Nota: si el usuario pasa una ruta sin padre (por ejemplo, `"enteros.bin")`, `getParent()` es `null`.
  Antes de crear, sería recomendable comprobar `if (path.getParent() != null) ...` para evitar el famoso `NullPointerException`.
- Después, llamamos al método `initializeSequentialInts(path, 20)`, que inicializa el archivo con 20 enteros contiguos (`0..19`).
- Luego, leemos el índice 9 con `readIntAtIndexpath, 9)` (es decir, el 10.º entero) y logueamos su valor.
- `writeIntAtIndex(path, 9, 99)` sobrescribe el entero en índice 9 con el valor `77`.
- Volvemos a leer y verificamos la actualización con el loguer.

Después de la función `main`, vamos a ver cómo funciona cada método que hemos usado en el ejercicio anterior uno por uno:

```java
    public static void initializeSequentialInts(Path file, int count) throws IOException {
        try (RandomAccessFile raf = new RandomAccessFile(file.toFile(), "rw")) {
            raf.setLength(0);
            for (int i = 0; i < count; i++) {
                raf.writeInt(i);
            }
            log.info("Escritos {} enteros en {}", count, file.toAbsolutePath());
        }
    }
```

En primer lugar, tenemos el método `initializeSequentialInts` para crear un fichero de registros fijos.
Abrimos un `RandomAccessFile` en modo `"rw"` (lectura y escritura) con *try-with-resources* (cierre garantizado).
`raf.setLength(0)` sirve para truncar el fichero (limpiarlo) si ya existía.
Después, con un bucle vamos escribiendo en el fichero con `raf.writeInt(i)`. Este método escribe un int (4 bytes).
Después mostramos cuántos cuántos enteros se han escrito y dónde.

Después tenemos el método que permite la lectura aleatoria `readIntAtIndex`:

```java
    public static int readIntAtIndex(Path file, int index) throws IOException {
        try (RandomAccessFile raf = new RandomAccessFile(file.toFile(), "r")) {
            long pos = (long) index * INT_BYTES;
            if (pos >= raf.length()) {
                throw new IOException("Índice fuera de rango: " + index);
            }
            raf.seek(pos);
            return raf.readInt();
        }
    }
```

Ahora, abrimos un `RandomAccessFile` en modo `"r"` (solo lectura), evitando escrituras accidentales.
Después, calculamos la posición con `index * INT_BYTES`, y hacemos casting a un `long` para evitar overflow si el fichero fuese muy grande.
Comprobamos por seguridad si la posición inicial fuese >= longitud del archivo, en cuyo caso el índice qudaría fuera de rango.
Ojo, si nuestro código admitiera huecos o offsets no alineados, tendríamos que hacer comprobaciones adicionales y sería un poco más complejo.
Por último, `raf.seek(pos)` mueve el puntero sin leer bytes previos (esta es la clave del acceso aleatorio).
Devolvemos `raf.readInt()`, que lee 4 bytes desde el puntero (curiosidad, la API usa int Big-Endian comoformato de orden de bytes).

Finalmente, vamos con el último método:

```java
    public static void writeIntAtIndex(Path file, int index, int value) throws IOException {
        try (RandomAccessFile raf = new RandomAccessFile(file.toFile(), "rw")) {
            long pos = (long) index * INT_BYTES;
            if (pos > raf.length()) {
                throw new IOException("Índice fuera de rango: " + index);
            }
            raf.seek(pos);
            raf.writeInt(value);
        }
    }
```

Volvemos a abrir un `RandomAccessFile` con `"rw"` (lectura/escritura).
Calculamos la posición (en long).
En nuestro ejercicio, no permitimos “agujeros” (huecos) en el fichero porque no funcionaría (es un ejemplo básico para ver el funcionamiento de la clase `RandomAccessFile`).
Si `pos > raf.length()`, estaríamos fuera de rango y se lanza excepción con `throw` que capturará el método `main` (desde donde hemos invocado `writeIntAtIndex(...)`.
Si la `pos == length`, es escribe al final extendiendo el fichero.
Por último, se posiciona y escribe el int (sobrescribe 4 bytes).

> Nota: Aunque `RandomAccessFile` permite por defecto escribir más allá del final, y los bytes intermedios se rellenan con ceros, en nuestro ejercicio no se realiza para mantener el fichero denso y sin huecos.

Con esto concluimos el ejercicio 4. Cabe destacar que la razón de ver acceso aleatorio no es “leer más rápido por leer”, sino poder saltar directamente a la parte del fichero que importa y modificar en el mismo lugar sin rehacerlo todo.
En tareas reales como ficheros de registros de tamaño fijo (clientes/productos) o índices auxiliares (`id → offset`), el patrón es siempre el mismo: calcula el offset y haz seek para leer/escribir solo esos bytes. Eso reduce un trabajo `O(n)` (recorrer todo el archivo secuencialmente o reescribirlo) a `O(1)` (ir directo a la posición), ahorrando tiempo, E/S y memoria, especialmente en ficheros grandes o con operaciones muy frecuentes.

Nuestro ejercicio es simple a propósito, pero ilustra bien todo esto: leer/modificar el elemento n sin tocar los anteriores.
Sí, con acceso secuencial podríamos llegar al 10.º entero recorriendo los nueve primeros, y para cambiarlo terminaríamos reescribiendo desde ahí o usando un temporal.
Sin embargo, eso escala mal y complica la consistencia. Con acceso aleatorio, actualizamos en el mismo sitio ese `int` (o un campo concreto de un registro) y listo.
Podemos llevar este mismo patrón a casos típicos: marcar una baja lógica en un registro o actualizar el saldo de un cliente. Ahí es donde el acceso aleatorio se convierte en una herramienta imprescindible.
