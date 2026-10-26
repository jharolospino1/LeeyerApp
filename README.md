# LeeyerApp

Aplicación para estudiar Derecho en Colombia. Incluye materias, biblioteca, casos prácticos, notas, foro, estadísticas y **Leeyer Pro**, la suscripción para prepararse para el **Examen de Estado de idoneidad profesional** (simulacros, diagnóstico, plan de estudio, LeeyerIA Tutor y más).

**Descargar la última versión (Android):**
<https://github.com/jharolospino1/LeeyerApp/releases/latest/download/LeeyerApp.apk>

---

## Contenido

1. [Qué incluye](#qué-incluye)
2. [Cómo está hecha](#cómo-está-hecha)
3. [Archivos del repositorio](#archivos-del-repositorio)
4. [Configuración inicial](#configuración-inicial)
5. [Compilar el APK](#compilar-el-apk)
6. [Publicar una actualización](#publicar-una-actualización)
7. [Inicio de sesión, huella y bloqueo](#inicio-de-sesión-huella-y-bloqueo)
8. [Botón atrás del celular](#botón-atrás-del-celular)
9. [Pagos de Leeyer Pro (Wompi)](#pagos-de-leeyer-pro-wompi)
10. [Solución de problemas](#solución-de-problemas)
11. [Seguridad](#seguridad)

---

## Qué incluye

**Para todos los usuarios**

- Materias, biblioteca digital, casos prácticos, simulacros básicos, notas, foro, favoritos y perfil.
- Barra inferior: Inicio, Materias, LeeyerIA, Favoritos y Perfil.
- Cuenta con correo y contraseña, y opción de entrar con Google, Apple o teléfono (cuando están activados en Supabase).
- Aviso de actualización dentro de la app.

**Leeyer Pro (según el plan del usuario)**

- Simulacros premium con preguntas actualizadas y análisis detallado.
- Diagnóstico inicial, plan de estudio, seguimiento y estadísticas.
- LeeyerIA Tutor y biblioteca premium.
- Sección «Mi acceso y mis pagos».
- Los planes se activan desde el servidor: el usuario nunca puede darse un plan a sí mismo desde la app.

**Seguridad de la sesión**

- Si el usuario pasa **más de 5 minutos fuera de la app**, debe volver a iniciar sesión.
- Puede registrar su **huella digital** para entrar con un toque.

**Panel de administración (solo administradores)**

- Usuarios, activación y retiro de planes, simulacros y preguntas, importación de preguntas, estadísticas y pagos.
- Ajustes para activar o desactivar los pagos en línea y elegir el entorno (pruebas o producción).

---

## Cómo está hecha

| Pieza | Qué es |
| --- | --- |
| **App** | Un solo archivo, `index.html` (HTML, CSS y JavaScript sin necesidad de compilar). |
| **Backend** | [Supabase](https://supabase.com): autenticación, base de datos (funciones `pro_*`), almacenamiento y Edge Functions. |
| **App Android** | [Capacitor 7](https://capacitorjs.com) envuelve `index.html` en un APK. |
| **Compilación** | GitHub Actions (`.github/workflows/build.yml`). No hace falta Android Studio. |
| **Pagos** | [Wompi](https://wompi.com), a través de una Edge Function de Supabase. |

Plugins nativos que incluye el APK:

- `@capacitor/app`: botón atrás del celular.
- `@capacitor/local-notifications` y `@capacitor/push-notifications`: avisos.
- `@capgo/capacitor-native-biometric`: lector de huella del teléfono.

---

## Archivos del repositorio

```
.
├── index.html                  ← toda la aplicación
├── icon-only.png               ← ícono de la app
├── icon-foreground.png         ← primer plano del ícono adaptable de Android
├── README.md                   ← este archivo
└── .github/
    └── workflows/
        └── build.yml           ← compila y publica el APK
```

La página de descarga y el archivo de versiones viven en **otro repositorio** (`leeyerapp-web`, publicado con GitHub Pages):

```
leeyerapp-web/
├── index.html                  ← página de descarga
└── version.json                ← versión publicada (avisos de actualización)
```

---

## Configuración inicial

### 1. Conexión con Supabase

Dentro de `index.html`, la app lee `window.LEEYER_CONFIG`:

```js
window.LEEYER_CONFIG = {
  supabaseUrl: "https://TU-PROYECTO.supabase.co",
  supabaseAnonKey: "TU_CLAVE_PUBLICABLE",   // clave pública (anon / publishable)
  requireLogin: true                         // obliga a iniciar sesión
};
```

Usa solo la clave **pública**. Nunca pongas la clave `service_role` en `index.html`.

### 2. Funciones del servidor (Edge Functions)

La app llama a funciones de Supabase por nombre. Los nombres están en `window.LEEYER_PRO_FN` (al inicio del bloque `leeyer-pro` de `index.html`):

```js
window.LEEYER_PRO_FN = { checkout: "swift-responder", tutor: "smart-api" };   // nombres de tus funciones en Supabase
```

- `checkout`: crea el cobro con Wompi y devuelve `checkout_url` y `reference`.
- `tutor`: LeeyerIA Tutor.

Si cambias el nombre de una función en Supabase, cámbialo aquí también.

### 3. Inicio de sesión con Google, Apple y teléfono

Se activan en **Supabase → Authentication → Providers**. Mientras un proveedor esté apagado, su botón aparece atenuado y la app avisa que todavía no está disponible.

**Google**

1. En Google Cloud → *Google Auth Platform* → *Clients* crea un cliente de tipo **Web application**.
2. *Authorized JavaScript origins*: solo el dominio (sin ruta).
3. *Authorized redirect URI*: `https://TU-PROYECTO.supabase.co/auth/v1/callback`.
4. Copia el *Client ID* y el *Client Secret* en Supabase → Providers → Google.
5. En Supabase → *URL Configuration* → *Redirect URLs* agrega las direcciones desde donde se abre la app.

> **Importante:** Google **bloquea** el inicio de sesión dentro de un WebView, que es lo que es el APK de Capacitor. Con el APK se puede entrar con correo y contraseña o con huella. Google funciona al abrir la app en Chrome. Para tenerlo dentro del APK haría falta un plugin nativo de Google y configurar Firebase.

**Apple**: requiere una cuenta de Apple Developer (de pago).

**Teléfono**: requiere un proveedor de SMS configurado en Supabase (por ejemplo Twilio, de pago).

---

## Compilar el APK

El APK se genera solo con GitHub Actions.

### Una sola vez: secretos del repositorio

En **Settings → Secrets and variables → Actions → New repository secret**:

| Secreto | Obligatorio | Para qué sirve |
| --- | --- | --- |
| `KEYSTORE_BASE64` | Sí | Tu archivo de firma (`.jks`) convertido a base64. El alias de la llave debe ser `leeyerapp`. |
| `KEYSTORE_PASSWORD` | Sí | Contraseña del keystore y de la llave. |
| `GOOGLE_SERVICES_JSON` | No | Contenido de `google-services.json` de Firebase. Sin él, el APK se compila **sin avisos push**. |

> Guarda una copia segura de tu keystore. Si lo pierdes no podrás publicar actualizaciones que se instalen encima de la versión anterior.

### Cada vez que quieras una versión nueva

1. Sube los cambios a este repositorio (por ejemplo, el `index.html` nuevo).
2. Ve a **Actions → Compilar LeeyerApp (APK) → Run workflow**.
3. Escribe la versión, con este formato: `3.1.3`.
4. Espera a que termine (unos minutos).
5. El APK queda en **Releases** y siempre se puede bajar desde el enlace fijo:

   <https://github.com/jharolospino1/LeeyerApp/releases/latest/download/LeeyerApp.apk>

El workflow hace todo esto por ti: instala Capacitor y los plugins, mete dentro del APK las librerías y la fuente (la app funciona sin depender de CDN), crea el proyecto Android, genera los íconos, agrega los permisos (notificaciones y huella), pone la versión, compila, alinea, firma y publica.

Datos útiles:

- Identificador de la app: `com.leeyerapp.app`.
- `versionName` = la versión que escribes; `versionCode` = el número de ejecución del workflow (siempre crece, por eso cada APK se instala sobre el anterior).
- El workflow falla con un mensaje claro si falta `index.html`, los íconos o los secretos obligatorios.

---

## Publicar una actualización

La app revisa `version.json` y, si la versión publicada es **mayor** que la instalada, muestra el aviso «¡Nueva versión disponible!».

**Pasos**

1. En `index.html`, dentro del bloque `leeyer-update`, sube `version` al número nuevo (por ejemplo `"3.1.3"`).
2. Compila el APK con ese mismo número (sección anterior).
3. En el repositorio `leeyerapp-web`, edita `version.json` con la versión nueva y súbelo.

**Ejemplo de `version.json`**

```json
{
  "version": "3.1.3",
  "mandatory": false,
  "downloadUrl": "https://jharolospino1.github.io/leeyerapp-web/",
  "title": "¡Nueva versión disponible!",
  "message": "Resumen corto de lo que cambia.",
  "notesTitle": "¿Qué hay de nuevo en la 3.1.3?",
  "notes": ["Primera mejora.", "Segunda mejora."],
  "title_en": "New version available!",
  "message_en": "Short summary of what changes.",
  "notesTitle_en": "What's new in 3.1.3?",
  "notes_en": ["First improvement.", "Second improvement."]
}
```

**Reglas**

- `version` debe ser mayor que la instalada; si es igual o menor no aparece nada.
- `mandatory: true` impide cerrar el aviso (úsalo solo si la versión vieja ya no sirve).
- `downloadUrl` es la página a la que lleva el botón del aviso.
- Límites de texto: título hasta 80 caracteres, mensaje hasta 300, hasta 6 notas de 140 caracteres cada una. Lo que pase de ahí se recorta.
- Si no hay internet, simplemente no se muestra el aviso.

---

## Inicio de sesión, huella y bloqueo

**Bloqueo a los 5 minutos.** La app guarda la hora a la que salió. Al volver, si pasaron más de 5 minutos, muestra la pantalla de inicio de sesión. El tiempo se cambia en la constante `TH` del módulo de autenticación dentro de `index.html`.

**Huella digital.** Después de iniciar sesión la app ofrece activar la huella. En el APK usa el lector nativo del teléfono (también acepta el PIN o patrón de pantalla si el teléfono no tiene huella). En un navegador usa WebAuthn, que necesita una dirección `https`.

La huella queda guardada solo en ese teléfono y solo sirve para desbloquear la sesión que ya existe; no reemplaza la contraseña de la cuenta. Se puede desactivar desde **Perfil**.

**Si dice «Huella no disponible»:**

| Mensaje | Causa | Solución |
| --- | --- | --- |
| «Este APK todavía no incluye el lector de huella» | El APK se compiló con un `build.yml` sin el plugin. | Usa el `build.yml` actual y compila de nuevo. |
| «Este dispositivo no tiene huella ni bloqueo de pantalla» | El teléfono no tiene huella registrada ni PIN. | Actívalo en Ajustes del teléfono. |
| «…desde un archivo (Descargas)» | Abriste el `index.html` suelto. | Instala el APK o abre la app por su dirección `https`. |

---

## Botón atrás del celular

El botón atrás hace lo mismo que el botón «Volver» de la pantalla: cierra el menú, las hojas y LeeyerIA, y luego retrocede por la navegación. Solo en la pantalla de inicio pide tocar **dos veces** para salir.

Dentro del APK depende del plugin `@capacitor/app`, que el workflow ya instala. Si compilas con un `build.yml` antiguo, el botón atrás puede cerrar la app.

---

## Pagos de Leeyer Pro (Wompi)

Cómo funciona el flujo:

1. El usuario elige un plan y pulsa **Ir a pagar**.
2. La app llama a la Edge Function de pagos (`checkout`) con el `plan_id`.
3. La función devuelve la dirección de pago de Wompi (`checkout_url`) y una `reference`.
4. La app abre la pasarela y consulta el estado del pago cada pocos segundos.
5. **El plan se activa solo cuando el servidor confirma el pago con Wompi.** Lo que muestre la pantalla no activa nada.

Para activar los pagos en producción:

1. Despliega en Supabase las funciones de pagos con tus llaves de Wompi guardadas como *secrets* del servidor (nunca en `index.html`).
2. Entra a la app como administrador → **Administrar Leeyer Pro → Ajustes**.
3. Elige el entorno (pruebas o producción) y activa **Pagos en línea**.

Mientras los pagos estén desactivados, la pantalla de compra avisa que todavía no se reciben pagos y no cobra nada. En el entorno de pruebas se muestra la etiqueta «ENTORNO DE PRUEBA».

También puedes activar un plan a mano desde el panel de administración, con el botón **Activar plan** del usuario.

---

## Solución de problemas

| Problema | Qué revisar |
| --- | --- |
| El workflow falla en «Verificar archivos y claves» | Faltan `index.html`, `icon-only.png`, `icon-foreground.png` o los secretos `KEYSTORE_BASE64` / `KEYSTORE_PASSWORD`. |
| «La versión debe verse así: 1.0.0» | Escribe solo números separados por puntos, por ejemplo `3.1.3`. |
| El APK no se instala encima del anterior | Se firmó con otro keystore. Hay que usar siempre el mismo. |
| No llegan avisos push | Falta el secreto `GOOGLE_SERVICES_JSON`, o la app Android en Firebase no se llama `com.leeyerapp.app`. |
| «Este proveedor no está activado» (Google/Apple/teléfono) | Actívalo en Supabase → Authentication → Providers. |
| Google no abre dentro del APK | Es una restricción de Google para WebViews. Usa correo o huella, o abre la app en Chrome. |
| «El servicio de pagos aún no está desplegado» | Falta desplegar la Edge Function de pagos o el nombre en `LEEYER_PRO_FN` no coincide. |
| Se cierra la app al dar atrás | Compila con el `build.yml` actual (incluye `@capacitor/app`). |
| No aparece el aviso de actualización | La versión de `version.json` no es mayor que la instalada, o no hay internet. |
| El aviso descarga el APK en vez de abrir la página | Revisa `downloadUrl` en `version.json`: debe ser la dirección de la página de descarga. |

---

## Seguridad

- **Nunca subas** el archivo `.jks`, contraseñas ni `service_role` al repositorio. Los secretos van solo en *Settings → Secrets*.
- La clave `anon` / *publishable* de Supabase sí puede estar en `index.html`: es pública por diseño. La protección real son las reglas de acceso (RLS) y las funciones de la base de datos.
- Las llaves privadas de Wompi viven solo en el servidor (secretos de Supabase).
- El administrador se valida en el servidor; los botones ocultos en la app no son una protección.

---

Hecho por **Jharol Ospino** · LeeyerApp
