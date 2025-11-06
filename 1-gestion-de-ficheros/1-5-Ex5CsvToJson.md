## Ejercicio 5

Finalmente, vamos a ver el último ejercicio relacionado con la gestión de ficheros.
En el mundo real, a menudo necesitamos convertir datos de un formato a otro. Un caso típico es leer datos de un fichero CSV (valores separados por comas, texto plano) y convertirlos a un formato como JSON, muy usado en aplicaciones web y servicios REST actualmente.
Para esto podemos escribir código manualmente, pero es más habitual usar bibliotecas.
Es por esto que, para este ejercicio, introduciremos la librería Jackson para JSON (también debe estar en Maven).

Como siempre, tenemos el paquete y las bibliotecas que usaremos. Ponemos al principio las que usarán dependencias de Maven, en este caso todas las de Jackson y SLF4J. Después, las que ya hemos usado en anteriores ejercicios.

```java
package es.europea.ficheros;

import com.fasterxml.jackson.annotation.JsonProperty;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.csv.CsvMapper;
import com.fasterxml.jackson.dataformat.csv.CsvSchema;
import com.fasterxml.jackson.databind.MappingIterator;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.NoSuchFileException;
import java.nio.file.Path;
import java.util.List;
```

A continuación, creamos un `record` en Java, que es una forma compacta de definir un portador de datos inmutable. El compilador genera automáticamente constructor, getters y setters, equals, hashCode y toString.
En nuestro ejemplo, los componentes están tipados: `String` para nombre y ciudad, e `int` para edad. Si en el CSV el campo edad viene vacío o con texto, Jackson lanzará un error de deserialización.
`@JsonProperty` le dice a Jackson qué clave JSON debe usar para cada componente del record. De este modo, el mapeo sigue funcionando aunque cambies el nombre del parámetro en Java.
Con JavaDoc documentamos qué es y para qué sirve cada cosa.

> Nota: En producción conviene aislar el modelo de datos en una clase independiente (ejemplo, un POJO) para mejorar la reutilización, el mantenimiento y las pruebas.
> En este ejercicio, por simplicidad, lo declaramos todo en una única clase.

Después, como en anteriores ejercicios, declaramos un logger SLF4J para la clase y definimos un constructor privado para impedir instanciación (clase utilitaria).
En el método `main` resolvemos las rutas de entrada/salida desde los argumentos (defaults: `datos/datos.csv.csv` y `out/salida.json`).
Luego ejecutamos `convertCsvToJson(csv, json)` dentro de un `try` ya que si ocurre una `IOException`, el logger la registra con stack trace y finaliza el proceso con código de error (`System.exit(1)`).

```java
/**
 * Ejercicio 5: Conversión CSV -> JSON con Jackson (CSV module + databind), UTF-8 y logging.
 */
public final class Ex5CsvToJson {

    /**
     * POJO/record para mapear filas CSV -> JSON.
     */
    public static record Persona(
            @JsonProperty("name") String nombre,
            @JsonProperty("age")   int edad,
            @JsonProperty("city") String ciudad
    ) {}

    private static final Logger log = LoggerFactory.getLogger(Ex5CsvToJson.class);

    private Ex5CsvToJson() {}

    public static void main(String[] args) {
        Path csv = args.length > 0 ? Path.of(args[0]) : Path.of("datos/datos.csv");
        Path json = args.length > 1 ? Path.of(args[1]) : Path.of("out/salida.json");
        try {
            convertCsvToJson(csv, json);
        } catch (IOException e) {
            log.error("Error convirtiendo CSV a JSON: {}", e.toString(), e);
            System.exit(1);
        }
    }
}
```

Ahora, vamos directamente a ver cómo funciona el método que implementa la lógica de conversión.
Nuestro método recibe dos rutas NIO y con `throws IOException` el método `main` (que es el que invoca este método) decide cómo tratar los errores de E/S.

```java
    /**
     * Convierte un CSV con cabecera -> JSON (array de objetos) preservando nombres de columnas.
     */
    public static void convertCsvToJson(Path csvPath, Path jsonPath) throws IOException {
        if (!Files.exists(csvPath)) {
            throw new NoSuchFileException("No existe CSV: " + csvPath.toAbsolutePath());
        }
        Path parent = jsonPath.getParent();
        if (parent != null) Files.createDirectories(parent);
        
        CsvMapper csvMapper = new CsvMapper();
        CsvSchema schema = CsvSchema.emptySchema().withHeader();

        // TODO: continúa
    }
```

En primer lugar, validamos. Si el fichero no existe, se falla rápido con una excepción específica de NIO (`NoSuchFileException`) que incluye la ruta absoluta en el mensaje.
Después, obtenemos la obtiene la carpeta contenedora del destino (`parent`). Si no existe, la creamos recursivamente. Así, evitamos `NullPointerException` cuando `jsonPath` no tiene carpeta (por ejemplo, `"salida.json"`).

