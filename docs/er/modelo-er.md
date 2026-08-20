# Modelo Entidad–Relación

**Proyecto:** NGT Platform (Neo Genesis Technology)
**Versión:** 1.1
**Estado:** Aprobado
**Origen:** Modelo lógico v1.1 y ADR 0001 (monolito modular)

---

# 1. Objetivo

Representar las entidades, atributos, claves y relaciones principales de NGT Platform antes de su transformación al modelo físico de SQL Server.

En esta etapa no se definen tipos de datos, índices, restricciones físicas ni configuraciones específicas de Entity Framework Core.

Las entidades se agrupan por módulos de negocio y las relaciones entre módulos se muestran de forma explícita.

---

# 2. Módulo Identity

## Usuario

* UsuarioId (PK)
* Nombre
* Email
* Estado
* FechaRegistro

## Rol

* RolId (PK)
* Nombre
* Descripcion

## UsuarioRol

* UsuarioId (PK, FK)
* RolId (PK, FK)

```mermaid
erDiagram
    Usuario ||--o{ UsuarioRol : posee
    Rol ||--o{ UsuarioRol : asigna
```

Relaciones:

* Usuario N ─── N Rol, mediante UsuarioRol.

---

# 3. Módulo Commerce — catálogo y datos comerciales

## Categoria

* CategoriaId (PK)
* Nombre
* Descripcion
* Activa

## Producto

* ProductoId (PK)
* CategoriaId (FK)
* Nombre
* Descripcion
* Precio
* Stock
* Activo

## DireccionUsuario

* DireccionUsuarioId (PK)
* UsuarioId (FK)
* Alias
* NombreDestinatario
* Telefono
* LineaDireccion1
* LineaDireccion2
* Pais
* Region
* Provincia
* Distrito
* CodigoPostal
* Referencia
* EsPredeterminada
* Activa

## Publicacion

* ProductoId (PK, FK)
* TipoPublicacion
* Autor
* Editorial
* ISBN
* Volumen

```mermaid
erDiagram
    Usuario ||--o{ DireccionUsuario : registra
    Categoria ||--o{ Producto : contiene
    Producto ||--o| Publicacion : especializa
```

Relaciones:

* Usuario 1 ─── N DireccionUsuario.
* Categoria 1 ─── N Producto.
* Producto 1 ─── 0..1 Publicacion.

Un producto asociado a `Publicacion` corresponde a la rama editorial de Commerce. La posible asociación con `ComponentePC` se representa en el módulo PcBuilder.

---

# 4. Módulo Commerce — carrito de compra

## Carrito

* CarritoId (PK)
* UsuarioId (FK)
* Estado
* FechaCreacion
* FechaActualizacion

## DetalleCarrito

* DetalleCarritoId (PK)
* CarritoId (FK)
* ProductoId (FK)
* Cantidad

```mermaid
erDiagram
    Usuario ||--o{ Carrito : posee
    Carrito ||--o{ DetalleCarrito : contiene
    Producto ||--o{ DetalleCarrito : incluye
```

Relaciones:

* Usuario 1 ─── N Carrito.
* Carrito 1 ─── N DetalleCarrito.
* Producto 1 ─── N DetalleCarrito.

La combinación Carrito + Producto debe ser única.

---

# 5. Módulo Commerce — pedidos, entrega y pagos

## Pedido

* PedidoId (PK)
* UsuarioId (FK)
* Subtotal
* CostoEnvio
* Total
* Estado
* FechaPedido

## DireccionPedido

* DireccionPedidoId (PK)
* PedidoId (FK)
* NombreDestinatario
* Telefono
* LineaDireccion1
* LineaDireccion2
* Pais
* Region
* Provincia
* Distrito
* CodigoPostal
* Referencia

## DetallePedido

* DetallePedidoId (PK)
* PedidoId (FK)
* ProductoId (FK)
* Cantidad
* PrecioUnitario

## Pago

