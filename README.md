# CRM ORM/ODM Lab

API REST de un CRM básico que combina un ORM (Sequelize + PostgreSQL) y un ODM (Mongoose + MongoDB).

## Stack

- Node.js 22, Express 5, CommonJS
- Sequelize + PostgreSQL 16 (`User`, `Company`, `Contact`)
- Mongoose + MongoDB 7 (`Activity`)
- Jest + Supertest
- GitHub Codespaces, Dev Containers, Docker Compose
- Supervisor (`npm run dev`)

## Arquitectura

```text
GitHub Codespace
│
├── app       Node.js 22  ──┬── Sequelize ──> postgres (PostgreSQL)
│                           └── Mongoose  ──> mongo    (MongoDB)
├── postgres
└── mongo
```

La aplicación se conecta por nombre de servicio (`postgres`, `mongo`). Las credenciales de desarrollo llegan como variables de entorno definidas en `.devcontainer/docker-compose.yml` (ver `.env.example`).

## Iniciar el Codespace

1. En GitHub: **Code → Codespaces → Create codespace on main**.
2. Espera a que se levanten los tres servicios (`app`, `postgres`, `mongo`). `postCreateCommand` ejecuta `npm install`.

## Instalar dependencias

```bash
npm install
```

## Seed y reset

```bash
npm run seed    # inserta datos deterministas (3 users, 4 companies, 8 contacts, 10 activities)
npm run reset   # elimina y recrea tablas/base de datos y vuelve a sembrar
```

## Iniciar la API

```bash
npm start       # node ./bin/www
npm run dev     # supervisor ./bin/www
```

Servidor en el puerto `3000` (variable `PORT`).

## Pruebas

```bash
npm test
```

Cada suite restablece PostgreSQL y MongoDB antes de ejecutarse y cierra las conexiones al terminar.

