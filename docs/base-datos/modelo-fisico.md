# Modelo Físico de Base de Datos

**Proyecto:** NGT Platform (Neo Genesis Technology)
**Versión:** 1.1
**Motor:** Microsoft SQL Server
**Estado:** Aprobado
**Origen:** Modelo lógico v1.0, modelo ER v1.0 y ADR 0001 (monolito modular)

---

# 1. Convenciones generales

## Identificadores

Las entidades propias del dominio utilizarán, salvo que se indique lo contrario:

* `int IDENTITY(1,1)` como identificador principal.

Los usuarios y roles administrados mediante ASP.NET Core Identity utilizarán:

* `uniqueidentifier`.

Las especializaciones que permanecen dentro de un mismo módulo podrán reutilizar la clave de su entidad padre como PK y FK.

En particular:

* `commerce.Publicacion` reutilizará `ProductoId` como PK y FK hacia `commerce.Producto`;
* `pcbuilder.ComponentePC` tendrá identidad propia mediante `ComponentePCId`;
* `pcbuilder.ComponentePC.ProductoId` será una referencia única al producto comercial asociado en `commerce.Producto`;
* los subtipos técnicos de PcBuilder reutilizarán `ComponentePCId` como PK y FK hacia `pcbuilder.ComponentePC`.

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

Las claves foráneas que crucen límites entre módulos utilizarán `NO ACTION` en la versión inicial, evitando que una operación de eliminación en un módulo provoque eliminaciones automáticas en otro.

---

## Organización modular y schemas

NGT Platform utilizará inicialmente una única base de datos SQL Server:

`NgtPlatform`

La base de datos se organizará mediante schemas que reflejan la propiedad de cada módulo:

| Módulo | Schema SQL | DbContext propietario |
| --- | --- | --- |
| Identity | `identity` | `IdentityDbContext` |
| Commerce | `commerce` | `CommerceDbContext` |
| PcBuilder | `pcbuilder` | `PcBuilderDbContext` |
| Game | `game` | `GameDbContext` |
| SoftwareServices | `services` | `SoftwareServicesDbContext` |
| Corporate | `corporate` | `CorporateDbContext` |

Cada módulo será propietario de sus tablas, reglas de persistencia y configuración de Entity Framework Core.

Un módulo no deberá:

* acceder directamente al `DbContext` de otro módulo;
* modificar tablas cuyo propietario sea otro módulo;
* utilizar entidades internas de otro módulo como mecanismo de comunicación entre módulos.

La comunicación entre módulos se realizará mediante contratos explícitos definidos por la aplicación.

Podrán existir claves foráneas físicas entre schemas cuando aporten integridad referencial y estén documentadas expresamente en este modelo. Estas relaciones no implican permiso para acceder directamente al `DbContext` del módulo propietario.

Cada `DbContext` administrará las migraciones correspondientes a las tablas de su propio módulo. Las dependencias entre migraciones que involucren claves foráneas entre módulos deberán respetar el orden de creación de las tablas principales.

---

# 2. ASP.NET Core Identity

**Módulo propietario:** Identity

**Schema:** `identity`

**DbContext:** `IdentityDbContext`

NGT Platform utilizará ASP.NET Core Identity para autenticación y autorización.

El proyecto utilizará `uniqueidentifier` como identificador de usuario y rol.

No se crearán tablas propias equivalentes a:

* Usuario;
* Rol;
* UsuarioRol.

ASP.NET Core Identity administrará sus tablas de autenticación y autorización dentro del schema `identity`, incluyendo las tablas estándar necesarias para usuarios, roles, relaciones, claims, logins y tokens.

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

La tabla principal de usuarios se referenciará físicamente como:

`identity.AspNetUsers`

---

# 3. commerce.DireccionUsuario

**Módulo propietario:** Commerce

**Schema:** `commerce`

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

FK entre módulos:

`UsuarioId → identity.AspNetUsers.Id`

Comportamiento de eliminación:

`NO ACTION`

Índices:

* índice por `UsuarioId`;
* índice único filtrado por `UsuarioId` cuando:

`EsPredeterminada = 1 AND Activa = 1`

Esto garantiza como máximo una dirección predeterminada activa por usuario.

---

# 4. commerce.Categoria

**Módulo propietario:** Commerce

**Schema:** `commerce`

