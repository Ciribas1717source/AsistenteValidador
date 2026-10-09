# Asistente de Consulta y Validador de Historias de Usuario

Aplicación web (React + Node.js, TypeScript) para consultar documentación corporativa en SharePoint y validar Historias de Usuario de Jira contra esa documentación, con evidencia trazable y cuatro estados de validación.

> Arquitectura, permisos, riesgos y plan: ver [`docs/ARQUITECTURA.md`](docs/ARQUITECTURA.md).

## Requisitos

- Node.js 20 o superior (`node -v`)
- npm 10 o superior

## Modo local (configuración actual): login propio + carpeta de OneDrive

La aplicación funciona **solo en tu computadora** (escucha en `127.0.0.1`), **sin Microsoft Entra ID ni Microsoft Graph**:

- **Login propio** con usuario y contraseña. La contraseña se guarda solo como hash scrypt con sal, en `data/local-users.json`, que está excluido de git. No hay usuarios predeterminados.
- **Sesiones en el backend** con cookie `HttpOnly` + `SameSite=Strict`, cierre por inactividad y protección CSRF. Tras 5 intentos fallidos, el usuario se bloquea 15 minutos.
- **Documentos** leídos en **solo lectura** de la carpeta configurada en `LOCAL_DOCS_ROOT` (sincronizada por OneDrive):
  - Solo esa carpeta y sus subcarpetas. Se rechazan `..`, rutas absolutas, `:` y enlaces simbólicos o junctions que salgan de la raíz.
  - Formatos que se leen: `.docx`, `.txt` y `.md`, hasta `LOCAL_MAX_FILE_MB` por archivo. Los demás formatos se listan pero no se leen.
  - La consulta y la validación usan un índice acotado (`LOCAL_MAX_INDEX_FILES`, `LOCAL_MAX_INDEX_MB`, `LOCAL_MAX_SCAN_DIRS`): no se carga toda la carpeta.
- **Jira y Claude en DEMO.** Con documentos reales:
  - La consulta documental devuelve fragmentos literales, sin respuesta redactada.
  - La validación da **NO VALIDABLE**, porque no hay IA real que evalúe. Los fragmentos reales se muestran solo como referencia.
  - Nunca se mezclan con los documentos DEMO.
- Para volver a ver los 4 estados con datos ficticios: `DOCS_SOURCE=demo`.

### Primera configuración en Windows (PowerShell)

```powershell
cd "RUTA\DEL\PROYECTO\asistente-hu"
npm install
Copy-Item .env.example .env          # el .env.example ya trae AUTH_MODE=local, DOCS_SOURCE=local y la ruta de OneDrive
notepad .env                         # revisá LOCAL_DOCS_ROOT (guardar en UTF-8)
npm run user:create                  # te pide usuario y contraseña (oculta, mín. 12 caracteres con letras y números)
npm run dev
```

Abrí http://localhost:5173 e ingresá con tu usuario. Para cambiar la contraseña, volvé a ejecutar `npm run user:create` con el mismo usuario.

### Cómo comprobar que se leen los documentos reales

1. Al iniciar, la consola del backend muestra `[documentos] OK — Carpeta documental local accesible` y la ruta. Si dice `ERROR`, el mensaje explica qué falta (ruta inexistente, OneDrive sin sincronizar o sin permisos).
2. Abrí la sección **Documentos**: las carpetas y archivos tienen que coincidir con los del Explorador de archivos.
3. Elegí un `.docx`, `.txt` o `.md` y tocá **Ver texto**: tiene que mostrar el contenido real.
4. Creá un archivo de prueba `.txt` en la carpeta, tocá **Actualizar** y verificá que aparece.
5. En **Consulta documental**, preguntá algo que esté en tus documentos: las fuentes listadas son archivos reales de la carpeta.

Si un archivo es "solo en línea" en OneDrive y no hay conexión, la lectura falla con un mensaje claro. Para tenerlo disponible siempre, en el Explorador usá *clic derecho → Mantener siempre en este dispositivo*.

## Datos DEMO (solo pruebas automáticas)

Jira se consulta siempre vía MCP, así que la app ya no muestra épicas ficticias. Los datos DEMO (`server/src/demo/demoData.ts`) se usan en `npm test` para verificar los cuatro estados de validación:

| Historia | Resultado esperado | Por qué |
|---|---|---|
| DEMO-245 | CUMPLE | Todo respaldado por citas de la documentación |
| DEMO-246 | REQUIERE REVISIÓN | Dos documentos se contradicen sobre quién aprueba |
| DEMO-247 | NO CUMPLE | Pide editar solicitudes Aprobadas; la regla lo prohíbe |
| DEMO-248 | NO VALIDABLE | No hay documentación sobre la exportación contable |

## Pruebas

```bash
npm test          # motor de validación: 4 estados, guardrails, fuentes caídas, ADF, capa MCP (sin red)
npm run typecheck
npm run mcp:check                 # estado REAL de los servidores MCP (inicia cada server)
npm run mcp:check -- MAART-19527  # además lee esa épica/historia vía fedpat-jira (solo lectura)
```

