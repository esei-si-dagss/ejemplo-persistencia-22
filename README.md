# Ejemplo MAVEN y JPA Básico  (DAGSS-2022)

* Ejemplo de creación de proyectos Maven y uso desde línea de comandos
* Ejemplo de mapeo JPA y DAO genérico


## PREVIO
### Requisitos previos

* Servidor de BD MySQL
* Maven (versión > 3.5.x)
* (opcional) GIT
* (opcional) IDE Java (Eclipse, Netbeans, IntelliJ, VS Code)

### Crear BD para los ejemplos

* Crear BD "pruebas_dagss" en MySQL

```
mysql -u root -p    [pedirá la contraseña de MySQL]

mysql> create database pruebas_dagss;
mysql> create user dagss@localhost identified by "dagss";
mysql> grant all privileges on pruebas_dagss.* to dagss@localhost;
```

Adicionalmente, puede ser necesario establecer un formato de fecha compatible
```
mysql> set @@global.time_zone = '+00:00';
mysql> set @@session.time_zone = '+00:00';
```



## CREAR Y CONFIGURAR PROYECTO MAVEN
### Crear un proyecto Maven usando el arquetipo `maven-archetype-quickstart` 
```sh
mvn archetype:generate   -DgroupId=es.uvigo.dagss \
                         -DartifactId=ejemplo-persistencia \
                         -Dversion=1.0 \
                         -Dpackage=es.uvigo.dagss.pedidos\
                         -DarchetypeArtifactId=maven-archetype-quickstart -DarchetypeVersion=1.4
                         
cd ejemplo-persistencia                         
```

* Comprobar la estructura de directorios creada con `tree ejemplo-persistencia` ó `ls -lR ejemplo-persistencia`
* Ajustar el archivo `pom.xml`generado (el arquetipo `maven-archetype-quickstart` establece por defecto plugins de versiones algo antiguas)
	1. Ajustar la versión de Java a utilizar, emplear al menos  Java 11 [la versión de Hibernate utilizada requiere Java 11 o superior]. También puede omitirse la especificación de versiones y usar la que esté instalada por defecto.
    ```xml
      <properties>
        <maven.compiler.target>11</maven.compiler.target>
        <maven.compiler.source>11</maven.compiler.source>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
      </properties>
    ```
    2. Eliminar el contenido del elemento `<dependencies>` (manteniendo las etiquetas externas que se rellenarán en el siguiente paso)
    3. Eliminar completamente la sección `<build>` (incluyendo contenido y etiquetas externas). Por defecto, se declaran versiones anticuadas de los plugins.

* Eliminar las clases de ejemplo incluidas en el arquetipo maven
```sh
rm src/main/java/es/uvigo/dagss/pedidos/App.java

rm src/test/java/es/uvigo/dagss/pedidos/AppTest.java
```

## CONFIGURACIÓN PARA JPA
### Declarar las dependencias necesarias en `pom.xml`
1 Declarar el uso de Hibernate como _provider_ JPA (ver. 5.6.14.Final) dentro de `<dependencies>...</dependencies>`

```xml
<project>
   ...
   <dependencies>
      <dependency>
         <groupId>org.hibernate</groupId>
         <artifactId>hibernate-core</artifactId>
         <version>5.6.14.Final</version>
       </dependency>
      ...
   </dependencies>
</project>
```
El resto de dependencias necesarias para `hibernate-core` serán descargadas e instaladas por Maven.

2 Declarar el _connector_ JDBC para MySQL (ver. 8.0.33) dentro de `<dependencies>...</dependencies>`

```xml
<project>
   ...
   <dependencies>
      ...
      <dependency>
         <groupId>mysql</groupId>
         <artifactId>mysql-connector-java</artifactId>
         <version>8.0.31</version>
      </dependency>
   </dependencies>
</project>
```

## MODELO E-R
![Modelo E-R](doc/modeloER_pedidos.png?raw=true "Modelo E-R del ejemplo")

## AÑADIR ENTIDADES

