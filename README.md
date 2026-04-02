# GuardianApp — Control Parental para Android (Kotlin Nativo)

Aplicación nativa Android escrita en Kotlin que actúa como agente de monitoreo en el dispositivo del menor. Requiere Android 8.0 (API 26) o superior.

---

## Estructura del Proyecto

```
guardian-android/
├── app/
│   ├── build.gradle.kts                  # Dependencias y configuración del módulo
│   └── src/main/
│       ├── AndroidManifest.xml           # Permisos y componentes declarados
│       ├── java/com/guardianapp/
│       │   ├── GuardianApplication.kt    # Canales de notificación
│       │   ├── data/
│       │   │   ├── api/
│       │   │   │   ├── Base44ApiService.kt   # Interfaz Retrofit (todos los endpoints)
│       │   │   │   ├── RetrofitClient.kt     # OkHttp + token Bearer
│       │   │   │   └── models/ApiModels.kt   # Request/Response data classes
│       │   │   ├── db/
│       │   │   │   ├── AppDatabase.kt        # Room database
│       │   │   │   ├── OfflineEventDao.kt    # Queries CRUD de la cola offline
│       │   │   │   └── OfflineEventEntity.kt # Tabla de eventos pendientes
│       │   │   └── prefs/
│       │   │       └── SecurePreferences.kt  # EncryptedSharedPreferences
│       │   ├── service/
│       │   │   ├── GuardianForegroundService.kt  # Foreground Service principal
│       │   │   ├── modules/
│       │   │   │   ├── GpsModule.kt          # GPS cada 2 min / 10 seg emergencia
│       │   │   │   ├── BatteryModule.kt      # Batería cada 5 min + alerta < 20%
│       │   │   │   ├── ScreenCaptureModule.kt# MediaProjection cada 10 min
│       │   │   │   └── CommandsModule.kt     # Polling de comandos cada 30 seg
│       │   │   └── sync/
│       │   │       └── OfflineSyncWorker.kt  # WorkManager: reenvío offline c/15 min
│       │   ├── receiver/
│       │   │   └── BootReceiver.kt           # BOOT_COMPLETED → inicia el servicio
│       │   └── ui/
│       │       ├── qr/QrScanActivity.kt      # Primera ejecución: escaneo QR
│       │       └── consent/ConsentActivity.kt # Consentimiento informado
│       └── res/
│           ├── layout/                       # activity_qr_scan.xml, activity_consent.xml
│           ├── values/                       # strings, colors, themes
│           └── drawable/                     # ic_notification.xml
├── build.gradle.kts                          # Root build file (plugin versions)
├── settings.gradle.kts                       # Módulos del proyecto
├── gradle/wrapper/gradle-wrapper.properties  # Gradle 8.4
└── local.properties.template                 # Plantilla de sdk.dir
```

---

## Requisitos previos

1. **Java 17** (o superior) instalado en tu sistema
2. **Android SDK** con:
   - Build Tools 34.0.0
   - Platform API 34
   - Google Play Services
3. **Android Studio Hedgehog** (o cualquier versión ≥ 2023.1) — opcional, puedes compilar desde terminal

---

## Configuración inicial

### 1. Clonar y configurar `local.properties`

```bash
cd guardian-android
cp local.properties.template local.properties
# Edita local.properties y pon la ruta correcta de tu SDK:
# sdk.dir=/home/TU_USUARIO/Android/Sdk
```

### 2. Variables de entorno (BASE44)

Las credenciales están en `app/build.gradle.kts` como `buildConfigField`:

```kotlin
buildConfigField("String", "BASE44_API_URL", "\"https://safe-guardian-sync.base44.app\"")
buildConfigField("String", "BASE44_API_KEY", "\"TU_API_KEY_REAL\"")
```

Cambia `"TU_API_KEY_REAL"` por tu clave antes de compilar, o usa Gradle properties:

```bash
# Compila pasando la clave como propiedad:
./gradlew assembleDebug -PBASE44_API_KEY="mi-clave-real"
```

---

## Compilar el APK

### Debug (recomendado para pruebas)

```bash
cd guardian-android

# Dar permisos al wrapper (Linux/Mac):
chmod +x gradlew

# Compilar APK debug:
./gradlew assembleDebug
```

El APK se genera en:
```
app/build/outputs/apk/debug/app-debug.apk
```

### Release (para distribución)

```bash
# 1. Crear un keystore (solo una vez):
keytool -genkey -v -keystore guardian-release.keystore \
  -alias guardian -keyalg RSA -keysize 2048 -validity 10000

# 2. Añadir en app/build.gradle.kts:
signingConfigs {
    create("release") {
        storeFile = file("../guardian-release.keystore")
        storePassword = "TU_STORE_PASSWORD"
        keyAlias = "guardian"
        keyPassword = "TU_KEY_PASSWORD"
    }
}

buildTypes {
    release {
        signingConfig = signingConfigs.getByName("release")
        isMinifyEnabled = true
        ...
    }
}

# 3. Compilar:
./gradlew assembleRelease
```

