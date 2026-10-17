# MotoGastos

App Flutter que usa la cámara del celular para reconocer una motocicleta
(marca, línea, cilindraje aproximado) con IA, y a partir de eso estima el
gasto anual de tenerla en Colombia: SOAT, revisión tecnomecánica, impuesto
vehicular, gasolina y mantenimiento. Todos los valores quedan editables
dentro de la app.

Este código corresponde al flujo ya validado en el prototipo de diseño
(Bienvenida → Cámara → Analizando → Resultado → Gastos), con la misma
paleta y tipografía (Space Grotesk + IBM Plex Sans).

## 1. Requisitos

- Flutter estable reciente (`flutter --version`), con soporte para Android
  y/o iOS.
- Un celular físico para probar la cámara (el emulador/simulador no tiene
  cámara real; puedes usar "Elegir foto de la galería" para probar el
  resto del flujo desde un emulador).

## 2. Generar el proyecto nativo (Android/iOS)

Este repositorio trae el código Dart (`lib/` y `pubspec.yaml`), pero no
versiona las carpetas nativas. Desde la raíz del repositorio, genera Android
en este mismo proyecto y descarga las dependencias:

```bash
flutter create --platforms=android .
flutter pub get
```

En `android/app/build.gradle.kts`, fija `compileSdk = 37`: la versión actual
de `permission_handler_android` requiere Android SDK 37. Para compilar iOS,
en una Mac ejecuta también `flutter create --platforms=ios .`.

Si Gradle/Kotlin falla en Windows con `this and base files have different
roots` y muestra el proyecto y la caché de Pub en unidades distintas, agrega
`kotlin.incremental=false` a `android/gradle.properties` y vuelve a ejecutar
`flutter clean` y `flutter run`. Kotlin documenta esta opción para desactivar
la compilación incremental mientras se evita ese problema de caché.

## 3. Permisos de cámara y galería

**Android** — en `android/app/src/main/AndroidManifest.xml`, dentro de
`<manifest>` (antes de `<application>`):

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.INTERNET" />
<uses-feature android:name="android.hardware.camera" android:required="false" />
```

**iOS** — en `ios/Runner/Info.plist`, dentro de `<dict>`:

```xml
<key>NSCameraUsageDescription</key>
<string>MotoGastos usa la cámara para reconocer tu moto.</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>MotoGastos necesita acceso a tus fotos para elegir una imagen de tu moto.</string>
```

(Revisa también las instrucciones de instalación de los paquetes `camera`
e `image_picker` en pub.dev por si la versión que instales pide algo
adicional — cambian de vez en cuando.)

## 4. Configurar el reconocimiento por IA

El servicio predeterminado usa Gemini con entrada de imagen y salida JSON.
El reconocimiento previo con Anthropic sigue disponible como alternativa.

1. Copia `env.json.example` a `env.json` (el archivo local está en
   `.gitignore` y no debe subirse al repositorio).
2. Configura `GEMINI_API_KEY` y el nombre del modelo:

   ```json
   {
     "GEMINI_API_KEY": "tu-api-key",
     "GEMINI_MODEL": "gemini-3.8-flash"
   }
   ```

   `gemini-1.5-flash` fue apagado por Google el 29 de septiembre de 2025;
   por eso `GEMINI_MODEL` permite escoger un modelo disponible. El código
   usa `gemini-3.8-flash` por defecto.

3. Corre o compila así:

   ```bash
   flutter run --dart-define-from-file=env.json
   ```

Gemini tiene prioridad cuando su API key está configurada; si no, la app
prueba Anthropic y finalmente usa `MockRecognitionService` para mantener
el flujo disponible en modo demostración.

### Para producción real

No publiques una API key dentro de la app móvil: `--dart-define` evita
subirla al repositorio, pero no evita que se extraiga del binario. Como
el proyecto ya usa Django, el siguiente paso recomendado es enviar la foto
a un endpoint autenticado del backend y llamar Gemini desde el servidor.
`GeminiMotoAiService` implementa `MotorcycleRecognitionService` y puede
reemplazarse por un cliente HTTP del backend sin cambiar las pantallas.
El paquete `google_generative_ai` se incluyó porque lo requiere esta
integración; Google recomienda migrar sus SDK cliente de Google AI a
Firebase AI Logic para nuevos proyectos Flutter.

## 5. Tarifas de Colombia usadas (2026)

En `lib/data/colombia_rates_2026.dart`, con las fuentes en los
comentarios del archivo. Resumen:

- **SOAT**: por rango de cilindraje (≤50, ≤100, 101–200, >200 cc).
- **Tecnomecánica**: valor promedio del rango oficial 2026.
- **Impuesto vehicular**: avalúo estimado × 1,5 % para motos de más de
  125 cc (exentas las de 125 cc o menos).
- **Gasolina**: se calcula como `(km al año ÷ rendimiento en km/galón) ×
  precio del galón`, con valores de referencia nacionales.
- **Mantenimiento**: estimado anual de aceite, frenos y ajustes.

Son valores de referencia, no una cotización oficial — la app deja
editar cada uno, y hay que revisarlos cada año porque cambian con
decretos, la UVT y el mercado.

## 6. Estructura del código

```
lib/
  main.dart               Punto de entrada
  app.dart                MaterialApp + Provider
  theme/                  Colores y tipografía
  models/                 Motorcycle, MotoData, ExpenseInputs
  data/                   Tarifas Colombia 2026
  services/               Gemini, Claude, mock e interfaz de reconocimiento
  state/                  AppState (estado compartido entre pantallas)
  screens/                Las 5 pantallas del flujo
  widgets/                Componentes reutilizables
```

## 7. Qué falta / próximos pasos sugeridos

- Backend propio para no exponer la API key (ver sección 4).
- Guardar un historial de motos escaneadas (hoy cada "Escanear otra
  moto" empieza de cero).
- Selección de ciudad para afinar el precio de la gasolina y el
  impuesto por municipio.
- Tests automatizados (`flutter test`) para `ExpenseInputs` y las
  tarifas.

## Nota sobre verificación

El flujo nuevo se revisó estáticamente. No se ejecutaron pruebas ni
`flutter analyze` en esta actualización. Antes de publicar, resuelve las
dependencias y verifica el proyecto con:

```bash
flutter pub get
flutter analyze
flutter run
```

y revisa cualquier diagnóstico de `flutter analyze`.
