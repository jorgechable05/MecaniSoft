# MecaniSoft — Mapa funcional y auditoría inicial

Análisis estático del código fuente proporcionado.

## Inventario

- 2600 archivos
- 608 PHP
- 504 JavaScript
- 27 controladores de negocio
- 24 modelos
- 31 vistas PHP
- 23 tablas detectadas en `db/database_developer.sql`
- Los PHP invocan múltiples procedimientos `SP_*`; el dump analizado no contiene `CREATE PROCEDURE`, por lo que debe verificarse que exista un dump adicional con los procedimientos.

## Arquitectura

`Vista → Controlador → Modelo → MySQL / Stored Procedures`

El proyecto usa PHP clásico con separación entre vistas, controladores y modelos. La persistencia se apoya fuertemente en procedimientos almacenados.

## Módulos funcionales

- Acceso: autenticación y permisos por módulo.
- Usuarios: cuentas, perfiles, contraseñas y widgets/dashboard.
- Roles: administración de roles.
- Bitácora: registro de acciones y accesos.
- Personas: datos personales y credenciales.
- Clientes: alta, edición y estado.
- Proveedores: proveedores y contactos.
- Vehículos: vehículos asociados a clientes.
- Productos: catálogo, códigos, ofertas e imágenes.
- Servicios: catálogo, ofertas e imágenes.
- Categorías, marcas, modelos, fabricantes y unidades de medida.
- Movimientos/inventario: movimientos y detalles.
- Ingresos: compras/ingresos a inventario y anulación.
- Cotizaciones: cabecera, detalle, edición y anulación.
- Órdenes: órdenes de trabajo, servicios/productos, tareas y facturación.
- Tareas: asignación/seguimiento de tareas.
- Ventas: ventas, detalle, numeración y anulación.
- Configuración: datos de la empresa.
- ISV/impuesto.
- Reportes PDF mediante MPDF.

## Flujo de negocio observado

### Cliente → vehículo → orden
Los modelos incluyen operaciones para registrar vehículos por cliente y las órdenes pueden asociar servicios/productos y tareas.

### Proveedor → ingreso → inventario
Los ingresos registran proveedor, comprobante, productos, cantidades y precios; existen operaciones para registrar/anular ingresos.

### Cotización → orden / venta
Las cotizaciones tienen cabecera y detalle. Las órdenes permiten registrar servicios/productos y contienen una operación `facturar`.

### Venta → detalle → reporte
Las ventas tienen cabecera/detalle, numeración, anulación y reportes PDF.

## Base de datos

Tablas detectadas:

`acceso`, `bitacora`, `bitacora_marca`, `categoria`, `cliente`, `configuracion`, `detalle_transaccion`, `fabricante`, `marca`, `modelo`, `modulo`, `persona`, `producto`, `producto_historial`, `proveedor`, `rol`, `tarea`, `transaccion_proveedor`, `transaccion_vehiculo`, `transacciones`, `unidad_medida`, `usuario`, `vehiculo`.

## Seguridad — hallazgos iniciales

### CRÍTICO
`conexion_global/r_conexion.php` contiene usuario MySQL `root` con contraseña vacía para el entorno de desarrollo. No debe utilizarse así en producción.

### ALTO
El README contiene credenciales de inicio de sesión de ejemplo. Si el repositorio es público, deben eliminarse/rotarse.

### ALTO
Los modelos construyen llamadas a procedimientos almacenados mediante interpolación de variables dentro de strings SQL. Aunque los SP pueden limitar parte del riesgo, es recomendable parametrizar/validar entradas y revisar cada procedimiento.

### MEDIO
Hay código que imprime directamente mensajes de excepción al usuario. En producción debe registrarse internamente y mostrar mensajes genéricos.

### POSITIVO
`modelo_usuario.php` utiliza `password_verify()` para comprobar contraseñas, lo que indica uso de hashes en el flujo de autenticación.

## Prioridades

1. Retirar credenciales del repositorio y usar variables de entorno.
2. Obtener/identificar el dump completo de procedimientos almacenados.
3. Auditar autorización por rol y módulo.
4. Revisar SQL injection, XSS y CSRF.
5. Revisar transacciones para ventas, inventario, órdenes y anulaciones.
6. Añadir pruebas automatizadas.
7. Documentar API/controladores/modelos.
8. Separar dependencias de terceros del código propio.

## Siguiente fase

Ejecutar MecaniSoft con PHP/MySQL y probar dinámicamente:

`login → permisos → cliente → vehículo → orden → tarea → inventario → venta/facturación → PDF → bitácora`

El objetivo es contrastar el comportamiento real contra este mapa estático y detectar errores funcionales que no pueden confirmarse leyendo solamente el código.
