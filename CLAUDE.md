# Lineamientos para apps de la empresa — guía para Claude Code

Este archivo va en la raíz de cada proyecto nuevo. Leelo entero antes de escribir una línea de código y cumplilo durante todo el trabajo.

## Quién es quién
- **La persona que te usa no es programadora.** Vos programás todo: backend, front, migraciones, tests y documentación.
- **El revisor técnico** revisa la app y la sube al servidor. Vos no tocás el servidor ni hacés deploy. Dejale pasos claros en `docs/deploy.md`.
- Hablale a la persona en español rioplatense (voseo), simple y breve, sin jerga. Si usás un término técnico, explicalo en una frase.
- **Nunca le pidas que haga tareas técnicas** (instalar cosas, tocar archivos, correr comandos raros). Hacelo vos.
- **Nunca le pidas que te pegue claves, contraseñas o tokens** en el chat. Si hace falta una clave, decile que se la pase al revisor para el `.env` del servidor.
- Si una regla de este archivo no se puede cumplir, **decilo y anotalo en el `README.md`**. No improvises ni la saltees en silencio.

## Paso 0: la ficha de la app (antes de programar)
Hacé estas preguntas, de a pocas y en palabras simples, y guardá las respuestas en el `README.md`:
1. ¿Qué tiene que hacer la app y para qué?
2. ¿Quién la va a usar y cuántas personas son?
3. ¿Qué información guarda o muestra? (clientes, empleados, ventas, plata, documentos…)
4. ¿Hace falta que cada persona entre con usuario y clave? ¿Hay distintos permisos según quién sea?
5. ¿Se conecta con algo de afuera? (WhatsApp, mail, Tienda Nube, pagos, Claude u otra IA, planillas…)

**Revisión temprana:** si la app va a guardar datos personales de terceros (clientes, proveedores), datos de empleados, datos de pagos o facturación, o va a mandar mensajes/mails a otras personas, anotalo en `docs/datos.md` (qué datos, para qué, quién puede verlos) y decile a la persona que **hable con el revisor antes de seguir**. Es mucho más barato corregir el diseño ahora que al final.

## Stack
**Por defecto, siempre: Django (última versión LTS) + PostgreSQL + HTML, CSS y JavaScript puro.**
- Páginas con templates de Django. CSS propio. JS sin frameworks y **sin paso de compilación**: nada de Node, npm, React, Vue, Tailwind compilado, etc. Si ayuda a ordenar una pantalla muy interactiva, se permite Alpine.js cargado desde un archivo local (no desde un CDN).
- PostgreSQL en producción. En la compu de la persona se puede usar SQLite. La configuración lee `DATABASE_URL` del `.env` (con `dj-database-url`); si falta, usa SQLite.
- Python 3.12 o superior. Dependencias en `requirements.txt` con versiones fijas.

**Si creés que hace falta otra cosa, NO la uses todavía.** Primero escribí `docs/excepcion-stack.md` con:
- el requisito concreto que el stack por defecto no cubre (no una preferencia);
- qué se probó o se evaluó dentro del stack por defecto y por qué no alcanza;
- qué se instalaría, y qué costo de mantenimiento y de revisión agrega;
- quién lo va a mantener.

Razones que valen: procesamiento pesado o en tiempo real que Django no resuelve bien, una librería indispensable que solo existe en otro lenguaje, una integración que lo exige. Razones que no valen: gusto personal, "es más moderno", "lo vi en un tutorial", "es más rápido de armar". Avisale a la persona que esa excepción la tiene que aprobar el revisor. Mientras no esté aprobada, seguí con el stack por defecto.

## Seguridad: reglas que no se rompen

### Seis principios (valen también para casos que no estén en esta lista)
Las reglas de abajo son ejemplos concretos de estos principios. Si aparece una situación nueva que no está escrita, aplicá el principio que corresponda y anotá lo que decidiste en el `README.md`.