| Campo       | Tipo          | Restricción         |
| ----------- | ------------- | ------------------- |
| CategoriaId | int           | PK, IDENTITY        |
| Nombre      | nvarchar(120) | NOT NULL            |
| Descripcion | nvarchar(500) | NULL                |
| Activa      | bit           | NOT NULL, DEFAULT 1 |

Restricción:

`UNIQUE (Nombre)`

---

# 5. commerce.Producto

**Módulo propietario:** Commerce

**Schema:** `commerce`

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

`CategoriaId → commerce.Categoria.CategoriaId`

CHECK:

`Precio >= 0`

`Stock >= 0`

Índices:

* `CategoriaId`;
* `Nombre`;
* `Activo`.

Commerce es propietario de la información comercial del producto: nombre, descripción, precio, stock, categoría y estado comercial.

Un producto no podrá estar asociado simultáneamente a `commerce.Publicacion` y `pcbuilder.ComponentePC`. Esta exclusividad será una regla coordinada por la lógica de aplicación y dominio, ya que puede involucrar más de un módulo.

---

# 6. commerce.Publicacion

**Módulo propietario:** Commerce

**Schema:** `commerce`

| Campo           | Tipo          | Restricción |
| --------------- | ------------- | ----------- |
| ProductoId      | int           | PK, FK      |
| TipoPublicacion | nvarchar(20)  | NOT NULL    |
| Autor           | nvarchar(200) | NULL        |
| Editorial       | nvarchar(150) | NULL        |
| ISBN            | nvarchar(20)  | NULL        |
| Volumen         | int           | NULL        |

FK:

`ProductoId → commerce.Producto.ProductoId`

CHECK:

`TipoPublicacion IN ('Comic', 'Manga')`

CHECK:

`Volumen IS NULL OR Volumen > 0`

Índice único filtrado:

`ISBN`

cuando:

`ISBN IS NOT NULL`

---

# 7. pcbuilder.ComponentePC

**Módulo propietario:** PcBuilder

**Schema:** `pcbuilder`

| Campo          | Tipo          | Restricción  |
| -------------- | ------------- | ------------ |
| ComponentePCId | int           | PK, IDENTITY |
| ProductoId     | int           | FK, NOT NULL |
| Marca          | nvarchar(100) | NOT NULL     |
| Modelo         | nvarchar(150) | NOT NULL     |

FK entre módulos:

`ProductoId → commerce.Producto.ProductoId`

Comportamiento de eliminación:

`NO ACTION`

UNIQUE:

`ProductoId`

La restricción `UNIQUE` garantiza que un mismo producto comercial no pueda representar más de un componente técnico.

PcBuilder es propietario de la información técnica del componente. Commerce continúa siendo propietario de precio, stock, disponibilidad comercial y demás datos generales del producto.

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

La exclusividad se garantizará mediante la lógica de dominio de PcBuilder.

---

# 8. pcbuilder.Procesador

**Módulo propietario:** PcBuilder

**Schema:** `pcbuilder`

| Campo                   | Tipo         | Restricción |
| ----------------------- | ------------ | ----------- |
| ComponentePCId          | int          | PK, FK      |
| Socket                  | nvarchar(30) | NOT NULL    |
| TdpWatts                | int          | NOT NULL    |
| ConsumoEstimadoWatts    | int          | NOT NULL    |
| TieneGraficosIntegrados | bit          | NOT NULL    |
| IncluyeRefrigeracion    | bit          | NOT NULL    |

FK:

`ComponentePCId → pcbuilder.ComponentePC.ComponentePCId`

CHECK:

`TdpWatts > 0`

`ConsumoEstimadoWatts > 0`

Índice:

`Socket`

---

# 9. pcbuilder.PlacaMadre

**Módulo propietario:** PcBuilder

**Schema:** `pcbuilder`

| Campo                    | Tipo         | Restricción |
| ------------------------ | ------------ | ----------- |
| ComponentePCId           | int          | PK, FK      |
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

`ComponentePCId → pcbuilder.ComponentePC.ComponentePCId`

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

# 10. pcbuilder.CompatibilidadProcesadorPlaca

**Módulo propietario:** PcBuilder

**Schema:** `pcbuilder`

| Campo                     | Tipo          | Restricción         |
| ------------------------- | ------------- | ------------------- |
| ProcesadorId              | int           | PK, FK              |
| PlacaMadreId              | int           | PK, FK              |
| RequiereActualizacionBios | bit           | NOT NULL, DEFAULT 0 |
| Observacion               | nvarchar(500) | NULL                |

