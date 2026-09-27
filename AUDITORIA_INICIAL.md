# MecaniSoft — Auditoría inicial

## Inventario
- Archivos totales del ZIP: **2600**
- PHP totales: **608**
- JavaScript totales: **504**
- PHP de aplicación (excluyendo dependencias): **94**
- Controladores: **27**
- Modelos: **24**
- Vistas PHP: **31**
- Funciones PHP detectadas en código de aplicación: **340**

## Arquitectura detectada
El proyecto usa una arquitectura PHP clásica separada en `controlador/`, `modelo/` y `vista/`, con JavaScript por módulo. También contiene `MPDF/` para generación de documentos PDF y `db/database_developer.sql` para la base de datos.

## Módulos de negocio detectados
PHPMailer, acceso, bitacora, categoria, cliente, configuracion, cotizacion, fabricante, ingreso, isv, marca, modelo, movimientos, orden, persona, producto, proveedor, rol, servicio, tareas, unidadmedida, usuario, vehiculo, venta

## Acceso y configuración
El README original indica una base de datos `db_mecanisoft` y configuración de conexión en `conexion_global/r_conexion.php` y `modelo/modelo_conexion.php`. **Las credenciales del README original no se reproducen aquí por seguridad.**

## Riesgos/áreas a revisar
1. Credenciales de conexión y secretos almacenados en código/configuración.
2. Autenticación, sesiones y autorización por rol.
3. Consultas SQL y posible inyección SQL.
4. Validación/escape de entradas y salidas HTML.
5. Subida/lectura de archivos y generación de PDF.
6. Protección CSRF y controles de acceso directos a vistas/controladores.
7. Registro de bitácora y trazabilidad de operaciones.
8. Integridad transaccional de ventas, ingresos, órdenes y movimientos.
9. Dependencias incluidas dentro del repositorio y archivos generados de gran tamaño.

## Próximo análisis recomendado
- Mapear cada módulo controlador → modelo → vista → JavaScript.
- Reconstruir el esquema de `database_developer.sql` y sus relaciones.
- Identificar funciones CRUD, procesos de venta/orden/ingreso y reportes.
- Ejecutar una auditoría de seguridad estática sobre autenticación, SQL, sesiones y permisos.
- Preparar una matriz de funciones existentes y funciones reutilizables para una futura modernización.

> Este documento corresponde a una revisión inicial del código original. No se ha modificado la lógica funcional del proyecto.
