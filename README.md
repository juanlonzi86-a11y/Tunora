# TUNORA · Listen to your world.

Streaming musical privado: **app** (Android APK + web para PC/tablet) + **ADMIN** (Python) + **Supabase**.

```
tunora/
├─ supabase/schema.sql        Tablas, RLS y buckets (ejecutar una vez)
├─ admin/                     ADMIN: panel de escritorio en Python
├─ app/                       App React + Capacitor (Android / PC / tablet)
└─ .github/workflows/android.yml   Compila el APK en la nube
```

> **Estado honesto:** el código está completo, pero no se ha podido compilar ni probar contra tu Supabase, Drive o Dropbox
> (sin red ni credenciales). Usa la lista de comprobación del final. ADMIN y los scripts Python sí pasan la comprobación de sintaxis.

---

## 1. Supabase (10 min)

1. Crea un proyecto en <https://supabase.com>.
2. **SQL Editor → New query**, pega `supabase/schema.sql` y pulsa **Run**.
3. **Authentication → Providers → Email**: déjalo activo (para pruebas puedes desactivar *Confirm email*).
4. **Project Settings → API**: copia **Project URL** y **anon public key**. *Nunca uses la `service_role` en ningún sitio de TUNORA.*
5. Crea tu cuenta (desde la app o en *Authentication → Users*) y hazla administradora:
   ```sql
   update public.users set is_admin = true where email = 'tu@email.com';
   ```
6. **Storage → Settings**: sube *Global file size limit* (el plan gratuito limita a 50 MB; los FLAC/WAV pueden superarlo).

**Seguridad:** el bucket `audio` es privado (la app usa URLs firmadas de 1 h); solo los administradores escriben en catálogo y buckets; cada usuario solo ve sus playlists, favoritos, historial y ajustes; `is_admin` no se puede modificar desde la app. ADMIN inicia sesión con tu cuenta de administrador y usa solo la anon key, así que **no hace falta service_role**.

## 2. ADMIN (Python 3.10+)

```bash
cd admin
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env                                   # rellena URL, anon key y tu email
python app.py
```
Pestañas: **Conexión** (login admin) → **Importar** (equipo / Drive / Dropbox, archivos o carpetas, progreso, duplicados) → **Biblioteca** (editar metadatos y portada, reemplazar audio, eliminar).

- Lee etiquetas y portadas incrustadas (MP3, FLAC, WAV, M4A/ALAC, AAC, OGG/Opus…). Las portadas se reducen a 1000 px; **el audio nunca se convierte**.
- Duplicados: por hash SHA-256 del archivo y por título + artista + duración.
- Tras importar, el PC puede apagarse: la música vive en Supabase Storage.

### Google Drive (OAuth)
1. <https://console.cloud.google.com> → crea proyecto → **APIs y servicios → Biblioteca → Google Drive API → Habilitar**.
2. **Pantalla de consentimiento OAuth**: tipo *Externo*, añade tu correo como *usuario de prueba*.
3. **Credenciales → Crear → ID de cliente OAuth → Aplicación de escritorio**. Descarga el JSON como `admin/client_secret.json`.
4. En ADMIN elige *Google Drive*: se abre el navegador, autorizas (solo lectura) y el token queda en `~/.tunora_admin/`.

### Dropbox (OAuth)
1. <https://www.dropbox.com/developers/apps> → **Create app** → *Scoped access* → *Full Dropbox*.
2. Pestaña **Permissions**: marca `files.metadata.read` y `files.content.read` → *Submit*.
3. Copia la **App key** a `DROPBOX_APP_KEY` (o a la pestaña Conexión). En ADMIN elige *Dropbox*, abre el enlace, autoriza y pega el código.

## 3. App web (PC, tablet, móvil)