PK compuesta:

`ProcesadorId + PlacaMadreId`

FK:

`ProcesadorId → pcbuilder.Procesador.ComponentePCId`

`PlacaMadreId → pcbuilder.PlacaMadre.ComponentePCId`

Índice adicional:

`PlacaMadreId`

---

# 11. pcbuilder.MemoriaRAM

**Módulo propietario:** PcBuilder

**Schema:** `pcbuilder`

| Campo                | Tipo         | Restricción |
| -------------------- | ------------ | ----------- |
| ComponentePCId       | int          | PK, FK      |
| TipoMemoria          | nvarchar(20) | NOT NULL    |
| CapacidadTotalGB     | int          | NOT NULL    |
| ModulosPorKit        | tinyint      | NOT NULL    |
| FrecuenciaMTs        | int          | NOT NULL    |
| ConsumoEstimadoWatts | int          | NOT NULL    |

FK:

`ComponentePCId → pcbuilder.ComponentePC.ComponentePCId`

CHECK:

`CapacidadTotalGB > 0`

`ModulosPorKit > 0`

`FrecuenciaMTs > 0`

`ConsumoEstimadoWatts > 0`

Índice:

`TipoMemoria`

---

# 12. pcbuilder.TarjetaGrafica

**Módulo propietario:** PcBuilder

**Schema:** `pcbuilder`

| Campo                          | Tipo | Restricción |
| ------------------------------ | ---- | ----------- |
| ComponentePCId                 | int  | PK, FK      |
| LongitudMm                     | int  | NOT NULL    |
| ConsumoEstimadoWatts           | int  | NOT NULL    |
| PotenciaFuenteRecomendadaWatts | int  | NOT NULL    |

FK:

`ComponentePCId → pcbuilder.ComponentePC.ComponentePCId`

CHECK:

`LongitudMm > 0`

`ConsumoEstimadoWatts > 0`

`PotenciaFuenteRecomendadaWatts > 0`

---

# 13. pcbuilder.Almacenamiento

**Módulo propietario:** PcBuilder

**Schema:** `pcbuilder`

| Campo                | Tipo         | Restricción |
| -------------------- | ------------ | ----------- |
| ComponentePCId       | int          | PK, FK      |
| TipoAlmacenamiento   | nvarchar(20) | NOT NULL    |
| Interfaz             | nvarchar(30) | NOT NULL    |
| CapacidadGB          | int          | NOT NULL    |
| ConsumoEstimadoWatts | int          | NOT NULL    |

FK:

`ComponentePCId → pcbuilder.ComponentePC.ComponentePCId`

CHECK:

`TipoAlmacenamiento IN ('HDD', 'SSD')`

CHECK:

`Interfaz IN ('SATA', 'M2NVMe')`

CHECK:

`CapacidadGB > 0`

CHECK:

`ConsumoEstimadoWatts > 0`

---

# 14. pcbuilder.FuentePoder

**Módulo propietario:** PcBuilder

**Schema:** `pcbuilder`

| Campo          | Tipo         | Restricción |
| -------------- | ------------ | ----------- |
| ComponentePCId | int          | PK, FK      |
| PotenciaWatts  | int          | NOT NULL    |
| FormatoFuente  | nvarchar(30) | NOT NULL    |

FK:

`ComponentePCId → pcbuilder.ComponentePC.ComponentePCId`

CHECK:

`PotenciaWatts > 0`

Índice:

`FormatoFuente`

---

# 15. pcbuilder.Gabinete

**Módulo propietario:** PcBuilder

**Schema:** `pcbuilder`

| Campo                       | Tipo | Restricción |
| --------------------------- | ---- | ----------- |
| ComponentePCId              | int  | PK, FK      |
| LongitudMaximaGpuMm         | int  | NOT NULL    |
| AlturaMaximaRefrigeracionMm | int  | NOT NULL    |

FK:

`ComponentePCId → pcbuilder.ComponentePC.ComponentePCId`

CHECK:

`LongitudMaximaGpuMm > 0`

CHECK:

`AlturaMaximaRefrigeracionMm > 0`

---

# 16. pcbuilder.GabineteFormatoPlaca

**Módulo propietario:** PcBuilder

**Schema:** `pcbuilder`

| Campo        | Tipo         | Restricción |
| ------------ | ------------ | ----------- |
| GabineteId   | int          | PK, FK      |
| FormatoPlaca | nvarchar(30) | PK          |

