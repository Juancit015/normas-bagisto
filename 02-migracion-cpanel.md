# 02 — Migración a cPanel (manual paso a paso)

Procedimiento probado para subir una tienda Bagisto 2.x desde desarrollo
local a hosting compartido cPanel. Los valores entre `<...>` se toman de
la ficha del proyecto (`bagisto-<tienda>/00-ficha-tienda.md`). NUNCA
anotar claves aquí: solo dónde van.

## Requisitos del hosting (verificar ANTES)

- PHP **8.4** en el dominio de la tienda (MultiPHP Manager o PHP Selector
  por dominio; el resto de dominios no se tocan).
- Extensiones: `bcmath, calendar, ctype, curl, dom, exif, fileinfo,
  gd, gmp, imagick, intl, mbstring, openssl, pdo_mysql, soap, sockets,
  zip, simplexml, xmlreader, xmlwriter, tokenizer, opcache`.
- Opciones: `memory_limit 512M`, `upload_max_filesize 100M`,
  `max_execution_time 300`.
- MySQL/MariaDB con BD + usuario con todos los privilegios.
- Cron disponible (tarea cada minuto) + Terminal/SSH.
- Elasticsearch: NO necesario (Bagisto usa la BD como respaldo).
- DNS del dominio/subdominio apuntando al hosting (verificar con `ping`
  → IP del servidor, o `whatsmydns.net`).

## Sesión 1 — Paquete (en local)

1. Dump BD: `mysqldump --single-transaction` → `<tienda>-bd.sql`.
2. Anotar `APP_KEY` local (se reutiliza: mismo proyecto).
3. Empaquetar `.tar.gz` (o `.zip`) EXCLUYENDO: `node_modules`, `.git`,
   `storage/framework/{cache,sessions,views}`, `storage/logs`,
   importaciones fuente, `*.hot`, y sobre todo **el `.env` local**.
   INCLUYENDO: `vendor/`, imágenes (`storage/app/public`), traducciones.
4. Verificar el paquete: sin `.env` dentro, con `vendor/autoload.php`,
   `public/index.php` e imágenes clave.
5. Receta `.env` producción: `APP_URL` real, MISMA `APP_KEY`,
   datos de BD nueva, `APP_DEBUG=false`, colas/sesión en archivo.

## Sesión 2 — Subida y encendido (en cPanel)

1. Subir paquete + SQL al Document Root del dominio.
2. Descomprimir por terminal (`tar -xzf`).
3. Importar BD (`mysql -u <usu> -p <bd> < <tienda>-bd.sql`).
4. Crear `.env` con la receta.
5. Comandos en orden: `storage:link`, `migrate --force`, `optimize`,
   `indexer:index`, limpiar cachés.
6. Document Root → `.../public`. Regenerar sitemap con dominio real.
7. Cron `schedule:run` cada minuto. SSL (AutoSSL) + forzar HTTPS.
8. Verificación E2E en producción (home, ficha, checkout de prueba,
   métodos de pago/envío, WhatsApp/correo, `/sitemap.xml`,
   `robots.txt`) antes de anunciar.

## Checklist pre-deploy (bloqueante, sin esto no hay migración)

1. Credenciales admin reales (fábrica eliminadas).
2. `APP_DEBUG=false`.
3. `APP_KEY` del proyecto (conservada, no inventada).
4. Pedidos/usuarios de prueba: borrados o declarados.
5. Principal/otros sitios del hosting verificados intactos.
