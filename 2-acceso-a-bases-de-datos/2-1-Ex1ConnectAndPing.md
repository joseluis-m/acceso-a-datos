# Introducción a JDBC y elección de la base de datos

**Java Database Connectivity (JDBC)** es la API estándar de Java para interactuar con bases de datos relacionales.
JDBC proporciona una interfaz unificada para trabajar con cualquier **sistema gestor de bases de datos relacional (SGBD)** desde Java.
Independientemente de que usemos MySQL, Oracle, PostgreSQL, etc., el código JDBC es prácticamente el mismo.
Solo necesitamos incluir el driver (conector) específico del motor de base de datos que vayamos a usar y cambiar la URL de conexión y credenciales según corresponda.
Gracias a este enfoque, nuestro programa Java no queda fuertemente ligado a un solo motor de base de datos, ya que podríamos reutilizar el código con otro SGBD simplemente cambiando el conector y la cadena de conexión (por ejemplo, de MySQL a PostgreSQL).

## ¿Qué es un conector JDBC?

Es una biblioteca (normalmente un `.jar`) que implementa el protocolo de comunicación específico de un SGBD.
Actúa como puente entre la aplicación Java y la base de datos.
En la actualidad, casi todos los SGBD disponen de un driver JDBC tipo 4 (100% Java), también llamado *thin driver*, que se incluye en la aplicación sin necesidad de instalar software nativo del lado del cliente.
Estos drivers puros en Java ofrecen buena portabilidad y rendimiento.

> Nota: Antiguamente existían otros tipos de drivers JDBC, como el tipo 1 que usaba un puente ODBC, pero tenían inconvenientes de rendimiento y hoy en día están en desuso en entornos de producción.

## Ventajas de usar conectores JDBC

Permiten a nuestra aplicación Java comunicarse de forma estándar con la base de datos, aprovechando una API común independientemente del proveedor.
Esto simplifica el desarrollo y mantenimiento, y evita tener que usar soluciones propietarias.
Además, los drivers JDBC modernos son eficientes y están optimizados para cada motor.
Por ejemplo, MySQL Connector/J es el conector JDBC oficial de MySQL (un driver de tipo 4) que implementa el protocolo de MySQL en Java.
Usando este conector, podemos conectar Java con MySQL de forma directa.
En general, los conectores JDBC ofrecen alto rendimiento, portabilidad y acceso completo a funcionalidades SQL del SGBD.

## Inconvenientes o consideraciones

Al usar JDBC directamente, el desarrollador debe manejar manualmente detalles como la gestión de conexiones, construcción de consultas SQL, manejo de errores SQL, etc.
Además, es fundamental incluir la versión correcta del driver JDBC en el proyecto, ya que un conector desactualizado o incompatible puede causar errores de conexión.
Por otra parte, si la aplicación debe soportar distintos SGBD, hay que tener un conector por cada uno y quizás cierta lógica para cada URL de conexión.
Sin embargo, en la práctica esto no suele ser un gran problema gracias a la estandarización de JDBC.

## Bases de datos embebidas e independientes

Un punto importante del acceso a datos es distinguir entre bases de datos embebidas (integradas) y bases de datos independientes (servidor externo).
Una base de datos embebida es aquella que funciona dentro de la propia aplicación, sin un proceso separado.
Algunas de sus ventajas son: no requieren instalar/configurar un servidor aparte, arrancan y se apagan junto con la aplicación, y suelen ser ligeras.
Además, son útiles para desarrollo, pequeñas aplicaciones de escritorio o pruebas. Ejemplos populares son H2, HSQLDB o SQLite.

En cambio, una base de datos independiente (como MySQL, Oracle, PostgreSQL, etc.) corre en un servicio separado (local o remoto) y la aplicación se conecta a través de la red (TCP/IP).
Estas permiten manejar grandes volúmenes de datos, múltiples usuarios concurrentes y compartición de datos entre varias aplicaciones.
En entornos empresariales es más común usar bases de datos independientes/servidor, ya que ofrecen mayor capacidad y escalabilidad, mientras que las embebidas se usan para testing o aplicaciones monousuario.
En los ejercicios que vamos a ver a continuación, usaremos MySQL como base de datos de ejemplo por ser ampliamente utilizada en la industria.
No obstante, veremos que el código JDBC sería muy similar si optáramos por otro SGBD (solo cambiaría el conector y la URL).

# Ejercicio 1

En este ejercicio, vamos a comprobar que podemos conectarnos a la BD usando un pool de conexiones (HikariCP) y el conector idóneo (H2 embebida o MySQL) y validar la conexión con un ping SQL (SELECT 1). Esto cubre:

> Nota: HikariCP es un pool de conexiones JDBC que mantiene algunas conexiones (TCP/IP con puertos) a la base de datos abiertas y las reutiliza para que tu aplicación sea más rápida y eficiente

## `pom.xml`

Antes de comenzar, vamos a ver cómo tenemos actualmente el archivo `pom.xml` con todas las dependencias configuradas para que funcione nuestro ejercicio (estamos usando JDK 25).

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>es.europea</groupId>
  <artifactId>acceso-a-datos</artifactId>
  <version>1.0.0</version>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.release>25</maven.compiler.release>
    <!-- Versiones centralizadas -->
    <slf4j.version>2.0.13</slf4j.version>
    <logback.version>1.5.20</logback.version>
    <jackson.version>2.17.2</jackson.version>
    <hikari.version>5.1.0</hikari.version>
    <mysql.connector.version>9.5.0</mysql.connector.version>
    <h2.version>2.2.224</h2.version>
  </properties>

  <dependencies>
    <!-- Logging -->
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

    <!-- Jackson JSON + CSV -->
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>${jackson.version}</version>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.dataformat</groupId>
      <artifactId>jackson-dataformat-csv</artifactId>
      <version>${jackson.version}</version>
    </dependency>

    <!-- Pool de conexiones -->
    <dependency>
      <groupId>com.zaxxer</groupId>
      <artifactId>HikariCP</artifactId>
      <version>${hikari.version}</version>
    </dependency>

    <!-- Conectores JDBC -->
    <dependency>
      <groupId>com.mysql</groupId>
      <artifactId>mysql-connector-j</artifactId>
      <version>${mysql.connector.version}</version>
      <scope>runtime</scope>
      <exclusions>
        <exclusion>
          <groupId>com.google.protobuf</groupId>
          <artifactId>protobuf-java</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
    <dependency>
      <groupId>com.h2database</groupId>
      <artifactId>h2</artifactId>
      <version>${h2.version}</version>
      <scope>runtime</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <!-- Compilador -->
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

## Código Java

Al inicio, tenemos el paquete donde está este ejercicio (es recomendable mantener una estructura por capas, en nuestro caso:
```java
package es.europea.jdbc;
```

Después, importamos `DatabaseType` para poder decidir si usar H2 o MySQL mediante una variable de entorno `DB_KIND`.
`DataSourceFactory` encapsula la creación del HikariDataSource (pool de conexiones) y la URL/driver adecuados (es decir, se aisla en una sola clase toda la configuración del pool/driver).

```java
import es.europea.jdbc.config.DataSourceFactory;
import es.europea.jdbc.config.DatabaseType;
```

SLF4J como API de logging, en runtime lo implementa Logback.
Se desacopla para poder cambiar si queremos la implementación sin tocar el código.

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
```

Ahora las interfaces JDBC estándar.
No dependemos de clases propietarias de un motor de BD, usamos lo siguiente:
- `DataSource`: punto de entrada recomendado y apto para pooling.
- `Connection`: sesión con la BD.
- `DatabaseMetaData`: metadatos de motor/driver (útil para diagnóstico).
- `Statement` y `ResultSet`: ejecutar SQL y leer resultados.

```java
import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.DatabaseMetaData;
import java.sql.ResultSet;
import java.sql.Statement;
```

Después, declaramos una clase `final` (evitamos herencia accidental) y nuestro `Logger`, que será único por clase, compartido e inmutable (`static final`).
En el método `main`, capturamos las posibles excepciones dentro de un bloque *try-catch*.

```java
public final class Ex1ConnectAndPing {
    private static final Logger log = LoggerFactory.getLogger(Ex1ConnectAndPing.class);

    public static void main(String[] args) {
        try {
            DatabaseType type = DatabaseType.fromEnv();
            DataSource ds = DataSourceFactory.create(type);

            try (Connection c = ds.getConnection()) {
                DatabaseMetaData md = c.getMetaData();
                log.info("Conectado a {} {} - Driver {} {}",
                        md.getDatabaseProductName(), md.getDatabaseProductVersion(),
                        md.getDriverName(), md.getDriverVersion());

                // Ping simple (dependiente de motor pero portable con SELECT 1)
                try (Statement st = c.createStatement();
                     ResultSet rs = st.executeQuery("SELECT 1")) {
                    rs.next();
                    int one = rs.getInt(1);
                    log.info("Ping OK => SELECT 1 = {}", one);
                }
            }
        } catch (Exception e) {
            log.error("Error: {}", e.getMessage(), e);
            System.exit(1);
        }
    }
}
```