PK:

`GabineteId + FormatoPlaca`

FK:

`GabineteId → pcbuilder.Gabinete.ComponentePCId`

---

# 17. pcbuilder.GabineteFormatoFuente

**Módulo propietario:** PcBuilder

**Schema:** `pcbuilder`

| Campo         | Tipo         | Restricción |
| ------------- | ------------ | ----------- |
| GabineteId    | int          | PK, FK      |
| FormatoFuente | nvarchar(30) | PK          |

PK:

`GabineteId + FormatoFuente`

FK:

`GabineteId → pcbuilder.Gabinete.ComponentePCId`

---

# 18. pcbuilder.GabineteRadiadorSoportado

**Módulo propietario:** PcBuilder

**Schema:** `pcbuilder`

| Campo            | Tipo     | Restricción |
| ---------------- | -------- | ----------- |
| GabineteId       | int      | PK, FK      |
| TamanoRadiadorMm | smallint | PK          |

PK:

`GabineteId + TamanoRadiadorMm`

FK:

`GabineteId → pcbuilder.Gabinete.ComponentePCId`

CHECK:

`TamanoRadiadorMm > 0`

---

# 19. pcbuilder.Refrigeracion

**Módulo propietario:** PcBuilder

**Schema:** `pcbuilder`

| Campo                | Tipo         | Restricción |
| -------------------- | ------------ | ----------- |
| ComponentePCId       | int          | PK, FK      |
| TipoRefrigeracion    | nvarchar(20) | NOT NULL    |
| AlturaMm             | int          | NULL        |
| TamanoRadiadorMm     | smallint     | NULL        |
| ConsumoEstimadoWatts | int          | NOT NULL    |

FK:

`ComponentePCId → pcbuilder.ComponentePC.ComponentePCId`

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

# 20. pcbuilder.RefrigeracionSocket

**Módulo propietario:** PcBuilder

**Schema:** `pcbuilder`

| Campo           | Tipo         | Restricción |
| --------------- | ------------ | ----------- |
| RefrigeracionId | int          | PK, FK      |
| Socket          | nvarchar(30) | PK          |

PK:

`RefrigeracionId + Socket`

FK:

`RefrigeracionId → pcbuilder.Refrigeracion.ComponentePCId`

---

# 21. commerce.Carrito

**Módulo propietario:** Commerce

**Schema:** `commerce`

| Campo              | Tipo             | Restricción                        |
| ------------------ | ---------------- | ---------------------------------- |
| CarritoId          | int              | PK, IDENTITY                       |
| UsuarioId          | uniqueidentifier | FK, NOT NULL                       |
| Estado             | nvarchar(25)     | NOT NULL                           |
| FechaCreacion      | datetime2        | NOT NULL, DEFAULT SYSUTCDATETIME() |
| FechaActualizacion | datetime2        | NOT NULL, DEFAULT SYSUTCDATETIME() |

FK entre módulos:

`UsuarioId → identity.AspNetUsers.Id`

Comportamiento de eliminación:

`NO ACTION`

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

# 22. commerce.DetalleCarrito

**Módulo propietario:** Commerce

**Schema:** `commerce`

| Campo            | Tipo | Restricción  |
| ---------------- | ---- | ------------ |
| DetalleCarritoId | int  | PK, IDENTITY |
| CarritoId        | int  | FK, NOT NULL |
| ProductoId       | int  | FK, NOT NULL |
| Cantidad         | int  | NOT NULL     |

FK:

`CarritoId → commerce.Carrito.CarritoId`

`ProductoId → commerce.Producto.ProductoId`

CHECK:

`Cantidad > 0`

UNIQUE:

`CarritoId + ProductoId`

---

# 23. commerce.Pedido

**Módulo propietario:** Commerce

**Schema:** `commerce`

| Campo       | Tipo             | Restricción                        |
| ----------- | ---------------- | ---------------------------------- |
| PedidoId    | int              | PK, IDENTITY                       |
| UsuarioId   | uniqueidentifier | FK, NOT NULL                       |
| Subtotal    | decimal(12,2)    | NOT NULL                           |
| CostoEnvio  | decimal(12,2)    | NOT NULL                           |
| Total       | decimal(12,2)    | NOT NULL                           |
| Estado      | nvarchar(25)     | NOT NULL                           |
| FechaPedido | datetime2        | NOT NULL, DEFAULT SYSUTCDATETIME() |