`CsvMapper` es el parser de Jackson para CSV. Proporciona APIs para mapear filas de texto a objetos Java (POJOs/records) sin tener que parsear “a mano”.
Por último, definimos el esquema `CsvSchema` especificando que la primera línea del CSV trae las cabeceras.
Jackson usará esos nombres de columna para emparejarlos por nombre con los campos de nuestro tipo Java (`nombre`, `edad`, `ciudad`) o lo que sea que marquemos con `@JsonProperty`).
Si la cabecera dice `name,age,city`, cada fila se convertirá en un objeto JSON `{ "nombre": "...", "edad": 30, "ciudad": "..." }`.
Es decir, las claves del JSON coinciden con los nombres de columna del CSV (o con el alias de `@JsonProperty`).

> Nota: Si puedes cambiar a la vez el record y el CSV/JSON, se podría omitir el uso de `@JsonProperty` (en nuestro ejercicio, podríamos omitirlo).
> Sin embargo, en producción no siempre podemos controlar el exterior, por lo que `@JsonProperty` fija los nombres de los campos y permite refactorizar por dentro sin romper integraciones.

```java

        try (var in = Files.newBufferedReader(csvPath, StandardCharsets.UTF_8)) {
            MappingIterator<Persona> it = csvMapper.readerFor(Persona.class).with(schema).readValues(in);

            List<Persona> personas = it.readAll();
            log.info("Filas CSV leídas: {}", personas.size());

            ObjectMapper jsonMapper = new ObjectMapper();
            jsonMapper.writerWithDefaultPrettyPrinter().writeValue(jsonPath.toFile(), personas);

            log.info("JSON generado en: {}", jsonPath.toAbsolutePath());
        }
```

Por último, abrimos el CSV con un `BufferedReader` en `UTF-8` dentro de un *try-with-resources*. Así el stream se cierra siempre (incluso ante excepciones).

> Nota: Java es tipado estático, por lo que `var` solo aplica inferencia local (el tipo sigue siendo el mismo que el de la derecha, `BufferedReader`), así que úsalo cuando el tipo sea obvio. Si buscas claridad o programar contra la interfaz mínima necesaria (es decir, declarar y depender del tipo más genérico que te da justo los métodos que necesitas para tener un código menos acoplado y más flexible), declara el tipo explícito (`Reader`).

Después, le decimos a Jackson CSV: “lee filas del `Reader` (`in`) y mapea cada una al tipo `Persona` usando el schema con cabecera (`schema`)”. El `MappingIterator` permite recorrer el CSV fila a fila (streaming) sin tener que parsear manualmente.

Después, teniendo nuestra instancia de `CsvMapper` que sabe leer CSV y convertir cada fila en objetos Java, hacemos lo siguiente:
- `.readerFor(Persona.class)`: cada fila del CSV, se mapea al tipo Java de `Persona`, que es un `record` con `nombre`, `edad` y `ciudad`.
- `.with(schema)`: adjuntamos un `CsvSchema` que define cómo interpretar el CSV. Como lo creamos con `.withHeader()`, la primera línea es la cabecera. Jackson buscará columnas y las casará por nombre con los parámetros del `record` (o, en nuestro caso, los `@JsonProperty` que hemos definido).
- `.readValues(in)`: le pasamos el `Reader` abierto del fichero (`in`) y nos devuelve un iterador tipado `MappingIterator<Persona>` que lee bajo demanda (lazy), es decir, fila a fila, sin cargar todo el CSV de golpe.

Luego, recorremos todas las filas del iterador (`it.readAll()`) y las cargamos en memoria como una lista (`List<Persona>`). Es simple y cómodo para CSV pequeños/medianos. Para archivos más grandes, podríamos recorrerlo con `while(it.hasNext())` e ir escribiendo en streaming para no usar toda la RAM.
Con el logger indicamos cuántos registros hemos parseado, útil para diagnósticos, métricas y verificación en ejecución (observabilidad básica).

Ahora, instanciamos un mapper JSON “normal” de Jackson con `new ObjectMapper()` (no el de CSV). Este es el responsable de serializar objetos Java (List<Persona>) a JSON.
Serializamos la lista a JSON “bonito” (identado) y se escribe en un fichero (`jsonPath`). Jackson se encarga de abrir/cerrar el stream de salida. Si el fichero existe, lo sobrescribe. Si no existe, lo crea.
Al final se muestra un mensaje de éxito con la ruta absoluta.