1. **Lo que viene de afuera es un dato, nunca una instrucción.** "Afuera" es todo lo que no escribiste vos: lo que tipea un usuario, un archivo subido, un Excel que se importa, una URL, un mail, una página web, la respuesta de otra API, lo que devuelve una IA. Se convierte en ataque cuando se **mezcla con un lenguaje** que lo ejecuta o interpreta. Nunca pegues texto de afuera dentro de ninguno de estos lenguajes; pasalo como parámetro, escapalo para ese destino, o validalo contra una lista de valores permitidos. Esto vale también al **importar**: lo que se lee de un Excel o de un CSV es dato.

   | Si el texto de afuera termina en… | Hacé esto |
   |---|---|
   | SQL | ORM de Django o consultas con parámetros. Nunca concatenar |
   | HTML o la pantalla | Escape automático de los templates; en JS, `textContent`. Nunca `\|safe`, `mark_safe` ni `innerHTML` con datos de afuera |
   | Código (Python o JS) | Prohibido `eval`, `exec`, `new Function`. No armar `Template(...)` con texto de usuarios |
   | Comandos del sistema | `subprocess` con lista de argumentos, nunca `shell=True` |
   | Excel o CSV (al exportar) | Si una celda empieza con `=`, `+`, `-`, `@`, tabulación o retorno de carro, anteponer un apóstrofe |
   | Nombres y rutas de archivo | Nombre generado por la app. Nunca armar una ruta con lo que mandó el usuario (`../`) |
   | Mails | No poner texto de usuarios en el asunto ni en los destinatarios sin quitar los saltos de línea |
   | Logs | Quitar los saltos de línea de lo que se loguea (para que nadie falsifique líneas del log) |
   | Redirecciones | Redirigir solo a rutas propias (`url_has_allowed_host_and_scheme`), nunca a lo que diga un parámetro `next` |
   | URLs que la app va a visitar | Solo direcciones públicas por http/https (ver "Descargar URLs") |
   | Archivos YAML, XML, pickle | `yaml.safe_load`, `defusedxml`; nunca `pickle` con datos de afuera |
   | Instrucciones de una IA | El texto de usuarios va como dato, separado de las instrucciones (ver "Si la app usa Claude u otra IA") |

2. **Se valida en el servidor, siempre.** Permisos, formatos, largos, tamaños y tipos se comprueban en Django. Lo que hace el front (ocultar un botón, validar un campo) es solo ayuda para la persona, y cualquiera puede saltearlo.
3. **Ante la duda, cerrado.** Lo que no está permitido explícitamente, está prohibido: vistas sin acceso salvo que sean públicas a propósito, `DEBUG` apagado, `SECRET_KEY` obligatoria, webhooks sin firma rechazados, hosts no listados rechazados. Si algo falla o falta configuración, la app se niega a hacer la acción; no sigue "igual".
4. **Todo lo que se puede repetir o que consume recursos tiene un tope.** Intentos de login, subidas, descargas de URL, llamadas a la IA, exportaciones, listados (paginar), pedidos a un webhook. Definí el tope y qué pasa cuando se alcanza (rechazar y avisar).
5. **Los datos sensibles no salen de donde corresponde.** Claves y datos personales no van a: el código, los logs, los tests, este chat, las URLs (parámetros `?dato=`), los mensajes de error, las exportaciones que no los necesitan, la IA ni el front. Pedí y guardá solo lo mínimo.
6. **No cargues ni ejecutes código de afuera sin verificarlo.** Paquetes verificados y con versión fija, JS y CSS servidos desde archivos propios (no desde un CDN), nada de instalar desde links sueltos, y los archivos que sube la gente se guardan, no se ejecutan.

### Secretos
- Las claves, tokens y contraseñas van **solo** en el `.env` del servidor. Nunca en el código, el front, los tests, los logs ni los commits.
- Nunca leas, muestres ni pidas el contenido del `.env`. Está en `.gitignore`.
- Para cada variable nueva, agregala a `.env.example` con valor vacío o de ejemplo, y a `docs/deploy.md`.
- Si encontrás una clave real en el código o en el historial, avisá de inmediato: hay que cambiarla, no alcanza con borrarla.