FK entre módulos:

`UsuarioId → identity.AspNetUsers.Id`

Comportamiento de eliminación:

`NO ACTION`

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

# 24. commerce.DireccionPedido

**Módulo propietario:** Commerce

**Schema:** `commerce`

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

`PedidoId → commerce.Pedido.PedidoId`

UNIQUE:

`PedidoId`

La restricción garantiza una dirección histórica por pedido.

---

# 25. commerce.DetallePedido

**Módulo propietario:** Commerce

**Schema:** `commerce`

| Campo           | Tipo          | Restricción  |
| --------------- | ------------- | ------------ |
| DetallePedidoId | int           | PK, IDENTITY |
| PedidoId        | int           | FK, NOT NULL |
| ProductoId      | int           | FK, NOT NULL |
| Cantidad        | int           | NOT NULL     |
| PrecioUnitario  | decimal(12,2) | NOT NULL     |

FK:

`PedidoId → commerce.Pedido.PedidoId`

`ProductoId → commerce.Producto.ProductoId`

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

# 26. commerce.Pago

**Módulo propietario:** Commerce

**Schema:** `commerce`

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

`PedidoId → commerce.Pedido.PedidoId`

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

# 27. commerce.HistorialPedido

**Módulo propietario:** Commerce

**Schema:** `commerce`

| Campo             | Tipo             | Restricción                        |
| ----------------- | ---------------- | ---------------------------------- |
| HistorialPedidoId | int              | PK, IDENTITY                       |
| PedidoId          | int              | FK, NOT NULL                       |
| UsuarioId         | uniqueidentifier | FK, NULL                           |
| EstadoAnterior    | nvarchar(25)     | NULL                               |
| EstadoNuevo       | nvarchar(25)     | NOT NULL                           |
| FechaCambio       | datetime2        | NOT NULL, DEFAULT SYSUTCDATETIME() |

FK interna:

`PedidoId → commerce.Pedido.PedidoId`

FK entre módulos:

`UsuarioId → identity.AspNetUsers.Id`

Comportamiento de eliminación de la relación entre módulos:

`NO ACTION`

CHECK:

`EstadoAnterior IS NULL OR EstadoAnterior IN ('Pendiente', 'Pagado', 'EnPreparacion', 'Enviado', 'Completado', 'Cancelado')`

CHECK:

`EstadoNuevo IN ('Pendiente', 'Pagado', 'EnPreparacion', 'Enviado', 'Completado', 'Cancelado')`

Índice compuesto:

`PedidoId + FechaCambio`

---

# 28. pcbuilder.ConfiguracionPC

**Módulo propietario:** PcBuilder

**Schema:** `pcbuilder`

| Campo                 | Tipo             | Restricción                        |
| --------------------- | ---------------- | ---------------------------------- |
| ConfiguracionPCId     | int              | PK, IDENTITY                       |
| UsuarioId             | uniqueidentifier | FK, NOT NULL                       |
| Nombre                | nvarchar(150)    | NOT NULL                           |
| EstadoCompatibilidad  | nvarchar(20)     | NOT NULL                           |
| FechaCreacion         | datetime2        | NOT NULL, DEFAULT SYSUTCDATETIME() |
| FechaActualizacion    | datetime2        | NOT NULL, DEFAULT SYSUTCDATETIME() |
| FechaUltimaValidacion | datetime2        | NULL                               |

FK entre módulos:

`UsuarioId → identity.AspNetUsers.Id`

Comportamiento de eliminación:

`NO ACTION`

CHECK:

`EstadoCompatibilidad IN ('Incompleta', 'Compatible', 'Incompatible')`

Índice:

`UsuarioId`

---

# 29. pcbuilder.ConfiguracionPCItem

**Módulo propietario:** PcBuilder

**Schema:** `pcbuilder`

| Campo                 | Tipo | Restricción  |
| --------------------- | ---- | ------------ |
| ConfiguracionPCItemId | int  | PK, IDENTITY |
| ConfiguracionPCId     | int  | FK, NOT NULL |
| ComponentePCId        | int  | FK, NOT NULL |
| Cantidad              | int  | NOT NULL     |

FK:

`ConfiguracionPCId → pcbuilder.ConfiguracionPC.ConfiguracionPCId`

`ComponentePCId → pcbuilder.ComponentePC.ComponentePCId`

CHECK:

`Cantidad > 0`

UNIQUE:

`ConfiguracionPCId + ComponentePCId`

