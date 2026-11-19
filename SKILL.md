---
name: revisar-app
description: Revisión legal (España/UE) y de seguridad/costes de cualquier app o web antes o después de lanzarla. Úsala cuando se pida "revisar app", "auditoría legal y seguridad", "checklist antes de lanzar", "/revisar-app", o revisar una app hecha con IA (vibe coding), WordPress, Next.js, Supabase/Neon, app móvil, tienda online, etc.
---

# Revisar app: legal + seguridad + costes

Guion para revisar CUALQUIER app/web (Next.js, WordPress, app móvil, panel interno, landing, tienda, API, automatización…). Parte de las 5 trampas típicas de las apps hechas con IA (factura sorpresa, accesibilidad, base de datos abierta, mensajes sin permiso, función infinita) y las amplía a lo que exige la ley española/UE y la seguridad básica.

> No sustituye asesoría jurídica. Los importes y fechas legales cambian: si algo es dudoso, márcalo como "verificar" en vez de afirmarlo.

## Reglas de trabajo

1. **Solo lectura primero.** Revisar todo y entregar informe. NO cambiar nada en producción, ni rotar claves, ni enviar correos, sin OK explícito.
2. **Evidencia siempre.** Cada hallazgo con archivo:línea, URL, cabecera o comando que lo demuestra. Nada de "probablemente".
3. **Saltar lo que no aplica** (p. ej. sin tienda → sin desistimiento), pero decir que se ha saltado y por qué.
4. **No atacar a terceros.** Pruebas solo contra la app propia, sin cargas masivas.
5. Al final: tabla de resultados + lista de arreglos priorizada. Arreglar solo tras OK, de uno en uno, y verificar cada arreglo.

## Paso 0 — Ficha de la app

Antes de revisar, rellenar:

- Nombre, URL(s), repo/ruta local, titular (empresa/autónomo).
- Stack: framework, hosting (Vercel, Netlify, hosting compartido…), BD (Neon, Supabase, MySQL…), auth, pagos (Stripe, Redsys…), correo (Resend, SES, Mailjet…), IA (Anthropic, OpenAI…), crons, almacenamiento de ficheros.
- Público: consumidores (B2C) o empresas (B2B). ¿Menores? ¿Ventas online? ¿App en App Store / Google Play?
- Datos personales que recoge (formularios, cuentas, pedidos, analítica, ubicación, salud, imágenes…).
- Tamaño del titular: ¿microempresa (<10 empleados y ≤2 M€ de facturación)? Afecta a la accesibilidad.

---

## BLOQUE A — Costes y facturas sorpresa

- [ ] **Límite de gasto en cada proveedor de pago por uso:** Vercel (Spend Management con pausa al llegar al límite), Netlify, Cloudflare, AWS/GCP (budgets + alertas; ojo: las alertas NO frenan el gasto), Neon (máximo de autoscaling), Supabase (spend cap), APIs de IA (límite mensual en la consola), SMS, correo.
- [ ] **Alertas** de facturación al correo del titular (50 %, 80 %, 100 %).
- [ ] **Ancho de banda:** ficheros grandes (vídeos, PDFs, imágenes sin optimizar) servidos desde un hosting que cobra por GB → CDN/almacenamiento barato o comprimir. Buscar ficheros > 1 MB en el repo y en `/public`.
- [ ] **Descarga en bucle / hotlinking** de ficheros pesados: caché, rate limit o URLs firmadas.
- [ ] **Recursión y bucles:** funciones que se llaman a sí mismas, webhooks que disparan el mismo webhook, triggers de BD que escriben en la misma tabla, crons que se solapan, `useEffect` que hace fetch en bucle.
- [ ] **Reintentos con tope y backoff** (nunca reintentar sin fin). Colas con dead-letter.
- [ ] **Timeouts** en funciones serverless y llamadas externas.
- [ ] **IA:** agentes o cadenas sin máximo de pasos/tokens; endpoints públicos que llaman al LLM sin auth ni rate limit (cualquiera gasta tu saldo).
- [ ] **Crons** con frecuencia razonable.
- [ ] **Endpoints públicos caros** (PDFs, imágenes, IA, envío de correos/SMS) protegidos con auth, rate limit y/o captcha.

