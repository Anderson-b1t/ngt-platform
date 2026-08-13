# Modelo Lógico de Datos

**Proyecto:** NGT Platform (Neo Genesis Technology)
**Estado:** Borrador inicial
**Nivel:** Modelo lógico, independiente de SQL Server y Entity Framework Core.

## 1. Objetivo

El presente documento define las entidades principales, sus atributos conceptuales y las relaciones necesarias para representar el dominio inicial de NGT Platform.

Este modelo no establece todavía tipos de datos SQL, índices, restricciones físicas ni detalles específicos de Entity Framework Core.

## 2. Criterios de diseño

* Las entidades se organizan según los módulos funcionales definidos en la arquitectura.
* El modelo lógico describe el negocio, no la implementación.
* Los estados se representan conceptualmente; su implementación física se decidirá posteriormente.
* Se evita duplicar información derivable salvo cuando sea necesaria para trazabilidad.
* Las credenciales y detalles internos de autenticación se definirán posteriormente junto con ASP.NET Core Identity.
* La eliminación física no será el comportamiento predeterminado para información transaccional o histórica.

---

# 3. Usuarios y autorización

## Usuario

Representa a una persona registrada en la plataforma.

Atributos:

* UsuarioId
* Nombre
* Email
* Estado
* FechaRegistro

Estados conceptuales:

* Activo
* Inactivo
* Bloqueado

> Los datos internos de autenticación, como hashes de contraseña, tokens y tablas propias de Identity, no forman parte todavía de este modelo lógico.

## Rol

Representa una función de autorización asignable a un usuario.

Atributos:

* RolId
* Nombre
* Descripcion

Roles iniciales:

* Usuario
* Administrador

## UsuarioRol

Representa la asignación de roles a usuarios.

Relaciones:

* Un Usuario puede tener uno o varios Roles.
* Un Rol puede pertenecer a uno o varios Usuarios.

Relación conceptual:

Usuario N ─── N Rol

---

# 4. Catálogo y comercio electrónico

## Categoria

Representa una clasificación comercial de productos.

Atributos:

* CategoriaId
* Nombre
* Descripcion
* Activa

Relación:

Categoria 1 ─── N Producto

## Producto

Representa cualquier artículo comercializable dentro de NGT Platform.

Atributos:

* ProductoId
* CategoriaId
* Nombre
* Descripcion
* Precio
* Stock
* TipoProducto
* Activo

Tipos conceptuales:

* ComponentePC
* Comic
* Manga

Relación:

* Una Categoria puede contener muchos Productos.
* Un Producto pertenece a una Categoria.

## ComponentePC

Representa la información técnica adicional de un Producto utilizado para ensamblar computadoras.

Atributos:

* ProductoId
* TipoComponente
* Marca
* Modelo

Tipos conceptuales iniciales:

* Procesador
* PlacaMadre
* MemoriaRAM
* TarjetaGrafica
* Almacenamiento
* FuentePoder
* Gabinete
* Refrigeracion

Relación:

Producto 1 ─── 0..1 ComponentePC

Solo los productos cuyo TipoProducto sea `ComponentePC` podrán poseer esta especialización.

> Las especificaciones técnicas necesarias para validar compatibilidad se detallarán posteriormente en el submodelo del ensamblador antes del modelo físico.

## Publicacion

Representa información editorial asociada a productos como cómics y mangas.

Atributos:

* ProductoId
* TipoPublicacion
* Autor
* Editorial
* ISBN
* Volumen

Tipos conceptuales:

* Comic
* Manga

Relación:

Producto 1 ─── 0..1 Publicacion

---

# 5. Carrito de compra

## Carrito

Representa una selección temporal de productos realizada por un usuario.

Atributos:

* CarritoId
* UsuarioId
* Estado
* FechaCreacion
* FechaActualizacion

Estados conceptuales:

* Activo
* ConvertidoEnPedido
* Abandonado

Relación:

Usuario 1 ─── N Carrito

Un usuario solo deberá mantener un carrito activo simultáneamente.

## DetalleCarrito

Representa un producto incluido dentro de un carrito.

Atributos:

* DetalleCarritoId
* CarritoId
* ProductoId
* Cantidad

Relaciones:

Carrito 1 ─── N DetalleCarrito

Producto 1 ─── N DetalleCarrito

---

# 6. Pedidos

## Pedido

Representa una compra confirmada por un usuario.

Atributos:

* PedidoId
* UsuarioId
* Total
* Estado
* FechaPedido

Estados conceptuales iniciales:

* Pendiente
* Pagado
* EnPreparacion
* Enviado
* Completado
* Cancelado

Relación:

Usuario 1 ─── N Pedido

## DetallePedido

Representa una línea individual dentro de un pedido.

Atributos:

* DetallePedidoId
* PedidoId
* ProductoId
* Cantidad
* PrecioUnitario

Relaciones:

Pedido 1 ─── N DetallePedido

Producto 1 ─── N DetallePedido

`PrecioUnitario` conserva el precio aplicado en el momento de la compra y no depende de cambios posteriores en el precio actual del producto.

## Pago

Representa un intento o transacción de pago asociado a un pedido.

Atributos:

* PagoId
* PedidoId
* Monto
* Metodo
* Estado
* ReferenciaExterna
* FechaPago

Estados conceptuales:

* Pendiente
* Aprobado
* Rechazado
* Reembolsado

Relación:

Pedido 1 ─── N Pago

Se permite más de un registro porque un pedido puede tener varios intentos de pago.