Crear el directorio para el paquete `entidades` y copiar los ficheros Java con la definición de las entidades (disponibles en [https://github.com/esei-si-dagss/ejemplo-persistencia-22/tree/main/src/main/java/es/uvigo/dagss/pedidos/entidades](https://github.com/esei-si-dagss/ejemplo-persistencia-22/tree/main/src/main/java/es/uvigo/dagss/pedidos/entidades))

```sh
mkdir -p src/main/java/es/uvigo/dagss/pedidos/entidades

    (por ahora no se crearán tests unitarios con JUnit)

cd src/main/java/es/uvigo/dagss/pedidos/entidades

wget https://raw.githubusercontent.com/esei-si-dagss/ejemplo-persistencia-22/main/src/main/java/es/uvigo/dagss/pedidos/entidades/{Articulo,Almacen,ArticuloAlmacen,ArticuloAlmacenId,Familia,Direccion,Cliente,Pedido,LineaPedido,EstadoPedido}.java

pushd
```

### Aspectos a revisar
1. Salvo en el caso de la entidad `Cliente`, que usa como clave primaria su DNI, las demás entidades usan claves autogeneradas de tipo `GenerationType.IDENTITY` (necesario para  mapear un atributo autoincremental de MySQL)
2. Salvo para la relación entre `Pedido` y su entidad débil `LineaPedido`, nos hemos limitado a relaciones unidireccionales.
    * Para asegurar que ambos extrmos de esa relación bidireccional se mantienen consistentes, se  incluyen los métodos `anadirLineaPedido()` y `anadirLineaPedidoInterno()` en `Pedido`, que se coordinan con el método `setPedido()`de `LineaPedido`para que en todo momento los dos lados de la relación sean correctos.
    * Más detalles en [jpa-implementation-patterns-bidirectional-assocations](https://xebia.com/blog/jpa-implementation-patterns-bidirectional-assocations/) y [Object_corruption,_one_side_of_the_relationship_is_not_updated_after_updating_the_other_side](https://en.wikibooks.org/wiki/Java_Persistence/Relationships#Object_corruption,_one_side_of_the_relationship_is_not_updated_after_updating_the_other_side)
  
3. La relación N:M entre `Articulo` y `Almacen` tiene un atributo propio, `stock`, por lo que se ha modelado esa relación como `@Entity`, empleando dos relaciones unidireccionales `@ManyToOne` hacia `Articulo` y `Almacen`.
    * Para gestionar el campo clave multiatributo se usa la clase auxiliar `ArticuloAlmacenId`, vinculada a la entidad que soporta la relación con atributos mediante la anotación`@IdClass`.
    * Más detalles en [Mapping_a_Join_Table_with_Additional_Columns](https://en.wikibooks.org/wiki/Java_Persistence/ManyToMany#Mapping_a_Join_Table_with_Additional_Columns)
    * 

### Configurar Persistence Unit de JPA

Definir fichero `persistence.xml` en `src/main/resources/META-INF`

```
mkdir -p src/main/resources/META-INF
nano src/main/resources/META-INF/persistence.xml
```

Contenido a incluir
```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence xmlns="http://xmlns.jcp.org/xml/ns/persistence"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence http://xmlns.jcp.org/xml/ns/persistence/persistence_2_2.xsd"
        version="2.2">

  <persistence-unit name="pedidos_PU" transaction-type="RESOURCE_LOCAL">
    <provider>org.hibernate.jpa.HibernatePersistenceProvider</provider>
    
    <class>es.uvigo.dagss.pedidos.entidades.Almacen</class>
    <class>es.uvigo.dagss.pedidos.entidades.Articulo</class>
    <class>es.uvigo.dagss.pedidos.entidades.ArticuloAlmacen</class>
    <class>es.uvigo.dagss.pedidos.entidades.Cliente</class>
    <class>es.uvigo.dagss.pedidos.entidades.Familia</class>
    <class>es.uvigo.dagss.pedidos.entidades.LineaPedido</class>
    <class>es.uvigo.dagss.pedidos.entidades.Pedido</class>
    
    <properties>
      <property name="javax.persistence.jdbc.url" value="jdbc:mysql://localhost:3306/pruebas_dagss?serverTimezone=UTC"/>
      <property name="javax.persistence.jdbc.user" value="dagss"/>
      <property name="javax.persistence.jdbc.password" value="dagss"/>
      <property name="javax.persistence.jdbc.driver" value="com.mysql.jdbc.Driver"/>
      <property name="javax.persistence.schema-generation.database.action" value="drop-and-create"/>
      
      <property name="hibernate.dialect" value="org.hibernate.dialect.MySQL8Dialect"/>
      <property name="hibernate.cache.provider_class" value="org.hibernate.cache.NoCacheProvider"/>
      <property name="hibernate.show_sql" value="true"/>
      <property name="hibernate.format_sql" value="true"/>
    </properties>
  </persistence-unit>

</persistence>
```


## DAO JPA Genérico

### Crear interface y clase del DAO

Crear el directorio para el paquete `daos` y copiar los ficheros Java con la definición de interfaces y clases de implementación.

```sh
mkdir -p src/main/java/es/uvigo/dagss/pedidos/daos
cd src/main/java/es/uvigo/dagss/pedidos/daos

wget https://raw.githubusercontent.com/esei-si-dagss/ejemplo-persistencia-22/main/src/main/java/es/uvigo/dagss/pedidos/daos/{GenericoDAO,GenericoDAOJPA,ClienteDAO,ClienteDAOJPA,PedidoDAO,PedidoDAOJPA,PedidosException}.java

pushd
```

#### Aspectos a revisar
1. Comprobar el interface _GenericoDAO_  con la definición de un DAO genérico (parametrizando la clase de la entidad y la de la clave) y su implementación _GenericoDAOJPA_ (recibe un _EntityManager_ en el constructor)
2. Comprobar los interfaces _ClienteDAO_ y _PedidoDAO_ 
	* Heredan de _GenericoDAO_ especificando los tipos correspondientes de entidad y clave
	* Añaden métodos adicionales específicos para el entidad
3. Revisar las  implementaciones de estos interfaces:
	* _ClienteDAOJPA_: hereda de _GenericoDAOJPA_ e implementa _ClienteDAO_
	* _PedidoDAOJPA_: hereda de _GenericoDAOJPA_ e implementa _PedidoDAO_
4. Comprobar en _PedidoDAOJPA_ la gestión de la relación 1:N con _LineaPedido_


## PROBAR EL PROYECTO
### Añadir clase con "main()" de ejemplo
```sh
cd src/main/java/es/uvigo/dagss/pedidos/

wget https://raw.githubusercontent.com/esei-si-dagss/ejemplo-persistencia-22/main/src/main/java/es/uvigo/dagss/pedidos/Main.java

pushd
```

#### Aspectos a revisar
1. Se crea un _EntityManagerFactory_ al que se le pedirá la creación de los respectivos _EntityManager_ con los que se realizarán las operaciones sobre la base de datos
2. Las operaciones realizadas por el _EntityManager_ sobre la base de datos se realizan dentro de una _transacción_ proporcionada por el _EntityEmanager_.
3. La transación es iniciada con `tx.begin()` y completada con éxito al hacer `tx.commit()`. En caso  de que durante la ejecución de esas acciones se lanze alguna excepcion, se ejecuta `tx.rollback()` para omitir los cambios realizados.
4. Dado que en `persistence.xml` se ha indicado la opción `drop-and-create`, en cada ejecución de estas clases de ejemplo, se eliminan los contenidos de la base de datos y se crea una nueva vacía.

## Ejecutar clase `Main` proporcionada
```sh
mvn package
mvn exec:java -Dexec.mainClass="es.uvigo.dagss.pedidos.Main"
```

* Comprobar el estado de la BD: tablas creadas y contenido de las mismas

```
mysql -u si -p   <con contraseña si>

mysql > use pruebas_dagss;
mysql > show tables;
mysql > describe Articulo;                # los nombres de las tablas pueden ser diferentes
mysql > describe Pedido;
mysql > select * from Articulo;
mysql > select * from Pedido;
mysql > ...

```



## TAREA EXTRA
* Comprobar la importación y uso del proyecto creado desde el IDE Java empleado habitualmente.

## Proyecto final resultante

Disponible en Github: [https://github.com/esei-si-dagss/ejemplo-persistencia-22](https://github.com/esei-si-dagss/ejemplo-persistencia-22)