* PagoId (PK)
* PedidoId (FK)
* Monto
* Metodo
* Estado
* ReferenciaExterna
* FechaCreacion
* FechaPago

## HistorialPedido

* HistorialPedidoId (PK)
* PedidoId (FK)
* UsuarioId (FK, opcional)
* EstadoAnterior
* EstadoNuevo
* FechaCambio

```mermaid
erDiagram
    Usuario ||--o{ Pedido : realiza

    Pedido ||--|| DireccionPedido : posee
    Pedido ||--|{ DetallePedido : contiene
    Producto ||--o{ DetallePedido : incluye

    Pedido ||--o{ Pago : registra
    Pedido ||--o{ HistorialPedido : genera
    Usuario o|--o{ HistorialPedido : realiza
```

Relaciones:

* Usuario 1 ─── N Pedido.
* Pedido 1 ─── 1 DireccionPedido.
* Pedido 1 ─── N DetallePedido.
* Producto 1 ─── N DetallePedido.
* Pedido 1 ─── N Pago.
* Pedido 1 ─── N HistorialPedido.
* Usuario 0..1 ─── N HistorialPedido.

`DireccionPedido` conserva la dirección utilizada al confirmar la compra y no depende de cambios posteriores en `DireccionUsuario`.

---

# 6. Módulo PcBuilder — componentes de PC

## ComponentePC

* ComponentePCId (PK)
* ProductoId (FK, único)
* Marca
* Modelo

Relación con Commerce:

* Producto 1 ─── 0..1 ComponentePC.

`ProductoId` es único en `ComponentePC`, de modo que un producto puede estar asociado como máximo a un componente técnico.


## Procesador

* ComponentePCId (PK, FK)
* Socket
* TdpWatts
* ConsumoEstimadoWatts
* TieneGraficosIntegrados
* IncluyeRefrigeracion

## PlacaMadre

* ComponentePCId (PK, FK)
* Socket
* Chipset
* FormatoPlaca
* TipoMemoria
* CantidadSlotsMemoria
* CapacidadMaximaMemoriaGB
* CantidadSlotsM2
* CantidadPuertosSata
* ConsumoEstimadoWatts

## MemoriaRAM

* ComponentePCId (PK, FK)
* TipoMemoria
* CapacidadTotalGB
* ModulosPorKit
* FrecuenciaMTs
* ConsumoEstimadoWatts

## TarjetaGrafica

* ComponentePCId (PK, FK)
* LongitudMm
* ConsumoEstimadoWatts
* PotenciaFuenteRecomendadaWatts

## Almacenamiento

* ComponentePCId (PK, FK)
* TipoAlmacenamiento
* Interfaz
* CapacidadGB
* ConsumoEstimadoWatts

## FuentePoder

* ComponentePCId (PK, FK)
* PotenciaWatts
* FormatoFuente

## Gabinete

* ComponentePCId (PK, FK)
* LongitudMaximaGpuMm
* AlturaMaximaRefrigeracionMm

## Refrigeracion

* ComponentePCId (PK, FK)
* TipoRefrigeracion
* AlturaMm
* TamanoRadiadorMm
* ConsumoEstimadoWatts

```mermaid
erDiagram
    Producto ||--o| ComponentePC : asocia
    ComponentePC ||--o| Procesador : especializa
    ComponentePC ||--o| PlacaMadre : especializa
    ComponentePC ||--o| MemoriaRAM : especializa
    ComponentePC ||--o| TarjetaGrafica : especializa
    ComponentePC ||--o| Almacenamiento : especializa
    ComponentePC ||--o| FuentePoder : especializa
    ComponentePC ||--o| Gabinete : especializa
    ComponentePC ||--o| Refrigeracion : especializa
```

Cada `ComponentePC` corresponde exactamente a uno de estos subtipos.

---

# 7. Módulo PcBuilder — compatibilidad procesador y placa madre

## CompatibilidadProcesadorPlaca

* ProcesadorId (PK, FK)
* PlacaMadreId (PK, FK)
* RequiereActualizacionBios
* Observacion