## Endpoints

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/health` | Health check |
| GET | `/users` | Listar usuarios |
| GET | `/users/:id` | Obtener usuario |
| POST | `/users` | Crear usuario |
| PUT | `/users/:id` | Actualizar usuario |
| DELETE | `/users/:id` | Eliminar usuario |
| GET | `/companies` | Listar compañías (`?industry=`) |
| GET | `/companies/:id` | Obtener compañía |
| POST | `/companies` | Crear compañía |
| PUT | `/companies/:id` | Actualizar compañía |
| DELETE | `/companies/:id` | Eliminar compañía |
| GET | `/contacts` | Listar contactos |
| GET | `/contacts/:id` | Obtener contacto |
| POST | `/contacts` | Crear contacto |
| PUT | `/contacts/:id` | Actualizar contacto |
| DELETE | `/contacts/:id` | Eliminar contacto |
| GET | `/activities` | Listar actividades (`?type=`) |
| GET | `/activities/:id` | Obtener actividad |
| POST | `/activities` | Crear actividad |
| PUT | `/activities/:id` | Actualizar actividad |
| DELETE | `/activities/:id` | Eliminar actividad |

Los errores se devuelven como JSON: `{ "error": "Contact not found" }`.

## Respuestas

**1. Dos motores.**
Activity contiene el campo metadata, que no siempre contiene el mismo tipo de información, sino que varía, por lo que si estuviera en una base de datos relacional, tendría muchas columnas con valores nulos, lo que no es óptimo. 
En cambio, Company y Contact, siempre tienen el mismo tipo de información, por lo que siempre se ocupan las mismas columnas y nunca se dejan vacías. En estas tablas se tienen datos fijos y que se pueden relacionar por tablas. 

**2. ORM vs ODM.**
Los dos sirven para trabajar con la base de datos usando objetos de JavaScript en lugar de escribir las querries a mano. Un ORM se usa con bases de datos relacionales, y aquí es Sequelize con PostgreSQL. Un ODM se usa con bases de datos de no relacionales con documentos, y aquí es Mongoose con MongoDB.
La diferencia es que en el ORM todos los registros tienen las mismas columnas y se pueden relacionar entre tablas, y en el ODM cada documento puede tener datos distintos.

**3. Configuración por variables de entorno.**
Se definen en el archivo .devcontainer/docker-compose.yml, en la parte de environment, y el código solo las lee. Es mala práctica ponerlas en los .js porque el código se sube públicamente y cualquiera podría ver la contraseña de la base de datos.
La app se conecta con DB_HOST=postgres y MONGODB_URI=mongodb://mongo:27017/crm. No es localhost porque las bases de datos no están en el mismo contenedor que la app, cada una tiene su propio contenedor, y se les llama por su nombre.

**4. Asociaciones.**
Según models/sequelize/index.js, una compañía puede tener muchos contactos, pero cada contacto solo pertenece a una compañía, es una relación de uno a muchos.
La llave foránea es companyId y está en la tabla contacts, porque cada contacto guarda el id de su compañía. El alias as: 'contacts' es el nombre de la relación, es el que se usa para pedir los contactos en el include y el nombre con el que aparecen en la respuesta.

**5. Eager loading.**
Si se hace en dos consultas, primero se pide la compañía y luego se vuelve a la base de datos a pedir sus contactos. Con include se pide todo de una sola vez y la compañía ya llega con sus contactos.
Es mejor usar include, porque solo se hace una consulta en lugar de dos y se escribe menos código.

**6. Instancia vs consulta.**
Buscar el contacto primero, como en el update de controllers/contacts.js, tiene la ventaja de que después de modificarlo, ya se tiene el contacto actualizado para regresarlo en la respuesta y si no existe puedo responder 404 desde antes.
Model.update directo tiene la ventaja de que lo hace todo en una sola consulta, pero solo regresa cuántos registros cambió y no el contacto, así que para regresarlo tendría que buscarlo otra vez.

**7. Esquema flexible.**
metadata es de tipo Mixed, entonces acepta cualquier objeto con los campos que sea. Por eso una llamada puede guardar la duración, un email el asunto y una reunión el lugar, aunque cada uno tenga datos diferentes.
La desventaja es que Mongoose no revisa lo que se guarda ahí entonces se podrían guardar datos mal escritos o del tipo equivocado sin que marque error.

**8. Sin ref.**
No se puede usar ref porque solo sirve para conectar documentos que están dentro de MongoDB, y los contactos y usuarios están guardados en PostgreSQL.
La consecuencia es que nadie revisa que esos ids existan. Por ejemplo, si se borra un User en PostgreSQL, sus actividades en MongoDB se quedan con un userId de alguien que ya no existe, y no aparece ningún error.

**9. Documento actualizado.**
Antes de corregirlo el cambio sí se guardaba, pero la respuesta regresaba el documento como estaba antes del cambio, porque findByIdAndUpdate funciona así si no se le dice otra cosa.
Lo que cambié fue agregarle new: true, para que regrese el documento ya actualizado y runValidators: true para que revise que los datos nuevos cumplan con el esquema.

**10. Pruebas de comportamiento.**
La ventaja es que no importa cómo escribas el código, solo que la API responda bien. Así puedes cambiar o mejorar tu código y las pruebas siguen pasando, mientras el resultado sea el mismo.

**11. Repetibilidad.**
Antes de cada suite tests/setup.js se conecta a las dos bases de datos, borra todo y vuelve a meter los datos iniciales. Al terminar cierra las conexiones.
Es necesario porque las pruebas crean y cambian datos. Si no se reiniciaran la siguiente vez habría datos diferentes y las pruebas fallarían.

**12. Tu experiencia.**
El reto más difícil fue el 5 porque era el más largo. Tuve que cambiar dos partes del archivo y revisar models/sequelize/index.js para saber cuál era el alias de la relación. Aparte no recordaba cómo traer los contactos junto con la compañía, así que tuve que investigar. Jest me marcaba Expected: true, Received: false en la prueba que revisa que contacts sea un arreglo y por eso entendí que la respuesta no traía los contactos.

## Evidencia
<img width="561" height="447" alt="Evidencia" src="https://github.com/user-attachments/assets/9177a93f-6674-44ef-9fbf-a318a9e05825" />
