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
> Sin embargo, en producción no siempre podemos controlar el exterior, por lo que @JsonProperty fija los nombres de los campos y permite refactorizar por dentro sin romper integraciones.

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

Finalmente, 