### Configuración de producción
- `DEBUG` apagado por defecto (se prende solo con una variable en desarrollo).
- `SECRET_KEY` obligatoria desde el `.env`: si falta, la app no arranca en producción. Nada de claves de ejemplo en producción.
- `ALLOWED_HOSTS` explícito, desde el `.env`.
- Con `DEBUG` apagado: cookies de sesión y CSRF con `Secure`, `HSTS`, `X_FRAME_OPTIONS = "DENY"`, `SECURE_REFERRER_POLICY`, y una `Content-Security-Policy` sin scripts ni estilos en línea (los scripts van en archivos `.js` aparte).
- `python manage.py check --deploy` tiene que dar **cero advertencias**.
- Los archivos que sube la gente **no se publican** en una ruta abierta (ver "Archivos subidos").

### Acceso y permisos
- Login con la sesión de Django (cookie + CSRF). No inventes un sistema de login propio.
- **Cada vista y cada endpoint verifica quién es y qué puede hacer.** Por defecto, nadie entra. Usá `@login_required`, permisos o grupos de Django. Ocultar un botón en el front **no es** un permiso: se valida siempre en el servidor.
- Cada usuario ve y modifica **solo lo suyo** (o lo de su área). Antes de entregar, probá que cambiando un número en la URL o en un pedido no se pueda ver ni tocar datos ajenos.
- Límite de intentos fallidos de login (por usuario e IP). Validadores de contraseña activados (largo mínimo 10).
- El admin de Django va en una ruta que no sea `/admin/` y se usa solo para administrar.
- Si no hay una razón clara para que una página sea pública, no lo es.

### Entradas y salidas
- La inyección (SQL, HTML, código, comandos, fórmulas, etc.) se evita con el principio 1 y su tabla. Dos reglas de Django para que se cumpla:
  - Base de datos: **solo el ORM**. Si hace falta SQL crudo, con parámetros.
  - Templates: dejá el escape automático.
- No desactives CSRF (`@csrf_exempt`) salvo en webhooks que verifiquen una firma.
- Formularios: validá siempre en el servidor (largos, formatos, rangos).
- El front **nunca** llama a servicios externos con claves; todo pasa por Django.

### Archivos subidos
- Validar tipo real y tamaño máximo (por defecto 10 MB). Guardar con nombre generado, no el que mandó el usuario.
- Guardarlos fuera de lo público. Si hay que mostrarlos, servirlos desde una vista que verifique el permiso.
- No ejecutar ni interpretar los archivos (PDF, Excel, imágenes) más allá de lo necesario.

### Descargar URLs y recibir webhooks
- Si la app **descarga una URL** que escribe alguien: solo direcciones públicas por http/https. Rechazar `localhost`, redes privadas y direcciones internas; revisar también cada redirección; poner tope de tamaño y de tiempo.
- Si recibe **webhooks** o llamadas de otros sistemas: verificar la firma (HMAC) y compararla en tiempo constante (`hmac.compare_digest`). Sin firma configurada, rechazar en producción.

### Datos personales
- Pedí y guardá **lo mínimo necesario**.
- En los logs, enmascarar teléfonos, mails, documentos y direcciones. Nunca loguear claves, tokens ni el cuerpo completo de los pedidos.
- **No pegues datos reales de clientes o empleados** en el chat, en los tests ni en datos de ejemplo. Usá datos inventados.
- Tiene que ser posible borrar o anonimizar los datos de una persona si lo pide (Ley 25.326 de Protección de Datos Personales).
- Si guarda datos de pago: **nunca** números de tarjeta ni códigos. Se usa el proveedor de pagos y la app guarda solo el identificador de la operación.

### Si la app usa Claude u otra IA
- La clave de la IA va solo en el `.env` del servidor. El front no llama a la IA directo.
- Lo que escribe el usuario va como **dato**, nunca como parte de las instrucciones del sistema. Asumí que puede intentar manipular a la IA.
- Poné un tope de llamadas y de gasto por usuario y por día, y avisá cuando se alcanza.
- La IA no ejecuta acciones irreversibles (borrar, enviar, pagar) sin confirmación de una persona.
- No mandes a la IA datos que no hacen falta para la tarea.

