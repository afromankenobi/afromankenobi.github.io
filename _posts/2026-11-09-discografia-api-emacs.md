---
layout: post
title: "Una API de discografías, rock chileno y Emacs"
tags: [java, spring, emacs, mongodb]
---

Este proyecto salió de un taller de Programación de Servicios Web: una API para guardar artistas y discos en MongoDB. Uso Doom Emacs, así que aproveché el módulo `rest` para probarla desde el editor. Las peticiones quedan en archivos de texto junto al código, y puedo repetir el recorrido sin copiar IDs entre respuestas.

Para los ejemplos elegí Los Jaivas, Los Prisioneros y Los Tres. Si vamos a llenar una base de datos con discos, al menos que sean discos que me gustan.

## Qué hace la API

La aplicación usa Java 21, Spring Boot 3.3.2 y Spring Data MongoDB. Corre localmente y guarda los documentos en Atlas. Permite crear, consultar, actualizar y eliminar artistas, además de registrar discos y consultar la discografía de cada artista.

La relación es simple: cada disco guarda el ID de su artista en `idArtista`. Antes de insertar un disco, el controlador comprueba que ese artista exista.

El código y las peticiones están en [discografia-api](https://github.com/afromankenobi/discografia-api). El README tiene la configuración de Atlas y el detalle de las rutas; aquí voy a concentrarme en cómo la pruebo desde Emacs.

## Primero, levantar la aplicación

Con JDK 21 seleccionado en `JAVA_HOME`, desde la carpeta del proyecto:

```sh
./gradlew bootWar
cp .env.example .env
```

Completa `MONGODB_URI` en `.env` con la URI de tu usuario de base de datos de Atlas, sin comillas ni `export`. Ese usuario necesita acceso a `discografica-db`, y Atlas debe permitir conexiones desde tu IP. Si las credenciales contienen caracteres especiales, codifícalos para una URL.

La aplicación carga el `.env` de la carpeta desde la que se ejecuta. El archivo está excluido de Git. Con eso listo:

```sh
java -jar build/libs/discografia-1.war
```

Por defecto escucha en `http://localhost:8080`.

## Las peticiones viven en un buffer

En Doom tengo habilitado `(rest +jq)`. El archivo [http/discografia.http](https://github.com/afromankenobi/discografia-api/blob/main/http/discografia.http) usa `restclient-mode` y contiene 19 casos, ordenados para reutilizar los datos que crea cada petición.

Ábrelo con `C-x C-f` y revisa la variable `:base`. El archivo usa el puerto `18082`, con el que se hizo la verificación. Si arrancaste con el comando anterior, cámbiala a:

```text
:base = http://127.0.0.1:8080/api
```

Las peticiones tienen esta forma:

```text
GET :base/artistas
Accept: application/json
```

Sitúa el cursor en la petición y pulsa `C-c C-c`, o `SPC m e` en Doom. Emacs muestra la respuesta en otro buffer.

## Guardar IDs y comprobar respuestas

Crear un artista devuelve un `_id`, que después necesito para consultar sus datos o agregar un disco. Dejé hooks en el archivo para guardar ese valor y comprobar el estado HTTP de cada respuesta.

Para permitir su ejecución solo en este buffer, pulsa `M-:` y evalúa:

```elisp
(setq-local restclient-enable-eval t)
```

Ahora ejecuta los casos en orden, esperando la respuesta antes de pasar al siguiente. El caso 01 guarda `:artista-id`; el 05 hace lo mismo con `:disco-id`. Las siguientes peticiones usan esas variables:

```text
GET :base/artista/:artista-id/discos
Accept: application/json
```

Los hooks comprueban el estado HTTP con `cl-assert`. Si una respuesta no coincide con lo esperado, aparece `cl-assertion-failed`. También hay comprobaciones para el ID de una actualización y para un filtro que debe devolver una lista vacía.

## Probar los errores también

La secuencia incluye nombres vacíos, recursos inexistentes e IDs duplicados. El caso 04 envía un ID distinto en el cuerpo de un `PUT` y comprueba que se conserve el de la URL.

Al final se elimina el artista creado al principio y se comprueba que ya no pueda consultarse, actualizarse ni eliminarse otra vez. Por eso conviene mantener el ID generado por la prueba: sustituirlo por el de un artista que quieras conservar haría que la secuencia lo eliminara.

El disco queda en la base. La API no tiene una ruta para eliminar discos ni borrado en cascada. Para repetir la secuencia, vuelve al caso 01 y sigue en orden; los nuevos POST reemplazan los IDs guardados en Emacs.

Así se ven las peticiones y sus respuestas en Doom Emacs, con la API conectada a Atlas. Puedes abrir cada captura para verla a tamaño completo.

[![Crear un artista devuelve HTTP 201]({{ '/assets/images/discografia-emacs/emacs-crear-artista.png' | relative_url }})]({{ '/assets/images/discografia-emacs/emacs-crear-artista.png' | relative_url }})

[![Consultar la discografía devuelve HTTP 200]({{ '/assets/images/discografia-emacs/emacs-consultar-discografia.png' | relative_url }})]({{ '/assets/images/discografia-emacs/emacs-consultar-discografia.png' | relative_url }})

[![Repetir un ID devuelve HTTP 409]({{ '/assets/images/discografia-emacs/emacs-id-duplicado.png' | relative_url }})]({{ '/assets/images/discografia-emacs/emacs-id-duplicado.png' | relative_url }})

## Ahora sí, los discos

En [http/rock-chileno.http](https://github.com/afromankenobi/discografia-api/blob/main/http/rock-chileno.http) dejé otra secuencia con Los Jaivas, Los Prisioneros y Los Tres. Incluye *Alturas de Machu Picchu*, *La voz de los '80*, *Pateando piedras*, *Los Tres* y *La espada y la pared*.

Se ejecuta de la misma forma: habilita los hooks, revisa `:base` y sigue las peticiones en orden. Ese archivo apunta al puerto `8080`. Cada POST inserta un registro nuevo; los ejemplos quedan guardados para seguir consultándolos.

Con eso tengo las peticiones, los ejemplos y las comprobaciones en el mismo lugar que el código. Para este proyecto, abrir un `.http` en Emacs me basta.