```mermaid
erDiagram
    Procesador ||--o{ CompatibilidadProcesadorPlaca : participa
    PlacaMadre ||--o{ CompatibilidadProcesadorPlaca : participa
```

Relación conceptual:

Procesador N ─── N PlacaMadre

mediante `CompatibilidadProcesadorPlaca`.

La existencia de una combinación representa compatibilidad técnica conocida entre ambos componentes.

---

# 8. Módulo PcBuilder — compatibilidad del gabinete

## GabineteFormatoPlaca

* GabineteId (PK, FK)
* FormatoPlaca (PK)

## GabineteFormatoFuente

* GabineteId (PK, FK)
* FormatoFuente (PK)

## GabineteRadiadorSoportado

* GabineteId (PK, FK)
* TamanoRadiadorMm (PK)

```mermaid
erDiagram
    Gabinete ||--o{ GabineteFormatoPlaca : admite
    Gabinete ||--o{ GabineteFormatoFuente : admite
    Gabinete ||--o{ GabineteRadiadorSoportado : admite
```

Relaciones:

* Gabinete 1 ─── N GabineteFormatoPlaca.
* Gabinete 1 ─── N GabineteFormatoFuente.
* Gabinete 1 ─── N GabineteRadiadorSoportado.

---

# 9. Módulo PcBuilder — compatibilidad de refrigeración

## RefrigeracionSocket

* RefrigeracionId (PK, FK)
* Socket (PK)

```mermaid
erDiagram
    Refrigeracion ||--o{ RefrigeracionSocket : soporta
```

Relación:

Refrigeracion 1 ─── N RefrigeracionSocket.

---

# 10. Módulo PcBuilder — configuraciones de PC

## ConfiguracionPC

* ConfiguracionPCId (PK)
* UsuarioId (FK)
* Nombre
* EstadoCompatibilidad
* FechaCreacion
* FechaActualizacion
* FechaUltimaValidacion

## ConfiguracionPCItem

* ConfiguracionPCItemId (PK)
* ConfiguracionPCId (FK)
* ComponentePCId (FK)
* Cantidad

```mermaid
erDiagram
    Usuario ||--o{ ConfiguracionPC : crea
    ConfiguracionPC ||--o{ ConfiguracionPCItem : contiene
    ComponentePC ||--o{ ConfiguracionPCItem : utiliza
```

Relaciones:

* Usuario 1 ─── N ConfiguracionPC.
* ConfiguracionPC 1 ─── N ConfiguracionPCItem.
* ComponentePC 1 ─── N ConfiguracionPCItem.

La combinación ConfiguracionPC + ComponentePC debe ser única.

---

# 11. Módulo Game — videojuego

## Juego

* JuegoId (PK)
* Nombre
* Descripcion
* Estado
* FechaAnuncio
* FechaLanzamientoEstimada

## PersonajeJuego

* PersonajeId (PK)
* JuegoId (FK)
* Nombre
* Descripcion

## MapaJuego

* MapaId (PK)
* JuegoId (FK)
* Nombre
* Descripcion

## PreregistroJuego

* PreregistroId (PK)
* JuegoId (FK)
* UsuarioId (FK, opcional)
* Email
* FechaRegistro

```mermaid
erDiagram
    Juego ||--o{ PersonajeJuego : contiene
    Juego ||--o{ MapaJuego : contiene
    Juego ||--o{ PreregistroJuego : recibe
    Usuario o|--o{ PreregistroJuego : realiza
```

Relaciones:

* Juego 1 ─── N PersonajeJuego.
* Juego 1 ─── N MapaJuego.
* Juego 1 ─── N PreregistroJuego.
* Usuario 0..1 ─── N PreregistroJuego.

La combinación Juego + Email debe ser única.

---

# 12. Módulo SoftwareServices — servicios de desarrollo

## ServicioDesarrollo

* ServicioId (PK)
* Nombre
* Descripcion
* PrecioBase
* Activo

