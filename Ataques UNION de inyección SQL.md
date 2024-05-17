
Cuando una aplicación es vulnerable a la inyección SQL y los resultados de la consulta se devuelven dentro de las respuestas de la aplicación, puede usar la `UNION`palabra clave para recuperar datos de otras tablas dentro de la base de datos. Esto se conoce comúnmente como ataque UNION de inyección SQL.

La `UNION`palabra clave le permite ejecutar una o más `SELECT`consultas adicionales y agregar los resultados a la consulta original. Por ejemplo:

`SELECT a, b FROM table1 UNION SELECT c, d FROM table2`

Esta consulta SQL devuelve un único conjunto de resultados con dos columnas, que contiene valores de columnas `a`y `b`en `table1`y columnas `c`y `d`en `table2`.

Para que una `UNION`consulta funcione, se deben cumplir dos requisitos clave:

- Las consultas individuales deben devolver el mismo número de columnas.
- Los tipos de datos de cada columna deben ser compatibles entre las consultas individuales.

Para llevar a cabo un ataque UNION de inyección SQL, asegúrese de que su ataque cumpla con estos dos requisitos. Esto normalmente implica descubrir:

- Cuántas columnas se devuelven de la consulta original.
- Qué columnas devueltas por la consulta original tienen un tipo de datos adecuado para contener los resultados de la consulta inyectada.

## Determinar el número de columnas necesarias

Cuando realiza un ataque UNION de inyección SQL, existen dos métodos efectivos para determinar cuántas columnas se devuelven de la consulta original.

Un método implica inyectar una serie de `ORDER BY`cláusulas e incrementar el índice de la columna especificada hasta que ocurra un error. Por ejemplo, si el punto de inyección es una cadena entre comillas dentro de la `WHERE`cláusula de la consulta original, deberá enviar:

`' ORDER BY 1-- ' ORDER BY 2-- ' ORDER BY 3-- etc.`

Esta serie de cargas útiles modifica la consulta original para ordenar los resultados por diferentes columnas en el conjunto de resultados. La columna de una `ORDER BY`cláusula se puede especificar por su índice, por lo que no es necesario conocer los nombres de ninguna columna. Cuando el índice de columna especificado excede el número de columnas reales en el conjunto de resultados, la base de datos devuelve un error, como por ejemplo:

`The ORDER BY position number 3 is out of range of the number of items in the select list.`

En realidad, la aplicación podría devolver el error de la base de datos en su respuesta HTTP, pero también puede emitir una respuesta de error genérica. En otros casos, es posible que simplemente no arroje ningún resultado. De cualquier manera, siempre que pueda detectar alguna diferencia en la respuesta, puede inferir cuántas columnas se devuelven de la consulta.

El segundo método implica enviar una serie de `UNION SELECT`cargas útiles que especifican un número diferente de valores nulos:

`' UNION SELECT NULL-- ' UNION SELECT NULL,NULL-- ' UNION SELECT NULL,NULL,NULL-- etc.`

Si el número de valores nulos no coincide con el número de columnas, la base de datos devuelve un error, como por ejemplo:

`All queries combined using a UNION, INTERSECT or EXCEPT operator must have an equal number of expressions in their target lists.`

Usamos `NULL`como valores devueltos por la `SELECT`consulta inyectada porque los tipos de datos en cada columna deben ser compatibles entre las consultas originales y las inyectadas. `NULL`es convertible a todos los tipos de datos comunes, por lo que maximiza la posibilidad de que la carga útil tenga éxito cuando el recuento de columnas sea correcto.

Al igual que con la `ORDER BY`técnica, la aplicación puede devolver el error de la base de datos en su respuesta HTTP, pero puede devolver un error genérico o simplemente no devolver ningún resultado. Cuando el número de valores nulos coincide con el número de columnas, la base de datos devuelve una fila adicional en el conjunto de resultados, que contiene valores nulos en cada columna. El efecto sobre la respuesta HTTP depende del código de la aplicación. Si tiene suerte, verá contenido adicional dentro de la respuesta, como una fila adicional en una tabla HTML. De lo contrario, los valores nulos podrían desencadenar un error diferente, como un archivo `NullPointerException`. En el peor de los casos, la respuesta podría tener el mismo aspecto que una respuesta provocada por un número incorrecto de valores nulos. Esto haría que este método fuera ineficaz.

