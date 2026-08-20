# Modelo Lógico de Datos

**Proyecto:** NGT Platform (Neo Genesis Technology)
**Versión:** 1.1
**Estado:** Aprobado
**Origen:** ADR 0001 (monolito modular) y modelo lógico v1.0
**Nivel:** Modelo lógico independiente de SQL Server, Entity Framework Core y detalles físicos de persistencia.

---

# 1. Objetivo

El presente documento define las entidades principales, sus atributos conceptuales, relaciones, restricciones y reglas de negocio necesarias para representar el dominio inicial de NGT Platform.

El modelo constituye la base para la posterior elaboración de:

* Modelo entidad–relación.
* Modelo físico de base de datos.
* Implementación con SQL Server.
* Mapeo con Entity Framework Core.
* Lógica de dominio del backend.

Este documento no define todavía:

* tipos de datos SQL;
* índices;
* nombres de constraints;
* estrategias de herencia de Entity Framework Core;
* estructuras internas de ASP.NET Core Identity;
* detalles específicos de infraestructura.

---

# 2. Criterios de diseño

* Las entidades se organizan según los módulos de negocio definidos en la arquitectura: Identity, Commerce, PcBuilder, Game, SoftwareServices y Corporate.
* Cada entidad pertenece conceptualmente a un módulo responsable de sus reglas de negocio.
* Las relaciones entre módulos se mantienen explícitas, evitando convertir una relación conceptual en acceso indiscriminado a datos internos de otro módulo.
* El modelo lógico representa el negocio y no la implementación técnica.
* Las relaciones se modelan de forma explícita cuando contienen información o reglas propias.
* Los estados se representan conceptualmente; su implementación física se decidirá posteriormente.
* Se evita almacenar información que pueda derivarse, salvo cuando sea necesaria para trazabilidad histórica.
* Las operaciones transaccionales importantes conservan su información histórica.
* No se utiliza borrado físico como comportamiento predeterminado para información histórica o transaccional.
* Las reglas de compatibilidad del ensamblador pertenecen a la lógica de dominio y no a la base de datos.
* La compatibilidad técnica y la disponibilidad comercial son conceptos independientes.
* La autenticación será implementada posteriormente mediante ASP.NET Core Identity.
* El diseño evita acoplar el dominio a las tablas internas de Identity.
* La arquitectura y el modelo evolucionarán mediante iteraciones controladas.

---

# 3. Módulo Identity

## Usuario

Representa a una persona registrada en NGT Platform.

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

Reglas:

* El email identifica de forma única conceptualmente a un usuario.
* Los datos internos de autenticación no pertenecen al modelo lógico del negocio.
* PasswordHash, tokens, recuperación de contraseña y demás elementos serán responsabilidad de ASP.NET Core Identity.

---

## Rol

Representa una función de autorización asignable a un usuario.

Atributos:

* RolId
* Nombre
* Descripcion

Roles iniciales:

* Usuario
* Administrador

La lista podrá evolucionar posteriormente hacia permisos más granulares.

---

## UsuarioRol

Representa la asignación de roles a usuarios.

Atributos conceptuales:

* UsuarioId
* RolId

Relaciones:

Usuario 1 ─── N UsuarioRol

Rol 1 ─── N UsuarioRol

Equivalencia conceptual:

Usuario N ─── N Rol

> En el modelo físico esta relación podrá ser proporcionada directamente por ASP.NET Core Identity. No se crearán tablas duplicadas si Identity ya proporciona esta funcionalidad.

---

# 4. Módulo Commerce — catálogo y datos comerciales

## Categoria

Representa una clasificación comercial de productos.

Atributos:

* CategoriaId
* Nombre
* Descripcion
* Activa

Relación:

Categoria 1 ─── N Producto

Una categoría puede contener múltiples productos.

---

## Producto

Representa cualquier artículo comercializable dentro de NGT Platform.

Atributos:

* ProductoId
* CategoriaId
* Nombre
* Descripcion
* Precio
* Stock
* Activo

Relación:

Categoria 1 ─── N Producto

Extensiones iniciales del producto:

