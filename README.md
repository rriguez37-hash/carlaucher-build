# Car Launcher (ZX DZ_01 · Android 11)

Launcher nativo (Kotlin, vistas Android clásicas, sin Compose) para pantalla multimedia de automóvil en horizontal:
mapa arriba, accesos rápidos a la derecha y reproductor integrado abajo.

> **Estado de verificación (léelo primero)**
> - El código **no se ha compilado ni ejecutado** todavía: el entorno donde se creó no tiene Android SDK ni red.
>   Se revisó leyendo el código contra las APIs de Android 11 y osmdroid 6.1.18 (revisión independiente incluida), pero **no hay APK incluido**.
> - **Nada se ha probado en tu unidad real.** La sección "Lista de verificación en la unidad" dice qué comprobar.

---

## 1. Qué es posible en este tipo de equipo (sin root, sin permisos de sistema)

Los datos que diste (Android 11.1, MCU `CVD1810-WJ_21.09.03_333`) corresponden a una unidad genérica con Android básico.
No se sabe aún si trae Google Play Services, Play Store ni qué apps propias del fabricante usa para radio y Bluetooth.
Por eso el proyecto incluye una pantalla de **Diagnóstico** que lo averigua.

### Funciones básicas (deberían funcionar en cualquier Android 11)

| Función | Cómo se resuelve |
|---|---|
| Ser la pantalla de inicio | Intent filter `HOME` + `DEFAULT`. El usuario lo elige en el selector de Android. |
| Botones grandes, tema oscuro, pantalla completa horizontal | Vistas nativas, `landscape` fijo, sin animaciones. |
| Accesos a navegación, teléfono, radio, Bluetooth y ajustes | Intents estándar + **elección manual de app** (mantener pulsado). |
| Acceso a YouTube Music | Abre `com.google.android.apps.youtube.music` si está instalada. |
| Botones anterior / reproducir-pausar / siguiente | Teclas multimedia del sistema (`AudioManager.dispatchMediaKeyEvent`). Funcionan sin permisos. |
| Lista de todas las apps y diagnóstico | `PackageManager` con `<queries>` (visibilidad de paquetes de Android 11). |

### Funciones opcionales (dependen del equipo o de un permiso del usuario)

| Función | Qué necesita | Limitación |
|---|---|---|
| Título, artista y carátula en vivo | El usuario concede **Acceso a notificaciones** al launcher (una vez). | Solo se ve lo que la app publique en su sesión de medios. Si la pantalla de ese permiso no existe en tu firmware, los botones siguen funcionando pero sin metadatos. |
| Mapa embebido con tu posición | GPS de la unidad + permiso de ubicación + internet. | Es **OpenStreetMap**, no Google Maps (ver abajo). |
| Música Bluetooth del teléfono | Que el sistema o la app Bluetooth del fabricante publique una sesión de medios. | Si el módulo Bluetooth lo gestiona el MCU/otra app sin sesión de medios, el launcher no verá título ni carátula; los botones pueden seguir funcionando vía teclas multimedia. Se debe comprobar en la unidad. |
| Radio | Elegir la app de radio del fabricante (mantener pulsado el botón Radio). | No existe una forma estándar de abrir "la radio"; hay que identificar su paquete (el Diagnóstico ayuda). |

### Lo que NO es posible (y qué se hace en su lugar)

- **Incrustar Google Maps real dentro del launcher.** Google no permite incrustar la app Maps. Su SDK (`MapView`) necesita Google Play Services en la unidad y una clave de API con facturación; muchas unidades genéricas no lo traen.
  - *Se hace:* mapa OpenStreetMap (sin clave, sin Play Services) y botón **"¿A dónde?"** / acceso Navegación que abre **Google Maps** a pantalla completa.
  - *Si el Diagnóstico muestra que hay Play Services*, el panel `MapPanel.kt` está aislado para sustituirlo por el SDK de Google más adelante.
- **Mostrar la guía de giros de Google Maps dentro del launcher.** Maps no expone esos datos a otras apps.
- **Poner Maps en pantalla dividida automáticamente.** Requiere APIs de sistema o soporte del firmware; no se asume.
- **Controlar el volumen del MCU, el ecualizador, la cámara trasera o los botones del volante.** Eso es del firmware del fabricante y se excluyó a propósito.

---

## 2. Decisiones técnicas

- **Kotlin + vistas nativas (`Activity`, sin AppCompat ni Compose).** Menos memoria, arranque más rápido y menos riesgo en unidades con poca RAM. Única dependencia externa: `osmdroid` 6.1.18.
- **minSdk 26, targetSdk 30 (Android 11), compileSdk 34.** Apuntar a 30 evita restricciones de versiones posteriores.
- **Sin servicios propios en segundo plano.** El listener de sesiones de medios y el GPS solo están activos mientras el launcher está visible (`onStart/onStop`, `onResume/onPause`).
- **Caché de mapa limitada a 100 MB**, carátula reducida a máx. 384 px y cacheada.
- **Seguridad:** sin `allowBackup`, sin tráfico HTTP en claro (los mosaicos OSM van por HTTPS), `DiagnosticsActivity` no exportada, permisos mínimos (`INTERNET`, `ACCESS_NETWORK_STATE`, `ACCESS_FINE_LOCATION`).
- **El servicio de notificaciones está vacío** (`MediaListenerService`): no lee ni guarda notificaciones; solo es la "identidad" que Android exige para leer sesiones de medios.
- **Estabilidad:** si el mapa falla al crearse, el resto del launcher sigue funcionando. El botón Atrás no cierra el launcher.

---

## 3. Compilar