## HistorialPedido

Registra cambios de estado relevantes de un pedido.

Atributos:

* HistorialPedidoId
* PedidoId
* UsuarioId
* EstadoAnterior
* EstadoNuevo
* FechaCambio

Relaciones:

Pedido 1 ─── N HistorialPedido

Usuario 0..1 ─── N HistorialPedido

`UsuarioId` puede ser opcional cuando el cambio haya sido realizado automáticamente por el sistema.

---

# 7. Ensamblador de PC

## ConfiguracionPC

Representa una configuración de computadora creada y guardada por un usuario.

Atributos:

* ConfiguracionPCId
* UsuarioId
* Nombre
* EstadoValidacion
* FechaCreacion
* FechaActualizacion

Estados conceptuales:

* Incompleta
* Valida
* Incompatible

Relación:

Usuario 1 ─── N ConfiguracionPC

## ConfiguracionPCItem

Representa un componente incluido dentro de una configuración.

Atributos:

* ConfiguracionPCItemId
* ConfiguracionPCId
* ProductoId
* Cantidad

Relaciones:

ConfiguracionPC 1 ─── N ConfiguracionPCItem

ComponentePC 1 ─── N ConfiguracionPCItem

Esta entidad sustituye una relación N-N directa entre `ConfiguracionPC` y `Producto`, ya que la configuración necesita información propia como la cantidad de cada componente.

## Reglas conceptuales del ensamblador

Una configuración solo podrá considerarse válida cuando:

* Contenga los componentes obligatorios definidos por las reglas del sistema.
* Los componentes sean compatibles entre sí.
* Las cantidades sean válidas.
* Los productos utilizados sean componentes activos y comercializables.

Una configuración incompleta o incompatible no podrá utilizarse para completar una compra como equipo ensamblado.

La matriz detallada de compatibilidad se definirá en una iteración específica del módulo de ensamblador.

---

# 8. Videojuego

## Juego

Representa un videojuego presentado dentro de NGT Platform.

Atributos:

* JuegoId
* Nombre
* Descripcion
* Estado
* FechaAnuncio
* FechaLanzamientoEstimada

Estados conceptuales:

* EnDesarrollo
* Alpha
* Beta
* Lanzado

## PersonajeJuego

Representa un personaje perteneciente a un juego.

Atributos:

* PersonajeId
* JuegoId
* Nombre
* Descripcion

Relación:

Juego 1 ─── N PersonajeJuego

## MapaJuego

Representa un escenario o mapa mostrado como contenido del videojuego.

Atributos:

* MapaId
* JuegoId
* Nombre
* Descripcion

Relación:

Juego 1 ─── N MapaJuego

## PreregistroJuego

Representa el preregistro de una persona interesada en un juego.

Atributos:

* PreregistroId
* JuegoId
* UsuarioId
* Email
* FechaRegistro

Relaciones:

Juego 1 ─── N PreregistroJuego

Usuario 0..1 ─── N PreregistroJuego

El preregistro podrá realizarse sin necesidad de tener una cuenta registrada.

La combinación Juego + Email deberá ser única conceptualmente para evitar preregistros duplicados.

---

# 9. Servicios de desarrollo

## ServicioDesarrollo

Representa un servicio profesional ofrecido por Neo Genesis Technology.

Atributos:

* ServicioId
* Nombre
* Descripcion
* PrecioBase
* Activo

## SolicitudServicio

Representa una solicitud realizada por un usuario respecto a un servicio de desarrollo.

Atributos:

* SolicitudId
* UsuarioId
* ServicioId
* DescripcionRequerimiento
* Estado
* FechaSolicitud

Estados conceptuales iniciales:

* Pendiente
* EnRevision
* Aprobada
* Rechazada
* EnProceso
* Completada
* Cancelada

Relaciones:

Usuario 1 ─── N SolicitudServicio

ServicioDesarrollo 1 ─── N SolicitudServicio

---

# 10. Contenido corporativo

## Empresa

Representa la información institucional de Neo Genesis Technology.

Atributos:

* EmpresaId
* Nombre
* Descripcion
* Mision
* Vision
* FechaCreacion
* Activa

Inicialmente se espera una única empresa activa, aunque el modelo no dependerá estrictamente de esa restricción.

---

# 11. Resumen de relaciones

Usuario N ─── N Rol

Categoria 1 ─── N Producto

Producto 1 ─── 0..1 ComponentePC

Producto 1 ─── 0..1 Publicacion

Usuario 1 ─── N Carrito

Carrito 1 ─── N DetalleCarrito

Producto 1 ─── N DetalleCarrito

Usuario 1 ─── N Pedido

Pedido 1 ─── N DetallePedido

Producto 1 ─── N DetallePedido

Pedido 1 ─── N Pago

Pedido 1 ─── N HistorialPedido

Usuario 0..1 ─── N HistorialPedido

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

# 12. Elementos deliberadamente fuera de esta iteración

No se modelan todavía en detalle:

* Direcciones y logística de entrega.
* Integración con pasarelas de pago específicas.
* Cupones y promociones.
* Reseñas de productos.
* Listas de deseos.
* Facturación electrónica.
* Inventario por múltiples almacenes.
* Especificaciones técnicas completas de cada tipo de componente.
* Reglas detalladas de compatibilidad del ensamblador.
* Gestión de archivos e imágenes.
* Notificaciones.
* Auditoría global del sistema.

Estos elementos se añadirán únicamente cuando una iteración funcional los requiera.