## SolicitudServicio

* SolicitudId (PK)
* UsuarioId (FK)
* ServicioId (FK)
* DescripcionRequerimiento
* Estado
* FechaSolicitud

```mermaid
erDiagram
    Usuario ||--o{ SolicitudServicio : realiza
    ServicioDesarrollo ||--o{ SolicitudServicio : recibe
```

Relaciones:

* Usuario 1 ─── N SolicitudServicio.
* ServicioDesarrollo 1 ─── N SolicitudServicio.

---

# 13. Módulo Corporate — contenido corporativo

## Empresa

* EmpresaId (PK)
* Nombre
* Descripcion
* Mision
* Vision
* FechaCreacion
* Activa

Actualmente `Empresa` no requiere relaciones con otras entidades del dominio.

---

# 14. Relaciones entre módulos

Las siguientes relaciones conectan módulos diferentes sin transferir la propiedad conceptual de sus entidades:

* `Usuario` se relaciona con entidades de Commerce, PcBuilder, Game y SoftwareServices como identidad del actor.
* `Producto` pertenece a Commerce y puede asociarse con un `ComponentePC` perteneciente a PcBuilder.
* Commerce conserva la propiedad de nombre, precio, stock, categoría y estado comercial del producto.
* PcBuilder conserva la propiedad de las especificaciones técnicas, compatibilidad y configuraciones.
* `ConfiguracionPCItem` referencia `ComponentePC`, no directamente `Producto`; la traducción al producto comercial asociado ocurre al iniciar una compra desde el ensamblador.

---

# 15. Resumen general de relaciones

Usuario N ─── N Rol
mediante UsuarioRol

Usuario 1 ─── N DireccionUsuario

Categoria 1 ─── N Producto

Producto 1 ─── 0..1 ComponentePC

Producto 1 ─── 0..1 Publicacion

Usuario 1 ─── N Carrito

Carrito 1 ─── N DetalleCarrito

Producto 1 ─── N DetalleCarrito

Usuario 1 ─── N Pedido

Pedido 1 ─── 1 DireccionPedido

Pedido 1 ─── N DetallePedido

Producto 1 ─── N DetallePedido

Pedido 1 ─── N Pago

Pedido 1 ─── N HistorialPedido

Usuario 0..1 ─── N HistorialPedido

ComponentePC 1 ─── 0..1 Procesador

ComponentePC 1 ─── 0..1 PlacaMadre

ComponentePC 1 ─── 0..1 MemoriaRAM

ComponentePC 1 ─── 0..1 TarjetaGrafica

ComponentePC 1 ─── 0..1 Almacenamiento

ComponentePC 1 ─── 0..1 FuentePoder

ComponentePC 1 ─── 0..1 Gabinete

ComponentePC 1 ─── 0..1 Refrigeracion

Procesador N ─── N PlacaMadre
mediante CompatibilidadProcesadorPlaca

Gabinete 1 ─── N GabineteFormatoPlaca

Gabinete 1 ─── N GabineteFormatoFuente

Gabinete 1 ─── N GabineteRadiadorSoportado

Refrigeracion 1 ─── N RefrigeracionSocket

Usuario 1 ─── N ConfiguracionPC

ConfiguracionPC 1 ─── N ConfiguracionPCItem

ComponentePC 1 ─── N ConfiguracionPCItem

Juego 1 ─── N PersonajeJuego

Juego 1 ─── N MapaJuego

Juego 1 ─── N PreregistroJuego

Usuario 0..1 ─── N PreregistroJuego

Usuario 1 ─── N SolicitudServicio

ServicioDesarrollo 1 ─── N SolicitudServicio

---

# 16. Estado del modelo

El modelo entidad–relación v1.1 define las entidades, atributos, claves y cardinalidades necesarias para representar el dominio inicial de NGT Platform.

Este modelo sirve como base conceptual del modelo físico v1.1, donde se definen tipos de datos, nulabilidad, restricciones, índices, schemas y demás características específicas de SQL Server.