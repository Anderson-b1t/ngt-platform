# Modelo Físico de Base de Datos

**Proyecto:** NGT Platform (Neo Genesis Technology)
**Versión:** 1.0
**Motor:** Microsoft SQL Server
**Estado:** Final
**Origen:** Modelo lógico v1.0 y modelo ER v1.0

---

# 1. Convenciones generales

## Identificadores

Las entidades propias del dominio utilizarán:

* `int IDENTITY(1,1)` como identificador principal.

Los usuarios y roles administrados mediante ASP.NET Core Identity utilizarán:

* `uniqueidentifier`.

Las especializaciones de `Producto` reutilizarán `ProductoId` como:

* clave primaria;
* clave foránea hacia su entidad padre.

---

## Fechas

Todas las fechas utilizarán:

`datetime2`

No se utilizará `datetime`.

Las fechas de creación generadas por el sistema podrán utilizar:

`SYSUTCDATETIME()`

como valor predeterminado.

---

## Valores monetarios

Los importes monetarios utilizarán:

`decimal(12,2)`

Aplicable a:

* precios;
* subtotales;
* costo de envío;
* totales;
* pagos;
* precio base de servicios.

---

## Texto

Se utilizará `nvarchar` para información textual.

`nvarchar(max)` se reservará únicamente para contenido cuya longitud pueda ser considerable.

---

## Estados

Los estados iniciales se almacenarán mediante `nvarchar` y restricciones `CHECK`.

Esto mantiene el modelo simple y evita crear tablas de catálogo innecesarias para estados pequeños y estables.

---

## Eliminación

La información histórica o transaccional no será eliminada físicamente durante el funcionamiento normal.

Cuando corresponda se utilizarán:

* `Activo`;
* `Activa`;
* `Estado`.

Las relaciones históricas utilizarán principalmente `NO ACTION`.

---

# 2. ASP.NET Core Identity

NGT Platform utilizará ASP.NET Core Identity para autenticación y autorización.

El proyecto utilizará `uniqueidentifier` como identificador de usuario y rol.

No se crearán tablas propias equivalentes a:

* Usuario;
* Rol;
* UsuarioRol.

Identity administrará sus tablas de autenticación y autorización.

La entidad de usuario de la aplicación extenderá al usuario de Identity con los siguientes campos propios:

| Campo         | Tipo             | Restricción                        |
| ------------- | ---------------- | ---------------------------------- |
| Id            | uniqueidentifier | PK                                 |
| Nombre        | nvarchar(150)    | NOT NULL                           |
| Estado        | nvarchar(20)     | NOT NULL                           |
| FechaRegistro | datetime2        | NOT NULL, DEFAULT SYSUTCDATETIME() |

`Email`, contraseña, tokens, bloqueo y demás información de autenticación serán administrados por Identity.

CHECK:

`Estado IN ('Activo', 'Inactivo', 'Bloqueado')`

Roles iniciales:

* Usuario
* Administrador

---

# 3. DireccionUsuario

| Campo              | Tipo             | Restricción         |
| ------------------ | ---------------- | ------------------- |
| DireccionUsuarioId | int              | PK, IDENTITY        |
| UsuarioId          | uniqueidentifier | FK, NOT NULL        |
| Alias              | nvarchar(80)     | NULL                |
| NombreDestinatario | nvarchar(150)    | NOT NULL            |
| Telefono           | nvarchar(25)     | NOT NULL            |
| LineaDireccion1    | nvarchar(200)    | NOT NULL            |
| LineaDireccion2    | nvarchar(200)    | NULL                |
| Pais               | nvarchar(100)    | NOT NULL            |
| Region             | nvarchar(100)    | NOT NULL            |
| Provincia          | nvarchar(100)    | NOT NULL            |
| Distrito           | nvarchar(100)    | NOT NULL            |
| CodigoPostal       | nvarchar(20)     | NULL                |
| Referencia         | nvarchar(300)    | NULL                |
| EsPredeterminada   | bit              | NOT NULL, DEFAULT 0 |
| Activa             | bit              | NOT NULL, DEFAULT 1 |

FK:

`UsuarioId → AspNetUsers.Id`

Índices:

* índice por `UsuarioId`;
* índice único filtrado por `UsuarioId` cuando:

`EsPredeterminada = 1 AND Activa = 1`

Esto garantiza como máximo una dirección predeterminada activa por usuario.

---

# 4. Categoria

| Campo       | Tipo          | Restricción         |
| ----------- | ------------- | ------------------- |
| CategoriaId | int           | PK, IDENTITY        |
| Nombre      | nvarchar(120) | NOT NULL            |
| Descripcion | nvarchar(500) | NULL                |
| Activa      | bit           | NOT NULL, DEFAULT 1 |

Restricción:

`UNIQUE (Nombre)`

---

# 5. Producto

| Campo       | Tipo           | Restricción         |
| ----------- | -------------- | ------------------- |
| ProductoId  | int            | PK, IDENTITY        |
| CategoriaId | int            | FK, NOT NULL        |
| Nombre      | nvarchar(200)  | NOT NULL            |
| Descripcion | nvarchar(2000) | NULL                |
| Precio      | decimal(12,2)  | NOT NULL            |
| Stock       | int            | NOT NULL            |
| Activo      | bit            | NOT NULL, DEFAULT 1 |

FK:

`CategoriaId → Categoria.CategoriaId`

CHECK:

`Precio >= 0`

`Stock >= 0`

Índices:

* `CategoriaId`;
* `Nombre`;
* `Activo`.

La exclusividad entre `ComponentePC` y `Publicacion` será una regla del dominio.

---

# 6. Publicacion

| Campo           | Tipo          | Restricción |
| --------------- | ------------- | ----------- |
| ProductoId      | int           | PK, FK      |
| TipoPublicacion | nvarchar(20)  | NOT NULL    |
| Autor           | nvarchar(200) | NULL        |
| Editorial       | nvarchar(150) | NULL        |
| ISBN            | nvarchar(20)  | NULL        |
| Volumen         | int           | NULL        |

FK:

`ProductoId → Producto.ProductoId`

CHECK:

`TipoPublicacion IN ('Comic', 'Manga')`

CHECK:

`Volumen IS NULL OR Volumen > 0`

Índice único filtrado:

`ISBN`

cuando:

`ISBN IS NOT NULL`

---

# 7. ComponentePC

| Campo      | Tipo          | Restricción |
| ---------- | ------------- | ----------- |
| ProductoId | int           | PK, FK      |
| Marca      | nvarchar(100) | NOT NULL    |
| Modelo     | nvarchar(150) | NOT NULL    |

FK:

`ProductoId → Producto.ProductoId`

El subtipo técnico será determinado por la especialización correspondiente.

Un componente debe pertenecer exactamente a uno de:

* Procesador;
* PlacaMadre;
* MemoriaRAM;
* TarjetaGrafica;
* Almacenamiento;
* FuentePoder;
* Gabinete;
* Refrigeracion.

La exclusividad se garantizará mediante la lógica de dominio.

---

# 8. Procesador

| Campo                   | Tipo         | Restricción |
| ----------------------- | ------------ | ----------- |
| ProductoId              | int          | PK, FK      |
| Socket                  | nvarchar(30) | NOT NULL    |
| TdpWatts                | int          | NOT NULL    |
| ConsumoEstimadoWatts    | int          | NOT NULL    |
| TieneGraficosIntegrados | bit          | NOT NULL    |
| IncluyeRefrigeracion    | bit          | NOT NULL    |

FK:

`ProductoId → ComponentePC.ProductoId`

CHECK:

`TdpWatts > 0`

`ConsumoEstimadoWatts > 0`

Índice:

`Socket`

---

# 9. PlacaMadre

| Campo                    | Tipo         | Restricción |
| ------------------------ | ------------ | ----------- |
| ProductoId               | int          | PK, FK      |
| Socket                   | nvarchar(30) | NOT NULL    |
| Chipset                  | nvarchar(50) | NOT NULL    |
| FormatoPlaca             | nvarchar(30) | NOT NULL    |
| TipoMemoria              | nvarchar(20) | NOT NULL    |
| CantidadSlotsMemoria     | tinyint      | NOT NULL    |
| CapacidadMaximaMemoriaGB | int          | NOT NULL    |
| CantidadSlotsM2          | tinyint      | NOT NULL    |
| CantidadPuertosSata      | tinyint      | NOT NULL    |
| ConsumoEstimadoWatts     | int          | NOT NULL    |

