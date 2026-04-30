# HelpDeskBot — Documentacion Tecnica

> Sistema de gestion de solicitudes internas de soporte sobre Telegram, automatizado con n8n Community Edition y persistido en Google Sheets.

---

## Tabla de contenido

1. [Descripcion general](#1-descripcion-general)
2. [Objetivo del sistema](#2-objetivo-del-sistema)
3. [Arquitectura del sistema](#3-arquitectura-del-sistema)
4. [Diagrama del flujo](#4-diagrama-del-flujo)
5. [Modelo de datos](#5-modelo-de-datos)
6. [Flujo conversacional](#6-flujo-conversacional)
7. [Validaciones implementadas](#7-validaciones-implementadas)
8. [Automatizaciones](#8-automatizaciones)
9. [Instalacion de n8n en Ubuntu](#9-instalacion-de-n8n-en-ubuntu)
10. [Importar el flujo en n8n](#10-importar-el-flujo-en-n8n)
11. [Credenciales requeridas](#11-credenciales-requeridas)

---

## 1. Descripcion general

HelpDeskBot es un bot conversacional disenado para gestionar solicitudes internas de soporte, tales como incidentes tecnicos, solicitudes administrativas y consultas generales, mediante flujos conversacionales controlados y automatizacion basica.

El bot no toma decisiones autonomas ni aprende de los datos. Su funcion es organizar, registrar y presentar informacion, guiando al usuario mediante opciones numericas claras y mensajes humanizados.

HelpDeskBot opera exclusivamente en Telegram, utilizando n8n Community Edition como motor de logica y automatizacion, y Google Sheets como sistema de persistencia de datos.

---

## 2. Objetivo del sistema

El sistema fue desarrollado con los siguientes propositos:

- Centralizar el registro de solicitudes de soporte en una organizacion, eliminando la comunicacion informal y no trazable.
- Proveer a los usuarios un canal estructurado y accesible desde Telegram para reportar incidentes o realizar consultas.
- Garantizar que cada solicitud quede registrada con tipo, prioridad, descripcion, estado y fecha, para su posterior seguimiento.
- Ofrecer al equipo responsable visibilidad sobre el estado de las solicitudes mediante reportes integrados.
- Registrar en bitacora cada interaccion del usuario para auditoria y trazabilidad.

---

## 3. Arquitectura del sistema

El sistema esta compuesto por tres capas:

**Capa de interfaz — Telegram**
El usuario interactua exclusivamente a traves del bot de Telegram. La comunicacion es unidireccional por mensajes de texto numerados. No se utilizan botones inline ni comandos slash; toda la navegacion se realiza mediante respuestas numericas.

**Capa de logica — n8n Community Edition**
n8n actua como motor de automatizacion. Cada mensaje del usuario dispara el webhook del trigger de Telegram, lo cual inicia la evaluacion del estado de sesion del usuario y enruta la ejecucion al nodo correspondiente. El estado de sesion se almacena en la memoria estatica global de n8n (`$getWorkflowStaticData('global')`), lo que permite mantener el contexto entre mensajes sin base de datos adicional.

**Capa de persistencia — Google Sheets**
Google Sheets actua como base de datos ligera. El documento `HelpDeskBot_DB` contiene tres hojas: `SOLICITUDES`, `USUARIOS` y `LOGS`. Todas las operaciones de lectura y escritura se realizan mediante el nodo nativo de Google Sheets en n8n.

---

## 4. Diagrama del flujo

El siguiente diagrama representa la arquitectura completa del flujo en n8n, incluyendo todos los estados de sesion, nodos de enrutamiento, ramas de validacion y puntos de escritura hacia Google Sheets.

![Diagrama del flujo HelpDeskBot](./Captura_de_pantalla__126_.png)

### Estados de sesion manejados por el Router

| Estado | Descripcion |
|---|---|
| `MENU` | El usuario se encuentra en el menu principal |
| `TIPO` | El sistema aguarda la seleccion del tipo de solicitud |
| `PRIORIDAD` | El sistema aguarda la seleccion de prioridad |
| `DESCRIPCION` | El sistema aguarda el texto descriptivo de la solicitud |
| `CONFIRMACION` | El sistema aguarda confirmacion final (SI / NO) |
| `CONFIG` | El usuario se encuentra en el menu de configuracion de perfil |
| `CONFIG_NOMBRE` | El sistema aguarda el nuevo nombre del usuario |

### Nodos principales del flujo

| Nodo | Tipo | Funcion |
|---|---|---|
| Recibir mensaje | Telegram Trigger | Punto de entrada de cada mensaje del usuario |
| Buscar usuario | Google Sheets | Verifica si el usuario existe en la hoja USUARIOS |
| Verificar usuario activo | IF | Bloquea el acceso si el campo `activo` no es `true` |
| Leer sesion | Code | Lee el estado y datos de sesion desde la memoria estatica global |
| Router de estado | Switch | Enruta la ejecucion segun el estado actual de la sesion |
| Validar opcion de menu | IF | Verifica que el mensaje sea una opcion valida del menu principal |
| Menu principal | Switch | Deriva la ejecucion a la rama correspondiente segun la opcion elegida |
| Iniciar creacion de solicitud | Code | Establece el estado `TIPO` e inicializa los datos de sesion |
| Seleccion de tipo | Switch | Valida y enruta el tipo de solicitud seleccionado |
| Guardar tipo seleccionado | Code | Almacena el tipo en la sesion y avanza al estado `PRIORIDAD` |
| Guardar prioridad | Code | Almacena la prioridad y avanza al estado `DESCRIPCION` |
| Guardar descripcion | Code | Valida y almacena la descripcion, avanza a `CONFIRMACION` |
| Verificar confirmacion | IF | Evalua si el usuario confirmo con SI |
| Registrar ticket | Google Sheets | Escribe la solicitud completa en la hoja SOLICITUDES |
| Leer perfil del usuario | Google Sheets | Obtiene nombre y rol del usuario para mostrar en configuracion |
| Iniciar configuracion | Code | Establece el estado `CONFIG` y carga los datos del perfil |
| Opciones de perfil | Switch | Enruta entre salir de configuracion (0) o cambiar nombre (1) |
| Guardar nuevo nombre | Code | Valida el nombre y lo almacena en la sesion |
| Actualizar nombre en Sheets | Google Sheets | Escribe el nuevo nombre en la hoja USUARIOS |
| Guardar en bitacora | Google Sheets | Registra cada interaccion en la hoja LOGS |

---

## 5. Modelo de datos

El documento de Google Sheets utilizado se denomina `HelpDeskBot_DB` y contiene las siguientes hojas:

### SOLICITUDES

Almacena todas las solicitudes de soporte creadas por los usuarios.

| Campo | Descripcion |
|---|---|
| `id_ticket` | Identificador unico generado con prefijo `TK-` y timestamp |
| `tipo` | Tipo de solicitud: Soporte tecnico, Solicitud administrativa o Consulta general |
| `prioridad` | Nivel de prioridad: Alta, Media o Baja |
| `descripcion` | Texto libre ingresado por el usuario, minimo 5 caracteres |
| `estado` | Estado actual: Abierto, En proceso o Cerrado |
| `creado_por` | ID de Telegram del usuario que creo la solicitud |
| `fecha_creacion` | Fecha y hora en formato ISO 8601 |

### USUARIOS

Almacena el registro de usuarios habilitados para usar el bot.

| Campo | Descripcion |
|---|---|
| `telegram_user` | ID numerico de Telegram del usuario |
| `nombre` | Nombre visible del usuario, editable desde el bot |
| `rol` | Rol asignado: `admin` o `usuario` |
| `activo` | Valor booleano (`true` / `false`). Solo los usuarios activos pueden operar |

### LOGS

Registra cada interaccion procesada por el bot para auditoria.

| Campo | Descripcion |
|---|---|
| `timestamp` | Fecha y hora exacta de la interaccion en ISO 8601 |
| `telegram_user` | ID del usuario que genero la interaccion |
| `pantalla` | Estado de sesion en el que se encontraba el usuario |
| `opcion` | Mensaje enviado por el usuario |
| `resultado` | Resultado del procesamiento (`OK`) |

---

## 6. Flujo conversacional

### Mensaje de bienvenida y menu principal

Al iniciar o al volver al menu principal, el bot presenta el siguiente mensaje:

```
Hola, soy HelpDeskBot.

Estoy aqui para ayudarte con solicitudes de soporte
de forma rapida y ordenada.
Escribe el numero de la opcion que quieras usar.

Menu principal:
0. Ayuda
1. Crear solicitud
2. Consultar estado de solicitud
3. Mis solicitudes
4. Reportes
5. Configuracion
```

### Opcion 1 — Crear solicitud (wizard de 4 pasos)

El flujo de creacion de solicitud es guiado y secuencial. El usuario no puede avanzar si los datos no son validos.

**Paso 1 — Tipo**
```
Vamos a crear una solicitud. Selecciona el tipo:

1. Soporte tecnico
2. Solicitud administrativa
3. Consulta general

9. Cancelar
```

**Paso 2 — Prioridad**
```
Tipo seleccionado: [tipo]

Selecciona la prioridad:

1. Alta
2. Media
3. Baja

9. Cancelar
```

**Paso 3 — Descripcion**
```
Prioridad seleccionada: [prioridad]

Escribe una descripcion detallada de tu solicitud.
Minimo 5 caracteres.

Escribe 9 para cancelar.
```

**Paso 4 — Confirmacion**
```
Resumen de tu solicitud:

Tipo: [tipo]
Prioridad: [prioridad]
Descripcion: [descripcion]

Escribe SI para confirmar o NO para cancelar.
```

### Opcion 2 — Consultar estado de solicitud

El sistema recupera todas las solicitudes no cerradas del usuario y las presenta con ID, tipo, prioridad, estado y fecha.

### Opcion 3 — Mis solicitudes

El sistema recupera todas las solicitudes del usuario (incluidas las cerradas) y las presenta con indicadores de estado diferenciados.

### Opcion 4 — Reportes

El sistema presenta un resumen estadistico de las solicitudes del usuario: total, cantidad de abiertas, en proceso y cerradas.

### Opcion 5 — Configuracion

```
Configuracion de tu perfil

Nombre: [nombre actual]
Rol: [rol]

1. Cambiar nombre
0. Volver al menu principal
```

Si el usuario selecciona la opcion 1, el sistema solicita un nuevo nombre (minimo 3 caracteres) y lo actualiza en la hoja USUARIOS.

---

## 7. Validaciones implementadas

El sistema aplica las siguientes validaciones en cada etapa del flujo:

| Validacion | Descripcion |
|---|---|
| Usuario activo | Solo los usuarios cuyo campo `activo` sea `true` en la hoja USUARIOS pueden interactuar con el bot |
| Opcion de menu valida | El sistema verifica que el mensaje sea 0, 1, 2, 3, 4 o 5 antes de enrutar |
| Tipo de solicitud valido | Solo se aceptan los valores 1, 2, 3 o 9 para cancelar |
| Prioridad valida | Solo se aceptan los valores 1, 2, 3 o 9 para cancelar |
| Descripcion minima | La descripcion debe tener al menos 5 caracteres |
| Nombre minimo | El nuevo nombre debe tener al menos 3 caracteres |
| Confirmacion explicita | El ticket no se registra hasta que el usuario responda SI; cualquier otra respuesta cancela el proceso |

---

## 8. Automatizaciones

Las siguientes acciones ocurren de forma automatica sin intervencion del usuario:

- **Registro del ticket:** Al confirmar con SI, el sistema genera un `id_ticket` unico y escribe la solicitud completa en la hoja SOLICITUDES con estado inicial `Abierto`.
- **Limpieza de sesion:** Al completar o cancelar cualquier flujo, el sistema restablece el estado de sesion a `MENU` y elimina los datos temporales.
- **Registro en bitacora:** Cada interaccion procesada exitosamente queda registrada en la hoja LOGS con timestamp, usuario, pantalla activa y opcion seleccionada.
- **Actualizacion de nombre:** Al cambiar el nombre desde configuracion, el sistema actualiza directamente el registro del usuario en la hoja USUARIOS mediante operacion `appendOrUpdate`.

---

## 9. Instalacion de n8n en Ubuntu

Los siguientes pasos describen como instalar n8n Community Edition en un servidor Ubuntu 20.04 o superior.

### Requisitos previos

- Ubuntu 20.04 LTS o superior
- Acceso root o usuario con privilegios sudo
- Node.js 18 o superior
- npm 8 o superior

### Paso 1 — Actualizar el sistema

```bash
sudo apt update && sudo apt upgrade -y
```

### Paso 2 — Instalar Node.js 18

```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
```

Verificar la instalacion:

```bash
node -v
npm -v
```

### Paso 3 — Instalar n8n globalmente

```bash
sudo npm install -g n8n
```

### Paso 4 — Ejecutar n8n

Para una prueba rapida:

```bash
n8n start
```

n8n quedara disponible en `http://localhost:5678`.

### Paso 5 — Ejecutar n8n como servicio con PM2 (recomendado para produccion)

Instalar PM2:

```bash
sudo npm install -g pm2
```

Iniciar n8n con PM2:

```bash
pm2 start n8n --name "n8n"
pm2 save
pm2 startup
```

Ejecutar el comando que PM2 indique para que el servicio inicie automaticamente al reiniciar el servidor.

### Paso 6 — Configurar variables de entorno (opcional pero recomendado)

Crear un archivo de entorno:

```bash
nano ~/.n8n/.env
```

Variables recomendadas:

```env
N8N_HOST=0.0.0.0
N8N_PORT=5678
N8N_PROTOCOL=http
WEBHOOK_URL=http://TU_IP_O_DOMINIO:5678/
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=tu_contrasena_segura
```

Reiniciar el servicio tras aplicar los cambios:

```bash
pm2 restart n8n
```

---

## 10. Importar el flujo en n8n

1. Abrir n8n en el navegador (`http://localhost:5678`).
2. En el panel principal, hacer clic en **"New Workflow"** o abrir un workflow existente.
3. Hacer clic en el menu de tres puntos (arriba a la derecha) y seleccionar **"Import from file"**.
4. Seleccionar el archivo `HelpDeskBot_v3_corregido.json`.
5. El flujo completo se importara con todos sus nodos y conexiones.
6. Configurar las credenciales (ver seccion siguiente) antes de activar el workflow.
7. Hacer clic en el toggle **"Active"** para activar el webhook de Telegram.

---

## 11. Credenciales requeridas

Para que el flujo funcione correctamente, se deben configurar las siguientes credenciales en n8n desde **Settings > Credentials**:

### Telegram Bot API

1. Crear un bot en Telegram a traves de `@BotFather` con el comando `/newbot`.
2. Copiar el token generado.
3. En n8n, crear una credencial de tipo **"Telegram API"** e ingresar el token.
4. Asignar esta credencial a todos los nodos de tipo Telegram del flujo.

### Google Sheets OAuth2

1. Crear un proyecto en Google Cloud Console.
2. Habilitar la API de Google Sheets y la API de Google Drive.
3. Crear credenciales de tipo **OAuth 2.0** para aplicacion de escritorio.
4. En n8n, crear una credencial de tipo **"Google Sheets OAuth2 API"** e iniciar el flujo de autorizacion.
5. Asignar esta credencial a todos los nodos de Google Sheets del flujo.
6. Compartir el documento `HelpDeskBot_DB` con la cuenta de Google autorizada, con permisos de edicion.

---

## Notas tecnicas

- El estado de sesion de cada usuario se almacena en la memoria estatica global de n8n (`$getWorkflowStaticData('global')`). Esta memoria es volatil y se pierde si el proceso de n8n se reinicia. Para entornos de produccion con alta disponibilidad, se recomienda migrar la gestion de sesion a una base de datos externa como Redis o PostgreSQL.
- El campo `chat_id` para enviar mensajes de Telegram se obtiene siempre desde el nodo `Leer sesion`, que es el unico nodo con acceso garantizado a ese valor en cualquier punto de la ejecucion del flujo.
- Todos los nodos que leen datos del usuario en estados intermedios (como `Opciones de perfil` en estado `CONFIG`) deben referenciar el mensaje original desde `$('Leer sesion').first().json.mensaje` en lugar de `$json.mensaje`, dado que los nodos de Telegram no propagan el contexto del input anterior.