---

# 30. game.Juego

**Módulo propietario:** Game

**Schema:** `game`

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

# 31. game.PersonajeJuego

**Módulo propietario:** Game

**Schema:** `game`

| Campo       | Tipo           | Restricción  |
| ----------- | -------------- | ------------ |
| PersonajeId | int            | PK, IDENTITY |
| JuegoId     | int            | FK, NOT NULL |
| Nombre      | nvarchar(150)  | NOT NULL     |
| Descripcion | nvarchar(2000) | NULL         |

FK:

`JuegoId → game.Juego.JuegoId`

Índice:

`JuegoId`

---

# 32. game.MapaJuego

**Módulo propietario:** Game

**Schema:** `game`

| Campo       | Tipo           | Restricción  |
| ----------- | -------------- | ------------ |
| MapaId      | int            | PK, IDENTITY |
| JuegoId     | int            | FK, NOT NULL |
| Nombre      | nvarchar(150)  | NOT NULL     |
| Descripcion | nvarchar(2000) | NULL         |

FK:

`JuegoId → game.Juego.JuegoId`

Índice:

`JuegoId`

---

# 33. game.PreregistroJuego

**Módulo propietario:** Game

**Schema:** `game`

| Campo         | Tipo             | Restricción                        |
| ------------- | ---------------- | ---------------------------------- |
| PreregistroId | int              | PK, IDENTITY                       |
| JuegoId       | int              | FK, NOT NULL                       |
| UsuarioId     | uniqueidentifier | FK, NULL                           |
| Email         | nvarchar(256)    | NOT NULL                           |
| FechaRegistro | datetime2        | NOT NULL, DEFAULT SYSUTCDATETIME() |

FK interna:

`JuegoId → game.Juego.JuegoId`

FK entre módulos:

`UsuarioId → identity.AspNetUsers.Id`

Comportamiento de eliminación de la relación entre módulos:

`NO ACTION`

UNIQUE:

`JuegoId + Email`

Índice:

`UsuarioId`

---

# 34. services.ServicioDesarrollo

**Módulo propietario:** SoftwareServices

**Schema:** `services`

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

# 35. services.SolicitudServicio

**Módulo propietario:** SoftwareServices

**Schema:** `services`

| Campo                    | Tipo             | Restricción                        |
| ------------------------ | ---------------- | ---------------------------------- |
| SolicitudId              | int              | PK, IDENTITY                       |
| UsuarioId                | uniqueidentifier | FK, NOT NULL                       |
| ServicioId               | int              | FK, NOT NULL                       |
| DescripcionRequerimiento | nvarchar(max)    | NOT NULL                           |
| Estado                   | nvarchar(25)     | NOT NULL                           |
| FechaSolicitud           | datetime2        | NOT NULL, DEFAULT SYSUTCDATETIME() |

FK entre módulos:

`UsuarioId → identity.AspNetUsers.Id`

Comportamiento de eliminación de la relación entre módulos:

`NO ACTION`

FK interna:

`ServicioId → services.ServicioDesarrollo.ServicioId`

CHECK:

`Estado IN ('Pendiente', 'EnRevision', 'Aprobada', 'Rechazada', 'EnProceso', 'Completada', 'Cancelada')`

Índices:

* `UsuarioId`;
* `ServicioId`;
* `Estado`.

---

# 36. corporate.Empresa

**Módulo propietario:** Corporate

**Schema:** `corporate`

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

## Principio general

Las claves foráneas internas de un módulo podrán utilizar el comportamiento de eliminación que corresponda a su ciclo de vida.

Toda clave foránea que cruce límites entre módulos utilizará `NO ACTION` en la versión 1.1.

Una clave foránea entre módulos proporciona integridad referencial física, pero no autoriza acceso directo al `DbContext`, entidades internas o tablas del módulo propietario.

---

## Eliminación en cascada

Se permitirá principalmente en entidades dependientes no históricas dentro del mismo módulo:

* `commerce.Carrito → commerce.DetalleCarrito`;
* `pcbuilder.ConfiguracionPC → pcbuilder.ConfiguracionPCItem`;
* `game.Juego → game.PersonajeJuego`;
* `game.Juego → game.MapaJuego`;
* `pcbuilder.Gabinete → pcbuilder.GabineteFormatoPlaca`;
* `pcbuilder.Gabinete → pcbuilder.GabineteFormatoFuente`;
* `pcbuilder.Gabinete → pcbuilder.GabineteRadiadorSoportado`;
* `pcbuilder.Refrigeracion → pcbuilder.RefrigeracionSocket`.