FK:

`ProductoId → ComponentePC.ProductoId`

CHECK:

`CantidadSlotsMemoria > 0`

`CapacidadMaximaMemoriaGB > 0`

`ConsumoEstimadoWatts > 0`

Índices:

* `Socket`;
* `Chipset`;
* `FormatoPlaca`;
* `TipoMemoria`.

`tinyint` ya impide valores negativos para slots y puertos.

---

# 10. CompatibilidadProcesadorPlaca

| Campo                     | Tipo          | Restricción         |
| ------------------------- | ------------- | ------------------- |
| ProductoIdProcesador      | int           | PK, FK              |
| ProductoIdPlacaMadre      | int           | PK, FK              |
| RequiereActualizacionBios | bit           | NOT NULL, DEFAULT 0 |
| Observacion               | nvarchar(500) | NULL                |

PK compuesta:

`ProductoIdProcesador + ProductoIdPlacaMadre`

FK:

`ProductoIdProcesador → Procesador.ProductoId`

`ProductoIdPlacaMadre → PlacaMadre.ProductoId`

Índice adicional:

`ProductoIdPlacaMadre`

---

# 11. MemoriaRAM

| Campo                | Tipo         | Restricción |
| -------------------- | ------------ | ----------- |
| ProductoId           | int          | PK, FK      |
| TipoMemoria          | nvarchar(20) | NOT NULL    |
| CapacidadTotalGB     | int          | NOT NULL    |
| ModulosPorKit        | tinyint      | NOT NULL    |
| FrecuenciaMTs        | int          | NOT NULL    |
| ConsumoEstimadoWatts | int          | NOT NULL    |

FK:

`ProductoId → ComponentePC.ProductoId`

CHECK:

`CapacidadTotalGB > 0`

`ModulosPorKit > 0`

`FrecuenciaMTs > 0`

`ConsumoEstimadoWatts > 0`

Índice:

`TipoMemoria`

---

# 12. TarjetaGrafica

| Campo                          | Tipo | Restricción |
| ------------------------------ | ---- | ----------- |
| ProductoId                     | int  | PK, FK      |
| LongitudMm                     | int  | NOT NULL    |
| ConsumoEstimadoWatts           | int  | NOT NULL    |
| PotenciaFuenteRecomendadaWatts | int  | NOT NULL    |

FK:

`ProductoId → ComponentePC.ProductoId`

CHECK:

`LongitudMm > 0`

`ConsumoEstimadoWatts > 0`

`PotenciaFuenteRecomendadaWatts > 0`

---

# 13. Almacenamiento

| Campo                | Tipo         | Restricción |
| -------------------- | ------------ | ----------- |
| ProductoId           | int          | PK, FK      |
| TipoAlmacenamiento   | nvarchar(20) | NOT NULL    |
| Interfaz             | nvarchar(30) | NOT NULL    |
| CapacidadGB          | int          | NOT NULL    |
| ConsumoEstimadoWatts | int          | NOT NULL    |

FK:

`ProductoId → ComponentePC.ProductoId`

CHECK:

`TipoAlmacenamiento IN ('HDD', 'SSD')`

CHECK:

`Interfaz IN ('SATA', 'M2NVMe')`

CHECK:

`CapacidadGB > 0`

CHECK:

`ConsumoEstimadoWatts > 0`

---

# 14. FuentePoder

| Campo         | Tipo         | Restricción |
| ------------- | ------------ | ----------- |
| ProductoId    | int          | PK, FK      |
| PotenciaWatts | int          | NOT NULL    |
| FormatoFuente | nvarchar(30) | NOT NULL    |

FK:

`ProductoId → ComponentePC.ProductoId`

CHECK:

`PotenciaWatts > 0`

Índice:

`FormatoFuente`

---

# 15. Gabinete

| Campo                       | Tipo | Restricción |
| --------------------------- | ---- | ----------- |
| ProductoId                  | int  | PK, FK      |
| LongitudMaximaGpuMm         | int  | NOT NULL    |
| AlturaMaximaRefrigeracionMm | int  | NOT NULL    |

FK:

`ProductoId → ComponentePC.ProductoId`

CHECK:

`LongitudMaximaGpuMm > 0`

CHECK:

`AlturaMaximaRefrigeracionMm > 0`

---

# 16. GabineteFormatoPlaca

| Campo              | Tipo         | Restricción |
| ------------------ | ------------ | ----------- |
| ProductoIdGabinete | int          | PK, FK      |
| FormatoPlaca       | nvarchar(30) | PK          |

PK:

`ProductoIdGabinete + FormatoPlaca`

FK:

`ProductoIdGabinete → Gabinete.ProductoId`

---

# 17. GabineteFormatoFuente

| Campo              | Tipo         | Restricción |
| ------------------ | ------------ | ----------- |
| ProductoIdGabinete | int          | PK, FK      |
| FormatoFuente      | nvarchar(30) | PK          |

PK:

`ProductoIdGabinete + FormatoFuente`

FK:

`ProductoIdGabinete → Gabinete.ProductoId`

---

# 18. GabineteRadiadorSoportado

| Campo              | Tipo     | Restricción |
| ------------------ | -------- | ----------- |
| ProductoIdGabinete | int      | PK, FK      |
| TamanoRadiadorMm   | smallint | PK          |

PK:

`ProductoIdGabinete + TamanoRadiadorMm`

FK:

`ProductoIdGabinete → Gabinete.ProductoId`

CHECK:

`TamanoRadiadorMm > 0`

---

# 19. Refrigeracion

| Campo                | Tipo         | Restricción |
| -------------------- | ------------ | ----------- |
| ProductoId           | int          | PK, FK      |
| TipoRefrigeracion    | nvarchar(20) | NOT NULL    |
| AlturaMm             | int          | NULL        |
| TamanoRadiadorMm     | smallint     | NULL        |
| ConsumoEstimadoWatts | int          | NOT NULL    |

FK:

`ProductoId → ComponentePC.ProductoId`

CHECK:

`TipoRefrigeracion IN ('Aire', 'LiquidaAIO')`

CHECK:

`ConsumoEstimadoWatts > 0`

CHECK conceptual:

Para refrigeración por aire:

* `AlturaMm IS NOT NULL`;
* `AlturaMm > 0`;
* `TamanoRadiadorMm IS NULL`.

Para refrigeración líquida AIO:

* `TamanoRadiadorMm IS NOT NULL`;
* `TamanoRadiadorMm > 0`;
* `AlturaMm IS NULL`.

---

# 20. RefrigeracionSocket

| Campo                   | Tipo         | Restricción |
| ----------------------- | ------------ | ----------- |
| ProductoIdRefrigeracion | int          | PK, FK      |
| Socket                  | nvarchar(30) | PK          |

PK:

`ProductoIdRefrigeracion + Socket`

FK:

`ProductoIdRefrigeracion → Refrigeracion.ProductoId`

---

# 21. Carrito

| Campo              | Tipo             | Restricción                        |
| ------------------ | ---------------- | ---------------------------------- |
| CarritoId          | int              | PK, IDENTITY                       |
| UsuarioId          | uniqueidentifier | FK, NOT NULL                       |
| Estado             | nvarchar(25)     | NOT NULL                           |
| FechaCreacion      | datetime2        | NOT NULL, DEFAULT SYSUTCDATETIME() |
| FechaActualizacion | datetime2        | NOT NULL, DEFAULT SYSUTCDATETIME() |

FK:

`UsuarioId → AspNetUsers.Id`

CHECK:

`Estado IN ('Activo', 'ConvertidoEnPedido', 'Abandonado')`

Índice:

`UsuarioId`

Índice único filtrado:

`UsuarioId`

cuando:

`Estado = 'Activo'`

Esto garantiza un único carrito activo por usuario.