* `Publicacion`, como especialización comercial dentro de Commerce.
* `ComponentePC`, como información técnica asociada administrada por PcBuilder.

Reglas:

* Todo producto pertenece a una categoría.
* El precio debe representar un valor comercial válido.
* El stock no puede representar una cantidad negativa.
* La desactivación de un producto impide nuevas ventas sin eliminar su historial.
* Un producto no puede estar asociado simultáneamente a `Publicacion` y `ComponentePC`.
* La información comercial común permanece en `Producto`.
* La información editorial pertenece a `Publicacion` dentro de Commerce.
* La información técnica de componentes pertenece a `ComponentePC` dentro de PcBuilder.

---

## Publicacion

Representa la información editorial asociada a un producto comercializado como cómic o manga.

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

Reglas:

* `TipoPublicacion` determina si el producto corresponde a un cómic o manga.
* Los datos comerciales generales continúan perteneciendo a `Producto`.

---

## DireccionUsuario

Representa una dirección guardada por un usuario para utilizarla posteriormente en sus compras.

Aunque se relaciona con `Usuario`, pertenece conceptualmente a Commerce porque su finalidad es apoyar el proceso de compra y entrega.

Atributos:

* DireccionUsuarioId
* UsuarioId
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

Relación:

Usuario 1 ─── N DireccionUsuario

Reglas:

* Un usuario puede registrar múltiples direcciones.
* Solo una dirección activa podrá considerarse predeterminada simultáneamente.
* Una dirección guardada puede modificarse sin alterar pedidos realizados anteriormente.

---

# 5. Módulo Commerce — carrito de compra

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

Reglas:

* Un usuario puede haber tenido múltiples carritos a lo largo del tiempo.
* Solo puede existir un carrito activo simultáneamente por usuario.
* Un carrito convertido en pedido no vuelve al estado activo.

---

## DetalleCarrito

Representa un producto incluido en un carrito.

Atributos:

* DetalleCarritoId
* CarritoId
* ProductoId
* Cantidad

Relaciones:

Carrito 1 ─── N DetalleCarrito

Producto 1 ─── N DetalleCarrito

Reglas:

* La cantidad debe ser mayor que cero.
* Un producto debe aparecer como máximo una vez dentro del mismo carrito; los incrementos actualizan la cantidad existente.
* El precio actual no se almacena como dato histórico en el carrito.
* Stock, actividad y precio se validarán nuevamente antes de confirmar la compra.

---

# 6. Módulo Commerce — pedidos, entrega y pagos

## Pedido

Representa una compra confirmada por un usuario.

Atributos:

* PedidoId
* UsuarioId
* Subtotal
* CostoEnvio
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

Reglas:

* Un pedido debe contener al menos un detalle.
* `Subtotal` representa la suma de sus líneas de pedido.
* `CostoEnvio` representa el costo de entrega aplicado.
* `Total` representa el importe definitivo del pedido.
* Conceptualmente:

Subtotal + CostoEnvio = Total

* Una vez confirmado, el pedido conserva la información comercial utilizada en ese momento.

---

## DireccionPedido

Representa una copia histórica de la dirección utilizada para entregar un pedido.

Atributos:

* DireccionPedidoId
* PedidoId
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

Relación:

Pedido 1 ─── 1 DireccionPedido

Reglas:

* La información se copia desde la dirección seleccionada durante la compra.
* No depende posteriormente de `DireccionUsuario`.
* Modificar o eliminar una dirección del usuario no modifica pedidos históricos.
* Esta entidad representa un snapshot transaccional.

---

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

Reglas:

* La cantidad debe ser mayor que cero.
* `PrecioUnitario` conserva el precio aplicado al producto en el momento de la compra.
* Cambios posteriores en `Producto.Precio` no modifican pedidos históricos.

---

## Pago

Representa un intento o transacción de pago asociado a un pedido.

Atributos:

* PagoId
* PedidoId
* Monto
* Metodo
* Estado
* ReferenciaExterna
* FechaCreacion
* FechaPago

Estados conceptuales:

* Pendiente
* Aprobado
* Rechazado
* Reembolsado
* Cancelado

Relación:

Pedido 1 ─── N Pago

Reglas:

* Un pedido puede tener múltiples intentos de pago.
* Solo los pagos aprobados representan dinero confirmado.
* `ReferenciaExterna` permitirá relacionar el pago con una futura pasarela.
* La integración concreta con proveedores de pago queda fuera del modelo lógico inicial.

---

## HistorialPedido

Registra los cambios relevantes de estado de un pedido.

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

Reglas:

* `UsuarioId` es opcional.
* Un valor ausente representa un cambio generado automáticamente por el sistema.
* El historial es append-only conceptualmente.
* Los registros históricos no deben sobrescribirse.

---

# 7. Módulo PcBuilder — ensamblador de PC

El ensamblador permite crear configuraciones de computadora y determinar si constituyen equipos funcionales y técnicamente compatibles.

La base de datos almacena especificaciones técnicas.

El backend aplica las reglas de compatibilidad.

---

## ComponentePC

Representa la entidad técnica de PcBuilder asociada a un producto comercializable de Commerce.

Atributos:

* ComponentePCId
* ProductoId
* Marca
* Modelo

Relación:

Producto 1 ─── 0..1 ComponentePC

Reglas:

* `ComponentePCId` identifica al componente dentro del dominio técnico de PcBuilder.
* `ProductoId` identifica el producto comercial asociado y mantiene la relación con Commerce.
* Un producto puede estar asociado como máximo a un `ComponentePC`.
* Un `ComponentePC` debe corresponder a exactamente uno de los subtipos técnicos definidos para el ensamblador.
* El subtipo técnico determina las especificaciones requeridas.
* No se almacena `TipoComponente` como segunda fuente de verdad; el subtipo correspondiente determina conceptualmente su naturaleza.

Subtipos iniciales:

* Procesador
* PlacaMadre
* MemoriaRAM
* TarjetaGrafica
* Almacenamiento
* FuentePoder
* Gabinete
* Refrigeracion

---

## Procesador

Representa las características técnicas de un procesador.

Atributos:

* ComponentePCId
* Socket
* TdpWatts
* ConsumoEstimadoWatts
* TieneGraficosIntegrados
* IncluyeRefrigeracion

Relación:

ComponentePC 1 ─── 0..1 Procesador

Reglas relevantes:

* El socket debe corresponder al soportado por la placa madre.
* La combinación procesador/placa debe estar reconocida como compatible.
* Si no posee gráficos integrados, deberá existir una tarjeta gráfica dedicada.
* Si no incluye refrigeración, deberá existir una solución de refrigeración compatible.

`TdpWatts` y `ConsumoEstimadoWatts` representan conceptos diferentes: el primero es una especificación térmica y el segundo se utiliza para estimaciones energéticas.

---

## PlacaMadre

Representa las características técnicas de una placa madre.

Atributos:

* ComponentePCId
* Socket
* Chipset
* FormatoPlaca
* TipoMemoria
* CantidadSlotsMemoria
* CapacidadMaximaMemoriaGB
* CantidadSlotsM2
* CantidadPuertosSata
* ConsumoEstimadoWatts

Relación:

ComponentePC 1 ─── 0..1 PlacaMadre

Reglas relevantes:

* Debe ser compatible con el procesador seleccionado.
* El tipo de memoria debe corresponder con la memoria RAM.
* Los módulos instalados no pueden superar los slots disponibles.
* La memoria instalada no puede superar la capacidad máxima.
* Su formato debe ser soportado por el gabinete.
* Los dispositivos de almacenamiento no pueden superar la conectividad disponible.

---

## CompatibilidadProcesadorPlaca

Representa una combinación técnicamente soportada entre un procesador y una placa madre.

Atributos:

* ProcesadorId
* PlacaMadreId
* RequiereActualizacionBios
* Observacion

Relaciones:

Procesador 1 ─── N CompatibilidadProcesadorPlaca

PlacaMadre 1 ─── N CompatibilidadProcesadorPlaca