## Producción local

```bash
npm run build     # compila el frontend en web/dist
npm start         # el backend sirve la API y el frontend en http://localhost:4000
```

## Pasar a integraciones reales

Cada integración se activa por separado en `.env`. La app **no mezcla** datos REAL y DEMO al validar.

### 1. Jira (vía MCP fedpat-jira, `JIRA_MODE=mcp`)

Usa el servidor MCP **fedpat-jira** (el mismo que usa Claude Code), definido en `.mcp.json` de la raíz. El backend lo inicia como proceso local (stdio) con la primera consulta.

- **Credenciales:** las gestiona fedpat-mcp en `~/.fedpat-mcp/config.json` (instalador `python setup_wizard.py` de fedpat-mcp). La app no lee ni copia ese archivo, y no le pasa al proceso las variables de este `.env`.
- **Solo lectura:** la app solo invoca `jira_search`, `jira_search_by_epic`, `jira_get_issue` y `jira_get_my_tasks` (para verificar la conexión). Las 12 tools de escritura del server y cualquier tool nueva quedan bloqueadas (`server/src/services/mcp/policy.ts`).
- **Requisitos:** `uv` en el PATH y la carpeta de fedpat-mcp indicada en `.mcp.json`. La restricción por IP de Atlassian aplica igual: hay que estar en una red autorizada.
- **Limitaciones de las tools:** los campos personalizados se muestran por id (la tool no devuelve sus nombres). El campo de criterios solo se usa si se configura `JIRA_ACCEPTANCE_CRITERIA_FIELD`; si no, se toman de las secciones «Criterios de aceptación…» de la descripción. La relación épica → historia usa la consulta fija `"Epic Link" = X OR parent = X`, con un máximo de 100 hijos.

Es el modo por defecto (`JIRA_MODE=mcp`). Verificá con `npm run mcp:check` y levantá la app con `npm run dev`. En **Conexiones → Servidores MCP** se ve el estado real de los tres servers.

El cliente Jira REST anterior (credenciales propias en `.env`) quedó **archivado** en `server/src/services/jira/realJiraProvider.ts`; solo se activa con `JIRA_MODE=real` y `JIRA_BASE_URL`/`JIRA_EMAIL`/`JIRA_API_TOKEN`.

### 1 ter. Doc-técnica de FlockTools (vía MCP fedpat-flocktools)

La **doc-técnica de Oracle Forms** de FlockTools se suma a la carpeta local como documentación autorizada, tanto en la **consulta documental** como en la **validación**.

- **Qué se indexa** (solo lectura, vía `flock_tech_espacios` → `flock_tech_forms` → `flock_tech_form_overview` → `flock_tech_get_detalle`): una ficha por form (descripción, seguridad, funcionalidades, tablas y paquetes), un documento por **evento** (descripción, permisos y validaciones, observaciones, parámetros, código sugerido y actual) y uno por paquete. Las columnas de tablas no se indexan.
- **Enlace con las historias:** cada evento tiene el campo **«incidencias»** con las URLs de Jira (`…/browse/MAART-123`). Al validar una historia, los eventos que la enlazan a ella o a su épica entran como **evidencia prioritaria**, marcados «Enlazado a …», aunque no coincidan por texto. El resultado muestra la sección «Doc-técnica de FlockTools enlazada». En la consulta, si la pregunta menciona una clave (p. ej. `MAART-21114`), primero van los eventos enlazados.
- **Índice en memoria:** se arma en segundo plano al iniciar el backend (~1 minuto, unos 600 documentos) y se reutiliza `FLOCKTOOLS_CACHE_MINUTES`. Si FlockTools no responde, la carpeta local se sigue usando y se avisa como observación.
- **Requisitos:** `FLOCKTOOLS_BASE_URL` y `FLOCKTOOLS_TOKEN` en la configuración de fedpat-mcp. Se desactiva con `FLOCKTOOLS_TECHDOC=false`.

**fedpat-doctec** se verifica pero no se usa como fuente: solo analiza CSV de documentación técnica (formato Fedpat), y la carpeta autorizada tiene esa documentación en `.xlsx`.

### 2. SharePoint en línea (`DOCS_SOURCE=sharepoint` + `AUTH_MODE=microsoft`) — opcional, hoy desactivado

Ubicación autorizada: `accionait.sharepoint.com/sites/FlockITequipo` › *Documentos compartidos* › `03. Federación Patronal/03. FedPat - Discovery ART`.
La app explora **solo** esa carpeta y sus subcarpetas, un nivel por vez. Los identificadores (sitio, biblioteca, carpeta) se obtienen de Microsoft Graph al conectarse; no se escriben a mano.

**Autenticación recomendada: delegada (`SP_AUTH_MODE=delegated`).** Iniciás sesión con tu cuenta corporativa en la página oficial de Microsoft (MFA y acceso condicional aplican) y la app ve únicamente lo que vos podés ver. No usa secretos. El token queda en memoria del backend; si reiniciás el servidor, volvés a iniciar sesión.