## BLOQUE B — Base de datos y datos expuestos

- [ ] **Row Level Security** activado en TODAS las tablas expuestas (Supabase: ninguna tabla "RLS disabled"; Neon Data API: RLS + políticas). RLS sin políticas correctas = igual de abierto. Probar con la clave pública/anon: un `select` de `users`, `profiles`, `orders`, `waitlist` debe devolver vacío o error.
- [ ] **Claves en el cliente:** en el JS público solo claves públicas (anon, publishable). Buscar en el bundle y el repo: `service_role`, `sk_live`, `SECRET`, `PRIVATE`, `DATABASE_URL`, `API_KEY`, tokens. En Next.js, nada sensible con prefijo `NEXT_PUBLIC_`.
- [ ] **Secretos en git:** `.env` fuera del control de versiones; revisar el historial. Si un secreto se filtró → **rotarlo** (no basta con borrarlo).
- [ ] **Autorización por objeto (IDOR):** cambiar el id en la URL/API (`/api/orders/123` → `124`) no puede dar datos de otro usuario. Cada endpoint filtra por el usuario de la sesión, no por el id que manda el cliente.
- [ ] **Rutas de admin** protegidas en el servidor (no solo ocultas en el menú).
- [ ] **Buckets de ficheros** privados de verdad (facturas, DNIs, fotos de clientes); URLs firmadas con caducidad.
- [ ] Sin listado de directorios, backups (`.sql`, `.zip`, `.bak`) ni `.git/` accesibles por web.
- [ ] **Respuestas de API** sin campos de más (hashes, emails de otros, notas internas).
- [ ] **Copias de seguridad** automáticas, fuera del mismo servidor, y restauración probada al menos una vez.

## BLOQUE C — Seguridad general

**Autenticación**
- [ ] Contraseñas con hash fuerte (bcrypt/argon2), nunca en claro ni MD5/SHA1.
- [ ] Rate limit / bloqueo en login, registro, recuperar contraseña y verificación de códigos.
- [ ] 2FA para administradores. Sin usuarios "admin" genéricos ni contraseñas compartidas.
- [ ] Cookies de sesión `HttpOnly`, `Secure`, `SameSite`; caducidad razonable.
- [ ] Recuperar contraseña: token de un solo uso con caducidad; no revela si el email existe.
- [ ] Freno a registros de bots (captcha/Turnstile, honeypot, verificación de email).

**Entrada y salida**
- [ ] SQL parametrizado (sin concatenar strings). Validación en el servidor (zod o similar).
- [ ] Sin XSS: nada de `dangerouslySetInnerHTML`/`innerHTML`/`echo` con datos del usuario sin escapar.
- [ ] Subidas: tipo y tamaño validados en el servidor, nombre regenerado, nada ejecutable (sin `.php` en uploads).
- [ ] CSRF en formularios con sesión (o SameSite + comprobación de origen).
- [ ] CORS sin `*` en endpoints con credenciales.
- [ ] Webhooks de pago (Stripe, Redsys…) con **firma verificada**; importe y estado del pago comprobados en el servidor, nunca desde el cliente.
- [ ] Redirecciones (`?redirect=`) limitadas al propio dominio.

