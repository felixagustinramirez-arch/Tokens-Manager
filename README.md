# Token Manager

Token Manager es una Progressive Web App (PWA) local-first para registrar manualmente los ciclos de uso de Claude. No se conecta a Claude, no lee credenciales y no intenta calcular el saldo real de tokens. La aplicación solo conserva los eventos que el usuario registra en el dispositivo.

> **Regla temporal:** el próximo corte se calcula como `primer mensaje + duración del ciclo`. Marcar los tokens como agotados nunca mueve el corte. Cuando el corte llega, la cuenta vuelve a `WAITING_FIRST_MESSAGE` y el siguiente primer mensaje inicia el intervalo posterior.

## Funciones incluidas

La aplicación permite agregar, editar, pausar y eliminar cuentas. Cada cuenta conserva su duración de ciclo, el próximo corte, el inicio del intervalo actual, el estado de tokens y un historial de eventos. El dashboard muestra métricas globales, cuenta regresiva en tiempo real y acciones rápidas para registrar el primer mensaje, marcar disponibilidad, marcar agotamiento o reiniciar manualmente.

Los datos estructurados se guardan en **IndexedDB**. La aplicación incluye backup JSON, restauración con validación de esquema y borrado completo de los datos locales. Los timestamps se almacenan como valores absolutos y se presentan con la zona horaria local del dispositivo.

El service worker proporciona cache offline y una base para recibir Web Push real. Las notificaciones locales se intentan cuando la aplicación está activa y el navegador concede permiso.

## Desarrollo local

Requisitos: Node.js 20 o posterior y pnpm.

```bash
pnpm install
pnpm dev
```

El servidor de desarrollo se abre en `http://localhost:3000`.

Antes de entregar cambios, ejecuta:

```bash
pnpm exec vitest run client/src/lib/time.test.ts
pnpm check
pnpm build
```

El build genera el frontend en `dist/public`. El scaffold conserva un servidor Express para la vista previa del entorno, pero la aplicación principal no depende de un backend.

## Publicar en GitHub Pages

El proyecto está configurado con `base: "./"`, `start_url: "."` y `scope: "."`. Estas rutas relativas permiten publicar tanto en un dominio raíz como en un Project Site, por ejemplo `https://usuario.github.io/repositorio/`.

### Publicación automática con GitHub Actions

El workflow `.github/workflows/deploy-pages.yml` compila `dist/public` y lo publica con GitHub Pages.

1. Sube este repositorio a GitHub.
2. En **Settings → Pages**, selecciona **GitHub Actions** como fuente de publicación.
3. Haz push a la rama `main`.
4. Espera a que termine el workflow **Deploy Token Manager to GitHub Pages**.
5. Abre la URL que GitHub muestre en la sección **Pages**.

### Publicación manual

Si no quieres usar Actions, ejecuta `pnpm build` y configura GitHub Pages para publicar el contenido de `dist/public`. No publiques las carpetas `.manus-logs`, `node_modules` ni archivos de entorno.

## Instalar en iPhone

Abre la URL publicada en Safari. Pulsa **Compartir**, elige **Añadir a pantalla de inicio** y confirma. La aplicación se abrirá en modo standalone y utilizará el almacenamiento local del dispositivo.

Safari debe tener permiso para las notificaciones si quieres recibir recordatorios mientras la PWA está activa. La disponibilidad de notificaciones puede variar según la versión de iOS y la configuración del dispositivo.

## Limitaciones de notificaciones

GitHub Pages sirve archivos estáticos. No ejecuta cron jobs ni un proceso capaz de enviar Web Push cuando la PWA está cerrada. Por eso Token Manager no finge que un service worker puede despertar la aplicación por sí solo.

El modo incluido funciona así:

- **Recordatorios locales:** se comprueban mientras la aplicación o la PWA está abierta y el navegador concede permiso.
- **Service worker:** mantiene cache offline y contiene el punto de extensión para el evento `push`.
- **Web Push real:** requiere un proveedor o servidor externo que almacene suscripciones y envíe notificaciones. Las claves privadas VAPID nunca deben guardarse en el frontend.

## Modelo temporal

El flujo correcto es el siguiente:

1. Una cuenta puede tener un próximo corte configurado o no tener ciclo iniciado.
2. Cuando llega el corte, la cuenta pasa a `WAITING_FIRST_MESSAGE`, sus tokens vuelven a `AVAILABLE` y no se inicia ningún intervalo automáticamente.
3. El usuario registra el primer mensaje. En ese instante se guarda `intervalStartedAt = now` y `nextResetAt = now + cycleDurationMinutes`.
4. Si los tokens se agotan antes del corte, solo cambia `tokenStatus` a `EXHAUSTED`.
5. El corte permanece intacto y se reconcilia al volver a abrir o activar la aplicación.

Ejemplo de aceptación: si el corte inicial es 05:30, el primer mensaje ocurre a 05:40 y los tokens se agotan a 08:00, el intervalo sigue siendo 05:40 → 10:40. Nunca se convierte en 08:00 → 13:00.

## Seguridad y privacidad

Token Manager nunca solicita ni almacena contraseñas, cookies, API keys, códigos de autenticación ni tokens de Claude. El backup JSON solo contiene cuentas, eventos y ajustes de notificaciones que el usuario haya registrado.

## Estructura principal

```text
client/
  public/
    manifest.json
    sw.js
    icon.svg
    icon-192.png
    icon-512.png
  src/
    hooks/useTokenManager.ts
    lib/db.ts
    lib/models.ts
    lib/notifications.ts
    lib/time.ts
    lib/time.test.ts
    pages/Home.tsx
    index.css
```

La lógica temporal, la persistencia, las notificaciones y la UI están separadas para que una futura integración de Web Push pueda añadirse sin cambiar el modelo local.

## Licencia

MIT.

## References

[1]: https://docs.github.com/en/pages "GitHub Pages documentation"
[2]: https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps "Progressive web apps on MDN"
[3]: https://developer.mozilla.org/en-US/docs/Web/API/Notifications_API "Notifications API on MDN"
