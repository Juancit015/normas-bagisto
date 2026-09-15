# 00 — Cómo trabaja el agente (obligatorio leer primero)

Manual operativo para cualquier agente (OpenCode u otro) que trabaje en un
sistema Bagisto 2.x. Estas formas de trabajo están por encima de cualquier
tarea puntual: si una instrucción del usuario las contradice, preguntar
antes de obedecer.

## 1. Dos modos: plan y ejecución

- **Modo plan**: solo leer e investigar. Cero ediciones, cero borrados,
  cero comandos que cambien nada. Al final se presenta el plan y se pide
  aprobación con preguntas concretas.
- **Modo ejecución**: se ejecuta lo aprobado. Si aparece algo no previsto,
  se vuelve a preguntar (no se improvisa).

## 2. Preguntar SIEMPRE antes de tocar

Antes de crear, borrar o editar cualquier dato (categoría, producto,
traducción, precio, filtro, pedido, configuración), preguntar al dueño y
esperar su OK explícito. Nunca tocar nada sin consentimiento. Después de
ejecutar, confirmar mostrando lo verificado en BD o en pantalla (nunca
"debería funcionar").

## 3. Apagar, nunca eliminar (regla de oro)

Ningún cambio elimina código ni funciones (ver `01-normas-universales.md`,
Norma 1). Todo lo que se apague queda intacto, reversible por
configuración y anotado en el Registro de desactivados de cada proyecto.
Si el dueño pregunta qué está desactivado, responder leyendo ese registro.

## 4. Todo cambio trae su mapa de impacto

Antes de ejecutar, listar a qué afecta (checkout, admin, correos,
traducciones, otros flujos) y verificarlo. Ejemplo real que justifica la
regla: hacer el email opcional reventó la creación de pedidos porque el
listener de "pedido creado" intentaba enviar el correo de confirmación.

## 5. Verificación obligatoria (nada se da por hecho)

- Tras cambios de datos: refrescar cachés en orden (flat/indexer,
  caché de listados API, FPC/páginas, caché de config) + reiniciar el
  servidor local si retiene valores.
- Tras cambios de código: formateador (`vendor/bin/pint`) en archivos
  tocados.
- Verificación E2E real: pedido completo de prueba o página servida con
  `curl` (nunca solo el archivo, nunca solo `tinker`). Servidor local
  vivo (`200`) antes de verificar.
- Probar el interruptor en ON y en OFF cuando se cree uno.

## 6. Cómo reportar al dueño

- Lenguaje simple, sin tecnicismos salvo que los pida. Tablas cortas.
- Estructura: qué se hizo → qué se verificó (con evidencia) → qué falta
  o qué decide el dueño.
- Ante un "sí" ambiguo, preguntar a qué se refiere. La duda se pregunta,
  no se adivina.
- Los secretos (claves, APP_KEY, contraseñas) NUNCA van en archivos ni en
  reportes: solo dónde van, jamás el valor.

## 7. Enseñar antes que hacer (Norma 9)

Si lo pedido existe en el admin, indicarle a Juan la ruta exacta para
que lo haga él y aprenda. Solo intervenir con código/BD si lo pide o si
la opción no existe.

## 8. Orden de lectura del agente nuevo

1. Este archivo (`00`).
2. `01-normas-universales.md`.
3. El manual del proyecto concreto (`bagisto-<tienda>/`).
4. Recién después, tocar algo (previo plan + aprobación).
