# Helpdesk ERP

Mesa de ayuda y éxito del cliente para el ERP/POS de la empresa ficticia Sistemas Integrados S.A. Proyecto de aula de Bases de Datos No Relacionales, Universidad Francisco de Paula Santander Ocaña. Docente: Edwin Barrientos Avendaño.

Los clientes reportan fallas de cinco paquetes del ERP: Ventas Online, Inventarios, Compras, Producción y POS. Cada ticket guarda datos técnicos distintos según el paquete, todos en la misma colección de MongoDB. Soporte resuelve el caso y deja la solución escrita en una base de conocimiento que se busca con texto libre. Si el pago de un paquete se rechaza, ese paquete queda suspendido y los demás siguen funcionando.

## Repositorios

| Repositorio | Contenido | Tecnología |
|---|---|---|
| [base-de-datos](https://github.com/Helpdesk-erp/base-de-datos) | Esquema con validación `$jsonSchema`, índices, datos de prueba, pruebas de las validaciones y reportes de agregación | MongoDB 8, mongosh |
| [backend](https://github.com/Helpdesk-erp/backend) | API REST con login JWT, permisos por rol, transacciones y los reportes de los dashboards | Spring Boot 4.1, Java 21 |
| [frontend](https://github.com/Helpdesk-erp/frontend) | Login, registro de clientes, un dashboard por rol y el buscador de la base de conocimiento | React 19, Vite, Material UI |

```
Navegador (React, puerto 5173)  ──/api──▶  Backend (Spring Boot, puerto 8080)  ──▶  MongoDB (Atlas o local)
```

## Despliegue local

### Requisitos

| Herramienta | Versión | Para qué |
|---|---|---|
| Git | cualquiera reciente | Clonar los repositorios |
| [mongosh](https://www.mongodb.com/try/download/shell) | 2.x | Cargar la base de datos |
| MongoDB | 8, en [Atlas](https://www.mongodb.com/atlas) (el plan gratuito sirve) o local como replica set | La base. Las transacciones no funcionan en un `mongod` standalone |
| JDK | 21 | El backend. Maven no hace falta: el repositorio trae `mvnw` |
| Node.js | 22 o superior | El frontend |
| pnpm | 10 o superior | El frontend usa pnpm, no npm. Se instala con `npm install -g pnpm` |

### 1. Clonar los tres repositorios

En una misma carpeta:

```bash
git clone https://github.com/Helpdesk-erp/base-de-datos.git
```

```bash
git clone https://github.com/Helpdesk-erp/backend.git
```

```bash
git clone https://github.com/Helpdesk-erp/frontend.git
```

### 2. Cargar la base de datos

En Atlas, agregue su IP en **Network Access**. Luego, desde la carpeta `base-de-datos`, conéctese con su usuario de base de datos (mongosh pide la contraseña):

```bash
mongosh "mongodb+srv://CLUSTER.mongodb.net/" --apiVersion 1 --username USUARIO
```

Dentro de mongosh:

```js
load("01_esquema.js")
load("02_datos_semilla.js")
load("03_pruebas.js")
```

`03_pruebas.js` debe terminar con `Fallidas: 0`. Para usar un MongoDB local en lugar de Atlas, el [README de base-de-datos](https://github.com/Helpdesk-erp/base-de-datos#en-un-mongodb-local) explica cómo levantarlo como replica set.

### 3. Arrancar el backend

En la carpeta `backend`, copie la plantilla de variables (en Linux o macOS, con `cp`):

```bash
copy .env.example .env
```

Abra `.env` y complete, sin comillas:

- `JWT_SECRET`: una clave de al menos 32 caracteres. El comando para generarla está en el mismo archivo.
- `MONGODB_URI`: la cadena de conexión de Atlas, con usuario y contraseña, o `mongodb://127.0.0.1:27018/?replicaSet=rs0` si usa MongoDB local.

Luego, en Windows:

```bash
mvnw.cmd spring-boot:run
```

En Linux o macOS:

```bash
./mvnw spring-boot:run
```

Cuando la consola muestre `Tomcat started on port 8080`, la API está lista. La primera vez Maven descarga las dependencias y tarda unos minutos. Las instrucciones para IntelliJ IDEA y los errores comunes están en el [README del backend](https://github.com/Helpdesk-erp/backend#ejecutar).

### 4. Arrancar el frontend

En otra terminal, en la carpeta `frontend`:

```bash
pnpm install
```

```bash
pnpm run dev
```

Abra `http://localhost:5173`.

### 5. Entrar

Todos los usuarios de prueba tienen la misma contraseña, que está en el [README de base-de-datos](https://github.com/Helpdesk-erp/base-de-datos#usuarios-de-prueba).

| Usuario | Rol | Qué se ve al entrar |
|---|---|---|
| `andres.carrascal` | Administrador | Bandeja de tickets sin atender, carga de cada agente, incidentes por paquete, membresías y pagos rechazados |
| `laura.pacheco` | Auditor | Tiempo de resolución, recaudo, asistencia a capacitaciones y el historial de cambios de los tickets |
| `jhon.ascanio`, `karen.sanguino` | Agente de soporte | Sus tickets ordenados por prioridad, las capacitaciones que dicta y el buscador de soluciones |
| `ferresantaclara` | Cliente | Sus paquetes, tickets, capacitaciones, pagos y el buscador de soluciones |
| `camilo.rolon` | Cliente | Lo mismo, con el paquete Ventas Online suspendido por un pago rechazado |

Desde la pantalla de login también se puede registrar un cliente nuevo.

## Dónde se cumple cada requisito del enunciado

| Requisito | Dónde está |
|---|---|
| Modelo híbrido: embeber o referenciar | Paquetes contratados, historial del ticket y cronograma de capacitaciones van embebidos; pagos, asistencias y usuarios van referenciados. La justificación de cada caso está en el [README de base-de-datos](https://github.com/Helpdesk-erp/base-de-datos#qué-se-embebe-y-qué-se-referencia) |
| Validación estricta con `$jsonSchema` | Las 12 colecciones de `01_esquema.js`, con `validationLevel: "strict"`. `03_pruebas.js` intenta 29 inserciones inválidas y MongoDB las rechaza todas |
| Documentos polimórficos | `tickets.detalle` exige campos distintos según el paquete (`oneOf`), y `paquetes.campos_ticket` describe el formulario de cada uno |
| Índices compuestos | Bandejas de tickets por estado, por agente y por cliente, todas con la fecha para ordenar sin la etapa `SORT`; pagos, capacitaciones y asistencias |
| Índice de texto | `base_conocimiento`, en español. El frontend busca mientras se escribe |
| Framework de Agregación (`$match`, `$group`, `$lookup`) | Los reportes de los cuatro dashboards, en `ReportesService.java` del backend y en `04_reportes.js` para correrlos en mongosh |
| `explain("executionStats")` | `04_reportes.js` lo corre sobre las consultas de los dashboards y resume el plan de cada una |
| Transacciones multi-documento | El registro de un cliente crea el cliente y su usuario en una transacción. La suspensión de un paquete por pago rechazado está en la [guía de CRUD](https://github.com/Helpdesk-erp/base-de-datos/blob/main/GUIA_CRUD.md#pagos-y-transacciones) |
| Control de acceso por roles con JWT | El token lleva el `_id` de la sesión y el backend revisa en cada petición que siga activa. Los permisos de cada rol se leen de la colección `roles` |

## Estado

Hecho: base de datos completa, login, registro, cierre de sesión, un dashboard por rol y la búsqueda en la base de conocimiento.

Pendiente: crear tickets con el formulario dinámico, asignarlos y cambiar su estado, registrar pagos desde la aplicación con la transacción que suspende o reactiva el paquete, y la inscripción a capacitaciones.

## Equipo

- Yenderson Enrique Martinez Mendez
- Marlon Quintero Rincon
- Fabian Eduardo Rojas Pallarez - 0192270
- Danuil Alejandro Neira Navarro
- Jordy Prada Yanes