---

## NO ACTION dentro de un módulo

Se utilizará para relaciones históricas, comerciales o cuya eliminación accidental debe evitarse:

* `commerce.Producto → commerce.DetallePedido`;
* `commerce.Pedido → commerce.DireccionPedido`;
* `commerce.Pedido → commerce.DetallePedido`;
* `commerce.Pedido → commerce.Pago`;
* `commerce.Pedido → commerce.HistorialPedido`;
* `services.ServicioDesarrollo → services.SolicitudServicio`;
* `game.Juego → game.PreregistroJuego`.

---

## NO ACTION entre módulos

Se mantendrán inicialmente las siguientes referencias físicas entre módulos:

* `identity.AspNetUsers → commerce.DireccionUsuario`;
* `identity.AspNetUsers → commerce.Carrito`;
* `identity.AspNetUsers → commerce.Pedido`;
* `identity.AspNetUsers → commerce.HistorialPedido`;
* `identity.AspNetUsers → pcbuilder.ConfiguracionPC`;
* `identity.AspNetUsers → game.PreregistroJuego`;
* `identity.AspNetUsers → services.SolicitudServicio`;
* `commerce.Producto → pcbuilder.ComponentePC`.

Los productos y usuarios utilizados históricamente no deben eliminarse físicamente durante el funcionamiento normal.

Si en el futuro un módulo se extrae a un servicio independiente, las claves foráneas que atraviesen su frontera deberán sustituirse por referencias externas y mecanismos de consistencia apropiados.

---

# 38. Índices y restricciones únicas

El modelo deberá garantizar:

* email de usuario mediante ASP.NET Core Identity;
* `commerce.Categoria.Nombre`;
* una dirección predeterminada activa por usuario;
* un carrito activo por usuario;
* `commerce.DetalleCarrito (CarritoId + ProductoId)`;
* `commerce.DireccionPedido.PedidoId`;
* `commerce.DetallePedido (PedidoId + ProductoId)`;
* `pcbuilder.ComponentePC.ProductoId`;
* `pcbuilder.ConfiguracionPCItem (ConfiguracionPCId + ComponentePCId)`;
* `game.PreregistroJuego (JuegoId + Email)`;
* `pcbuilder.CompatibilidadProcesadorPlaca (ProcesadorId + PlacaMadreId)`;
* `pcbuilder.GabineteFormatoPlaca (GabineteId + FormatoPlaca)`;
* `pcbuilder.GabineteFormatoFuente (GabineteId + FormatoFuente)`;
* `pcbuilder.GabineteRadiadorSoportado (GabineteId + TamanoRadiadorMm)`;
* `pcbuilder.RefrigeracionSocket (RefrigeracionId + Socket)`;
* ISBN cuando no sea NULL;
* `commerce.Pago (Metodo + ReferenciaExterna)` cuando exista referencia externa.

---

# 39. Reglas que permanecerán en el dominio y la aplicación

Las siguientes reglas no se intentarán resolver únicamente mediante restricciones SQL:

* un producto no puede estar asociado simultáneamente a `commerce.Publicacion` y `pcbuilder.ComponentePC`;
* pertenencia de un `pcbuilder.ComponentePC` a exactamente un subtipo técnico;
* compatibilidad completa entre componentes;
* composición mínima de una PC funcional;
* cálculo de potencia requerida;
* recálculo de `EstadoCompatibilidad`;
* comprobación de stock durante checkout;
* validación de que el producto continúa activo;
* actualización automática de `FechaActualizacion`;
* transición válida entre estados;
* creación del snapshot de `commerce.DireccionPedido`;
* cálculo del subtotal del pedido;
* control transaccional del stock;
* interacción con pasarelas de pago.

Las reglas exclusivamente técnicas del configurador pertenecen a PcBuilder.

Las reglas de precio, stock, carrito, pedidos y pagos pertenecen a Commerce.

Cuando una regla requiera información de más de un módulo, la coordinación se realizará mediante contratos explícitos de aplicación y no mediante acceso directo al `DbContext` del otro módulo.

---

# 40. Integridad transaccional

## Checkout de Commerce

La creación definitiva de un pedido deberá realizarse dentro de una operación transaccional del módulo Commerce que permita:

1. validar productos activos;
2. validar stock;
3. obtener precios actuales;
4. calcular subtotal;
5. calcular costo de envío;
6. calcular total;
7. crear `commerce.Pedido`;
8. crear `commerce.DetallePedido`;
9. crear `commerce.DireccionPedido`;
10. actualizar stock;
11. crear el estado inicial en `commerce.HistorialPedido`.

Si la operación falla, no deberán persistirse cambios parciales.

---

## Compra originada desde PcBuilder

Cuando una compra se origine desde una configuración de PC:

1. PcBuilder validará que la configuración esté completa y sea técnicamente compatible;
2. PcBuilder proporcionará a Commerce los identificadores de producto y cantidades necesarios mediante un contrato explícito;
3. Commerce volverá a validar estado comercial, stock y precios actuales;
4. Commerce ejecutará el checkout dentro de su propia transacción.

PcBuilder no modificará directamente stock, pedidos ni pagos de Commerce.

Commerce no accederá directamente al `PcBuilderDbContext` para ejecutar el checkout.

---

# 41. Resumen de tablas por módulo

El modelo físico v1.1 incluye 34 tablas propias del dominio, además de las tablas administradas por ASP.NET Core Identity.

## Identity — schema `identity`

ASP.NET Core Identity administrará sus tablas estándar dentro del schema `identity`.

La tabla principal de usuarios será:

* `identity.AspNetUsers`.

---

## Commerce — schema `commerce`

1. `commerce.DireccionUsuario`
2. `commerce.Categoria`
3. `commerce.Producto`
4. `commerce.Publicacion`
5. `commerce.Carrito`
6. `commerce.DetalleCarrito`
7. `commerce.Pedido`
8. `commerce.DireccionPedido`
9. `commerce.DetallePedido`
10. `commerce.Pago`
11. `commerce.HistorialPedido`

Total: 11 tablas.

---

## PcBuilder — schema `pcbuilder`

1. `pcbuilder.ComponentePC`
2. `pcbuilder.Procesador`
3. `pcbuilder.PlacaMadre`
4. `pcbuilder.CompatibilidadProcesadorPlaca`
5. `pcbuilder.MemoriaRAM`
6. `pcbuilder.TarjetaGrafica`
7. `pcbuilder.Almacenamiento`
8. `pcbuilder.FuentePoder`
9. `pcbuilder.Gabinete`
10. `pcbuilder.GabineteFormatoPlaca`
11. `pcbuilder.GabineteFormatoFuente`
12. `pcbuilder.GabineteRadiadorSoportado`
13. `pcbuilder.Refrigeracion`
14. `pcbuilder.RefrigeracionSocket`
15. `pcbuilder.ConfiguracionPC`
16. `pcbuilder.ConfiguracionPCItem`

Total: 16 tablas.

---

## Game — schema `game`

1. `game.Juego`
2. `game.PersonajeJuego`
3. `game.MapaJuego`
4. `game.PreregistroJuego`

Total: 4 tablas.

---

## SoftwareServices — schema `services`

1. `services.ServicioDesarrollo`
2. `services.SolicitudServicio`

Total: 2 tablas.

---

## Corporate — schema `corporate`

1. `corporate.Empresa`

Total: 1 tabla.

---

Total de tablas del dominio:

`11 + 16 + 4 + 2 + 1 = 34`

---

# 42. Estado

El modelo físico v1.1 establece la estructura base de persistencia de NGT Platform para Microsoft SQL Server y la alinea con la arquitectura de monolito modular definida en el ADR 0001.

Quedan definidos:

* tablas del dominio;
* propiedad de datos por módulo;
* schemas SQL por módulo;
* `DbContext` propietario de cada módulo;
* tipos de datos;
* claves primarias;
* claves foráneas;
* relaciones físicas entre módulos;
* nulabilidad;
* restricciones `CHECK`;
* restricciones `UNIQUE`;
* índices principales;
* comportamiento general de eliminación;
* integración con ASP.NET Core Identity;
* separación entre integridad de base de datos y reglas de dominio;
* separación entre persistencia interna de los módulos y contratos de comunicación.

La base de datos inicial seguirá siendo única:

`NgtPlatform`

La existencia de una base de datos compartida no autoriza a los módulos a acceder directamente a la persistencia interna de otros módulos.

El siguiente paso corresponde a implementar la estructura modular del backend en .NET, configurar los `DbContext` de Entity Framework Core y crear las migraciones iniciales de forma controlada.