---

# 22. DetalleCarrito

| Campo            | Tipo | Restricción  |
| ---------------- | ---- | ------------ |
| DetalleCarritoId | int  | PK, IDENTITY |
| CarritoId        | int  | FK, NOT NULL |
| ProductoId       | int  | FK, NOT NULL |
| Cantidad         | int  | NOT NULL     |

FK:

`CarritoId → Carrito.CarritoId`

`ProductoId → Producto.ProductoId`

CHECK:

`Cantidad > 0`

UNIQUE:

`CarritoId + ProductoId`

---

# 23. Pedido

| Campo       | Tipo             | Restricción                        |
| ----------- | ---------------- | ---------------------------------- |
| PedidoId    | int              | PK, IDENTITY                       |
| UsuarioId   | uniqueidentifier | FK, NOT NULL                       |
| Subtotal    | decimal(12,2)    | NOT NULL                           |
| CostoEnvio  | decimal(12,2)    | NOT NULL                           |
| Total       | decimal(12,2)    | NOT NULL                           |
| Estado      | nvarchar(25)     | NOT NULL                           |
| FechaPedido | datetime2        | NOT NULL, DEFAULT SYSUTCDATETIME() |

FK:

`UsuarioId → AspNetUsers.Id`

CHECK:

`Subtotal >= 0`

CHECK:

`CostoEnvio >= 0`

CHECK:

`Total >= 0`

CHECK:

`Total = Subtotal + CostoEnvio`

CHECK:

`Estado IN ('Pendiente', 'Pagado', 'EnPreparacion', 'Enviado', 'Completado', 'Cancelado')`

Índices:

* `UsuarioId`;
* `Estado`;
* `FechaPedido`.

---

# 24. DireccionPedido

| Campo              | Tipo          | Restricción  |
| ------------------ | ------------- | ------------ |
| DireccionPedidoId  | int           | PK, IDENTITY |
| PedidoId           | int           | FK, NOT NULL |
| NombreDestinatario | nvarchar(150) | NOT NULL     |
| Telefono           | nvarchar(25)  | NOT NULL     |
| LineaDireccion1    | nvarchar(200) | NOT NULL     |
| LineaDireccion2    | nvarchar(200) | NULL         |
| Pais               | nvarchar(100) | NOT NULL     |
| Region             | nvarchar(100) | NOT NULL     |
| Provincia          | nvarchar(100) | NOT NULL     |
| Distrito           | nvarchar(100) | NOT NULL     |
| CodigoPostal       | nvarchar(20)  | NULL         |
| Referencia         | nvarchar(300) | NULL         |

FK:

`PedidoId → Pedido.PedidoId`

UNIQUE:

`PedidoId`

La restricción garantiza una dirección histórica por pedido.

---

# 25. DetallePedido

| Campo           | Tipo          | Restricción  |
| --------------- | ------------- | ------------ |
| DetallePedidoId | int           | PK, IDENTITY |
| PedidoId        | int           | FK, NOT NULL |
| ProductoId      | int           | FK, NOT NULL |
| Cantidad        | int           | NOT NULL     |
| PrecioUnitario  | decimal(12,2) | NOT NULL     |

FK:

`PedidoId → Pedido.PedidoId`

`ProductoId → Producto.ProductoId`

CHECK:

`Cantidad > 0`

CHECK:

`PrecioUnitario >= 0`

UNIQUE:

`PedidoId + ProductoId`

Índices:

* `PedidoId`;
* `ProductoId`.

---

# 26. Pago

| Campo             | Tipo          | Restricción                        |
| ----------------- | ------------- | ---------------------------------- |
| PagoId            | int           | PK, IDENTITY                       |
| PedidoId          | int           | FK, NOT NULL                       |
| Monto             | decimal(12,2) | NOT NULL                           |
| Metodo            | nvarchar(30)  | NOT NULL                           |
| Estado            | nvarchar(20)  | NOT NULL                           |
| ReferenciaExterna | nvarchar(150) | NULL                               |
| FechaCreacion     | datetime2     | NOT NULL, DEFAULT SYSUTCDATETIME() |
| FechaPago         | datetime2     | NULL                               |

