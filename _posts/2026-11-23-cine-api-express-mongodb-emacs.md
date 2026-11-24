---
published: false
layout: post
title: "Una API de películas con Express, MongoDB y Emacs"
tags: [javascript, express, mongodb, docker, emacs]
mermaid: true
---

En el taller anterior hice una API para guardar discos. En este tocó hacer algo parecido con películas y actores, esta vez en JavaScript con Express. Para probarla elegí dos películas que me gustan, *Matrix* y *Una mente brillante*. Quería poder consultar el reparto de cada una desde la API y comprobar que MongoDB devolviera a los actores de la película que pedí. [La API de discografías]({{ '/2026-11-09-discografia-api-emacs/' | relative_url }})

### Primero dejo que MongoDB responda

Partí con `server.js` y `node --watch` para que Node lo reinicie cuando cambie el código. En ese archivo configuro CORS, habilito la lectura de JSON y formularios, y monto bajo `/api` las rutas de películas y actores. Dejé un `GET /` con un mensaje de bienvenida para poder comprobar el arranque de inmediato.

¿Para qué abrir el puerto si cada consulta depende de MongoDB? En `src/common/db.js` creo un `MongoClient` con `MONGODB_URI` y lo comparto entre los controladores. Antes de llamar a `app.listen()`, la función `connect()` hace esto.

```js
await client.connect();
await client.db('admin').command({ ping: 1 });
```

Si falla el `ping`, Express termina el arranque sin abrir el puerto. Si responde, ya puedo hacer una petición desde el cliente HTTP.

### ¿Dónde guardo el vínculo?

En `src/pelicula` y `src/actor` dejé los esquemas, las rutas y los controladores. Los esquemas me sirven para validar lo que llega por HTTP antes de guardarlo. El año de estreno y la edad, por ejemplo, quedan como enteros BSON de 32 bits.

<pre class="mermaid">
flowchart LR
    request["Cliente HTTP"] --> express["Express /api"]
    express --> movies["Controlador de películas"]
    express --> actors["Controlador de actores"]
    movies --> movieDocuments[("peliculas")]
    actors -->|"Busca la película"| movieDocuments
    actors --> actorDocuments[("actores")]
</pre>

Cuando agrego a alguien al reparto, envío `nombrePelicula`. El controlador busca ese título en `peliculas` y copia el `_id` que encontró a `idPelicula` en `actores`. Si hay dos películas con el mismo título, la API devuelve `409`. En ese caso mando también el ID para elegir una.

Me importa el tipo de ese valor porque lo uso en dos consultas. En `peliculas`, `_id` es un `ObjectId`. En `actores`, `idPelicula` es texto. Cuando pido un reparto, convierto el parámetro de la URL a `ObjectId` para consultar la película y lo paso a texto para buscar a sus actores.

### Dos consultas que caían en la misma ruta

La pauta proponía `GET /api/actor/:id` para buscar a un actor y `GET /api/actor/:pelicula` para buscar el reparto. Si llamo a `/api/actor/123`, Express ve un segmento variable después de `/actor/` en los dos casos. El nombre solo cambia si lo leo como `req.params.id` o `req.params.pelicula`.

Dejé `/api/actor/:id` para el actor y puse el reparto debajo de la película. Así queda explícito en la URL qué quiero pedir.

```http
GET /api/actor/<id-del-actor>
GET /api/pelicula/<id-de-la-pelicula>/actores
```

Para editar una película uso `PUT /api/pelicula/:id`. El controlador aplica `$set` a los campos que le mando. Para cambiar el título, envío solo `nombre` y el año y los géneros siguen como estaban.

### Ahora lo pruebo en local

Uso Node.js 22 o posterior. Desde la raíz del proyecto, levanto MongoDB en Docker y arranco la API.

```sh
docker run --rm -d --name cine-api-mongo -p 127.0.0.1:27017:27017 mongo:7
npm ci
MONGODB_URI=mongodb://127.0.0.1:27017 npm start
```

La variable en el comando le dice a Express que use el contenedor durante esa ejecución. Así puedo probar en Docker sin sobrescribir la conexión que tengo guardada en `.env`. Un `curl http://localhost:3000/` devuelve «Bienvenido al cine Iplacex».

### Las peticiones de prueba en Emacs

Como en la API de discografías, dejé las peticiones en un archivo de texto, esta vez en `http/cine.http`. En Doom Emacs puedo abrirlo con `restclient-mode`, habilitar los hooks con `M-:` y ejecutar la petición bajo el cursor con `C-c C-c`.

```elisp
(setq-local restclient-enable-eval t)
```

Primero creo la película y agrego un actor. Los `POST` del archivo guardan los IDs de las respuestas como `:pelicula-id` y `:actor-id`. Así puedo consultar el reparto sin copiar el ID desde otro buffer.

```text
GET :host/api/pelicula/:pelicula-id/actores
```

En la prueba HTTP usé *La memoria del agua*. Al cambiar el título a *La memoria del agua (reestreno)*, el `GET` del reparto devolvió `200` con el mismo actor y el mismo `idPelicula`. Eso era lo que quería comprobar. El nombre me sirve para encontrar la película al inicio y, desde ahí, la relación queda guardada por ID.

Al final del archivo borro la película de prueba. El actor queda en MongoDB y la consulta del reparto devuelve `404` porque la película ya no existe. Cuando termino, cierro Express y hago `docker stop cine-api-mongo`. Como inicié el contenedor con `--rm`, puedo partir de nuevo con una base vacía.

### Matrix y Una mente brillante en Atlas

Para arrancar la API con Atlas, guardo en `.env` la URI de `eva-u3-express` y ejecuto `npm start`. El usuario de la API tiene lectura y escritura sobre `cine-db`. Con la API ya conectada, creé las dos películas y cargué cinco integrantes de cada reparto con `POST /api/actor`. Envié el título y el ID de la película en cada alta. Para `edad` usé la que tenía cada persona al estreno.

Para repetir la consulta desde un buffer de `restclient-mode`, primero ejecuto `GET :host/api/peliculas` y uso los `_id` de *Matrix* (1999) y *Una mente brillante* (2001) en las variables de estas dos peticiones.

```text
:host = http://127.0.0.1:3000
:matrix-id = <id de Matrix>
:mente-brillante-id = <id de Una mente brillante>

### Buscar los IDs
GET :host/api/peliculas

### Matrix
GET :host/api/pelicula/:matrix-id/actores

### Una mente brillante
GET :host/api/pelicula/:mente-brillante-id/actores
```

En la prueba HTTP contra Atlas, las dos consultas devolvieron `200`. Estos son los nombres que salieron en cada respuesta.

- **Matrix** · Keanu Reeves, Laurence Fishburne, Carrie-Anne Moss, Hugo Weaving y Joe Pantoliano. [Ficha de AFI](https://catalog.afi.com/Catalog/moviedetails/61230)
- **Una mente brillante** · Russell Crowe, Jennifer Connelly, Ed Harris, Paul Bettany y Adam Goldberg. [Ficha de Universal Pictures](https://www.universalpicturesathome.com/movies/a-beautiful-mind)

Comprobé que cada arreglo trajera cinco documentos, que cada nombre apareciera una sola vez y que todos los `idPelicula` coincidieran con el ID de la URL. Dejé las dos películas y sus repartos en Atlas como datos de prueba.

### Referencias

- [Guía de enrutamiento de Express](https://expressjs.com/en/guide/routing/)
- [Conexiones del driver de MongoDB para Node.js](https://www.mongodb.com/docs/drivers/node/v6.x/connect/connection-options/connection-pools/)
