# MecaniSoft — Auditoría funcional fase 1

## Alcance
Auditoría estática/funcional sobre el código fuente original `MecaniSoft-main`. No se realizaron cambios a la lógica del sistema.

> Estado: **análisis de código completado; ejecución end-to-end pendiente** porque el entorno no tiene un servidor MySQL/MariaDB configurado con `db_mecanisoft` y sus procedimientos almacenados.

## 1. Resultado de sintaxis
- PHP 8.4.23 disponible.
- Archivos PHP del proyecto (excluyendo `MPDF/vendor`) pasaron `php -l` sin errores de sintaxis.
- El proyecto depende de MySQL/MariaDB mediante `mysqli` y procedimientos almacenados.

## 2. Mapa funcional

| Módulo | Controlador | Modelo | BD/SP | Estado estático |
|---|---|---|---|---|
| Login/usuarios | `controlador/usuario/` | `modelo_usuario.php` | Sí | Implementado |
| Permisos/accesos | `controlador/acceso/` | `modelo_acceso.php` | Sí | Implementado |
| Bitácora | `controlador/bitacora/` | `modelo_bitacora.php` | Sí | Implementado |
| Clientes | `controlador/cliente/` | `modelo_cliente.php` | Sí | Implementado |
| Personas | `controlador/persona/` | `modelo_persona.php` | Sí | Implementado |
| Vehículos | `controlador/vehiculo/` | `modelo_vehiculo.php` | Sí | Implementado |
| Productos | `controlador/producto/` | `modelo_producto.php` | Sí | Implementado |
| Servicios | `controlador/servicio/` | `modelo_servicio.php` | Sí | Implementado |
| Inventario/movimientos | `controlador/movimientos/` | `modelo_movimientos.php` | Sí | Implementado |
| Proveedores | `controlador/proveedor/` | `modelo_proveedor.php` | Sí | Implementado |
| Ingresos/compras | `controlador/ingreso/` | `modelo_ingreso.php` | Sí | Implementado |
| Cotizaciones | `controlador/cotizacion/` | `modelo_cotizacion.php` | Sí | Implementado |
| Órdenes de trabajo | `controlador/orden/` | `modelo_orden.php` | Sí | Implementado |
| Tareas | `controlador/tareas/` | `modelo_tarea.php` | Sí | Implementado |
| Ventas | `controlador/venta/` | `modelo_venta.php` | Sí | Implementado |
| Configuración | `controlador/configuracion/` | `modelo_configuracion.php` | Sí | Implementado |
| Roles | `controlador/rol/` | `modelo_rol.php` | Sí | Implementado |
| Catálogos | categoría/marca/modelo/fabricante/unidad | modelos correspondientes | Sí | Implementado |
| PDF/reportes | `MPDF/` | varios | Sí | Implementado |

## 3. Flujo principal

```text
vista/*.php + js/*.js
        ↓ AJAX / POST
controlador/*/controlador_*.php
        ↓
modelo/modelo_*.php
        ↓ mysqli->query("CALL SP_...")
MySQL/MariaDB
        ↓
resultado JSON/HTML
        ↓
vista / DataTables / SweetAlert
```

## 4. Login y permisos

### Lo que funciona según el código
- Contraseñas nuevas: `password_hash(..., PASSWORD_DEFAULT)`.
- Login: `password_verify()` contra el hash recuperado.
- Al crear sesión se cargan `S_IDUSUARIO`, `S_USUARIO`, `S_ROL`, `S_ACCESOS`.
- Se registra la entrada en bitácora.
- La interfaz oculta módulos según `S_ACCESOS`.

### Hallazgo crítico
`controlador/usuario/controlador_usuario.php`, operación `crearSesion`, recibe por POST `idusuario`, `usuario` y `rol` y los coloca directamente en sesión después de consultar accesos para ese `idusuario`.

Esto significa que **la autorización no debe considerarse segura solo por ocultar opciones en la interfaz**. Un atacante que pueda llamar directamente al endpoint podría intentar manipular esos identificadores. Debe validarse en servidor que el usuario autenticado corresponde al registro que se está estableciendo en sesión y que sus permisos provienen exclusivamente de BD.

### Otros puntos
- No se encontró `session_regenerate_id()` durante el establecimiento de sesión.
- No se identificó una capa central de autorización en todos los controladores; varios endpoints dependen implícitamente de sesión/interfaz.
- No se identificó protección CSRF sistemática.