FK:

`PedidoId → Pedido.PedidoId`

CHECK:

`Monto > 0`

CHECK:

`Estado IN ('Pendiente', 'Aprobado', 'Rechazado', 'Reembolsado', 'Cancelado')`

Índices:

* `PedidoId`;
* `Estado`.

Índice único filtrado:

`Metodo + ReferenciaExterna`

cuando:

`ReferenciaExterna IS NOT NULL`

---

# 27. HistorialPedido

| Campo             | Tipo             | Restricción                        |
| ----------------- | ---------------- | ---------------------------------- |
| HistorialPedidoId | int              | PK, IDENTITY                       |
| PedidoId          | int              | FK, NOT NULL                       |
| UsuarioId         | uniqueidentifier | FK, NULL                           |
| EstadoAnterior    | nvarchar(25)     | NULL                               |
| EstadoNuevo       | nvarchar(25)     | NOT NULL                           |
| FechaCambio       | datetime2        | NOT NULL, DEFAULT SYSUTCDATETIME() |

FK:

`PedidoId → Pedido.PedidoId`

`UsuarioId → AspNetUsers.Id`

CHECK:

`EstadoAnterior IS NULL OR EstadoAnterior IN ('Pendiente', 'Pagado', 'EnPreparacion', 'Enviado', 'Completado', 'Cancelado')`

CHECK:

`EstadoNuevo IN ('Pendiente', 'Pagado', 'EnPreparacion', 'Enviado', 'Completado', 'Cancelado')`

Índice compuesto:

`PedidoId + FechaCambio`

---

# 28. ConfiguracionPC

| Campo                 | Tipo             | Restricción                        |
| --------------------- | ---------------- | ---------------------------------- |
| ConfiguracionPCId     | int              | PK, IDENTITY                       |
| UsuarioId             | uniqueidentifier | FK, NOT NULL                       |
| Nombre                | nvarchar(150)    | NOT NULL                           |
| EstadoCompatibilidad  | nvarchar(20)     | NOT NULL                           |
| FechaCreacion         | datetime2        | NOT NULL, DEFAULT SYSUTCDATETIME() |
| FechaActualizacion    | datetime2        | NOT NULL, DEFAULT SYSUTCDATETIME() |
| FechaUltimaValidacion | datetime2        | NULL                               |

FK:

`UsuarioId → AspNetUsers.Id`

CHECK:

`EstadoCompatibilidad IN ('Incompleta', 'Compatible', 'Incompatible')`

Índice:

`UsuarioId`

---

# 29. ConfiguracionPCItem

| Campo                 | Tipo | Restricción  |
| --------------------- | ---- | ------------ |
| ConfiguracionPCItemId | int  | PK, IDENTITY |
| ConfiguracionPCId     | int  | FK, NOT NULL |
| ProductoId            | int  | FK, NOT NULL |
| Cantidad              | int  | NOT NULL     |

FK:

`ConfiguracionPCId → ConfiguracionPC.ConfiguracionPCId`

`ProductoId → ComponentePC.ProductoId`

CHECK:

`Cantidad > 0`

UNIQUE:

`ConfiguracionPCId + ProductoId`

---

# 30. Juego

| Campo                    | Tipo           | Restricción  |
| ------------------------ | -------------- | ------------ |
| JuegoId                  | int            | PK, IDENTITY |
| Nombre                   | nvarchar(200)  | NOT NULL     |
| Descripcion              | nvarchar(2000) | NULL         |
| Estado                   | nvarchar(20)   | NOT NULL     |
| FechaAnuncio             | datetime2      | NULL         |
| FechaLanzamientoEstimada | datetime2      | NULL         |

CHECK:

`Estado IN ('EnDesarrollo', 'Alpha', 'Beta', 'Lanzado')`

---

# 31. PersonajeJuego

| Campo       | Tipo           | Restricción  |
| ----------- | -------------- | ------------ |
| PersonajeId | int            | PK, IDENTITY |
| JuegoId     | int            | FK, NOT NULL |
| Nombre      | nvarchar(150)  | NOT NULL     |
| Descripcion | nvarchar(2000) | NULL         |

