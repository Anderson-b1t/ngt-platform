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

## 5. Módulos funcionales iniciales

### Usuarios
Responsable de las cuentas, autenticación, autorización y gestión básica de usuarios.

### Catálogo
Responsable de productos, categorías, precios, stock y disponibilidad.

### Ensamblador de PC
Responsable de las configuraciones de PC y sus reglas de compatibilidad y validez.

### Pedidos
Responsable de pedidos, detalles de pedido, estados e historial.

### Videojuego
Responsable del contenido público relacionado con el videojuego, personajes, mapas y preregistros.

### Servicios
Responsable del catálogo de servicios de desarrollo y las solicitudes realizadas por los usuarios.

### Corporativo
Responsable del contenido institucional de Neo Genesis Technology.

### Administración
Proporcionará las operaciones administrativas necesarias sobre los diferentes módulos.

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

## 8. Estructura general del repositorio

ngt-platform/
├── backend/
├── frontend/
├── docker/
├── docs/
├── .gitattributes
├── .gitignore
└── README.md

## 9. Estado

Arquitectura inicial definida.

La estructura interna definitiva del backend y frontend se establecerá cuando se creen sus respectivos proyectos, evitando definir carpetas prematuramente sin conocer las necesidades reales de implementación.