#### Pedido a IT (App Registration en Microsoft Entra ID)

1. Registrar una aplicación de **un solo inquilino**, por ejemplo «Asistente Validador HU (local)».
2. En **Autenticación** → *Agregar una plataforma* → **Aplicaciones móviles y de escritorio** → URI de redirección: `http://localhost:4000/api/sharepoint/auth/callback`. No usar las plataformas *Web* ni *SPA*.
3. En **Autenticación** → *Configuración avanzada* → **Permitir flujos de clientes públicos: Sí**.
4. En **Permisos de API** → Microsoft Graph → **Permisos delegados**: `User.Read` y `Sites.Read.All` (solo lectura). No hace falta ningún permiso de aplicación.
5. Si la organización no permite que los usuarios den consentimiento: **Conceder consentimiento de administrador** para esos dos permisos.
6. Si la app requiere asignación de usuarios: asignar tu usuario en *Aplicaciones empresariales*.
7. Informarte el **Id. de directorio (inquilino)** y el **Id. de aplicación (cliente)**. No son secretos. No se necesita client secret.

#### Configurar y probar en Windows (PowerShell)

```powershell
cd C:\ruta\a\asistente-hu
npm install
Copy-Item .env.example .env      # solo si todavía no tenés .env
notepad .env
```

En el `.env` (guardado en UTF-8, que es lo predeterminado del Bloc de notas):

```
AUTH_MODE=microsoft
DOCS_SOURCE=sharepoint
SP_AUTH_MODE=delegated
MS_TENANT_ID=<Id. de directorio que te pase IT>
MS_CLIENT_ID=<Id. de aplicación que te pase IT>
```

Dejá `AI_MODE=demo`. Las líneas `SP_HOSTNAME`, `SP_SITE_PATH`, `SP_DRIVE_NAME` y `SP_FOLDER_PATH` ya vienen completas.

```powershell
npm run dev
```

1. En la consola tiene que aparecer: `Modos → Jira: demo | SharePoint: real (auth delegated) | IA: demo`.
2. Abrí http://localhost:5173 → menú **SharePoint**. Debe decir *Falta iniciar sesión* y mostrar la URI de redirección.
3. Hacé clic en **Iniciar sesión con Microsoft**, elegí tu cuenta corporativa y aceptá. Volvés a la app con *Sesión iniciada como …*.
4. En **Ubicación autorizada** aparecen el sitio, la biblioteca y la carpeta con sus identificadores **obtenidos de Microsoft Graph**.
5. En **Documentos de la carpeta autorizada** ves las subcarpetas y archivos con nombre, ruta, tamaño, fecha y enlace **Abrir** (abre el documento en SharePoint).

**Cómo comprobar que la conexión es real:**

- Abrí un enlace **Abrir**: tiene que llevarte al documento en `accionait.sharepoint.com`.
- Compará el listado con la carpeta en el navegador: mismos nombres y fechas.
- Renombrá o subí un archivo de prueba en SharePoint (si tenés permiso) y tocá **Actualizar**: el cambio aparece.
- Consultá el estado técnico: `Invoke-RestMethod http://localhost:4000/api/sharepoint/status | ConvertTo-Json -Depth 6`.
- En la consola del backend vas a ver `GET /browse → 200` por cada carpeta que abras.

**Errores frecuentes:** la pantalla muestra el código `AADSTS…` y qué pedirle a IT (consentimiento faltante, URI de redirección mal registrada, flujos públicos deshabilitados, acceso condicional). Si la carpeta o la biblioteca no se encuentran, se listan las bibliotecas visibles para corregir `SP_DRIVE_NAME`.

**Alternativa sin sesión de usuario (`SP_AUTH_MODE=app`):** permiso de aplicación `Sites.Selected` concedido solo sobre el sitio FlockITequipo (`POST /sites/{site-id}/permissions` con rol `read`) y un client secret en `MS_CLIENT_SECRET`. Es útil para un servidor compartido, pero no respeta los permisos de cada usuario.

En esta etapa la validación y la consulta documental siguen en DEMO: no usan los documentos reales hasta que se active la lectura de contenido.

### 3. IA (`AI_MODE=real`)

1. Confirmá con Seguridad/Legales que se pueden enviar fragmentos de documentación y de historias a la API de Claude.
2. Obtené una API key en la consola de Anthropic de la organización.
3. Completá `ANTHROPIC_API_KEY` y `ANTHROPIC_MODEL` (lista en https://docs.claude.com/en/docs/about-claude/models).

## Seguridad

- Sin secretos en el frontend; `.env` está en `.gitignore`.
- Jira: lista blanca de rutas de solo lectura en `server/src/services/jira/jiraClient.ts`.
- SharePoint: solo se recorre la carpeta configurada; nunca búsqueda en todo el tenant.
- Logs: método, ruta, estado y duración. Nunca cuerpos, tokens ni contenido.
- El contenido de Jira/SharePoint se trata como datos, no como instrucciones.
