# Arquitectura inicial de NGT Platform

## 1. Descripción

NGT Platform (Neo Genesis Technology) es una plataforma web full-stack que integra diferentes áreas de una empresa tecnológica dentro de un único ecosistema.

El proyecto tiene como objetivos principales servir como plataforma funcional, entorno de aprendizaje avanzado y proyecto profesional de portfolio.

## 2. Objetivos funcionales

La plataforma contempla inicialmente:

- Venta de componentes para PC.
- Ensamblador de PC con validación de compatibilidad y configuración funcional.
- Venta de cómics y mangas.
- Presentación de un videojuego en desarrollo.
- Preregistro de usuarios para el videojuego.
- Contratación de servicios de desarrollo de software.
- Presentación institucional de Neo Genesis Technology.
- Administración del contenido y operaciones de la plataforma.

## 3. Arquitectura inicial

La primera versión utilizará una arquitectura de monolito modular.

El backend se desplegará inicialmente como una única aplicación, pero su lógica se organizará mediante módulos funcionales con responsabilidades claramente separadas.

La evolución prevista es:

Monolito modular → arquitectura distribuida cuando exista una necesidad real → posible extracción de módulos a microservicios.

La migración hacia microservicios no se realizará únicamente por complejidad técnica o aprendizaje, sino cuando los límites, carga o necesidades de despliegue de un módulo justifiquen su separación.

## 4. Arquitectura general

La solución estará compuesta inicialmente por:

- Frontend web desarrollado con Angular.
- Backend desarrollado con ASP.NET Core.
- API HTTP para la comunicación entre frontend y backend.
- Entity Framework Core como ORM.
- SQL Server como base de datos relacional.
- Docker para infraestructura local.
- Git y GitHub para control de versiones.

Flujo general:

Usuario
    ↓
Angular
    ↓
ASP.NET Core API
    ↓
Lógica de negocio / módulos
    ↓
Entity Framework Core
    ↓
SQL Server

## 5. Módulos de negocio iniciales

La división principal del backend se realizará por capacidades de
negocio y no únicamente por capas técnicas.

### Identity

Responsable de las cuentas de usuario, autenticación, roles y
autorización.

### Commerce

Responsable del comercio electrónico de la plataforma, incluyendo
catálogo de productos, categorías, publicaciones, carrito, pedidos,
pagos y datos necesarios para el proceso de compra.

### PcBuilder

Responsable de los componentes técnicos de PC, reglas de compatibilidad,
validación de configuraciones y configuraciones creadas por los usuarios.

### Game

Responsable del contenido relacionado con el videojuego, personajes,
mapas y preregistros.

### SoftwareServices

Responsable del catálogo de servicios de desarrollo de software y las
solicitudes realizadas por los usuarios.

### Corporate

Responsable del contenido institucional de Neo Genesis Technology.

### Administración

La administración no constituye un módulo de negocio independiente.

Las operaciones administrativas serán proporcionadas por cada módulo y
protegidas mediante las políticas de autenticación y autorización
correspondientes.

## 6. Roles iniciales

### Usuario
Representa clientes, compradores y usuarios registrados de la plataforma.

### Administrador
Gestiona productos, pedidos, servicios, contenido, usuarios y preregistros según las políticas de autorización definidas.

Los permisos podrán evolucionar hacia un modelo más granular en futuras iteraciones.

## 7. Principios arquitectónicos

- Separación clara de responsabilidades.
- Bajo acoplamiento entre módulos.
- Alta cohesión dentro de cada módulo.
- Evitar dependencias innecesarias.
- No introducir microservicios antes de que exista una necesidad justificable.
- Mantener la lógica de negocio independiente de la interfaz de usuario.
- Mantener las decisiones de infraestructura separadas del dominio cuando sea posible.
- Priorizar mantenibilidad, testabilidad y evolución.
- Implementar de manera incremental.
- Documentar decisiones arquitectónicas relevantes.

## 8. Organización del monolito modular

El backend se ejecutará como una única aplicación ASP.NET Core.

Cada módulo de negocio se implementará inicialmente como un proyecto
.NET independiente dentro de la misma solución.

Los módulos no podrán acceder directamente al DbContext ni a las
implementaciones internas de otros módulos.

La comunicación entre módulos se realizará mediante contratos
explícitos.

La persistencia utilizará inicialmente una única base de datos SQL
Server denominada NgtPlatform.

Cada módulo dispondrá de su propio DbContext y será responsable de las
entidades que administra.

La base de datos podrá organizar sus tablas mediante schemas asociados
a los módulos.

Las relaciones físicas que crucen límites entre módulos se evaluarán
individualmente durante la revisión del modelo físico.

La estructura interna de cada módulo podrá organizarse mediante Domain,
Features, Infrastructure y Presentation cuando dichas divisiones sean
necesarias.

No se introducirán microservicios, brokers de mensajería ni
infraestructura distribuida mientras no exista una necesidad concreta
que justifique dicha complejidad.

## 9. Estructura general del repositorio

ngt-platform/
├── backend/
├── frontend/
├── docker/
├── docs/
├── .gitattributes
├── .gitignore
└── README.md

## 10. Estado

Arquitectura inicial definida.

La estructura interna definitiva del backend y frontend se establecerá cuando se creen sus respectivos proyectos, evitando definir carpetas prematuramente sin conocer las necesidades reales de implementación.