### Opción A — Android Studio (recomendada)
1. Instala Android Studio (versión reciente) con JDK 17 (incluido).
2. `File > Open` y elige la carpeta `CarLauncher`. Deja que sincronice Gradle (necesita internet).
3. Instala el SDK Platform 34 si lo pide.
4. `Build > Build APK(s)`. Resultado: `app/build/outputs/apk/debug/app-debug.apk`.

### Opción B — Línea de comandos
```bash
# Requiere JDK 17+ y Android SDK (ANDROID_HOME configurado, plataforma 34)
./gradlew :app:assembleDebug      # APK de prueba
./gradlew :app:assembleRelease    # APK reducido (firmado con clave de depuración, solo para uso personal)
```

### Opción C — Sin instalar nada (GitHub Actions)
1. Sube la carpeta a un repositorio de GitHub.
2. Pestaña **Actions > Build APK > Run workflow**.
3. Descarga el artefacto `CarLauncher-apk`.

---

## 4. Instalar en la unidad (con cuidado)

1. Copia el APK a un USB o instálalo con `adb install app-debug.apk` si la unidad lo permite. Habilita "Orígenes desconocidos" si lo pide.
2. Abre **Car Launcher** desde la lista de apps (aparece como app normal, **sin fijarlo todavía**).
3. Revisa que todo se ve bien en tu pantalla.
4. En el menú `⋮` pulsa **"Usar como pantalla de inicio"**, o pulsa el botón Home físico y elige el launcher con **"Solo una vez"** primero.
5. Cuando estés conforme, elige **"Siempre"**.

**Recuperación si algo sale mal:** Ajustes > Aplicaciones > Car Launcher > "Borrar valores predeterminados" (o desinstálalo). Desde el propio launcher siempre tienes el botón **Ajustes** y `⋮ > Todas las aplicaciones`. **No quites el launcher original del fabricante.**

### Permisos a conceder
- **Ubicación** (se pide una sola vez al abrir; si lo rechazas, usa `⋮ > Permiso de ubicación`).
- **Acceso a notificaciones** (`⋮ > Acceso a notificaciones`, o toca el área del reproductor): activa "Car Launcher". Sin esto, solo funcionan los botones por teclas multimedia.

---

## 5. Uso

- **Tocar** un acceso lo abre. **Mantener pulsado** abre la lista de apps para reasignarlo (el nombre de la app elegida aparece bajo el icono). "Predeterminado" lo restaura.
- **Botón Música (izquierda del reproductor):** abre YouTube Music (o la app que elijas con pulsación larga).
- **Zona de título/artista:** abre la app que está reproduciendo.
- **`⋮` arriba a la derecha:** todas las apps, usar como inicio, permisos, diagnóstico, restablecer accesos.

---

## 6. Lista de verificación en la unidad (nada de esto está probado)

1. **Diagnóstico** (`⋮ > Diagnóstico del equipo`): ¿hay Google Play Services? ¿Maps y YouTube Music instalados y visibles? Copia el informe: sirve para decidir los siguientes pasos.
2. ¿La resolución y el tamaño de los botones son cómodos? (se diseñó en dp; pueden requerir ajuste en `dimens.xml`).
3. ¿El GPS de la unidad da posición al mapa? ¿Hay internet (hotspot del teléfono) para los mosaicos?
4. Reproduce algo por **Bluetooth desde el teléfono** y por **YouTube Music**, pulsa "Actualizar" en Diagnóstico: ¿aparece una sesión de medios? ¿Cambian título/carátula y responden anterior/play/siguiente?
5. ¿Existe la pantalla "Acceso a notificaciones" en los ajustes de este firmware?
6. ¿Funciona "Usar como pantalla de inicio" y el botón Home físico? Algunas unidades lo capturan el MCU/firmware.
7. Identifica el paquete de la **radio** y de la app **Bluetooth** del fabricante (Diagnóstico > "Posibles apps del equipo") y asígnalos.
8. Con el vehículo estacionado, revisa consumo de RAM tras un rato de uso.

---

## 7. Estructura

```
app/src/main/
├── AndroidManifest.xml               HOME launcher, permisos, <queries>
├── java/com/carlauncher/app/
│   ├── MainActivity.kt               pantalla principal y orquestación
│   ├── map/MapPanel.kt               mapa OSM oscuro + ubicación (aislado, sustituible)
│   ├── media/MediaBridge.kt          sesiones de medios + teclas multimedia
│   ├── media/MediaListenerService.kt servicio vacío (identidad para el permiso)
│   ├── shortcuts/Shortcuts.kt        accesos configurables (SharedPreferences)
│   └── ui/AppPicker.kt, DiagnosticsActivity.kt
└── res/                              layout, colores, vectores
```

## 8. Personalización rápida
- **Colores:** `res/values/colors.xml` (acento azul `#2F7BFF`).
- **Tamaños de barras y botones:** `res/values/dimens.xml` y `res/layout/activity_main.xml`.
- **Centro inicial del mapa:** constantes `INITIAL_LAT/INITIAL_LON` en `MapPanel.kt` (solo se usa hasta tener GPS).
- **Servidor de mosaicos:** el predeterminado es el público de OSM, pensado para uso ligero. Para uso diario intenso, usa un proveedor propio o de pago con `TileSourceFactory`/`XYTileSource` en `MapPanel.kt` (respeta su política de uso y atribución).

## 9. Licencias y atribución
- Mapa: © OpenStreetMap contributors (se muestra en pantalla).
- Librería: osmdroid (Apache 2.0).
- Iconos: trazados vectoriales estilo Material Design (Apache 2.0).
