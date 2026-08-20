# ADR 0001: Adoptar una arquitectura de monolito modular

**Estado:** Aceptada

**Fecha:** 2026-08-20

## Contexto

NGT Platform integra varias capacidades de negocio dentro de una misma
plataforma: comercio electrónico, configuración de PC, presentación y
preregistro de un videojuego, servicios de desarrollo de software,
identidad de usuarios y contenido corporativo.

Aunque estas capacidades presentan límites funcionales diferenciados,
la primera versión del sistema no requiere la complejidad operativa de
una arquitectura de microservicios.

El proyecto debe permitir mantener responsabilidades claramente
separadas y facilitar una posible extracción futura de módulos cuando
existan razones reales de escalabilidad, despliegue independiente,
aislamiento o evolución del negocio.

## Decisión

NGT Platform se implementará inicialmente como un monolito modular.

El backend se ejecutará y desplegará como una única aplicación
ASP.NET Core.

Los módulos iniciales serán:

- Identity
- Commerce
- PcBuilder
- Game
- SoftwareServices
- Corporate

Cada módulo representará una capacidad de negocio y se implementará
inicialmente como un proyecto .NET independiente dentro de la misma
solución.

Cada módulo será responsable de su lógica y persistencia.

Se utilizará inicialmente una única base de datos SQL Server denominada
NgtPlatform, organizando la persistencia mediante schemas asociados a
los módulos.

Cada módulo dispondrá de su propio DbContext de Entity Framework Core.

Un módulo no podrá acceder directamente al DbContext ni a las
implementaciones internas de otro módulo. La comunicación entre módulos
se realizará mediante contratos explícitos.

Las relaciones físicas entre datos pertenecientes a módulos diferentes
se evaluarán individualmente durante la revisión del modelo físico.

La capacidad administrativa será transversal y estará controlada
mediante autenticación y autorización; no constituirá un módulo de
negocio independiente.

## Organización interna

La organización principal del sistema será por módulos de negocio.

Dentro de cada módulo podrán utilizarse separaciones como Domain,
Features, Infrastructure y Presentation cuando sean necesarias.

Las funcionalidades se organizarán preferentemente de forma cercana a
su caso de uso, evitando una separación global exclusivamente por capas
técnicas.

## Consecuencias

La aplicación tendrá un único proceso y un único despliegue inicial.

Los módulos tendrán límites explícitos y propiedad clara sobre su
lógica y datos.

Se introduce mayor disciplina arquitectónica que en un monolito
tradicional, pero se evita la complejidad operacional de los
microservicios.

La extracción futura de un módulo requerirá cambios de infraestructura
y persistencia, pero sus límites estarán definidos desde el diseño
inicial.

## Evolución

Un módulo podrá convertirse en un servicio independiente únicamente
cuando exista una necesidad justificada.

La adopción futura de microservicios será incremental y no requerirá
convertir todos los módulos simultáneamente.