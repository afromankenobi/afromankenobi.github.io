---
layout: post
title: "Una biblioteca con Blazor y SQL Server"
tags: [csharp, blazor, sqlserver, dotnet]
---

Este proyecto salió de un taller de Programación Multiplataforma: una aplicación para administrar los libros de la biblioteca William Wallace. Había que agregar, listar, editar y eliminar registros con Blazor, y guardarlos en SQL Server usando Entity Framework Core.

## Por qué usé Windows y macOS

La ejecución original de Blazor se hizo en Windows ARM64, mientras que SQL Server quedó en un contenedor Docker sobre el Mac. Esa distribución tiene una razón concreta: la aplicación y el motor de base de datos tienen requisitos de arquitectura distintos.

.NET 8 tiene soporte para Windows ARM64, de modo que pude compilar y ejecutar allí el servidor ASP.NET Core y abrir el cliente Blazor en el navegador. El motor de SQL Server 2022, en cambio, [requiere un procesador x64 para su instalación en Windows](https://learn.microsoft.com/en-us/sql/sql-server/install/hardware-and-software-requirements-for-installing-sql-server-2022?view=sql-server-ver16). Tener Windows instalado no resuelve esa diferencia de arquitectura.

Para completar el ejercicio con el equipo disponible, ejecuté la imagen Linux de SQL Server con `platform: linux/amd64` en Docker sobre el Mac ARM. Blazor se conectó a ese motor por TCP. La aplicación siguió usando SQL Server y Entity Framework Core, tal como pedía la pauta; cambió la ubicación del motor.

La emulación permitió ejecutar y comprobar el CRUD en este laboratorio. Eso no la convierte en una plataforma soportada: [Microsoft admite sus contenedores de SQL Server en hosts Linux Intel/AMD x86-64 y excluye los entornos de emulación](https://learn.microsoft.com/en-us/sql/linux/install-upgrade/setup?view=sql-server-ver16). En un equipo Windows x64 se puede instalar el motor localmente; con un servidor SQL x64 disponible también se puede usar una conexión remota desde Windows o macOS.

El recorrido documentado en las capturas es, por tanto, navegador y aplicación en Windows, con la base en el Mac. Después también compilé el proyecto y ejecuté sus nueve pruebas en macOS. La misma solución sirve para ambos sistemas; las capturas del CRUD corresponden a Windows.

## Preparar Windows

### Instalar las herramientas

Desde PowerShell, primero compruebo la arquitectura e instalo el SDK y las herramientas de trabajo:

```powershell
[System.Runtime.InteropServices.RuntimeInformation]::OSArchitecture
winget install --id Microsoft.DotNet.SDK.8 --exact --source winget
winget install --id Git.Git --exact --source winget
winget install --id Microsoft.VisualStudioCode --exact --scope user --source winget
```

Después de aceptar los permisos del instalador, abro una terminal nueva y verifico:

```powershell
dotnet --list-sdks
git --version
code --version
```

Necesito un SDK de la banda `8.0.4xx`. El proyecto tiene un `global.json` que pide `8.0.400` y permite parches de esa misma banda; se comprobó con `8.0.427`. Tener solamente .NET 9 o 10 no satisface esa configuración. La [guía de Microsoft para Windows](https://learn.microsoft.com/en-us/dotnet/core/install/windows) explica las opciones de instalación y las arquitecturas disponibles.

Abro la carpeta `evaluacion-2` en el editor y ejecuto los comandos siguientes desde ella. Si trabajo con una carpeta compartida que da problemas de permisos o rutas, uso una copia en el disco local de Windows.

### Conectar SQL Server

En Windows ARM64 uso una instancia remota. Para reproducir la distribución de este laboratorio, el Mac debe tener el contenedor preparado y el puerto accesible desde Windows; más abajo explico cómo levantarlo. En Windows x64 también puedo instalar SQL Server 2022 Developer con el motor de base de datos y autenticación mixta, y usarlo localmente.

En una instancia nueva, me conecto con una cuenta administradora desde SSMS o desde la extensión SQL Server de Visual Studio Code y ejecuto, en este orden:

```text
sql/01-crear-biblioteca.sql
sql/02-verificar-biblioteca.sql
```

El primer script crea el login SQL `AdminBiblioteca`, con la contraseña académica `Admin12345` y `master` como base predeterminada. También crea `BibliotecaDB`, asigna ese login como propietario y prepara `dbo.Libros`. El segundo consulta la configuración y los registros para comprobar el resultado. Si la base se preparó con el script de Docker incluido en este proyecto, estos pasos ya se ejecutaron.

Los scripts conservan los datos existentes. Tampoco cambian la contraseña de un login que ya exista ni corrigen silenciosamente una tabla con otro esquema: por eso conviene revisar la salida de la verificación.

Para conectar Blazor, creo `src/BlazorBiblioteca/appsettings.Local.json` con esta estructura:

```json
{
  "ConnectionStrings": {
    "BibliotecaDB": "Server=IP_DEL_MAC,14333;Database=BibliotecaDB;User Id=AdminBiblioteca;Password=Admin12345;Encrypt=True;TrustServerCertificate=True;Connect Timeout=5"
  }
}
```

Reemplazo `IP_DEL_MAC` por la dirección accesible desde Windows. Si uso otra instancia, cambio también el puerto; para SQL local con TCP en 1433, el valor es `localhost,1433`. Una instancia nombrada puede usar `localhost\\SQLEXPRESS` dentro del JSON, si así está configurada. Cuando SQL está en el Mac, `localhost` en Windows sigue apuntando a Windows.

El archivo local sobrescribe la cadena de `appsettings.json` y queda fuera de Git y del ZIP. Las credenciales y `TrustServerCertificate=True` corresponden al laboratorio. Para comprobar el puerto remoto antes de abrir la aplicación:

```powershell
Test-NetConnection -ComputerName IP_DEL_MAC -Port 14333
```

`TcpTestSucceeded: True` confirma conectividad TCP; el inicio de sesión y la consulta del catálogo comprueban el acceso a la base.

### Compilar y abrir la aplicación

Desde `evaluacion-2`:

```powershell
dotnet restore BlazorBiblioteca.sln --locked-mode
dotnet build BlazorBiblioteca.sln --no-restore
dotnet test BlazorBiblioteca.sln --no-build
dotnet dev-certs https --trust
dotnet run --project src/BlazorBiblioteca --launch-profile https
```

Acepto la confianza del certificado de desarrollo cuando Windows lo solicita y abro `https://localhost:7149`. La terminal del servidor debe seguir abierta mientras uso la aplicación.

## Preparar macOS

### Instalar .NET con asdf

Para manejar el SDK en el Mac uso asdf y dejo la versión definida por proyecto. Estos pasos asumen que [asdf ya está configurado en la terminal](https://asdf-vm.com/guide/getting-started.html), con su carpeta de shims en el `PATH`.

Agrego el [plugin `dotnet`](https://github.com/hensou/asdf-dotnet) una sola vez. Si aparece en `asdf plugin list`, puedo saltar este comando:

```sh
asdf plugin add dotnet https://github.com/hensou/asdf-dotnet.git
```

Desde `evaluacion-2`, instalo y selecciono el SDK:

```sh
asdf install dotnet 8.0.427
asdf set dotnet 8.0.427
asdf current dotnet
dotnet --version
```

El último comando debe mostrar `8.0.427`. Si no coincide, reviso que `command -v dotnet` apunte al shim de asdf. `asdf set` guarda la selección en el `.tool-versions` de esa carpeta:

```text
dotnet 8.0.427
```

Conviene guardar ese archivo junto al código. En otro equipo con asdf y el plugin preparados, `asdf install` instala la versión declarada. asdf selecciona la instalación y `global.json` limita el SDK que acepta la solución; ambos quedan alineados con .NET 8.

### Levantar SQL Server

Necesito Docker con Compose y un motor activo, por ejemplo Docker Desktop o Colima. En Mac ARM, el motor debe poder ejecutar imágenes `linux/amd64` mediante emulación. Compruebo que Docker responde:

```sh
docker info
docker compose version
```

Para una instalación nueva que se usará solamente desde el Mac, ejecuto:

```sh
bash scripts/iniciar-sql-mac.sh
```

El script crea `.env` con una contraseña aleatoria para `sa`, publica SQL en `127.0.0.1:14333`, espera a que el motor responda y ejecuta los dos scripts SQL. Los datos quedan en el volumen `biblioteca_sql`. La contraseña de `sa` permanece en `.env`; Blazor usa el login académico `AdminBiblioteca`.

Si también voy a conectar Windows, en el primer arranque indico la IP privada del Mac que Windows pueda alcanzar. Por ejemplo, si esa dirección fuera `192.168.1.50`:

```sh
BIBLIOTECA_BIND_IP=192.168.1.50 bash scripts/iniciar-sql-mac.sh
```

Esa IP es un ejemplo: hay que sustituirla por la del equipo. La publicación debe quedar limitada a la red del laboratorio y el firewall debe permitir la conexión desde Windows. Con Colima también hay que comprobar el reenvío del puerto desde su máquina virtual al Mac; que el contenedor esté activo no demuestra por sí solo que Windows lo alcance.

Si `.env` ya existe, el script conserva sus valores: pasar otra IP en el comando no reemplaza la guardada. Para cambiarla, edito únicamente `BIBLIOTECA_BIND_IP` en `.env` y vuelvo a ejecutar el script. No elimino el archivo ni el volumen para repetir la práctica. Reviso el resultado con:

```sh
docker compose ps
cat evidencias/sql/conexion.json
```

Si el motor no arranca, el script deja el diagnóstico en `evidencias/sql/arranque-error.txt`. En ese caso hay que resolver la compatibilidad de la emulación o usar un SQL Server x64 accesible antes de probar el formulario.

### Conectar y ejecutar Blazor

Creo `src/BlazorBiblioteca/appsettings.Local.json` con la dirección publicada por Docker. Para el arranque local por defecto:

```json
{
  "ConnectionStrings": {
    "BibliotecaDB": "Server=127.0.0.1,14333;Database=BibliotecaDB;User Id=AdminBiblioteca;Password=Admin12345;Encrypt=True;TrustServerCertificate=True;Connect Timeout=5"
  }
}
```

Si publiqué SQL en la IP privada del Mac para compartirlo con Windows, uso esa dirección en lugar de `127.0.0.1`. No reutilizo una dirección antigua sin comprobar el contenedor actual.

Desde la misma carpeta compilo, pruebo y levanto la aplicación:

```sh
dotnet restore BlazorBiblioteca.sln --locked-mode
dotnet build BlazorBiblioteca.sln --no-restore
dotnet test BlazorBiblioteca.sln --no-build
dotnet dev-certs https --trust
dotnet run --project src/BlazorBiblioteca --launch-profile https
```

Acepto en macOS la confianza del certificado de desarrollo y abro `https://localhost:7149`. El navegador del Mac consulta la misma API y la misma base que usaría Windows; el código no necesita una variante por sistema operativo.

## Cómo se conectan las piezas


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

## Comprobar el recorrido completo

Las pruebas automatizadas cubren el CRUD de la API, datos inválidos, IDs manipulados, registros inexistentes y el esquema generado por el proveedor SQL Server. Los nueve casos pasaron tanto en la ejecución original de Windows como en la revisión de macOS del 10 de noviembre de 2026.

Las pruebas de la API usan SQLite en memoria para aislar los datos. La prueba del esquema genera SQL sin conectarse al motor. Para comprobar la persistencia real, también hice el recorrido desde el navegador y consulté SQL Server directamente.

Estos son los datos iniciales de la pauta:

| Libro | Autor | Páginas | Publicación |
| --- | --- | ---: | --- |
| El ojo del mundo | Robert Jordan | 824 | 15/01/1990 |
| Orgullo o prejuicio | Jane Austen | 500 | 28/01/1813 |
| Juego de tronos | George R.R. Martin | 800 | 01/08/1996 |

Sí, el enunciado decía «Orgullo o prejuicio». Dejé ese nombre para mantener los datos del ejercicio.

Para repetir la demostración en una base de práctica vacía:

1. Abro la aplicación y compruebo el mensaje de catálogo vacío con conexión válida.
2. Envío el formulario sin nombre ni autor, con cero páginas y luego con fecha vacía para comprobar las validaciones.
3. Agrego los tres libros y capturo el mensaje de ingreso y la lista con sus IDs.
4. Recargo la página y verifico que los registros siguen presentes.
5. Pulso EDITAR en El ojo del mundo, pruebo cancelar una edición y después guardo el cambio de 824 a 825 páginas. El ID debe mantenerse.
6. Elimino Juego de tronos y compruebo el mensaje y su ausencia en el catálogo.
7. Ejecuto `sql/02-verificar-biblioteca.sql` y comparo la consulta con la pantalla.

La base usada en la demostración ya conserva El ojo del mundo con 825 páginas y Orgullo o prejuicio con 500; el tercer libro fue eliminado. Antes de repetir el recorrido reviso los registros existentes para no duplicarlos ni borrar datos que quiera conservar. No hace falta vaciar la base para consultar el resultado final.

Si el catálogo no carga, reviso la salida del servidor, la dirección y el puerto SQL y el inicio de sesión de `AdminBiblioteca`. Si falta `BibliotecaDB` o `Libros`, compruebo que los scripts se hayan ejecutado en la instancia correcta. El mensaje de error de conexión no equivale a una lista vacía.

Con eso puedo seguir una operación de principio a fin: guardar desde el formulario, recargar, editar el mismo ID y consultar la base. Para una biblioteca de tres libros, alcanza para ver qué está haciendo cada pieza.