El APK firmado estará en:
```
app/build/outputs/apk/release/app-release.apk
```

---

## Instalar en el dispositivo

```bash
# Conecta el dispositivo con USB debug activado:
adb install app/build/outputs/apk/debug/app-debug.apk

# O directamente:
./gradlew installDebug
```

---

## Flujo de primera ejecución

```
[Abrir app] → QrScanActivity
    → Escanear QR (formato: {"childUserId":"..","familyId":".."} o "childId:familyId")
    → GET /users/{childUserId}  — valida usuario en Base44
    → Guarda childUserId, familyId, authToken en EncryptedSharedPreferences
    → ConsentActivity
        → Usuario llena nombre + checkbox
        → Solicita permisos: Location, Camera, Microphone, Storage
        → Solicita permiso MediaProjection (captura de pantalla)
        → POST /consent-records
        → Inicia GuardianForegroundService
        → La app pasa a segundo plano (finish())
```

---

## Módulos de monitoreo

| Módulo | Intervalo | Endpoint |
|--------|-----------|----------|
| GPS normal | 2 minutos | POST /locations |
| GPS emergencia | 10 segundos | POST /locations |
| Batería | 5 minutos | POST /battery-status |
| Alerta batería baja (<20%) | Automático | POST /alerts |
| Captura de pantalla | 10 minutos | POST /screen-captures |
| Polling comandos | 30 segundos | GET /commands |
| Audio (START_RECORDING) | Bajo demanda | POST /audio-files |
| Foto (TAKE_PHOTO) | Bajo demanda | POST /camera-captures |

---

## Comandos remotos disponibles

Enviar desde el panel de Base44 a través del endpoint `/commands`:

```json
{"type": "START_RECORDING", "params": {"durationSeconds": "60"}}
{"type": "STOP_RECORDING"}
{"type": "TAKE_PHOTO", "params": {"facing": "front"}}
{"type": "TAKE_PHOTO", "params": {"facing": "back"}}
{"type": "EMERGENCY_MODE"}
```

Cuando se activa cámara o micrófono, **siempre aparece una notificación visible** en el dispositivo del menor.

---

## Sincronización offline

- Si falla cualquier envío a Base44, el evento se guarda en Room (SQLite local).
- **WorkManager** ejecuta `OfflineSyncWorker` cada 15 minutos cuando hay conexión.
- Los eventos se reenvían en orden FIFO.
- Después de 10 reintentos fallidos, el evento se descarta.

---

## Notas importantes sobre permisos Android

### ACCESS_BACKGROUND_LOCATION (Android 10+)
- Requiere aprobación manual del usuario en Ajustes > Aplicaciones > GuardianApp > Permisos > Ubicación > **Permitir siempre**.

### MediaProjection (captura de pantalla)
- El usuario debe aprobar el diálogo del sistema en cada arranque del servicio (limitación de Android).
- Si el usuario cancela, la captura de pantalla queda desactivada (el resto de módulos sigue activo).

### Optimización de batería
- Para garantizar que el servicio no sea matado, ir a: Ajustes > Aplicaciones > GuardianApp > Batería > **Sin restricciones**.

---

## Dependencias principales

| Librería | Versión | Uso |
|----------|---------|-----|
| Retrofit | 2.9.0 | HTTP client |
| OkHttp | 4.12.0 | Interceptor Authorization |
| Room | 2.6.1 | Cola offline SQLite |
| WorkManager | 2.9.0 | Sync periódico |
| FusedLocationProvider | 21.2.0 | GPS |
| CameraX | 1.3.2 | Fotos sin preview |
| ZXing Android Embedded | 4.3.0 | Escaneo QR |
| EncryptedSharedPreferences | 1.1.0-alpha06 | Almacenamiento seguro |
| Kotlinx Coroutines | 1.7.3 | Async/background |

---

## Solución de problemas

**Gradle no encuentra el SDK:**
```
Error: SDK location not found
→ Asegúrate de que local.properties tiene sdk.dir correcto
```

**`permission denied` al ejecutar gradlew en Linux/Mac:**
```bash
chmod +x gradlew
```

**El servicio se mata en segundo plano:**
```
→ Desactivar optimización de batería para GuardianApp
→ En algunos fabricantes (Xiaomi, Huawei) hay pasos adicionales en AutoStart
```

**MediaProjection se desconecta al reiniciar:**
```
→ Android invalida el token de MediaProjection en cada reinicio
→ El usuario debe abrir la app una vez después de cada reinicio para conceder el permiso de pantalla
→ El resto de módulos (GPS, batería, comandos) sí arrancan automáticamente en el boot
```