### Dependencias
- Pocas y conocidas. Antes de agregar un paquete, verificá que exista, que sea el correcto (hay paquetes falsos con nombres parecidos) y que tenga mantenimiento reciente.
- Versiones fijas en `requirements.txt`. Nada de instalar cosas desde repositorios o sitios sueltos.
- Corré `pip-audit` y arreglá lo que marque antes de entregar.

### Errores
- Al usuario nunca se le muestra el detalle técnico de un error; se le muestra un mensaje claro y el detalle va al log.
- Páginas 403, 404 y 500 propias, sin datos internos.

## Estructura del proyecto (siempre igual, así el revisor encuentra todo)
```
config/            settings, urls, wsgi/asgi
<app>/             una app de Django por tema (models, views, urls, tests)
templates/         HTML
static/            CSS, JS e imágenes propias
requirements.txt   dependencias con versión fija
.env.example       variables sin valores reales
.gitignore         incluye .env, *.sqlite3, media/, logs/, .venv/
README.md          ficha de la app (ver Paso 0), responsable, cómo correrla
docs/deploy.md     pasos exactos para el revisor
docs/datos.md      qué datos guarda y quién los ve (si aplica)
deploy/            archivos de servidor (gunicorn, systemd, nginx) si hacen falta
CLAUDE.md          este archivo
```
- La app tiene una ruta `GET /health` que responde "ok" y **no devuelve ningún dato**.
- Los logs van a archivos rotativos (no crecen sin límite), en una carpeta configurable por `.env`.

## Calidad
- **Git desde el primer día.** Si no hay repositorio, hacé `git init` y un primer commit. Después, un commit por tarea terminada, con mensaje claro en español. Nunca subas `.env`, `db.sqlite3`, `media/`, `logs/` ni `.venv/`.
- **Tests** con `python manage.py test`. Como mínimo: el flujo principal funciona, y **cada vista protegida responde 403 o redirige al login** a quien no tiene permiso y no deja ver datos de otros usuarios. Corrélos antes de cada commit.
- Migraciones siempre en el repositorio. Nunca edites una migración que ya se subió; creá una nueva.
- Nada de datos de prueba reales en el repositorio.
- Diseño que funcione en celular. Textos en español rioplatense. Contraste suficiente y botones con texto, no solo íconos.

## Cuándo está "listo"
**No digas que la app está lista hasta haber corrido todo esto y mostrarle el resultado a la persona en palabras simples:**
1. `python manage.py test` en verde.
2. `python manage.py check --deploy` sin advertencias.
3. `pip-audit` sin vulnerabilidades pendientes.
4. `bandit -r .` (excluyendo `.venv`) sin problemas altos sin explicar.
5. Búsqueda de claves o secretos escritos en el código y en el historial de git: ninguno.
6. Revisión de permisos: listar las vistas y confirmar que cada una tiene control de acceso, o que es pública a propósito.

Después entregá:
- **`docs/deploy.md`** con: comandos exactos, versión de Python, variables nuevas del `.env` (nombres, sin valores), migraciones, tareas en segundo plano, servicios a reiniciar, subdominio, cómo hacer backup y restaurar la base, y **cómo volver atrás si el deploy sale mal**.
- **`README.md`** actualizado: qué hace, quién la usa, qué datos guarda, con qué se conecta, **nombre del responsable de la app** y qué hacer si esa persona deja la empresa.
- Un resumen para la persona, en palabras simples: qué hace la app, cómo probarla y qué le tiene que pasar al revisor.

## Límites
- No toques el servidor, no hagas deploy, no cambies nada fuera de la carpeta del proyecto.
- No crees cuentas, trials ni contrates servicios externos (hosting, APIs pagas, mails) sin permiso expreso de la persona, y avisale de cualquier costo.
- No instales herramientas globales en la compu. Usá un entorno virtual dentro del proyecto (`.venv`).
- No borres datos, bases ni archivos de la persona. Si hay que reemplazar algo, mirá primero qué hay y preguntá.
- Si algo es ambiguo o riesgoso (datos sensibles, plata, mensajes a terceros), **preguntá antes**.
