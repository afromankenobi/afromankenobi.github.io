---
layout: post
title: "Slash codes de SAP GUI para tener a mano"
tags: [sap, sap-gui, abap]
---

Si estoy editando un programa en `SE38` y necesito consultar una tabla en el Diccionario ABAP, escribo `/oSE11`. El diccionario se abre en otra sesión y el editor queda donde estaba. Con `/nSE11`, en cambio, termino la transacción del editor y abro el diccionario en esa misma sesión.

## Cambiar de transacción

Escribe el comando en el **campo de comandos de SAP GUI** y pulsa **Enter**. El prefijo y el código de transacción van juntos, sin espacios.

| Quiero… | Escribo | Qué ocurre |
|---|---|---|
| Volver al inicio | `/n` | Termina la transacción actual. La sesión sigue abierta. |
| Cambiar de tarea en la misma sesión | `/nSE11` | Termina la transacción actual y abre `SE11`. Guarda antes los cambios que quieras conservar. |
| Consultar algo en paralelo | `/oSE11` | Abre `SE11` en otra sesión y conserva la pantalla actual. |

Puedes sustituir `SE11` por el código que necesites. Si el código ya empieza con una barra, consérvala. Por ejemplo, para consultar el registro de errores de SAP Gateway en otra sesión, escribe `/o/IWFND/ERROR_LOG`. La primera barra pertenece al comando `/o` y la segunda al nombre de la transacción. [Códigos de transacción en SAP GUI](https://help.sap.com/docs/sap_netweaver_702/fe1b9b0e6c5510149c8680d5b2058e7f/f9e1a442dc030e31e10000000a1550b0.html) · [Registro de errores de Gateway](https://help.sap.com/docs/SUPPORT_CONTENT/erphcm/3354686138.html)

## Ver y cerrar sesiones

| Quiero… | Escribo | Qué ocurre |
|---|---|---|
| Ver qué sesiones tengo abiertas | `/o` | Muestra la lista de sesiones del sistema. Desde ahí puedes crear o cerrar una. |
| Cerrar la sesión en la que estoy | `/i` | Cierra la sesión actual. Si es la última, sales del sistema. |
| Cerrar otra sesión por número | `/i3` | Cierra la sesión 3. Comprueba el número con `/o` antes de ejecutarlo. |
| Salir del sistema con confirmación | `/nend` | Pide confirmación para cerrar todas las sesiones de esa conexión. |
| Salir del sistema de inmediato | `/nex` | **Cierra todas las sesiones de esa conexión sin preguntar. Se pierden los cambios sin guardar.** |

Si abriste el diccionario con `/oSE11`, puedes volver al editor cerrando la ventana del diccionario con `/i`. [Gestión de sesiones en SAP GUI](https://help.sap.com/docs/SAP_NETWEAVER_740/9737050ef01843f19572591b42128f1b/9b9efa42d7d448f68bf6a8baddccbc8f.html)

## Depurar una acción ABAP

En una pantalla clásica de SAP GUI (Dynpro), escribe el comando, pulsa **Enter** y realiza la acción que quieres investigar. Por ejemplo, para seguir la lógica de un informe desde su pantalla de selección, introduce `/h` y luego pulsa **Ejecutar**. Necesitas autorización de depuración.

| Comando | Dónde empieza la depuración |
|---|---|
| `/h` | En la primera instrucción del procesamiento de la pantalla. |
| `/ha` | En la primera instrucción ABAP, saltando la lógica de flujo de la pantalla. |
| `/hs` | Activa la depuración de sistema para poder seguir también el código de sistema. |

SAP detalla los puntos de entrada de los tres comandos en su [guía del depurador ABAP para pantallas Dynpro](https://help.sap.com/saphelp_SNC700_ehp01/helpdata/en/e4/fc840c8c09403c87501c68f80fa716/content.htm).
