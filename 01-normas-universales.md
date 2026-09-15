# Skill: bagisto-universal

Normas universales de trabajo para **cualquier proyecto Bagisto 2.x**.
Destiladas de producción real (tienda peruana, 1239 productos, checkout por
WhatsApp). Cargar junto a la skill específica de cada tienda. Sin datos de
ninguna tienda concreta: adaptar nombres, números y textos a cada proyecto.

## Norma 1 — Apagar, nunca eliminar

Ningún cambio elimina código ni funciones. Todo lo que se apague debe
quedar (a) intacto en código, (b) reversible por configuración o admin, y
(c) anotado en un Registro de desactivados.

- Los bloques de Blade se condicionan (`@if`), no se borran. Los marcos
  vacíos no se dibujan, pero la función sigue viva y el admin la controla.
- Los comportamientos de negocio que hoy van fijos en código llevan
  interruptor en `core_config` (clave propia, ej.
  `mitienda.checkout.solo_pais`), con valor por defecto = comportamiento
  actual. Verificar el ON y el OFF.
- Si el usuario pregunta qué está desactivado, responder leyendo el
  Registro (no adivinar).

### Registro de desactivados (plantilla por proyecto)

| Función | Estado | Dónde se revierte |
|---|---|---|
| (ej. Boletín) | OFF (`clave.valor=0`) | Admin → … o `=1` |
| (ej. Pasarelas no usadas) | OFF (`....active=0`) | Admin Ventas → Métodos de pago |
| (ej. Sección footer) | inactiva (`status=0`) | Admin Apariencia → Temas |
| Pedidos/usuarios de prueba | (listar IDs) | Preguntar antes de borrar |

## Norma 2 — Todo cambio trae su mapa de impacto

Antes de ejecutar, listar a qué afecta (checkout, admin, correos,
traducciones, otros flujos) y verificarlo. Ejemplo real: hacer el email
opcional reventó la creación de pedidos porque el listener de "pedido
creado" enviaba el correo de confirmación (`TypeError`, que el `catch`
no atrapa). El mapa lo habría anticipado; la prueba E2E lo confirmó.

## Norma 3 — Checklist pre-deploy / seguridad

Antes de migrar a producción, verificar TODO:

1. Credenciales admin reales (eliminar las de fábrica).
2. `APP_DEBUG=false` en el `.env` de producción.
3. `APP_KEY` generada para ese servidor. OJO: al migrar el MISMO
   proyecto se conserva la `APP_KEY` (si cambia, las contraseñas de
   clientes dejan de funcionar). La regla de "no reusar" aplica ENTRE
   proyectos distintos.
4. Decidir qué pedidos/usuarios de prueba se borran (con cascada
   controlada: items, pagos, direcciones, notificaciones) y cuáles quedan.
5. Correos SMTP configurados o explícitamente pospuestos.

## Norma 4 — Email opcional (patrón)

Si la venta se cierra por otro canal (WhatsApp), el email del checkout
puede ser opcional, pero con resguardos:

- Validación `nullable|email` (nunca eliminar el campo).
- Resguardos `empty($...->customer_email)` en TODOS los listeners que
  envían correo al cliente (pedido creado/cancelado/comentado, envío,
  factura, reembolso): sin email no se intenta enviar.
- El mensaje alternativo (ej. WhatsApp) debe tolerar email vacío (`N/A`).
- Registro de cuentas: ahí el email sigue obligatorio (login).

## Norma 5 — SEO base (Fase 1)

1. Meta descriptions de productos: generarlas desde la descripción corta
   (corte en palabra, 160 máx) + `indexer:index`. Fallback del Blade con
   corte en palabra (nunca `Str::limit` seco).
2. Sitemap: crear la fila + disparar el job; verificar `/sitemap.xml`.
   REGENERAR al cambiar de dominio.
3. Rich snippets ON (`catalog.rich_snippets.*`): JSON-LD en fichas.
4. Canonical en fichas y categorías (dinámico: cambia solo con dominio).
5. NO hacer: meta keywords (Google lo ignora), hreflang/robots/hostname
   hasta tener el dominio final.

## Norma 6 — GEO mínimo

- JSON-LD `Organization` + `Store/LocalBusiness` en el layout (nombre,
  teléfono, dirección, moneda, pagos aceptados).
- `public/llms.txt`: ficha comercial (quiénes, categorías con URLs,
  contacto, sitemap). Estático, sin código.
- Expectativa honesta: suma para IAs, no garantiza citas.

## Norma 7 — Migración cPanel + recambio de catálogo

### Migración