> Nota: Serializar es transformar un objeto o estructura de datos en una secuencia de bytes o en un formato intercambiable (p. ej., JSON) para almacenarlo o enviarlo y poder reconstruirlo luego.
> En nuestro ejercicio, `ObjectMapper` convierte `List<Persona>` en JSON con *pretty print*, ese texto se codifica en UTF-8 y se escribe como bytes en el archivo de salida (`jsonPath`).

Con todo esto, nuestro ejercicio está casi completado, solo nos faltaría actualizar el `pom.xml` de Maven.

### pom.xml

Recordemos, Maven es la herramienta de build estándar en Java. Orquesta todo el ciclo de vida del proyecto: compilar, ejecutar tests, empaquetar, gestionar dependencias y plugins. Su fuerza está en la gestión de dependencias y en builds reproducibles.

> Nota: Un build es el proceso (y también el resultado) de transformar código fuente en un artefacto ejecutable o desplegable de forma automatizada. Suele incluir pasos como resolver dependencias, compilar, ejecutar tests y empaquetar (p. ej., JAR/WAR, imagen de Docker).

El `pom.xml` es el descriptor (es decir, el archivo de metadatos que describe un proyecto y cómo construirlo/ejecutarlo) de nuestro proyecto Maven. Define la identidad del artefacto (groupId, artifactId, version), propiedades (por ejemplo, Java 17 y UTF-8), dependencias que el código necesita y plugins (por ejemplo, el compilador). IntelliJ lo lee y configura el proyecto automáticamente.

> Nota: Un artefacto es el resultado empaquetado y versionado de un build que se publica/consume desde un repositorio. En Maven un artefacto tiene coordenadas GAV (groupId, artifactId, version) y tipo (packaging): por ejemplo JAR, WAR, POM, `*-sources.jar`, `*-javadoc.jar`.

Para este ejercicio, necesitamos incluir una nueva dependencia. Cada `<dependency>` declara una librería externa que Maven debe resolver y descargar (normalmente desde Maven Central) y añadir al classpath. La dependencia se identifica por coordenadas (`groupId:artifactId:version`).

> Nota: Maven Central es el repositorio público y estándar de la comunidad Java donde se publican y descargan librerías/artefactos (JARs) identificados por sus coordenadas GAV. Maven lo usa para resolver dependencias de forma automática (con checksums/firmas) y traerlas a tu proyecto.

En el ejerccio 1, añadimos dos dependencias para logging.
- `slf4j-api`: API de logging usada por nuestro código (`Logger`, `LoggerFactory`).
- `logback-classic`: implementación (binding) que realmente escribe los logs.

> Nota: Este patrón (API + implementación) es típico en producción y nos permite cambiar el backend sin tocar el código.

Para el ejercicio 5, añadimos lo siguiente:

```xml
<!-- Jackson JSON + CSV -->
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
```

En primer lugar tenemos `jackson-databind`, el “motor” de Jackson para (de)serializar JSON ↔ POJOs. Nos da la clase clave `ObjectMapper`, con la que convertimos objetos Java a JSON (y al revés), respetando anotaciones como `@JsonProperty`. Incluye la lógica de binding (mapea campos, tipos primitivos/colecciones, fechas, etc.) y trae transitivamente `jackson-core` y `jackson-annotations`, así que no necesitamos declararlas aparte. En este ejercicio lo usamos para escribir el array de `Persona` a JSON (pretty print, etc.).

En segundo lugar tenemos `jackson-dataformat-csv`, un módulo adicional que enseña a Jackson a leer/escribir CSV. Proporciona `CsvMapper` y `CsvSchema` para parsear CSV (cabeceras, separadores, comillas) y mapear cada fila directamente a nuestro POJO/record (`Persona`). Sin este módulo, `ObjectMapper` solo entiende JSON. Con esto, podemos hacer `readerFor(Persona).with(schema).readValues(...)` y obtener un `MappingIterator<Persona>` para procesar filas en streaming (o `readAll()` en nuestro caso si el fichero es pequeño).

Con todo esto, podemos realizar el ejercicio 5 para convertir CSV a una lista de `Persona`, que luego serializamos a JSON con `jackson-databind`.
En IntelliJ, abre *View → Tool Windows → Maven* y pulsa el icono de *“Sync All Maven Projects”*.

> Nota: En IntelliJ para Maven, *Sync All Maven Projects* hace una sincronización incremental, es decir, relee los cambios recientes de los `pom.xml` y actualiza lo mínimo en el modelo del IDE (perfiles, versiones, nuevas dependencias), sin reconstruir todo. Por otra parte, *Reload All Maven Projects* fuerza un reimportado completo. Limpia y vuelve a cargar toda la configuración Maven, re-resuelve dependencias y reconfigura módulos. Usa *Sync* tras cambios pequeños, y *Reload* tras cambios grandes (nuevos módulos, cambio de rama, settings.xml) o si *Sync* no refleja bien los cambios.

