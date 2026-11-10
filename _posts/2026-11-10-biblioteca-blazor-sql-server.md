---
layout: post
title: "Una biblioteca con Blazor y SQL Server"
tags: [csharp, blazor, sqlserver, dotnet]
---

Este proyecto salió de un taller de Programación Multiplataforma: una aplicación para administrar los libros de la biblioteca William Wallace. Había que agregar, listar, editar y eliminar registros con Blazor, y guardarlos en SQL Server usando Entity Framework Core.

La pauta traía los libros: *El ojo del mundo*, *Orgullo o prejuicio* y *Juego de tronos*. Sí, decía «Orgullo o prejuicio». Dejé ese nombre para mantener los datos del ejercicio.

El formulario es pequeño, pero permite seguir el recorrido completo de un dato: desde lo que escribo en el navegador hasta la fila que queda guardada en la base. Aquí voy a concentrarme en ese recorrido y en cómo comprobar que una edición conserva el libro correcto.

## Primero, dónde corre cada cosa

La solución usa .NET 8 y tiene tres proyectos:

- `BlazorBiblioteca.Client`: los componentes que se ejecutan en el navegador con WebAssembly.
- `BlazorBiblioteca`: el servidor ASP.NET Core, el controlador y la conexión a SQL Server.
- `BlazorBiblioteca.Shared`: la clase `Libro`, con las propiedades y validaciones que comparten cliente y servidor.

La plantilla es Blazor Web App, con interactividad WebAssembly global. En `App.razor`, la configuración de las rutas queda así:

```razor
<Routes @rendermode="new InteractiveWebAssemblyRenderMode(prerender: false)" />
```