- Hosting: PHP **8.4** + extensiones (`intl, gd, zip, bcmath, soap,
  imagick…`), docroot a `/public`, MySQL/MariaDB, cron cada minuto,
  terminal/SSH, 512M. Elasticsearch: opcional.
- Paquete: `.zip` con `vendor/` e imágenes, SIN `node_modules`, cachés,
  logs ni `.git`. Descomprimir por terminal.
- Datos: dump completo (nunca solo archivos: el catálogo vive en la BD).
- `.env` prod: `APP_URL` real, MISMA `APP_KEY`, `APP_DEBUG=false`.
- Encendido: `storage:link`, `migrate --force`, `optimize`,
  `indexer:index`, regenerar sitemap, cron `schedule:run`, SSL + HTTPS.
- Cron `schedule:run` cada minuto (`* * * * *`, cPanel → Trabajos de cron):
  comando `/usr/local/bin/php /home/<cuenta>/<tienda>/artisan schedule:run
  >>/dev/null 2>&1`. OJO con el binario: usar el PHP que SÍ tenga `intl`
  (verificado con `php -m`); el `ea-phpXX` de los ejemplos de cPanel puede
  no traerlo. El `>>/dev/null 2>&1` evita un correo por minuto.
  Verificación: `grep -i "schedule\|cron" storage/logs/laravel.log`
  (vacío = trabaja en silencio). Revisar capturas del formulario antes de
  guardar (frecuencia + comando completo, el campo suele verse cortado).
- Verificación E2E en producción antes de anunciar.

### Recambio de catálogo

- Archivos ↔ archivos, datos ↔ datos. Subir archivos NO cambia
  productos/precios (viven en BD).
- Importador DataTransfer: upsert por SKU (`append`), borrado selectivo
  por archivo (`delete`). NO hay modo "reemplazar todo": orquestar
  append + delete + regenerar (flat, sitemap, cachés). Pedidos y clientes
  quedan intactos.
- Mantener SKUs estables entre catálogos para que el upsert funcione.

## Norma 8 — Cachés y verificación (siempre)

Tras cambios de datos, en orden: flat/indexer, `CatalogApiCache`,
`responsecache:clear`, `cache:clear` (+ reiniciar `artisan serve` si
retiene valores). Tras cambios de código: `vendor/bin/pint` en tocados.
Verificación: pedido E2E real (no solo tinker), HTML servido con `curl`
(no el archivo), y Artisan serve vivo (`curl` 200) antes de verificar.

## Norma 9 — Enseñar la autopista del admin (2026-09-10)

Si lo pedido existe como opción en el admin de Bagisto, NO hacerlo
directo: decirle a Juan la ruta exacta (`Admin → ...`) para que lo haga
él y aprenda sin depender de la IA después. Solo intervenir si pide
ayuda o si la opción no existe y requiere código/BD. Frase tipo:
"Juan, esto lo haces tú en [ruta], así aprendes el camino."

## Norma 10 — Solución de problemas post-deploy (cPanel)

Síntomas reales de una migración y su causa. Diagnosticar SIEMPRE con
evidencia (log, `curl`, archivo de prueba), nunca adivinando.

1. **Categorías/listados "cargando" infinito o API 500.** Mirar
   `storage/logs/laravel.log` (con `APP_DEBUG=false` el error va ahí).
   - `NumberFormatter ... install the intl extension`: el PHP **web** no
     tiene `intl` aunque la terminal sí. Suele haber 2 gestores de PHP
     desalineados (MultiPHP `ea-phpXX` vs CloudLinux PHP Selector
     `alt-phpXX`). Confirmar con un archivo temporal:
     `printf '%s' '<?php header("Content-Type: text/plain"); echo PHP_VERSION."|intl:".(extension_loaded("intl")?"YES":"NO")."|".php_ini_loaded_file()."|".php_sapi_name();' > public/_diag.php`
     y `curl` a `/_diag.php`. Solución: poner el dominio en el gestor
     que sí tenga `intl` (CloudLinux PHP Selector). Borrar `_diag.php`.
2. **Sitemap/robots con `localhost` o dominio viejo.** El host sale de
   `channels.hostname`, no solo de `APP_URL`. Corregir:
   `DB::table('channels')->update(['hostname' => 'https://dominio'])` y
   **regenerar el sitemap**. `APP_URL` solo no basta.
3. **Archivos del sitemap dan 403.** El generador los crea en `600`
   (el servidor web no puede leerlos). `SITE CHMOD` por FTP suele ser
   ignorado y re-subir por FTP mantiene `600` (umask 077). Solución
   fiable: un script PHP temporal (corre como el usuario de la cuenta)
   que haga `chmod($archivo, 0644)` y `chmod($dir, 0755)`; borrarlo luego.