```bash
cd app
cp .env.example .env        # URL y anon key (solo claves públicas)
npm install
npm run dev                 # desarrollo en http://localhost:5173
npm run build               # genera app/dist (hosting estático: Netlify, Vercel, GitHub Pages, Cloudflare Pages…)
```
Es una PWA: en Chrome/Edge puedes “Instalar TUNORA”. Atajos en PC: **Espacio** play/pausa · **←/→** ±5 s · **Shift+←/→** anterior/siguiente · **M** silenciar · **/** buscar.

## 4. APK de Android

### Opción A · GitHub Actions (recomendada, no necesitas instalar nada)
1. Sube la carpeta `tunora` a un repositorio de GitHub (privado está bien).
2. **Settings → Secrets and variables → Actions → New repository secret**:
   `VITE_SUPABASE_URL` y `VITE_SUPABASE_ANON_KEY`.
3. Pestaña **Actions → Build Android APK → Run workflow** (también se lanza solo con cada `push`).
4. Al terminar (~6 min), en la ejecución descarga el artefacto **tunora-apk** → `app-debug.apk`.
5. Cópialo al móvil e instálalo (permite “instalar apps desconocidas”).

**Firma estable (para actualizar sin desinstalar):** sin esto, cada compilación usa una clave temporal distinta y Android rechaza actualizar encima. Genera la clave una vez y guárdala como secreto:
```bash
keytool -genkeypair -v -keystore debug.keystore -alias androiddebugkey -storepass android -keypass android \
  -keyalg RSA -keysize 2048 -validity 10000 -dname "CN=TUNORA"
base64 -w0 debug.keystore      # macOS: base64 -i debug.keystore
```
Pega el resultado en el secreto `ANDROID_DEBUG_KEYSTORE_BASE64`. **Guarda `debug.keystore` en lugar seguro.**

### Opción B · En tu PC
Requisitos: Node 20+, JDK 17 y Android SDK (Android Studio).
```bash
cd app && npm install && npm run build
npx cap add android
python scripts/patch_android.py          # permisos + audio en segundo plano
npx cap sync android
cd android && ./gradlew assembleDebug    # Windows: gradlew.bat assembleDebug
# → android/app/build/outputs/apk/debug/app-debug.apk
```

### Nuevas versiones
Cambia el código y haz `push` (Actions sube el número de versión solo), o repite la Opción B. Con la misma clave de firma, el APK nuevo se instala encima del anterior.

### Publicar en Play Store (opcional)
Genera un **AAB firmado con tu propia clave de subida**: `./gradlew bundleRelease` y configura `signingConfigs` en `android/app/build.gradle`.

## 5. Cómo está resuelto cada requisito clave

| Requisito | Solución |
|---|---|
| Audio en segundo plano, notificación y pantalla de bloqueo | `@jofr/capacitor-media-session` (servicio en primer plano + Media Session) y `patch_android.py` |
| Auriculares / botones multimedia | Acciones play, pausa, anterior, siguiente y seek registradas en el plugin |
| Calidad sin conversión | `<audio>` reproduce el archivo original por URL firmada con Range (streaming) |
| Recuperar última canción | Cola y posición en `localStorage` y en `user_settings` |
| Offline seguro | Android: `Filesystem.downloadFile` en el almacenamiento privado de la app. Web: Cache Storage |
| Rendimiento | Solo metadatos al abrir; audio por streaming; listas por tramos + `content-visibility`; animaciones solo con `transform`/`opacity` y detenidas en pausa; portadas diferidas y reducidas |
| Tres layouts | Móvil: barra inferior + mini reproductor · Tablet: carril lateral + maestro/detalle · PC: barra lateral + cola + reproductor inferior |

**Limitaciones que debes conocer**
- La licencia del plugin multimedia es **GPL-3.0**: sin problema para uso personal; si distribuyes la app públicamente, revisa sus implicaciones.
- El ecualizador usa Web Audio: en algunos Android puede pausar el audio con la pantalla apagada (por eso está desactivado por defecto).
- «Calidad de reproducción» no transcodifica (el servidor no convierte audio): elige *Original* o *Ahorro de datos* (sin precarga y priorizando descargas).
- Las ondas de sonido son animadas, no un análisis espectral real (ahorra batería).
- Sin iconos propios en Android: se usa el de Capacitor. Para el tuyo: `npm i -D @capacitor/assets` y `npx capacitor-assets generate --android` con `assets/icon-only.png`.

## 6. Lista de comprobación (en tu entorno)

- [ ] `schema.sql` ejecutado sin errores y tu usuario con `is_admin = true`.
- [ ] ADMIN conecta y muestra “Conectado como … (admin)”.
- [ ] Importas 1 MP3 y 1 FLAC: aparecen en la pestaña Biblioteca con portada y calidad.
- [ ] Reimportar el mismo archivo lo marca como duplicado.
- [ ] La app (web) inicia sesión, lista y reproduce; la barra de progreso salta.
- [ ] Desde otra cuenta no admin, intentar escribir en `songs` falla (RLS).
- [ ] En el móvil: bloquear pantalla → la música sigue y hay controles; cambiar de canción desde la notificación.
- [ ] Descargar una canción, poner modo avión y reproducirla.