También se aplica ese modo a `HeadOutlet`. En este proyecto desactivé el prerenderizado y el componente carga su lista desde el navegador. Microsoft explica la configuración global en la [documentación de modos de renderizado de Blazor](https://learn.microsoft.com/en-us/aspnet/core/blazor/components/render-modes?view=aspnetcore-8.0).

El cliente envía peticiones con `HttpClient` a `api/Libro`. El controlador recibe un `LibroDBContext` y usa Entity Framework Core para consultar o modificar la base. La cadena de conexión queda en el proyecto del servidor.

## La tabla manda

La base se llama `BibliotecaDB` y contiene una tabla `Libros`. La definición del ejercicio es bastante concreta:

```sql
CREATE TABLE dbo.Libros
(
    Id int IDENTITY(1,1) NOT NULL CONSTRAINT PK_Libros PRIMARY KEY,
    NombreLibro varchar(50) NOT NULL,
    Autor varchar(50) NOT NULL,
    NumPaginas int NOT NULL,
    FechaPublicacion date NOT NULL
);
```

En el modelo uso `DateOnly` para la fecha de publicación. En el contexto configuré los nombres como `varchar(50)` mediante `HasMaxLength(50).IsUnicode(false)`, y la fecha como `date`. Así el modelo de Entity Framework coincide con la tabla pedida.

El ID lo genera SQL Server. Al agregar un libro, el controlador rechaza una petición que intente enviar un ID distinto de cero. Al actualizar, comprueba que el ID de la URL coincida con el del cuerpo de la petición.

## Un formulario para agregar y editar

`Home.razor` usa `EditForm`, `DataAnnotationsValidator` y un componente de entrada por campo: `InputText`, `InputNumber` e `InputDate`. El nombre y el autor son obligatorios, admiten hasta 50 caracteres y el número de páginas debe ser mayor que cero.

La estructura del formulario es esta, omitiendo los campos para mostrar lo que conecta las piezas:

```razor
<EditForm Model="modelo" OnValidSubmit="Guardar"
          FormName="libro" @key="modelo" novalidate>
    <DataAnnotationsValidator />
    @* Campos y botones del formulario *@
</EditForm>
```

`Guardar` decide qué operación corresponde. Un modelo con ID cero es un libro nuevo; uno con ID existente se envía a actualizar. Durante la petición, los controles quedan deshabilitados para evitar otro envío desde el formulario.

Las validaciones también se aplican en la API. Aunque alguien envíe una petición sin pasar por la pantalla, el servidor debe rechazar un libro sin nombre o con cero páginas.

## Editar una copia

Al pulsar EDITAR, copio los valores de la fila al modelo del formulario:

```csharp
modelo = new Libro
{
    Id = libro.Id,
    NombreLibro = libro.NombreLibro,
    Autor = libro.Autor,
    NumPaginas = libro.NumPaginas,
    FechaPublicacion = libro.FechaPublicacion
};
```

Eso permite cancelar la edición sin cambiar lo que muestra la tabla. Si el formulario usara directamente el objeto de la lista, escribir en un campo también modificaría ese objeto antes de guardar.

Para probar la actualización cambié *El ojo del mundo* de 824 a 825 páginas. La petición fue un `PUT` a `api/Libro/1`, y el registro mantuvo su ID. El cambio de páginas es solo parte de la demostración.

[![El ojo del mundo conserva el ID 001 después de actualizar sus páginas a 825]({{ '/assets/images/biblioteca-blazor/actualizar.png' | relative_url }})]({{ '/assets/images/biblioteca-blazor/actualizar.png' | relative_url }})

Después de guardar, el formulario vuelve al modo de ingreso y consulta nuevamente la lista.

## Qué hace @key aquí

Cada fila de la tabla tiene `@key="libro.Id"`. Esa clave permite que Blazor mantenga la correspondencia entre el registro y su fila al volver a renderizar la lista.

El formulario tiene otra clave: `@key="modelo"`. Cuando reemplazo el modelo por una nueva instancia, Blazor crea de nuevo el formulario con ese modelo, incluyendo su estado de validación.

La identidad de la fila en la base sigue dependiendo de la clave primaria y de `IDENTITY(1,1)`. `@key` se ocupa de la interfaz; no genera IDs en SQL Server.

## Recargar también es parte de la prueba

La carga inicial ocurre en el método de ciclo de vida del componente:

```csharp
protected override async Task OnInitializedAsync() => await Listar();
```

`Listar` hace un GET a la API. Si la consulta funciona y devuelve una lista vacía, aparece «No existen libros en la biblioteca actualmente». Si la consulta falla, la pantalla muestra que no pudo cargar el catálogo. Son dos resultados distintos que conviene poder reconocer al probar la conexión.

Para la demostración ingresé los tres libros y recargué la página. Los registros siguieron ahí:

[![Catálogo con los tres libros del ejercicio, sus identificadores, páginas y fechas]({{ '/assets/images/biblioteca-blazor/catalogo.png' | relative_url }})]({{ '/assets/images/biblioteca-blazor/catalogo.png' | relative_url }})

Después actualicé el primer libro y eliminé *Juego de tronos*. Una consulta directa a SQL Server confirmó los dos registros restantes y las 825 páginas del ID 1. Las capturas corresponden a la ejecución del 9 de noviembre de 2026, con Blazor en Windows ARM64 y SQL Server en Docker sobre el Mac.

## Instalar .NET con asdf

Para manejar el SDK uso asdf y dejo la versión definida por proyecto. Estos pasos asumen que [asdf ya está configurado en la terminal](https://asdf-vm.com/guide/getting-started.html), con su carpeta de shims en el `PATH`.

Primero agrego el [plugin `dotnet`](https://github.com/hensou/asdf-dotnet). Este paso se hace una sola vez; si aparece en `asdf plugin list`, puedes saltarlo:

```sh
asdf plugin add dotnet https://github.com/hensou/asdf-dotnet.git
```

Desde la carpeta `evaluacion-2`, instalo y selecciono el SDK con el que se verificó el proyecto:

```sh
asdf install dotnet 8.0.427
asdf set dotnet 8.0.427
asdf current dotnet
dotnet --version
```

El último comando debe mostrar `8.0.427`. Si no coincide, revisa que `command -v dotnet` apunte al shim de asdf.

`asdf set` guarda la selección en el archivo `.tool-versions` de esa carpeta:

```text
dotnet 8.0.427
```

Conviene guardar ese archivo junto al código. En otro equipo con asdf y el plugin preparados, basta ejecutar `asdf install` desde la carpeta para instalar la versión declarada. La [documentación de asdf](https://asdf-vm.com/manage/versions.html) explica cómo selecciona las versiones por directorio.

El proyecto también tiene un `global.json` que pide `8.0.400` con `rollForward: latestPatch`, por lo que acepta `8.0.427`, dentro de la banda `8.0.4xx`. asdf selecciona la instalación disponible y `global.json` limita el SDK que puede usar la solución. Mantengo ambos alineados con .NET 8, que es lo que exige el taller.

## Las pruebas quedan junto al código

Con el SDK seleccionado mediante asdf, desde la misma carpeta:

```sh
dotnet restore BlazorBiblioteca.sln --locked-mode
dotnet build BlazorBiblioteca.sln --no-restore
dotnet test BlazorBiblioteca.sln --no-build
```

Los nueve casos cubren el recorrido CRUD, datos inválidos, IDs manipulados, registros inexistentes y el esquema que genera el proveedor SQL Server. Las pruebas de la API usan SQLite en memoria para aislar los datos; la comprobación del esquema genera SQL sin conectarse al motor. La ejecución del navegador y la consulta directa a SQL Server completan la verificación de persistencia.

Para levantar la aplicación hay que preparar la base con `sql/01-crear-biblioteca.sql` y configurar su dirección en `appsettings.json` del servidor, o en `appsettings.Local.json`, que queda fuera de Git. Con esa conexión lista:

```sh
dotnet dev-certs https --trust
dotnet run --project src/BlazorBiblioteca --launch-profile https
```

La aplicación abre en `https://localhost:7149`. Las credenciales del ejercicio y la ausencia de autenticación corresponden a una práctica local; la base debe quedar restringida al entorno de laboratorio. Además, el contenedor x64 usado sobre Mac ARM funciona por emulación, una configuración que [Microsoft no soporta oficialmente](https://learn.microsoft.com/sql/linux/install-upgrade/setup).

Con eso tengo un recorrido que puedo seguir de principio a fin: guardar desde el formulario, recargar, editar el mismo ID y consultar la base. Para una biblioteca de tres libros, alcanza para ver qué está haciendo cada pieza.
