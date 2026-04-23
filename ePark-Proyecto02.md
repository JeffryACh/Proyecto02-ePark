# ePark – Guía de Desarrollo Android
> Curso IC-5821 Requerimientos de Software | I Semestre 2026  
> Sistema de gestión de parquímetros inteligentes para la Municipalidad de Cartago

---

## 1. Arquitectura de la Aplicación

### MVVM (Model-View-ViewModel)

Alineada con el modelo de objetos del curso (responsabilidades separadas):

```
Vista (Activity/Fragment / Composable)
    ↕
ViewModel (lógica de presentación)
    ↕
Repository (fuente de verdad)
    ↕
Room (local) / Supabase (remoto)
```

**Relación con el modelo de objetos del PDF:**

| Clase del modelo | Capa en MVVM |
|---|---|
| `AppDelUsuario` | View + ViewModel |
| `GestorDeCobro` | Repository / Use Case |
| `Parquimetro` | Model / Entidad |
| `SistemaCobros` | Supabase client |
| `Municipalidad` | Backend remoto (Supabase) |

---

## 2. Base de Datos

### Estrategia: **Dual (Local + Remota)**

El diagrama de secuencia muestra que hay objetos locales (`App`, `Parquimetro`) y remotos (`SistemaCobros`, `Municipalidad`), por lo que se necesitan dos capas de persistencia.

---

### 2.1 Base de Datos Local: **Room (SQLite)**

Room es la librería oficial de Android para persistencia local, construida sobre SQLite.

**Uso en ePark:**
- Caché de sesiones de parqueo activas (HU 7.1)
- Historial de pagos recientes sin conexión (HU 7.3)
- Configuración del usuario y vehículos registrados

**Ejemplo de entidad para el proyecto:**
```kotlin
@Entity(tableName = "sesion_parqueo")
data class SesionParqueo(
    @PrimaryKey val idSesion: String,
    val idZona: String,
    val horaInicio: Long,
    val idUsuario: String,
    val estadoPago: String
)
```

---

### 2.2 Base de Datos Remota: **Supabase (PostgreSQL)**

Supabase es la alternativa open source a Firebase, basada en **PostgreSQL real**. Para ePark es la elección ideal porque la HU 7.3 requiere queries con filtros por fecha, cliente y método de pago — exactamente donde SQL brilla y las bases NoSQL sufren.

