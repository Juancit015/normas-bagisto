# 03 — Recambio de catálogo (manual)

Principio madre: **archivos ↔ archivos, datos ↔ datos**. Subir archivos
NO cambia productos/precios/stock (viven en la BD). Este manual cubre los
3 casos, del más común al más drástico. Todo cambio de catálogo requiere
aprobación previa (Norma de preguntar siempre) y verificación posterior.

## Caso A — Cambios puntuales (precios, stock, textos)

1. Editar en admin local o directo en BD (con plan aprobado).
2. Regenerar: índice/flat del producto, caché de listados, FPC, caché
   de config. Reiniciar servidor local si retiene valores.
3. Verificar en listado (`/api/products?category_id=X`) Y en ficha.
4. Llevar a producción por la vía que corresponda (archivo si fue
   plantilla/traducción; dato si fue BD: SQL dirigido o re-importación).

## Caso B — Lote nuevo en Excel/CSV (importador DataTransfer)

- Ruta admin: Ajustes → Data Transfer → Imports. Formatos: csv, xls,
  xlsx, xml.
- `action=append` = crea + ACTUALIZA (upsert por SKU: si el SKU existe,
  se actualiza; si no, se crea). Es la vía normal para catálogos nuevos.
- `action=delete` = borra SOLO los SKU listados en el archivo (previa
  validación). No existe "reemplazar todo": lo que no viene en el archivo
  se conserva.
- NO hay importador de categorías: las categorías deben existir antes
  (el import las asigna por nombre `Padre/Hija`).
- En hosting sin worker de colas: modo síncrono (lotes de 100 en el
  navegador) o cron con `queue:work`.
- Regla de oro entre catálogos: **mantener SKUs estables**. Mismo SKU =
  actualización limpia; SKU nuevo = duplicado.

## Caso C — Catálogo en PDF que reemplaza todo

El importador no lee PDFs: el pipeline es manual, igual que la carga
inicial del proyecto (ver ficha de la tienda para su tabla de
reubicaciones y su tono de descripciones):

1. Extraer productos del PDF (nombre con specs reales, sin inventar).
2. Crear/actualizar por SKU (upsert): lo que coincide se actualiza.
3. Borrar descontinuados: lista de SKUs + `delete`, o borrado masivo
   desde el admin. Pedidos y clientes quedan intactos.
4. Regenerar TODO: flat de cada producto, listados, sitemap, cachés.
5. Verificación: conteos (productos, imágenes), filtros por categoría,
   5 fichas al azar (nombre, precio, foto, URL), pedido E2E.

## Verificación mínima de cualquier recambio

- N productos e imágenes (conteo BD vs esperado).
- Filtros obligatorios de cada categoría (los de la ficha de tienda).
- Sin URLs rotas (las `url_key` existentes NO se tocan al renombrar).
- Pedido E2E de un producto nuevo y uno viejo.