Equivalencia conceptual:

Procesador N ─── N PlacaMadre

Reglas:

* El socket debe coincidir.
* La existencia de una combinación registrada confirma la compatibilidad de plataforma.
* Puede registrarse si la combinación requiere una actualización de BIOS.
* La información deberá provenir de especificaciones técnicas previamente validadas.

---

## MemoriaRAM

Representa las características técnicas de un producto de memoria RAM.

Atributos:

* ComponentePCId
* TipoMemoria
* CapacidadTotalGB
* ModulosPorKit
* FrecuenciaMTs
* ConsumoEstimadoWatts

Relación:

ComponentePC 1 ─── 0..1 MemoriaRAM

Reglas relevantes:

* El tipo de memoria debe coincidir con el soportado por la placa madre.
* Los módulos instalados se calculan considerando `ModulosPorKit × Cantidad`.
* La cantidad total de módulos no puede superar los slots de memoria.
* La capacidad total instalada no puede superar el máximo soportado.

---

## TarjetaGrafica

Representa las características técnicas de una tarjeta gráfica.

Atributos:

* ComponentePCId
* LongitudMm
* ConsumoEstimadoWatts
* PotenciaFuenteRecomendadaWatts

Relación:

ComponentePC 1 ─── 0..1 TarjetaGrafica

Reglas relevantes:

* La longitud no puede superar la admitida por el gabinete.
* La fuente debe proporcionar potencia suficiente.
* Una configuración sin gráficos integrados necesita una GPU dedicada.

Alcance inicial:

* Máximo una tarjeta gráfica dedicada por configuración.
* Multi-GPU queda fuera de la versión inicial.

---

## Almacenamiento

Representa las características técnicas de una unidad de almacenamiento.

Atributos:

* ComponentePCId
* TipoAlmacenamiento
* Interfaz
* CapacidadGB
* ConsumoEstimadoWatts

Tipos conceptuales:

* HDD
* SSD

Interfaces iniciales:

* SATA
* M2NVMe

Relación:

ComponentePC 1 ─── 0..1 Almacenamiento

Reglas relevantes:

* Un equipo funcional requiere al menos una unidad de almacenamiento.
* Los dispositivos SATA no pueden superar los puertos SATA disponibles.
* Los dispositivos M.2 no pueden superar los slots M.2 disponibles.

---

## FuentePoder

Representa las características técnicas de una fuente de alimentación.

Atributos:

* ComponentePCId
* PotenciaWatts
* FormatoFuente

Relación:

ComponentePC 1 ─── 0..1 FuentePoder

Reglas relevantes:

* Debe proporcionar potencia suficiente para toda la configuración.
* Su formato debe ser soportado por el gabinete.

---

## Gabinete

Representa las características físicas de un gabinete.

Atributos:

* ComponentePCId
* LongitudMaximaGpuMm
* AlturaMaximaRefrigeracionMm

Relación:

ComponentePC 1 ─── 0..1 Gabinete

Un gabinete puede admitir varios formatos de placa madre, fuentes de poder y tamaños de radiador.

---

## GabineteFormatoPlaca

Representa un formato de placa madre admitido por un gabinete.

Atributos:

* GabineteId
* FormatoPlaca

Relación:

Gabinete 1 ─── N GabineteFormatoPlaca

Valores conceptuales iniciales:

* ATX
* MicroATX
* MiniITX

---

## GabineteFormatoFuente

Representa un formato de fuente admitido por un gabinete.

Atributos:

* GabineteId
* FormatoFuente

Relación:

Gabinete 1 ─── N GabineteFormatoFuente

Valores conceptuales iniciales:

* ATX
* SFX

---

## GabineteRadiadorSoportado

Representa un tamaño de radiador admitido por un gabinete.

Atributos:

* GabineteId
* TamanoRadiadorMm

Relación:

Gabinete 1 ─── N GabineteRadiadorSoportado

Valores conceptuales iniciales:

* 120
* 240
* 280
* 360

---

## Refrigeracion

Representa una solución de refrigeración para el procesador.

Atributos:

* ComponentePCId
* TipoRefrigeracion
* AlturaMm
* TamanoRadiadorMm
* ConsumoEstimadoWatts

Tipos conceptuales:

* Aire
* LiquidaAIO

Relación:

ComponentePC 1 ─── 0..1 Refrigeracion

Reglas:

* `AlturaMm` aplica principalmente a refrigeración por aire.
* `TamanoRadiadorMm` aplica a refrigeración líquida AIO.
* Debe ser compatible con el socket del procesador.
* Debe caber físicamente en el gabinete.

---

## RefrigeracionSocket

Representa un socket admitido por una solución de refrigeración.

Atributos:

* RefrigeracionId
* Socket

Relación:

Refrigeracion 1 ─── N RefrigeracionSocket

Una solución de refrigeración puede soportar múltiples sockets.

---

## ConfiguracionPC

Representa una configuración de computadora creada y guardada por un usuario.

Atributos:

* ConfiguracionPCId
* UsuarioId
* Nombre
* EstadoCompatibilidad
* FechaCreacion
* FechaActualizacion
* FechaUltimaValidacion

Estados conceptuales:

* Incompleta
* Compatible
* Incompatible

Relación:

Usuario 1 ─── N ConfiguracionPC

Reglas:

* El estado representa el resultado de la última validación técnica.
* El estado debe recalcularse cuando cambien sus componentes.
* El precio total no necesita almacenarse porque puede calcularse a partir de los productos seleccionados.

---

## ConfiguracionPCItem

Representa un componente incluido en una configuración.

Atributos:

* ConfiguracionPCItemId
* ConfiguracionPCId
* ComponentePCId
* Cantidad

Relaciones:

ConfiguracionPC 1 ─── N ConfiguracionPCItem

ComponentePC 1 ─── N ConfiguracionPCItem

Reglas:

* La cantidad debe ser mayor que cero.
* Un componente debe aparecer como máximo una vez en una configuración.
* `ComponentePCId` debe corresponder a un componente técnico existente en PcBuilder.

---

## Reglas de composición

Una configuración funcional deberá contener como mínimo:

* un procesador;
* una placa madre;
* memoria RAM;
* al menos una unidad de almacenamiento;
* una fuente de poder;
* un gabinete;
* una solución gráfica;
* refrigeración del procesador cuando sea necesaria.

La solución gráfica podrá provenir de:

* gráficos integrados en el procesador; o
* una tarjeta gráfica dedicada.

La refrigeración podrá provenir de:

* la solución incluida con el procesador; o
* un componente de refrigeración independiente.

Alcance inicial:

* exactamente un procesador;
* exactamente una placa madre;
* exactamente una fuente de poder;
* exactamente un gabinete;
* como máximo una tarjeta gráfica dedicada;
* uno o más productos de memoria RAM;
* una o más unidades de almacenamiento;
* como máximo una solución de refrigeración adicional.

---

## Reglas de compatibilidad

### Procesador y placa madre

Deben cumplirse ambas condiciones:

* SocketProcesador = SocketPlacaMadre.
* La combinación debe existir en `CompatibilidadProcesadorPlaca`.

---

### Memoria RAM y placa madre

Debe cumplirse:

TipoMemoriaRAM = TipoMemoriaPlacaMadre

Además:

* módulos instalados ≤ slots disponibles;
* memoria instalada ≤ capacidad máxima soportada.

---

### Placa madre y gabinete

El formato de la placa madre debe existir entre los formatos admitidos por el gabinete.

---

### Tarjeta gráfica y gabinete

LongitudGPU ≤ LongitudMaximaGpuGabinete

---

### Fuente y gabinete

El formato de la fuente debe existir entre los formatos admitidos por el gabinete.

---

### Refrigeración y procesador

El socket del procesador debe existir entre los sockets soportados por la refrigeración seleccionada.

---

### Refrigeración por aire y gabinete

AlturaRefrigeracion ≤ AlturaMaximaRefrigeracionGabinete

---

### Refrigeración líquida y gabinete

El tamaño del radiador debe encontrarse entre los tamaños soportados por el gabinete.