4. **`Access denied for user '...'@'localhost'` tras crear el `.env`.**
   Casi siempre la clave trae `#`, `$`, espacios o comillas que el parser
   del `.env` malinterpreta. Usar clave **solo letras, números y `_`**,
   sin comillas. Probar `php artisan migrate --force`.
5. **Document Root debe apuntar a `/public`.** Si se ve "Index of /", el
   docroot está en la raíz del proyecto y hay que agregarle `/public`.
6. **DNS.** Dominio nuevo puede tardar. Verificar con `ping` (debe dar la
   IP del hosting). Un dominio propio exige registrarlo (costo); un
   subdominio del dominio ya contratado es gratis. Si el DNS se maneja
   fuera del hosting, agregar registro `A` en el proveedor.
7. **No duplicar subdominios.** Un solo destino para la tienda; borrar
   los subdominios/carpetas de prueba para no confundir la subida.

## Norma 11 — Deploy de archivos (FTP / cPanel), sin SSH

**Recomendación para todo proyecto Bagisto:** habilitar que el agente
(OpenCode) pueda reemplazar archivos por FTP sin depender de SSH. En la
práctica, el dueño crea una cuenta FTP acotada al proyecto, entrega las
credenciales en la sesión, y el agente respalda, sube y verifica los
archivos por su cuenta. Comprobado en producción real (SSH cerrado, FTP
abierto). Habilitación en un proyecto nuevo:

1. **Probar puertos** desde la máquina del agente (`21` FTP, `2083` API
   cPanel; `22`/`2222` SSH suelen estar cerrados). Si el 21 está abierto,
   el deploy de archivos es posible sin tocar nada del hosting.
2. **Crear cuenta FTP acotada** (ver abajo).
3. El agente opera: respaldar → subir → `diff` → (config por AP/admin) →
   limpiar cacheados → borrar temporales → avisar para rotar claves.

Cuando SSH (puerto 22) está cerrado pero FTP (21) y cPanel API (2083) están
abiertos, se reemplazan archivos por FTP.

1. Verificar puertos abiertos desde la máquina local:
   `for P in 21 2083 22; do timeout 5 bash -c "echo > /dev/tcp/HOST/$P" && echo "$P abierto"; done`
2. Crear una cuenta FTP en cPanel (Cuentas FTP) **limitada a la carpeta del
   proyecto**. OJO: si el Directorio queda con una subcarpeta con el nombre
   del usuario (ej. `/ruta/proyecto/deploy`), la cuenta queda enjaulada y no
   llega al proyecto; corregir el Directorio a la raíz del proyecto.
3. Flujo seguro: **respaldar** el archivo remoto, **subir**, **verificar**:
   - `curl -u 'user:pass' "ftp://HOST/ruta/archivo" -o backup`
   - `curl -u 'user:pass' -T archivo_local "ftp://HOST/ruta/archivo"`
   - `diff archivo_local backup` (deben coincidir con el nuevo).
4. La **configuración y los datos viven en la BD**, no viajan por FTP: los
   cambios de config se hacen por Admin o terminal (tinker), y luego se
   limpian cachés (`optimize:clear`, `optimize`, `responsecache:clear`).
5. **Permisos**: archivos creados por FTP o por trabajos en segundo plano
   pueden salir en `600`. Si el web da 403 en un archivo legítimo, revisar
   permisos; el `chmod` fiable se hace con un script PHP temporal, no por
   FTP.
6. **Credenciales**: no se escriben en los manuales; se rotan tras el deploy
   (una clave pegada en un chat queda expuesta). Cuenta FTP acotada al
   proyecto, nunca a todo el hosting.

## Norma 10b — POST JSON con 500 genérico: revisar CSRF primero (2026-09-15)

El `Handler` convierte `TokenMismatchException` en el JSON 500 genérico
cuando el request lleva `Accept: application/json` (el 419 solo sale en
HTML). Antes de depurar el controlador: probar con sesión+token CSRF
válidos generados en el servidor. Un 500 sin rastro en `laravel.log`
suele ser CSRF, no código.

## Norma 11b — Deploy solo con autorización (2026-09-15)

NUNCA subir nada al servidor (FTP, scripts, config) sin autorización
expresa de Juan para ese cambio. Todo cambio se desarrolla y prueba en
local; el deploy se propone por popup y se ejecuta solo con su OK.

## Norma 12 — Preguntar siempre con popup (2026-09-15)

Toda pregunta, opcion o confirmacion se hace con ventanas popup
(question tool), NUNCA solo en texto plano: Juan elige opciones o
escribe su respuesta ahi. Asi lo pidio expresamente.