| Característica | Descripción |
|---|---|
| **Base de datos** | PostgreSQL completo con soporte SQL nativo |
| **Auth integrado** | Login con email/contraseña, Google OAuth, magic link |
| **Realtime** | Suscripciones en tiempo real sobre tablas PostgreSQL |
| **Storage** | Almacenamiento de archivos (comprobantes, fotos) |
| **Open source** | Self-hosteable o cloud gratuito en [supabase.com](https://supabase.com) |
| **SDK Kotlin** | `supabase-kt`, librería oficial mantenida por Supabase |

**Esquema de tablas sugerido para ePark:**

```sql
-- Usuarios (manejado por Supabase Auth automáticamente)
auth.users

-- Vehículos del usuario
CREATE TABLE vehiculos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    usuario_id UUID REFERENCES auth.users(id),
    placa TEXT NOT NULL,
    tipo TEXT NOT NULL  -- 'carro', 'moto', 'scooter'
);

-- Zonas de parqueo (parquímetros)
CREATE TABLE parquimetros (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    zona TEXT NOT NULL,
    tarifa_por_hora DECIMAL NOT NULL,
    latitud DECIMAL,
    longitud DECIMAL,
    disponible BOOLEAN DEFAULT true
);

-- Sesiones de estacionamiento (HU 7.1)
CREATE TABLE sesiones (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    usuario_id UUID REFERENCES auth.users(id),
    parquimetro_id UUID REFERENCES parquimetros(id),
    hora_inicio TIMESTAMPTZ NOT NULL,
    hora_fin TIMESTAMPTZ,
    estado TEXT DEFAULT 'activa'  -- 'activa', 'finalizada'
);

-- Pagos con tarjeta (HU 7.3)
CREATE TABLE pagos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sesion_id UUID REFERENCES sesiones(id),
    usuario_id UUID REFERENCES auth.users(id),
    monto DECIMAL NOT NULL,
    metodo TEXT NOT NULL,           -- 'tarjeta_credito', 'tarjeta_debito'
    fecha TIMESTAMPTZ DEFAULT now(),
    confirmado BOOLEAN DEFAULT false
);
```

**Query de ejemplo para HU 7.3** (todos los pagos por tarjeta de crédito de un cliente en un día):

```sql
SELECT * FROM pagos
WHERE usuario_id = 'uuid-del-usuario'
  AND metodo = 'tarjeta_credito'
  AND DATE(fecha) = '2026-04-23'
ORDER BY fecha DESC;
```

> Esto con Firestore requeriría múltiples llamadas y lógica manual. Con Supabase es una sola query.

**Realtime para HU 7.2** — Supabase permite suscribirse a cambios en una tabla:

```kotlin
supabase.realtime.createChannel("sesion-activa")
    .postgresChangeFlow<PostgresAction.Update>(schema = "public") {
        table = "sesiones"
        filter = "id=eq.$idSesion"
    }
    .onEach { change ->
        // Reaccionar cuando la sesión se actualiza (tiempo venciendo)
    }
    .launchIn(viewModelScope)
```

---

### 2.3 Notificaciones Push con Supabase: **OneSignal**

Supabase no incluye un servicio de push propio (a diferencia de Firebase FCM). La solución recomendada es **OneSignal**, que es gratuito y tiene SDK oficial para Android.

**Flujo para HU 7.2 (aviso de 5 minutos):**

```
WorkManager (Android)
    → cuenta regresiva local
    → a los 5 min restantes dispara notificación local

Supabase Edge Function (opcional para notif remota)
    → llama a OneSignal REST API
    → OneSignal envía push al dispositivo del usuario
```

> Para el curso, la **notificación local con WorkManager** es suficiente y más simple de implementar.

---

### 2.4 Comparación final de capas de datos

| Criterio | Room (local) | Supabase (PostgreSQL) |
|---|---|---|
| Trabajo offline | ✅ | ⚠️ Requiere Room como caché |
| Tiempo real | ❌ | ✅ Con Realtime subscriptions |
| Queries complejas (HU 7.3) | ⚠️ Básico | ✅ SQL completo |
| Autenticación | ❌ | ✅ Integrada |
| Escalabilidad | N/A | ✅ |
| Complejidad de setup | Baja | Media |
| Datos transaccionales | ⚠️ Básico | ✅ Óptimo (PostgreSQL) |
| Costo | Gratis | Gratis (plan free generoso) |

> **Estrategia final:** Room para caché local y trabajo offline + Supabase como fuente de verdad remota.

---

## 3. Librerías y Herramientas Clave

### Comunicación y Red
| Librería | Uso |
|---|---|
| **Retrofit 2** | Llamadas HTTP a APIs REST (`SistemaCobros`) |
| **OkHttp** | Cliente HTTP base con interceptores de logging |
| **Gson / Moshi** | Serialización JSON de objetos del modelo |

### Notificaciones (HU 7.2)
| Herramienta | Uso |
|---|---|
| **WorkManager** | Temporizador en background, dispara notificación local a los 5 min |
| **AlarmManager** | Alarmas precisas para avisos de tiempo |
| **OneSignal SDK** | Notificaciones push remotas (alternativa a FCM sin Firebase) |

### Mapas y GPS (HU 7.1)
| Librería | Uso |
|---|---|
| **Google Maps SDK** | Visualizar zonas de parqueo |
| **FusedLocationProvider** | Obtener ubicación GPS del vehículo |

### UI / Diseño
| Herramienta | Uso |
|---|---|
| **Jetpack Compose** | UI declarativa moderna (recomendado) |
| **Material Design 3** | Componentes visuales estándar |
| **Navigation Component** | Navegación entre pantallas |

### Pagos (HU 7.3)
| Librería | Uso |
|---|---|
| **Stripe SDK** | Procesamiento de pagos con tarjeta |
| **Google Pay API** | Alternativa de pago integrada |

---

## 4. Resumen de Stack Tecnológico

```
┌─────────────────────────────────────────┐
│           APLICACIÓN ANDROID            │
│                                         │
│  Lenguaje:     Kotlin                   │
│  IDE:          Android Studio           │
│  UI:           Jetpack Compose          │
│  Arquitectura: MVVM + Repository        │
│                                         │
│  BD Local:     Room (SQLite)            │
│  BD Remota:    Supabase (PostgreSQL)    │
│  Auth:         Supabase Auth            │
│  Notif Push:   WorkManager + OneSignal  │
│  Realtime:     Supabase Realtime        │
│  Mapas/GPS:    Google Maps + Fused      │
│  Pagos:        Stripe SDK               │
└─────────────────────────────────────────┘
```

---

## 5. Relación con el Modelo de Objetos (PDF)

| Objeto del modelo (PDF) | Implementación en Android |
|---|---|
| `AppDelUsuario` | Activity/Fragment + ViewModel |
| `Parquimetro` | Entidad Room + tabla Supabase `parquimetros` |
| `SistemaCobros` | Supabase client → tabla `pagos` |
| `Municipalidad` | Supabase backend (PostgreSQL) |
| `GestorDeCobro` | Repository class en Android |
| `SensorDeParqueo` | FusedLocationProvider + GPS |
| Notificación (HU 7.2) | WorkManager + OneSignal |
| Registro de pagos (HU 7.3) | Room DAO + Supabase tabla `pagos` (SQL) |

---

## 6. Configuración de Android Studio para ePark

Esta sección detalla cómo dejar Android Studio listo para trabajar en equipo de forma correcta y estable, basándose en las decisiones del proyecto.

---

### Paso 1 – Instalar Android Studio

1. Descargar la versión **Ladybug Feature Drop (2024.2.2)** o superior desde [developer.android.com/studio](https://developer.android.com/studio)
2. Durante la instalación del wizard inicial seleccionar **Standard** (instala todo lo necesario automáticamente)
3. Componentes que deben quedar instalados — verificar en `File → Settings → Languages & Frameworks → Android SDK → SDK Tools`:
   - ✅ Android SDK Build-Tools
   - ✅ Android Emulator
   - ✅ Android SDK Platform-Tools
   - ✅ Google Play services
   - ✅ Intel x86 Emulator Accelerator (HAXM) — si el equipo tiene CPU Intel

---

### Paso 2 – Crear el Proyecto Correctamente

1. Abrir Android Studio → `New Project`
2. En la pantalla de plantillas (primera imagen): seleccionar **Phone and Tablet** en el panel izquierdo → elegir **Empty Activity**
   > ⚠️ Empty Activity crea el proyecto con **Jetpack Compose** preconfigurado. No elegir "Empty Views Activity" (esa usa XML, el sistema antiguo).

3. En la pantalla de configuración (segunda imagen), usar **exactamente estos valores**:

| Campo | Valor correcto | ⚠️ Error común a evitar |
|---|---|---|
| **Name** | `ePark` | No usar espacios ni guiones |
| **Package name** | `cr.ac.tec.epark` | ❌ No usar `com.example` (es genérico) |
| **Save location** | Carpeta del repositorio Git | Debe ser la misma en todos los equipos del grupo |
| **Minimum SDK** | **API 26 (Android 8.0)** | ❌ No usar API 36 — solo cubre 7.5% de dispositivos |
| **Build configuration language** | **Kotlin DSL (build.gradle.kts)** ✅ Recommended | Dejar el que viene seleccionado |

4. Hacer clic en **Finalizar** y esperar la sincronización de Gradle (~2-3 minutos la primera vez)

> ⚠️ Si aparece el aviso `'Proyecto' already exists at the specified Save location and it is not empty` significa que la carpeta ya tiene archivos (por ejemplo, del repositorio Git). Esto es normal si ya hicieron `git clone` antes. Hacer clic en **Finalizar** de todas formas.

---

### Paso 3 – Verificar la Instalación del SDK

Ir a `File → Settings → Languages & Frameworks → Android SDK → SDK Platforms` y confirmar que esté instalado al menos:

- ✅ **Android 8.0 (API 26)** — mínimo del proyecto
- ✅ **Android 14.0 (API 34)** — para el emulador principal
- ✅ **Android 16.0 (API 36)** — si el equipo quiere probar en la versión más nueva

Si alguno no está, marcarlo y hacer clic en **Apply**.

---

### Paso 4 – Configurar el Emulador (AVD)

1. Ir a `Tools → Device Manager → Create Device`
2. Seleccionar hardware: **Pixel 7** → Siguiente
3. Seleccionar imagen del sistema: **API 34 (Android 14.0) — x86_64** (descargar si no está)
4. En AVD Configuration:
   - RAM: mínimo **2048 MB**
   - Internal Storage: mínimo **4 GB**
5. Hacer clic en **Finish**
6. Iniciar el emulador con ▶ y verificar que llegue a la pantalla de inicio de Android antes de correr la app

> Si el equipo tiene un **teléfono físico Android**, es preferible usarlo para pruebas: activar `Ajustes → Acerca del teléfono → Número de compilación` (tocar 7 veces) para habilitar el Modo Desarrollador, luego activar `Depuración USB` en las opciones de desarrollador.

---

### Paso 5 – Configurar Git desde Android Studio

Android Studio tiene integración nativa con Git. Configurarla correctamente evita conflictos entre integrantes.

**Vincular el repositorio existente:**
```bash
# En la terminal integrada (View → Tool Windows → Terminal)
git init
git remote add origin https://github.com/su-org/epark.git
git pull origin main
```

**Verificar que el `.gitignore` generado por Android Studio incluya:**
```gitignore
# Archivos locales que NO deben subirse al repo
local.properties        ← contiene las keys de Supabase
*.jks                   ← keystore de firma de la app
.gradle/
build/
.idea/
```

> ⚠️ **Crítico:** El archivo `local.properties` contiene las credenciales de Supabase. Nunca debe llegar al repositorio. Verificar que esté en `.gitignore` antes del primer `git push`.

**Flujo de ramas recomendado:**
```
main          ← versión estable (solo el líder hace merge aquí)
develop       ← rama de integración del equipo
feature/auth       ← Integrante 1
feature/hu71       ← Integrante 2
feature/hu72       ← Integrante 3
feature/hu73       ← Integrante 4
```

---

### Paso 6 – Agregar Dependencias en `build.gradle.kts`

Abrir el archivo `app/build.gradle.kts` y reemplazar el bloque `dependencies` con:

```kotlin
dependencies {
    // Jetpack Compose
    implementation(platform("androidx.compose:compose-bom:2024.05.00"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("androidx.activity:activity-compose:1.9.0")
    debugImplementation("androidx.compose.ui:ui-tooling")

    // ViewModel
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.7.0")

    // Navigation
    implementation("androidx.navigation:navigation-compose:2.7.7")

    // Room (base de datos local / caché offline)
    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    kapt("androidx.room:room-compiler:2.6.1")

    // Supabase (base de datos remota + Auth + Realtime)
    implementation(platform("io.github.jan-tennert.supabase:bom:2.5.4"))
    implementation("io.github.jan-tennert.supabase:postgrest-kt")
    implementation("io.github.jan-tennert.supabase:auth-kt")
    implementation("io.github.jan-tennert.supabase:realtime-kt")
    implementation("io.ktor:ktor-client-android:2.3.12")

    // Google Maps y GPS
    implementation("com.google.maps.android:maps-compose:4.3.3")
    implementation("com.google.android.gms:play-services-location:21.2.0")

    // WorkManager (temporizador background para HU 7.2)
    implementation("androidx.work:work-runtime-ktx:2.9.0")

    // OneSignal (notificaciones push remotas)
    implementation("com.onesignal:OneSignal:5.1.6")

    // Stripe (pagos con tarjeta)
    implementation("com.stripe:stripe-android:20.48.0")

    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.0")

    // Testing
    testImplementation("junit:junit:4.13.2")
    androidTestImplementation("androidx.test.ext:junit:1.1.5")
}
```

También verificar que al inicio del archivo `app/build.gradle.kts` estén los plugins:

```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.compose)
    id("kotlin-kapt")  // ← necesario para Room
}
```

Después: `File → Sync Project with Gradle Files` y esperar que termine sin errores.

---

### Paso 7 – Conectar Supabase

1. Ir a [supabase.com](https://supabase.com) → `New Project`
2. Nombre: `epark`, seleccionar región más cercana, establecer contraseña de base de datos
3. Esperar ~2 minutos al aprovisionamiento
4. Ir a `Project Settings → API` y copiar:
   - **Project URL** → `https://xxxx.supabase.co`
   - **anon/public key** → empieza con `eyJhbGci...`
5. Abrir `local.properties` en la raíz del proyecto y agregar al final:
   ```
   SUPABASE_URL=https://xxxx.supabase.co
   SUPABASE_KEY=eyJhbGci...
   ```
6. En `app/build.gradle.kts`, dentro de `android { defaultConfig { ... } }`, agregar:
   ```kotlin
   val props = java.util.Properties().apply {
       load(rootProject.file("local.properties").inputStream())
   }
   buildConfigField("String", "SUPABASE_URL", "\"${props["SUPABASE_URL"]}\"")
   buildConfigField("String", "SUPABASE_KEY", "\"${props["SUPABASE_KEY"]}\"")
   ```
   Y habilitar BuildConfig:
   ```kotlin
   buildFeatures {
       compose = true
       buildConfig = true  // ← agregar esta línea
   }
   ```
7. Crear `app/src/main/java/cr/ac/tec/epark/EparkApp.kt`:
   ```kotlin
   class EparkApp : Application() {
       val supabase = createSupabaseClient(
           supabaseUrl = BuildConfig.SUPABASE_URL,
           supabaseKey = BuildConfig.SUPABASE_KEY
       ) {
           install(Postgrest)
           install(Auth)
           install(Realtime)
       }
   }
   ```
8. Registrar la clase Application en `AndroidManifest.xml`:
   ```xml
   <application
       android:name=".EparkApp"
       ...>
   ```
9. Crear las tablas en Supabase: `SQL Editor` en el dashboard → ejecutar el script SQL de la sección 2.2
10. Sincronizar Gradle nuevamente

---

### Paso 8 – Verificar que Todo Funciona

Antes de que cada integrante empiece a trabajar en su módulo, verificar esta lista:

- [ ] `Run` en el emulador muestra la pantalla inicial sin errores
- [ ] `Build → Make Project` (`Ctrl+F9`) compila sin warnings críticos
- [ ] `git status` no muestra `local.properties` como archivo a commitear
- [ ] El proyecto aparece en el dashboard de Supabase con las tablas creadas
- [ ] Todos los integrantes pueden hacer `git clone` y correr la app sin pasos adicionales

---

### Configuraciones adicionales de Android Studio recomendadas

Estas configuraciones mejoran la experiencia de desarrollo en equipo:

**Formateo de código uniforme** (`File → Settings → Editor → Code Style → Kotlin`):
- Indent: **4 espacios** (no tabs)
- Continuation indent: **4 espacios**
- Hacer clic en `Set from... → Kotlin Style Guide`

**Auto-import** (`File → Settings → Editor → General → Auto Import`):
- ✅ Add unambiguous imports on the fly
- ✅ Optimize imports on the fly

**Inspecciones útiles** (`File → Settings → Editor → Inspections`):
- ✅ Kotlin → Probable bugs → habilitarlas todas
- ✅ Android → Lint → habilitar las de performance

**Compartir configuración con el equipo:** El archivo `.editorconfig` en la raíz del proyecto fuerza el mismo estilo en todos:
```ini
root = true

[*.kt]
indent_style = space
indent_size = 4
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true
```

---

## 7. Planificación con MoSCoW y Fechas Reales

### ¿Qué es MoSCoW?

MoSCoW es una técnica de priorización que divide los requerimientos en 4 categorías, ayudando al equipo a enfocarse en lo que realmente importa cuando el tiempo es limitado:

| Categoría | Significado | Regla práctica |
|---|---|---|
| 🔴 **Must Have** | Obligatorio. Sin esto, el sistema no funciona. | Si no está en la demo, fallamos. |
| 🟠 **Should Have** | Importante pero no crítico. | Debe estar en la entrega final. |
| 🟡 **Could Have** | Deseable si hay tiempo. | Solo si el Must y Should están 100% listos. |
| ⚪ **Won't Have** | Fuera del alcance de este ciclo. | Se documenta para versiones futuras. |

---

### Clasificación MoSCoW para ePark

#### 🔴 Must Have — Obligatorio para ambas entregas

Estas funcionalidades deben estar en la **demo del 22 de mayo** en alguna forma funcional, y completas para el **5 de junio**.

| Funcionalidad | HU | Justificación |
|---|---|---|
| Login y registro de usuario | — | Sin autenticación no hay sistema |
| Estacionar un vehículo en la calle | 7.1 | Flujo principal del diagrama de secuencia |
| Registro de la sesión en Supabase | 7.1 | Base de todo el resto del sistema |
| Notificación local a los 5 minutos | 7.2 | Requerimiento explícito del profesor |
| Visualizar pagos del día por tarjeta | 7.3 | Requerimiento explícito del profesor |

#### 🟠 Should Have — Necesario para la entrega final (5 de junio)

Pueden estar ausentes o en mock data en la demo, pero deben estar completos en la entrega final.

| Funcionalidad | HU | Justificación |
|---|---|---|
| Mapa con Google Maps mostrando zonas | 7.1 | Mejora la usabilidad y la presentación |
| Temporizador visual de sesión activa | 7.2 | Feedback visual al usuario |
| Filtros por fecha en historial de pagos | 7.3 | Completa el requerimiento 7.3 |
| Manejo de errores y estados de carga | — | UX mínima aceptable |
| Caché offline con Room | — | Robustez del sistema |

#### 🟡 Could Have — Si hay tiempo libre antes del 5 de junio

| Funcionalidad | Justificación |
|---|---|
| Notificaciones push remotas con OneSignal | Agrega valor pero la notificación local ya cumple el requisito |
| Integración real con Stripe | Modo simulado es suficiente para el curso |
| Historial de sesiones pasadas | Complementa la HU 7.3 pero no es estrictamente requerido |
| Modo oscuro / temas visuales | Pulido estético, no funcional |

#### ⚪ Won't Have — Fuera del alcance de este semestre

| Funcionalidad | Nota |
|---|---|
| Panel de administración (Municipalidad) | Sistema separado, requiere backend propio |
| Integración con parquímetros físicos reales | Requiere hardware |
| Soporte para múltiples vehículos por sesión | Complejidad innecesaria para el prototipo |
| App iOS | Solo se desarrolla Android |

---

### Fechas Clave

| Evento | Fecha | Qué debe estar listo |
|---|---|---|
| 🟢 Inicio del proyecto | 23 abril 2026 | Repositorio y proyecto creado |
| 🔵 Demo / Evaluación intermedia | **22 mayo 2026** | Todos los **Must Have** funcionando |
| 🔴 Entrega final | **5 junio 2026** | Must Have + Should Have completos |

---

### Roadmap Ajustado a las Fechas Reales

El período total es de **~6 semanas** (23 abril → 5 junio), dividido en dos fases:

**Fase 1 – Hacia la demo (23 abril → 22 mayo) = 4 semanas**
**Fase 2 – Hacia la entrega final (23 mayo → 5 junio) = 2 semanas**

---

#### Semana 1 · 23–29 abril · Esqueleto y Setup

**Meta:** El proyecto compila, todos tienen acceso al repositorio y pueden correr la app.

- Integrante 1: Crear proyecto Android Studio, configurar Git, estructura de carpetas MVVM, inicializar Supabase client
- Integrante 1: Pantallas de Login y Registro conectadas a Supabase Auth
- Integrante 2: Crear entidades Room (`SesionParqueo`, `Parquimetro`) y DAOs base
- Integrante 3: Configurar WorkManager y canal de notificaciones Android
- Integrante 4: Crear entidad `Pago` en Room y pantalla de historial con datos mock

**Criterio de salida:** `git clone` + `Run` funciona en el dispositivo de todos sin errores.

---

#### Semana 2 · 30 abril–6 mayo · HU 7.1 Core

**Meta:** Un usuario puede estacionarse y la sesión queda guardada en Supabase.

- Integrante 2: Flujo completo de estacionamiento (selección de zona → confirmar → insertar en Supabase tabla `sesiones`)
- Integrante 2: Guardar sesión en Room localmente
- Integrante 1: Pantalla de perfil de usuario con datos desde Supabase Auth
- Integrante 3: Temporizador local iniciado al confirmar estacionamiento
- Integrante 4: Query básica de pagos desde Supabase (sin filtros aún)

**Criterio de salida:** Demo interna del flujo HU 7.1 de punta a punta.

---

#### Semana 3 · 7–13 mayo · HU 7.2 + Mapa

**Meta:** La notificación de 5 minutos funciona y el mapa muestra zonas.

- Integrante 2: Integrar Google Maps Compose con zonas de parqueo reales (desde Supabase tabla `parquimetros`)
- Integrante 3: Notificación local disparada por WorkManager a los 5 minutos ✅ **Must Have completo**
- Integrante 3: Pantalla de sesión activa con countdown visual en tiempo real
- Integrante 4: Filtros por fecha y método en el historial de pagos

**Criterio de salida:** Notificación aparece automáticamente en el dispositivo.

---

#### Semana 4 · 14–21 mayo · HU 7.3 + Pulido para Demo

**Meta:** Los 3 flujos principales funcionan. Preparar la demo del 22 de mayo.

- Integrante 4: Historial de pagos por tarjeta de crédito en un día, filtrado por cliente ✅ **Must Have HU 7.3 completo**
- Integrante 4: Pago simulado con Stripe en modo test
- Todo el equipo: Eliminar datos mock de las pantallas principales
- Todo el equipo: Manejo básico de errores (sin conexión, login fallido)
- Integrante 1: Ensayo de la demo — recorrido completo del flujo

> ⚠️ **Regla de la semana 4:** No se agregan features nuevas. Solo se estabilizan las existentes.

**Criterio de salida:** Demo completa grabada o ensayada el 21 de mayo.

---

#### 🔵 22 mayo — Evaluación Intermedia (Demo / Beta)

**Debe funcionar en vivo:**
1. Login de usuario
2. Estacionar un vehículo → sesión guardada en Supabase
3. Notificación local a los 5 minutos
4. Ver historial de pagos del día filtrado por tarjeta

---

#### Semana 5 · 23–29 mayo · Should Have + Robustez

**Meta:** Completar todo lo que quedó pendiente de la categoría Should Have.

- Supabase Realtime suscrito a cambios en `sesiones` (Integrante 3)
- Caché offline completo con Room sincronizado (Integrante 2)
- Estados de carga `Loading` / `Error` / `Success` en todas las pantallas (todos)
- OneSignal si hay tiempo disponible — Could Have (Integrante 3)

---

#### Semana 6 · 30 mayo–5 junio · QA y Entrega Final

**Meta:** La app es estable, sin crashes, y lista para entregar.

- Pruebas en dispositivo físico (no solo emulador)
- Corregir bugs encontrados en la semana 5
- Revisar que todos los requerimientos del profesor estén cubiertos
- Documentación mínima del proyecto (README con instrucciones de instalación)
- **No se agregan features nuevas después del 2 de junio**

> ⚠️ **Regla de las últimas 48 horas:** Solo corrección de bugs críticos. Nada nuevo.

---

#### 🔴 5 junio — Entrega Final

---

### Resumen Visual del Calendario

```
ABRIL                    MAYO                         JUNIO
23  24  25  26  27  28  29  30  1   2   3   4   5   6   7 ...22  23...5jun
│←────── Semana 1 ──────→│←──── Semana 2 ────→│       │         │
                                               │←── S3 →│←── S4 →│ ← DEMO 22/5
                                                                  │←── S5 ──→│←S6→│
                                                                              ↑ Entrega
                                                                             5 jun
```

```
Semana  Fechas           Foco principal              MoSCoW
──────  ──────────────   ─────────────────────────   ───────
  1     23–29 abril      Setup + Auth + Esqueleto     🔴 Must
  2     30 abr–6 mayo    HU 7.1 Core (estacionar)     🔴 Must
  3     7–13 mayo        HU 7.2 (notificaciones)      🔴 Must
  4     14–21 mayo       HU 7.3 + pulido demo         🔴 Must
  ─     22 mayo          ★ DEMO / EVALUACIÓN BETA     ─────
  5     23–29 mayo       Should Have + robustez       🟠 Should
  6     30 mayo–5 jun    QA + correcciones + entrega  🟠 Should
  ─     5 junio          ★ ENTREGA FINAL              ─────
```

---

## 8. División de Responsabilidades para 4 Personas

Cada integrante es dueño de un módulo vertical completo (UI + lógica + datos), más una responsabilidad transversal compartida.

---

### Integrante 1 – Líder de Arquitectura y Autenticación

**Módulo principal:** Base del proyecto + Login/Registro

**Responsabilidades:**
- Configurar el proyecto en Android Studio y el repositorio Git
- Definir y mantener la estructura de carpetas MVVM
- Implementar Supabase Auth (login con email/contraseña y Google OAuth)
- Crear las clases base: `BaseViewModel`, `BaseRepository`, `AppDatabase` (Room)
- Inicializar el cliente Supabase (`SupabaseClient`) y compartirlo via inyección
- Pantallas: Login, Registro, Perfil de usuario
- Revisar Pull Requests y resolver conflictos de merge

**Archivos clave:**
```
data/local/AppDatabase.kt
data/remote/ApiClient.kt
ui/auth/LoginScreen.kt
ui/auth/AuthViewModel.kt
```

---

### Integrante 2 – HU 7.1: Flujo de Estacionamiento

**Módulo principal:** Estacionar un vehículo (diagrama de secuencia completo)

**Responsabilidades:**
- Implementar la pantalla de mapa con Google Maps Compose
- Integrar `FusedLocationProvider` para obtener GPS del usuario
- Crear las entidades Room: `SesionParqueo`, `Parquimetro`
- Implementar el `GestorEstacionamiento` (Repository) que orquesta el flujo del PDF
- Insertar y consultar sesiones en Supabase (tabla `sesiones`)
- Pantallas: Mapa de zonas, Detalle de zona, Confirmación de estacionamiento

**Archivos clave:**
```
domain/model/SesionParqueo.kt
domain/model/Parquimetro.kt
data/local/SesionDao.kt
data/repository/EstacionamientoRepository.kt
ui/estacionar/MapaScreen.kt
ui/estacionar/EstacionarViewModel.kt
```

---

### Integrante 3 – HU 7.2: Notificaciones y Temporizador

**Módulo principal:** Sistema de alertas y gestión del tiempo activo

**Responsabilidades:**
- Configurar OneSignal SDK para notificaciones push remotas
- Implementar `WorkManager` para temporizador de background (5 minutos)
- Crear la notificación local con canal de notificaciones Android
- Suscribirse a Supabase Realtime en la tabla `sesiones` para detectar cambios
- Pantalla de sesión activa con countdown visual en tiempo real
- Pantalla de historial de notificaciones recibidas
- Manejar permisos de notificaciones en Android 13+

**Archivos clave:**
```
service/TemporizadorWorker.kt
service/OneSignalManager.kt
ui/sesion/SesionActivaScreen.kt
ui/sesion/SesionViewModel.kt
ui/notificaciones/NotificacionesScreen.kt
```

---

### Integrante 4 – HU 7.3: Pagos e Historial

**Módulo principal:** Registro, procesamiento y visualización de pagos

**Responsabilidades:**
- Implementar la entidad `Pago` en Room con DAO completo
- Integrar Stripe SDK en modo test para simular cobros
- Crear queries SQL en Supabase filtradas por fecha, cliente y método (HU 7.3)
- Pantalla de historial de pagos con filtros interactivos
- Pantalla de detalle de pago individual
- Exportar o mostrar resumen del día

**Archivos clave:**
```
domain/model/Pago.kt
data/local/PagoDao.kt
data/repository/PagoRepository.kt
data/remote/StripeService.kt
ui/pagos/PagosScreen.kt
ui/pagos/PagosViewModel.kt
ui/pagos/DetallePagoScreen.kt
```

---

### Resumen de la División

| Integrante | Módulo | HU Principal | Responsabilidad transversal |
|---|---|---|---|
| **1** | Arquitectura + Auth | Login/Registro | Setup del proyecto, Supabase client, revisión de PRs |
| **2** | Estacionamiento | HU 7.1 | Modelo de datos compartido (Room + tablas Supabase) |
| **3** | Notificaciones | HU 7.2 | OneSignal + Supabase Realtime |
| **4** | Pagos | HU 7.3 | Queries SQL en Supabase + Stripe |

### Normas de colaboración

- Cada integrante trabaja en su rama `feature/nombre` y hace Pull Request a `develop`
- Las reuniones de sincronización deben ocurrir al inicio de cada nueva versión (v0.1, v0.2, etc.)
- El Integrante 1 debe aprobar todos los PRs antes de hacer merge a `develop`
- Los datos mock los define el Integrante 1 para que todos puedan trabajar en paralelo desde la v0.1