---

### Almacenamiento y placa madre

CantidadM2Seleccionada ≤ CantidadSlotsM2

CantidadSataSeleccionada ≤ CantidadPuertosSata

---

### Potencia

La fuente debe cumplir la potencia mínima requerida por la configuración.

El cálculo deberá considerar como mínimo:

* consumo estimado del procesador;
* consumo estimado de la placa madre;
* consumo de la tarjeta gráfica;
* consumo de memoria;
* consumo del almacenamiento;
* consumo de refrigeración cuando corresponda;
* consumo adicional estimado del sistema;
* margen de seguridad.

La fórmula exacta pertenece a la lógica de dominio.

---

## Compatibilidad y disponibilidad comercial

La compatibilidad técnica no depende del stock.

Una configuración puede encontrarse técnicamente en estado `Compatible` aunque uno o más productos estén agotados.

Para permitir la compra deben cumplirse simultáneamente:

1. La configuración debe estar técnicamente `Compatible`.
2. Todos los productos deben encontrarse activos.
3. Debe existir stock suficiente para las cantidades seleccionadas.
4. Los precios deben ser validados nuevamente al momento de compra.

---

## Compra desde el ensamblador

Cuando una configuración compatible sea añadida al carrito:

* cada `ComponentePC` se traduce al `ProductoId` comercial asociado antes de incorporarse al carrito;
* se respeta la cantidad definida en `ConfiguracionPCItem`;
* se vuelve a validar stock;
* se vuelve a validar que los productos estén activos;
* se vuelve a ejecutar la validación técnica antes de confirmar la compra.

En la versión inicial no es obligatorio persistir dentro del pedido que las líneas procedieron de una configuración específica.

---

## Resultado de validación

El resultado detallado de una validación no constituye inicialmente una entidad persistente.

El backend podrá generar resultados como:

* componente obligatorio faltante;
* procesador no soportado por placa madre;
* socket incompatible;
* memoria incompatible;
* exceso de módulos RAM;
* capacidad de RAM excedida;
* placa incompatible con gabinete;
* tarjeta gráfica demasiado larga;
* refrigeración incompatible;
* fuente insuficiente;
* almacenamiento sin conectividad disponible.

Los resultados serán enviados al frontend para explicar los problemas encontrados.

Un historial persistente de validaciones solo se incorporará si aparece una necesidad real de auditoría.

---

# 8. Módulo Game — videojuego

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

---

## PersonajeJuego

Representa un personaje perteneciente al videojuego.

Atributos:

* PersonajeId
* JuegoId
* Nombre
* Descripcion

Relación:

Juego 1 ─── N PersonajeJuego

---

## MapaJuego

Representa un escenario o mapa mostrado como contenido del videojuego.

Atributos:

* MapaId
* JuegoId
* Nombre
* Descripcion

Relación:

Juego 1 ─── N MapaJuego

---

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

Reglas:

* El preregistro puede realizarse sin tener una cuenta.
* `UsuarioId` es opcional.
* El email es obligatorio.
* La combinación Juego + Email debe ser única conceptualmente.

---

# 9. Módulo SoftwareServices — servicios de desarrollo

## ServicioDesarrollo

Representa un servicio profesional ofrecido por Neo Genesis Technology.

Atributos:

* ServicioId
* Nombre
* Descripcion
* PrecioBase
* Activo

---

## SolicitudServicio

Representa una solicitud realizada por un usuario respecto a un servicio de desarrollo.

Atributos:

* SolicitudId
* UsuarioId
* ServicioId
* DescripcionRequerimiento
* Estado
* FechaSolicitud

Estados conceptuales:

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

Reglas:

* Una solicitud siempre pertenece a un servicio existente.
* La solicitud conserva el servicio solicitado aunque posteriormente este sea desactivado.

---

# 10. Módulo Corporate — contenido corporativo

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

Inicialmente se espera una única empresa activa, aunque el modelo no depende técnicamente de esa restricción.

---

# 11. Reglas transversales del modelo

## Unicidad conceptual

Deben ser únicos conceptualmente:

* Usuario.Email
* Juego + PreregistroJuego.Email
* Carrito + Producto dentro de DetalleCarrito
* ConfiguracionPC + ComponentePC dentro de ConfiguracionPCItem
* Procesador + PlacaMadre dentro de CompatibilidadProcesadorPlaca
* Gabinete + FormatoPlaca
* Gabinete + FormatoFuente
* Gabinete + TamanoRadiador
* Refrigeracion + Socket

---

## Cantidades

Las cantidades de:

* DetalleCarrito;
* DetallePedido;
* ConfiguracionPCItem;

deben ser mayores que cero.

---

## Valores monetarios

Los valores monetarios no pueden representar importes negativos salvo que una futura operación específica lo requiera.

---

## Stock

El stock no puede ser negativo.

La disponibilidad se valida nuevamente en el momento de confirmar una compra.

---

## Historial

Los datos históricos importantes no se recalculan a partir del estado actual.

Ejemplos:

* PrecioUnitario de DetallePedido.
* DireccionPedido.
* HistorialPedido.
* Pagos realizados.

---

## Administración

`Administrador` representa una función de autorización.

No existe una entidad `Administracion`.

El panel administrativo utilizará capacidades de los diferentes módulos de acuerdo con las políticas de autorización.

---

## Propiedad y colaboración entre módulos

* Identity es propietario conceptual de usuarios, roles y autorización.
* Commerce es propietario de catálogo comercial, direcciones de compra, carrito, pedidos, pagos y disponibilidad comercial.
* PcBuilder es propietario de especificaciones técnicas, compatibilidad y configuraciones de PC.
* Game es propietario del contenido y preregistro del videojuego.
* SoftwareServices es propietario del catálogo de servicios y sus solicitudes.
* Corporate es propietario del contenido institucional.
* Las referencias a `Usuario` desde otros módulos representan la identidad del actor, pero no transfieren la propiedad de la autenticación.
* La relación `Producto` — `ComponentePC` conecta la identidad comercial con la identidad técnica sin convertir a PcBuilder en propietario del precio, stock o estado comercial.
* Una compra originada desde PcBuilder debe utilizar el producto comercial asociado y delegar en Commerce las reglas de stock, precio, carrito, pedido y pago.

---

# 12. Resumen de relaciones

Usuario 1 ─── N DireccionUsuario

Usuario N ─── N Rol
mediante UsuarioRol

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

# 13. Elementos deliberadamente fuera de la versión 1.1

No forman parte del modelo lógico inicial:

* integración concreta con Mercado Pago, PayPal, Stripe u otras pasarelas;
* proveedores de transporte;
* tracking de paquetes;
* tarifas logísticas avanzadas;
* cupones;
* promociones;
* reseñas de productos;
* listas de deseos;
* facturación electrónica;
* devoluciones y RMA;
* inventario distribuido en múltiples almacenes;
* gestión detallada de imágenes y archivos multimedia;
* notificaciones;
* auditoría global;
* historial de precios;
* impuestos complejos por jurisdicción;
* conectores eléctricos avanzados del ensamblador;
* configuraciones multi-GPU;
* sistemas custom de refrigeración líquida;
* gestión interna de proyectos derivados de solicitudes de servicios;
* datos internos del gameplay del videojuego;
* mecánicas gacha del juego.

Estos elementos podrán incorporarse mediante nuevas iteraciones cuando exista un requerimiento funcional que lo justifique.

---

# 14. Estado del modelo

El modelo lógico v1.1 establece la base funcional inicial de NGT Platform para:

* usuarios y autorización;
* direcciones;
* catálogo;
* productos;
* cómics y mangas;
* carrito;
* pedidos;
* entrega;
* pagos;
* trazabilidad de pedidos;
* componentes de PC;
* ensamblador y compatibilidad;
* videojuego y preregistro;
* servicios de desarrollo;
* contenido corporativo.

El modelo entidad–relación v1.1 deberá representar estas entidades, claves y cardinalidades manteniendo los límites entre módulos sin introducir tipos de datos específicos de SQL Server.