**Infraestructura**
- [ ] HTTPS forzado + HSTS. Cabeceras `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `frame-ancestors`/`X-Frame-Options`, `Referrer-Policy`.
- [ ] Dependencias sin vulnerabilidades críticas (`npm audit`, `composer audit`); PHP/Node en versiones con soporte.
- [ ] Producción sin stack traces ni modo debug.
- [ ] Logs sin contraseñas, tokens ni datos personales innecesarios.
- [ ] Mínimo privilegio: usuario de BD sin permisos de superusuario; tokens con el alcance mínimo.

**WordPress (si aplica)**
- [ ] Núcleo, tema y plugins al día; fuera los plugins abandonados (>1 año sin actualizar) o desactivados.
- [ ] `DISALLOW_FILE_EDIT`; `xmlrpc.php` bloqueado si no se usa; enumeración de usuarios cerrada (`/?author=1`, `/wp-json/wp/v2/users`).
- [ ] Sin admins desconocidos; revisar mu-plugins y uploads por puertas traseras; permisos de ficheros correctos; `wp-config.php` no accesible.
- [ ] Firewall y escaneo de malware.

## BLOQUE D — Comunicaciones comerciales (LSSI art. 21-22)

- [ ] **Consentimiento previo y expreso** para correos, SMS o WhatsApp comerciales. Apuntarse a una lista de espera ≠ aceptar publicidad, salvo que se dijera claramente. Casilla **sin marcar** por defecto y separada de la aceptación de la política de privacidad.
- [ ] Excepción de cliente previo: solo productos/servicios **similares** a los ya comprados, con opción de baja en cada envío.
- [ ] B2B también: la LSSI exige consentimiento para correo comercial aunque el destinatario sea una empresa (salvo la excepción anterior).
- [ ] **Prueba del consentimiento** guardada (fecha, IP, texto aceptado, origen).
- [ ] **Baja** fácil y gratuita en cada envío (enlace + cabecera `List-Unsubscribe` one-click) y aplicada de verdad.
- [ ] Remitente identificado y mensaje identificado como publicitario.
- [ ] Doble opt-in recomendado (evita bots y altas falsas).
- [ ] SPF, DKIM y DMARC correctos; los envíos masivos, desde un subdominio o dominio separado.
- [ ] Referencia: los envíos masivos sin consentimiento son infracción grave de la LSSI (hasta 150.000 €), además del RGPD.

## BLOQUE E — Accesibilidad

- [ ] **¿Es obligatoria?** Ley 11/2023 (Acta Europea de Accesibilidad): desde el 28-06-2025 la cumplen el comercio electrónico y otros servicios a consumidores (EN 301 549 ≈ WCAG 2.1 AA). Las microempresas de servicios están exentas, aunque se recomienda cumplir. En el sector público es obligatoria siempre (RD 1112/2018).
- [ ] Todo usable **solo con teclado** (Tab, Shift+Tab, Enter, Esc): menús, modales, carrito, checkout y formularios. Foco visible.
- [ ] Imágenes con `alt` descriptivo (decorativas con `alt=""`); botones-icono con `aria-label`.
- [ ] Contraste suficiente (4.5:1 texto normal, 3:1 texto grande).
- [ ] Formularios con `<label>`; errores explicados en texto, no solo con color.
- [ ] Un `h1`, jerarquía de encabezados, `lang` en `<html>` y landmarks.
- [ ] Vídeos con subtítulos; nada que parpadee más de 3 veces por segundo; respetar `prefers-reduced-motion`.
- [ ] Zoom al 200 % sin romper; usable en móvil.
- [ ] Lighthouse/axe con la puntuación anotada. Si aplica la ley: **declaración de accesibilidad** publicada, con un canal de quejas.

## BLOQUE F — Protección de datos (RGPD + LOPDGDD)

- [ ] **Política de privacidad** completa: responsable y contacto, finalidades, base legal, plazos de conservación, destinatarios/proveedores, transferencias fuera de la UE, derechos y cómo ejercerlos, reclamación ante la AEPD.
- [ ] Primera capa informativa en **cada formulario** (responsable, finalidad, derechos y enlace a la política).
- [ ] **Recoger solo lo necesario.**
- [ ] **Encargados del tratamiento:** DPA con cada proveedor que toca datos (hosting, BD, correo, CRM, analítica, IA). Para EE. UU.: Data Privacy Framework o cláusulas contractuales tipo.
- [ ] **IA:** no enviar datos personales a APIs de IA sin base legal, sin informar al usuario y sin proveedor con DPA (y sin que entrenen con ellos).
- [ ] **Registro de actividades de tratamiento** si aplica; evaluación de impacto si hay datos sensibles a gran escala.
- [ ] **Derechos** operativos: acceso, rectificación, supresión y portabilidad. **Borrar la cuenta** desde la app (obligatorio en App Store y Google Play).
- [ ] **Menores:** en España el consentimiento propio vale desde los 14 años (verificar si ya está en vigor la subida a 16); por debajo, consentimiento de padres o tutores.
- [ ] **Brechas:** procedimiento para notificar a la AEPD en 72 h.
- [ ] Borrado o anonimización de los datos antiguos según los plazos declarados.

## BLOQUE G — Cookies y analítica

- [ ] El banner **bloquea** las cookies no técnicas (analítica, publicidad, píxeles) **hasta que se aceptan**. Comprobar en incógnito: sin clic no hay `_ga`, `_fbp`… ni peticiones a esos dominios.
- [ ] Botón **Rechazar** al mismo nivel y en el mismo paso que Aceptar (guía AEPD). Configuración por categorías.
- [ ] Retirar el consentimiento fácilmente (enlace permanente). Renovarlo como máximo cada 24 meses.
- [ ] Google Consent Mode v2 coherente con el banner (denied por defecto).
- [ ] **Política de cookies** con la lista real (nombre, proveedor, finalidad, duración).
- [ ] Tracking por servidor (CAPI, etc.) también con base legal; sin enviar datos de quien rechazó.

## BLOQUE H — Aviso legal y comercio electrónico

- [ ] **Aviso legal (LSSI art. 10):** nombre o razón social, NIF, domicilio, email, datos registrales y colegio profesional si aplica.
- [ ] **Condiciones de contratación** (si se vende): proceso de compra, precios **con IVA** y gastos de envío antes de pagar, plazos de entrega, formas de pago, garantía legal (3 años en bienes).
- [ ] **Desistimiento de 14 días naturales** con formulario modelo y sus excepciones.
- [ ] Botón final inequívoco, tipo "**Pedido con obligación de pago**".
- [ ] Confirmación del pedido por email.
- [ ] El enlace a la plataforma europea ODR **ya no es obligatorio**: la plataforma cerró en julio de 2025. Quitarlo si sigue puesto.
- [ ] Hojas de reclamaciones e información de resolución de conflictos según la comunidad autónoma (verificar).
- [ ] Suscripciones: precio de renovación claro, aviso antes de cobrar y **baja tan fácil como el alta**.
- [ ] Sin reseñas falsas; las rebajas se calculan sobre el precio más bajo de los últimos 30 días.
- [ ] **Facturación** (si la app emite facturas): reglamento de sistemas de facturación (RD 1007/2023, VERI*FACTU). Verificar la fecha de obligatoriedad vigente, porque se ha aplazado.

## BLOQUE I — IA, apps móviles y otros

- [ ] **IA (AI Act art. 50):** avisar de que se habla con una IA (chatbots) y marcar el contenido generado o manipulado cuando aplique. Verificar las fechas de aplicación vigentes.
- [ ] Los chatbots no prometen precios, plazos ni condiciones vinculantes; tienen límite de uso.
- [ ] **App Store / Google Play:** las etiquetas de privacidad / "Seguridad de los datos" coinciden con la realidad; borrar la cuenta en la app y por web; política de privacidad enlazada; permisos justificados y pedidos en el momento de usarlos.
- [ ] Notificaciones push comerciales con opt-in y desactivables.
- [ ] Imágenes, fuentes, música y textos con licencia.
- [ ] Términos de uso si hay cuentas o contenido de usuarios.

---

## Formato del informe

```
# Revisión <app> — <fecha>
Ficha: <stack, URL, público, datos>

## Resumen
🔴 Críticos: N · 🟠 Altos: N · 🟡 Medios: N · ✅ OK: N · ➖ No aplica: N

## Hallazgos (por gravedad)
| # | Bloque | Gravedad | Qué pasa | Evidencia (archivo:línea / URL / comando) | Arreglo propuesto | Esfuerzo |

## Lo que está bien
## No aplica (y por qué)
## Pendiente de verificar
```

- 🔴 **Crítico:** datos expuestos, claves filtradas, gasto sin límite, cobro manipulable o spam masivo activo. Arreglar hoy.
- 🟠 **Alto:** incumplimiento legal claro (cookies antes del consentimiento, sin aviso legal, sin baja) o fallo explotable.
- 🟡 **Medio/bajo:** mejoras de accesibilidad, cabeceras, documentación.

Al terminar el informe, preguntar qué arreglos aplicar. Hacerlos uno a uno, verificarlos y actualizar la tabla.