## 5. Base de datos

El `database_developer.sql` contiene:
- 23 tablas.
- 139 procedimientos almacenados definidos.
- 134 procedimientos almacenados referenciados por el código.
- Solo **1 procedimiento referenciado no fue localizado por nombre exacto:** `SP_NUM_COMPROBANTE_VENTA`.

El dump sí contiene los procedimientos almacenados. La siguiente prueba debe importar el dump en una instancia limpia y ejecutar cada flujo.

## 6. Ventas / órdenes / inventario

Se observan operaciones para:
- registrar y anular ventas;
- registrar ingresos de proveedores;
- movimientos manuales de inventario;
- registrar órdenes de trabajo;
- agregar productos/servicios a órdenes;
- tareas de órdenes;
- facturar órdenes;
- cotizaciones;
- generación de número/serie de comprobantes.

La integridad real de estas operaciones debe probarse con transacciones/concurrencia directamente en MySQL, porque gran parte de la lógica está encapsulada en procedimientos almacenados.

## 7. Carga de archivos — riesgo alto

Se encontraron varios `move_uploaded_file()` en:
- productos;
- servicios;
- usuarios;
- vehículos;
- configuración.

En los fragmentos revisados se usa el nombre enviado por el cliente y no se observó una validación robusta de:
- MIME real;
- extensión permitida;
- tamaño máximo;
- contenido de imagen;
- nombre aleatorio seguro;
- almacenamiento fuera del directorio ejecutable.

Debe auditarse antes de exponer el sistema públicamente.

## 8. SQL

La aplicación utiliza procedimientos almacenados, lo cual centraliza una parte importante de la lógica. Sin embargo, los parámetros se interpolan directamente en strings como:

```php
$sql = "call SP_REGISTRAR_PRODUCTO('$producto', ... )";
```

Aunque exista `htmlspecialchars()`, **eso no sustituye consultas parametrizadas ni constituye protección SQL**. El riesgo final depende de cómo estén implementados los procedimientos y del modo en que MySQL procese esos argumentos.

Debe migrarse progresivamente a parámetros preparados o a una capa de acceso segura.

## 9. PDF

Se detectaron reportes para:
- ventas;
- órdenes;
- facturas de órdenes;
- ingresos;
- cotizaciones;
- historial de productos;
- historial de servicios;
- movimientos;
- estadísticas de ventas;
- códigos de barras.

Usa mPDF y paquetes relacionados mediante Composer.

## 10. Bitácora

Hay integración explícita de bitácora en varias operaciones: login, altas, ediciones, anulaciones y cambios de permisos. Esto es positivo.

Pendiente: comprobar que la bitácora no pueda falsificarse mediante parámetros enviados por el cliente y que registre correctamente usuario, IP, acción, entidad y fecha.

## 11. Priorización de pruebas reales

### P0 — seguridad
1. Intentar crear sesión con `idusuario` de otro usuario.
2. Intentar modificar `rol` por POST.
3. Acceder directamente a controladores sin sesión.
4. Probar CSRF en operaciones de escritura.
5. Probar subida de PHP/HTML/SVG/archivos maliciosos.
6. Probar manipulación de IDs de clientes, órdenes, ventas e inventario.

### P1 — transacciones
1. Registrar compra y comprobar inventario.
2. Registrar venta y comprobar decremento de inventario.
3. Anular venta y comprobar reversión.
4. Crear orden con productos/servicios.
5. Completar tareas y facturar orden.
6. Ejecutar dos ventas concurrentes del mismo stock.

### P2 — documentos
1. Generar PDF de venta.
2. Generar PDF de orden.
3. Generar factura.
4. Generar cotización.
5. Validar totales, impuestos, descuentos y numeración.

## 12. Conclusión de fase 1

**MecaniSoft sí tiene una arquitectura funcional completa**, no es únicamente una maqueta. La lógica de negocio está fuertemente apoyada en procedimientos almacenados y cubre el ciclo de taller desde clientes/vehículos hasta órdenes, tareas, inventario, ventas y documentos.

Antes de modificar o modernizar el sistema, la prioridad es ejecutar una instancia limpia y validar los flujos críticos. Los principales riesgos preliminares son **autorización de sesión, carga de archivos, ausencia de una capa CSRF central y construcción de SQL mediante interpolación**.