FK:

`JuegoId → Juego.JuegoId`

Índice:

`JuegoId`

---

# 32. MapaJuego

| Campo       | Tipo           | Restricción  |
| ----------- | -------------- | ------------ |
| MapaId      | int            | PK, IDENTITY |
| JuegoId     | int            | FK, NOT NULL |
| Nombre      | nvarchar(150)  | NOT NULL     |
| Descripcion | nvarchar(2000) | NULL         |

FK:

`JuegoId → Juego.JuegoId`

Índice:

`JuegoId`

---

# 33. PreregistroJuego

| Campo         | Tipo             | Restricción                        |
| ------------- | ---------------- | ---------------------------------- |
| PreregistroId | int              | PK, IDENTITY                       |
| JuegoId       | int              | FK, NOT NULL                       |
| UsuarioId     | uniqueidentifier | FK, NULL                           |
| Email         | nvarchar(256)    | NOT NULL                           |
| FechaRegistro | datetime2        | NOT NULL, DEFAULT SYSUTCDATETIME() |

FK:

`JuegoId → Juego.JuegoId`

`UsuarioId → AspNetUsers.Id`

UNIQUE:

`JuegoId + Email`

Índice:

`UsuarioId`

---

# 34. ServicioDesarrollo

| Campo       | Tipo           | Restricción         |
| ----------- | -------------- | ------------------- |
| ServicioId  | int            | PK, IDENTITY        |
| Nombre      | nvarchar(150)  | NOT NULL            |
| Descripcion | nvarchar(2000) | NULL                |
| PrecioBase  | decimal(12,2)  | NOT NULL            |
| Activo      | bit            | NOT NULL, DEFAULT 1 |

CHECK:

`PrecioBase >= 0`

---

# 35. SolicitudServicio

| Campo                    | Tipo             | Restricción                        |
| ------------------------ | ---------------- | ---------------------------------- |
| SolicitudId              | int              | PK, IDENTITY                       |
| UsuarioId                | uniqueidentifier | FK, NOT NULL                       |
| ServicioId               | int              | FK, NOT NULL                       |
| DescripcionRequerimiento | nvarchar(max)    | NOT NULL                           |
| Estado                   | nvarchar(25)     | NOT NULL                           |
| FechaSolicitud           | datetime2        | NOT NULL, DEFAULT SYSUTCDATETIME() |

FK:

`UsuarioId → AspNetUsers.Id`

`ServicioId → ServicioDesarrollo.ServicioId`

CHECK:

`Estado IN ('Pendiente', 'EnRevision', 'Aprobada', 'Rechazada', 'EnProceso', 'Completada', 'Cancelada')`

Índices:

* `UsuarioId`;
* `ServicioId`;
* `Estado`.

---

# 36. Empresa

| Campo         | Tipo           | Restricción                        |
| ------------- | -------------- | ---------------------------------- |
| EmpresaId     | int            | PK, IDENTITY                       |
| Nombre        | nvarchar(200)  | NOT NULL                           |
| Descripcion   | nvarchar(2000) | NULL                               |
| Mision        | nvarchar(2000) | NULL                               |
| Vision        | nvarchar(2000) | NULL                               |
| FechaCreacion | datetime2      | NOT NULL, DEFAULT SYSUTCDATETIME() |
| Activa        | bit            | NOT NULL, DEFAULT 1                |

---

# 37. Comportamiento de claves foráneas

## Eliminación en cascada

Se permitirá principalmente en entidades dependientes no históricas:

* Carrito → DetalleCarrito.
* ConfiguracionPC → ConfiguracionPCItem.
* Juego → PersonajeJuego.
* Juego → MapaJuego.
* Gabinete → tablas de formatos soportados.
* Refrigeracion → RefrigeracionSocket.

---

## NO ACTION

Se utilizará para relaciones históricas, comerciales o cuya eliminación accidental debe evitarse:

* Usuario → Pedido.
* Usuario → HistorialPedido.
* Producto → DetallePedido.
* Pedido → DireccionPedido.
* Pedido → DetallePedido.
* Pedido → Pago.
* Pedido → HistorialPedido.
* ServicioDesarrollo → SolicitudServicio.
* Juego → PreregistroJuego.

Los productos y usuarios utilizados históricamente no deben eliminarse físicamente durante el funcionamiento normal.

---

# 38. Índices únicos

El modelo deberá garantizar:

* email de usuario mediante Identity;
* `Categoria.Nombre`;
* una dirección predeterminada activa por usuario;
* un carrito activo por usuario;
* `CarritoId + ProductoId`;
* `PedidoId` en DireccionPedido;
* `PedidoId + ProductoId` en DetallePedido;
* `ConfiguracionPCId + ProductoId`;
* `JuegoId + Email`;
* `ProductoIdProcesador + ProductoIdPlacaMadre`;
* `ProductoIdGabinete + FormatoPlaca`;
* `ProductoIdGabinete + FormatoFuente`;
* `ProductoIdGabinete + TamanoRadiadorMm`;
* `ProductoIdRefrigeracion + Socket`;
* ISBN cuando no sea NULL;
* `Metodo + ReferenciaExterna` de pago cuando exista referencia externa.

---

# 39. Reglas que permanecerán en el dominio

Las siguientes reglas no se intentarán resolver únicamente mediante restricciones SQL:

* exclusividad entre `ComponentePC` y `Publicacion`;
* pertenencia de un `ComponentePC` a exactamente un subtipo técnico;
* compatibilidad completa entre componentes;
* composición mínima de una PC funcional;
* cálculo de potencia requerida;
* recálculo de EstadoCompatibilidad;
* comprobación de stock durante checkout;
* validación de que el producto continúa activo;
* actualización automática de FechaActualizacion;
* transición válida entre estados;
* creación del snapshot de DireccionPedido;
* cálculo del subtotal del pedido;
* control transaccional del stock;
* interacción con pasarelas de pago.

Estas reglas serán responsabilidad de la capa de aplicación y dominio del backend.

---

# 40. Integridad transaccional

La creación definitiva de un pedido deberá realizarse dentro de una operación transaccional que permita:

1. validar productos activos;
2. validar stock;
3. obtener precios actuales;
4. calcular subtotal;
5. calcular costo de envío;
6. calcular total;
7. crear Pedido;
8. crear DetallePedido;
9. crear DireccionPedido;
10. actualizar stock;
11. crear el estado inicial del historial.

Si la operación falla, no deberán persistirse cambios parciales.

---

# 41. Resumen de tablas del dominio

El esquema inicial incluye:

1. DireccionUsuario
2. Categoria
3. Producto
4. Publicacion
5. ComponentePC
6. Procesador
7. PlacaMadre
8. CompatibilidadProcesadorPlaca
9. MemoriaRAM
10. TarjetaGrafica
11. Almacenamiento
12. FuentePoder
13. Gabinete
14. GabineteFormatoPlaca
15. GabineteFormatoFuente
16. GabineteRadiadorSoportado
17. Refrigeracion
18. RefrigeracionSocket
19. Carrito
20. DetalleCarrito
21. Pedido
22. DireccionPedido
23. DetallePedido
24. Pago
25. HistorialPedido
26. ConfiguracionPC
27. ConfiguracionPCItem
28. Juego
29. PersonajeJuego
30. MapaJuego
31. PreregistroJuego
32. ServicioDesarrollo
33. SolicitudServicio
34. Empresa

A estas tablas se añadirán las generadas por ASP.NET Core Identity.

---

# 42. Estado

El modelo físico v1.0 establece la estructura base de persistencia de NGT Platform para Microsoft SQL Server.

Quedan definidos:

* tablas del dominio;
* tipos de datos;
* claves primarias;
* claves foráneas;
* nulabilidad;
* restricciones `CHECK`;
* restricciones `UNIQUE`;
* índices principales;
* comportamiento general de eliminación;
* integración con ASP.NET Core Identity;
* límites entre integridad de base de datos y reglas de dominio.

El siguiente paso corresponde a implementar la infraestructura de desarrollo y levantar SQL Server mediante Docker.