## Sintaxis específica de la base de datos

En Oracle, cada `SELECT`consulta debe utilizar la `FROM`palabra clave y especificar una tabla válida. Hay una tabla integrada en Oracle llamada `dual`que se puede utilizar para este propósito. Entonces, las consultas inyectadas en Oracle deberían verse así:

`' UNION SELECT NULL FROM DUAL--`

Las cargas útiles descritas utilizan la secuencia de comentarios de doble guión `--`para comentar el resto de la consulta original después del punto de inyección. En MySQL, la secuencia de doble guión debe ir seguida de un espacio. `#`Alternativamente, se puede utilizar el carácter almohadilla para identificar un comentario.

Para obtener más detalles sobre la sintaxis específica de la base de datos, consulte la [hoja de referencia de inyección SQL](https://portswigger.net/web-security/sql-injection/cheat-sheet)

## Encontrar columnas con un tipo de datos útil

Un ataque UNION de inyección SQL le permite recuperar los resultados de una consulta inyectada. Los datos interesantes que desea recuperar normalmente están en forma de cadena. Esto significa que necesita encontrar una o más columnas en los resultados de la consulta original cuyo tipo de datos sea, o sea compatible con, datos de cadena.

Después de determinar la cantidad de columnas requeridas, puede sondear cada columna para probar si puede contener datos de cadena. Puede enviar una serie de `UNION SELECT`cargas útiles que coloquen un valor de cadena en cada columna por turno. Por ejemplo, si la consulta devuelve cuatro columnas, deberá enviar:

`' UNION SELECT 'a',NULL,NULL,NULL-- ' UNION SELECT NULL,'a',NULL,NULL-- ' UNION SELECT NULL,NULL,'a',NULL-- ' UNION SELECT NULL,NULL,NULL,'a'--`

Si el tipo de datos de la columna no es compatible con los datos de cadena, la consulta inyectada provocará un error en la base de datos, como por ejemplo:

`Conversion failed when converting the varchar value 'a' to data type int.`

Si no se produce un error y la respuesta de la aplicación contiene algún contenido adicional, incluido el valor de cadena inyectado, entonces la columna relevante es adecuada para recuperar datos de cadena.

## Usando un ataque UNION de inyección SQL para recuperar datos interesantes

Cuando haya determinado el número de columnas devueltas por la consulta original y haya encontrado qué columnas pueden contener datos de cadena, estará en condiciones de recuperar datos interesantes.

Suponer que:

- La consulta original devuelve dos columnas, las cuales pueden contener datos de cadena.
- El punto de inyección es una cadena entre comillas dentro de la `WHERE`cláusula.
- La base de datos contiene una tabla llamada `users`con las columnas `username`y `password`.

En este ejemplo, puede recuperar el contenido de la `users`tabla enviando la entrada:

`' UNION SELECT username, password FROM users--`

Para poder realizar este ataque, necesitas saber que hay una tabla llamada `users`con dos columnas llamadas `username`y `password`. Sin esta información, tendrías que adivinar los nombres de las tablas y columnas. Todas las bases de datos modernas ofrecen formas de examinar la estructura de la base de datos y determinar qué tablas y columnas contienen.

## Recuperar múltiples valores dentro de una sola columna

En algunos casos, es posible que la consulta del ejemplo anterior solo devuelva una única columna.

Puede recuperar varios valores juntos dentro de esta única columna concatenando los valores. Puede incluir un separador que le permita distinguir los valores combinados. Por ejemplo, en Oracle podrías enviar la entrada:

`' UNION SELECT username || '~' || password FROM users--`

Esto utiliza la secuencia de doble canalización, `||`que es un operador de concatenación de cadenas en Oracle. La consulta inyectada concatena los valores de los campos `username`y `password`, separados por el `~`carácter.

Los resultados de la consulta contienen todos los nombres de usuario y contraseñas, por ejemplo:

`... administrator~s3cure wiener~peter carlos~montoya ...`

Diferentes bases de datos utilizan diferentes sintaxis para realizar la concatenación de cadenas. Para obtener más detalles, consulte la [hoja de referencia de inyección SQL](https://portswigger.net/web-security/sql-injection/cheat-sheet) .

## Examinando la base de datos en ataques de inyección SQL

Para explotar las vulnerabilidades de inyección SQL, a menudo es necesario encontrar información sobre la base de datos. Esto incluye:

- El tipo y versión del software de base de datos.
- Las tablas y columnas que contiene la base de datos.

## Consultar el tipo y la versión de la base de datos.

Potencialmente, puede identificar tanto el tipo como la versión de la base de datos inyectando consultas específicas del proveedor para ver si alguna funciona.

Las siguientes son algunas consultas para determinar la versión de la base de datos para algunos tipos de bases de datos populares:

|   |   |
|---|---|
|Tipo de base de datos|Consulta|
|Microsoft, MySQL|`SELECT @@version`|
|Oráculo|`SELECT * FROM v$version`|
|PostgreSQL|`SELECT version()`|

Por ejemplo, podrías utilizar un `UNION`ataque con la siguiente entrada:

`' UNION SELECT @@version--`

Esto podría devolver el siguiente resultado. En este caso, puedes confirmar que la base de datos es Microsoft SQL Server y ver la versión utilizada:

`Microsoft SQL Server 2016 (SP2) (KB4052908) - 13.0.5026.0 (X64) Mar 18 2018 09:11:49 Copyright (c) Microsoft Corporation Standard Edition (64-bit) on Windows Server 2016 Standard 10.0 <X64> (Build 14393: ) (Hypervisor)`

## Listado del contenido de la base de datos.

La mayoría de los tipos de bases de datos (excepto Oracle) tienen un conjunto de vistas llamado esquema de información. Esto proporciona información sobre la base de datos.

Por ejemplo, puede realizar una consulta `information_schema.tables`para enumerar las tablas de la base de datos:

`SELECT * FROM information_schema.tables`

Esto devuelve un resultado como el siguiente:

`TABLE_CATALOG TABLE_SCHEMA TABLE_NAME TABLE_TYPE ===================================================== MyDatabase dbo Products BASE TABLE MyDatabase dbo Users BASE TABLE MyDatabase dbo Feedback BASE TABLE`

Este resultado indica que hay tres tablas, llamadas `Products`, `Users`y `Feedback`.

Luego puede realizar consultas `information_schema.columns`para enumerar las columnas en tablas individuales:

`SELECT * FROM information_schema.columns WHERE table_name = 'Users'`

Esto devuelve un resultado como el siguiente:

`TABLE_CATALOG TABLE_SCHEMA TABLE_NAME COLUMN_NAME DATA_TYPE ================================================================= MyDatabase dbo Users UserId int MyDatabase dbo Users Username varchar MyDatabase dbo Users Password varchar`

Este resultado muestra las columnas de la tabla especificada y el tipo de datos de cada columna.

## Lab
1. Utilice Burp Suite para interceptar y modificar la solicitud que establece el filtro de categoría de producto.
2. Determine la cantidad de columnas que devuelve la consulta y qué columnas contienen datos de texto. Verifique que la consulta devuelva dos columnas, las cuales contengan texto, utilizando una carga útil como la siguiente en el `category`parámetro:
    
    `'+UNION+SELECT+'abc','def'--`
3. Utilice la siguiente carga útil para recuperar la lista de tablas en la base de datos:
    
    `'+UNION+SELECT+table_name,+NULL+FROM+information_schema.tables--`
4. Busque el nombre de la tabla que contiene las credenciales de usuario.
5. Utilice la siguiente carga útil (que reemplaza el nombre de la tabla) para recuperar los detalles de las columnas de la tabla:
    
    `'+UNION+SELECT+column_name,+NULL+FROM+information_schema.columns+WHERE+table_name='users_abcdef'--`
6. Busque los nombres de las columnas que contienen nombres de usuario y contraseñas.
7. Utilice la siguiente carga útil (que reemplaza los nombres de tablas y columnas) para recuperar los nombres de usuario y contraseñas de todos los usuarios:
    
    `'+UNION+SELECT+username_abcdef,+password_abcdef+FROM+users_abcdef--`
8. Busque la contraseña del `administrator`usuario y úsela para iniciar sesión.

