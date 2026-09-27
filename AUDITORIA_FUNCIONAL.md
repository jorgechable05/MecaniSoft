# Auditoría funcional — MecaniSoft

**Fecha:** 2026-09-27  
**Entorno:** PHP 8.4.23 CLI + servidor PHP integrado  
**Base prevista:** MySQL/MariaDB (`db_mecanisoft`)  
**Código:** copia original `MecaniSoft-main`  
**Regla:** no se modificó la lógica del proyecto.

## Resumen ejecutivo

La aplicación puede arrancar y servir la pantalla de login, pero **no es posible ejecutar los flujos P0/P1 completos en este entorno** porque la instalación disponible de PHP no tiene la extensión `mysqli` y el proyecto depende de `mysqli_*` para conectarse a MySQL.

Se comprobó una falla real de ejecución al invocar el endpoint de autenticación: HTTP 500, seguido de `Call to undefined function mysqli_connect()` y posteriormente `Call to a member function query() on null`.

Por tanto, los resultados de esta fase deben interpretarse como una **auditoría funcional bloqueada por dependencia de infraestructura**, no como evidencia de que los módulos de negocio estén defectuosos.

## Clasificación

| Flujo | Estado | Evidencia / motivo |
|---|---|---|
| Login — página | 🟢 | `GET /vista/login.php` devuelve HTTP 200. |
| Login — autenticación | 🔴 | `POST controlador_usuario.php?opcion=verificarUsuario` devuelve HTTP 500 por ausencia de `mysqli`. |
| Sesiones | ⚪ | Requiere completar autenticación y conexión DB. |
| Roles/permisos | ⚪ | Requiere sesión + DB. |
| Clientes | ⚪ | Requiere DB. |
| Vehículos | ⚪ | Requiere DB. |
| Productos | ⚪ | Requiere DB. |
| Inventario | ⚪ | Requiere DB. |
| Compras/ingresos | ⚪ | Requiere DB. |
| Órdenes de trabajo | ⚪ | Requiere DB. |
| Tareas | ⚪ | Requiere DB. |
| Ventas | ⚪ | Requiere DB. |
| Anulaciones | ⚪ | Requiere DB. |
| Facturación | ⚪ | Requiere DB. |
| PDF | ⚪ | Requiere datos/DB para validar documentos reales. |
| Bitácora | ⚪ | Requiere DB y flujo autenticado. |

## Pruebas ejecutadas

### 1. Sintaxis PHP

- Archivos PHP analizados: **608**
- Errores de sintaxis encontrados: **0**
- Resultado: 🟢

Los archivos PHP pasan `php -l` en PHP 8.4.23, pero esto no garantiza compatibilidad funcional con todas las extensiones/dependencias ni con la versión de MySQL del proyecto.

### 2. Pantalla de login

`GET /vista/login.php` → HTTP **200**. Resultado: 🟢.

### 3. Autenticación

`POST /controlador/usuario/controlador_usuario.php?opcion=verificarUsuario` → HTTP **500**.

Mensaje observado: `Call to undefined function mysqli_connect()` y posteriormente `Call to a member function query() on null`.

Resultado: 🔴 **bloqueo de infraestructura**.

No se interpreta como fallo de las credenciales o del algoritmo de autenticación, porque la petición no alcanza una conexión válida con MySQL.

## Base de datos

El dump `database_developer.sql` contiene:

- **23 tablas**
- **139 procedimientos almacenados**
- **1 función** (`strSplit`)
- **3 triggers**

El análisis estático encontró **140 nombres `SP_*` utilizados por el código** frente a 139 procedimientos definidos. El nombre de negocio que requiere revisión es `SP_NUM_COMPROBANTE_VENTA`.

## Hallazgos de seguridad/robustez pendientes de validación dinámica

1. 🔒 **Creación de sesión:** el endpoint `crearSesion` recibe identificadores/rol mediante POST. Debe comprobarse que el servidor no confíe en valores manipulados por el cliente.
2. 🔒 **Carga de archivos:** existen usos de `move_uploaded_file()`. Deben validarse MIME real, extensión, tamaño, nombre generado y ubicación.
3. 🟠 **Sesión:** debe comprobarse el uso de `session_regenerate_id()` después de autenticación.
4. 🟠 **CSRF:** no se identificó una protección CSRF centralizada en la revisión estática.
5. 🟠 **SQL:** varios modelos construyen llamadas `CALL SP_...` dinámicamente. Debe comprobarse escape/validación de parámetros.
6. 🟠 **Errores de conexión:** el fallo actual expone detalles internos al cliente, lo cual no debería ocurrir en producción.

## Qué falta para completar P0/P1

Se necesita un runtime con:

- PHP con `mysqli` habilitado.
- MySQL/MariaDB compatible.
- Importación del `database_developer.sql`.
- Permisos para crear/usar `db_mecanisoft`.
- Dependencias PHP necesarias para PDF y correo.

Después se ejecutarán pruebas aisladas con datos de prueba y se registrará para cada flujo:

**entrada → endpoint → controlador → modelo → procedimiento → resultado DB → respuesta HTTP/UI → efectos secundarios**.

## Decisión

**No se modificó el código fuente.** El bloqueo actual es de infraestructura de prueba. El siguiente paso correcto es preparar el runtime MySQL/MariaDB con `mysqli` y repetir exactamente los flujos P0/P1.
