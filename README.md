# ePark - Proyecto 02 (IC-5821)

Sistema Android para gestion de parquimetros inteligentes de la Municipalidad de Cartago.

Este README resume **como es hoy** el proyecto (estado actual) y **como va a ser** (trayectoria planificada), junto con la arquitectura, la estructura de datos y el flujo de datos para las HU 7.1, 7.2 y 7.3.

---

## 1) Estado del proyecto y trayectoria

Este documento combina dos objetivos complementarios:
- **Academico**: justificar decisiones de modelado por responsabilidades, mensajes y cohesion.
- **Tecnico-practico**: guiar implementacion, integracion y validacion incremental del equipo.

## Como es hoy (estado actual)
- Existe una guia funcional y tecnica en `ePark-Proyecto02.md`.
- Ya estan definidas:
  - la arquitectura objetivo (`MVVM + Repository`),
  - la estrategia de persistencia (`Room + Supabase`),
  - las historias de usuario clave (HU 7.1, 7.2, 7.3),
  - la division de responsabilidades del equipo.
- El repositorio esta en fase de base documental y planificacion de implementacion.

## Como va a ser (estado objetivo)
- Una app Android en Kotlin con Jetpack Compose que permita:
  1. Estacionar un vehiculo y registrar la sesion (HU 7.1).
  2. Notificar al usuario cuando falten 5 minutos (HU 7.2).
  3. Consultar pagos por tarjeta de credito en un dia por cliente (HU 7.3).
- Con sincronizacion entre almacenamiento local y remoto:
  - Local: Room (offline/caché).
  - Remoto: Supabase PostgreSQL (fuente de verdad).

## Trayectoria (linea de tiempo)
| Fase | Fechas | Resultado esperado |
|---|---|---|
| Semana 1 | 23-29 abril 2026 | Setup, auth y esqueleto MVVM |
| Semana 2 | 30 abril-6 mayo 2026 | HU 7.1 operativa (estacionar + sesion) |
| Semana 3 | 7-13 mayo 2026 | HU 7.2 (temporizador + notificacion local) |
| Semana 4 | 14-21 mayo 2026 | HU 7.3 + estabilizacion demo |
| Demo intermedia | 22 mayo 2026 | Must Have funcionales |
| Semana 5 | 23-29 mayo 2026 | Should Have + robustez |
| Semana 6 | 30 mayo-5 junio 2026 | QA, correcciones, entrega final |

---

## 2) Arquitectura integrada

Arquitectura principal:

```text
Vista (Activity/Fragment/Compose)
    <->
ViewModel (estado UI + casos de uso de presentacion)
    <->
Repository (orquestacion y reglas de negocio)
    <->
Room (local) + Supabase (remoto)
```

### Componentes y su rol
- **UI (Compose)**: renderiza estado y captura eventos del usuario.
- **ViewModel**: transforma eventos en acciones de dominio, expone `UiState`.
- **Repository**: decide lectura/escritura local-remota y aplica reglas.
- **Room**: cache de sesiones activas, datos de soporte y trabajo offline.
- **Supabase**: autenticacion, tablas de negocio, consultas SQL y realtime.
- **WorkManager**: timers y ejecucion en background para HU 7.2.

### Integracion entre capas
- El flujo siempre entra por UI y baja por ViewModel -> Repository.
- Room evita bloquear la UX sin red y acelera lecturas locales.
- Supabase consolida datos oficiales y permite consultas complejas (HU 7.3).
- Realtime sincroniza cambios relevantes de `sesiones` al cliente.

### Criterios de modelado aplicados (academico)
- **Estado (atributos)**: cada objeto conserva solo la informacion que le corresponde.
- **Comportamiento (metodos)**: cada capa ejecuta responsabilidades claras (sin clases "ManejadorDeTodo").
- **Identidad**: entidades con identificadores unicos (`id`, `usuario_id`, `sesion_id`).
- **Mensajeria**: la colaboracion ocurre por mensajes/llamadas entre objetos y capas.

---

## 3) Estructura de datos

## Modelo conceptual (objetos del dominio)
- `AppDelUsuario` (interaccion de usuario y experiencia de app)
- `Parquimetro` (zona, tarifa, disponibilidad)
- `Sesion` (inicio, fin, estado)
- `Pago` (monto, metodo, fecha, confirmado)
- `GestorDeCobro` (orquestacion de cobro y confirmaciones)

## Estructura de persistencia

### Local - Room (cache y soporte offline)
- Entidades principales sugeridas:
  - `SesionParqueo`
  - `Parquimetro`
  - `Pago` (resumen local/consulta reciente)

### Remota - Supabase (PostgreSQL)
Tablas sugeridas:
- `vehiculos`
- `parquimetros`
- `sesiones`
- `pagos`

Relaciones clave:
- `sesiones.usuario_id -> auth.users.id`
- `sesiones.parquimetro_id -> parquimetros.id`
- `pagos.sesion_id -> sesiones.id`
- `pagos.usuario_id -> auth.users.id`

---

## 4) Flujo de datos por historia de usuario

Cada flujo se define como secuencia de mensajes entre actores de dominio (`Usuario`, `AppDelUsuario`, `Parquimetro`, `SistemaCobros`, `Municipalidad`) y componentes tecnicos (ViewModel, Repository, Room, Supabase).

## HU 7.1 - Estacionar un vehiculo
1. Usuario selecciona zona desde app.
2. ViewModel solicita al Repository iniciar sesion.
3. Repository registra en Supabase (`sesiones`) y guarda cache local en Room.
4. UI muestra sesion activa y datos de cobro estimado.

## HU 7.2 - Aviso cuando faltan 5 minutos
1. Al iniciar sesion, se programa un trabajo con WorkManager.
2. Worker calcula tiempo restante y dispara notificacion local al umbral.
3. (Opcional) Realtime actualiza estado de sesion si cambia en backend.

## HU 7.3 - Consultar pagos del dia por cliente y metodo
1. Usuario aplica filtros (cliente, fecha, tarjeta de credito).
2. Repository ejecuta query SQL en Supabase sobre `pagos`.
3. Resultado se ordena por fecha y se presenta en UI.
4. Room puede conservar un snapshot local para consulta rapida.

---

## 5) Como esta integrada su estructura (codigo, datos y equipo)

## Integracion tecnica
- **Presentacion**: Compose + Navigation + Material 3.
- **Dominio/Orquestacion**: ViewModels + Repositories.
- **Datos**: Room (local) + Supabase (remoto) con estrategia dual.
- **Servicios transversales**: WorkManager (tiempo/notificaciones), Maps/GPS, pagos.

## Integracion de equipo
- Rama principal estable: `main`.
- Integracion continua del equipo: `develop`.
- Desarrollo por funcionalidad: `feature/auth`, `feature/hu71`, `feature/hu72`, `feature/hu73`.
- Integrante 1 lidera arquitectura y revisiones de PR.

---

## 6) Estructura propuesta del proyecto Android

```text
app/src/main/java/cr/ac/tec/epark/
  data/
    local/         # Room DB, DAOs, entidades locales
    remote/        # Supabase client, servicios externos
    repository/    # Repositorios (orquestacion local/remoto)
  domain/
    model/         # Modelos de dominio
    usecase/       # Casos de uso (opcional)
  ui/
    auth/
    estacionar/
    sesion/
    pagos/
    common/
  service/
    TemporizadorWorker.kt
    NotificacionManager.kt
  EparkApp.kt
```

---

## 7) Arranque rapido (cuando inicie implementacion)

1. Crear proyecto Android con Kotlin + Compose (API minima 26).
2. Configurar dependencias: Room, Supabase, WorkManager, Maps, Stripe.
3. Agregar variables en `local.properties`:

```properties
SUPABASE_URL=https://xxxx.supabase.co
SUPABASE_KEY=eyJhbGci...
```

4. Exponer claves en `BuildConfig` desde `app/build.gradle.kts`.
5. Crear tablas en Supabase (`vehiculos`, `parquimetros`, `sesiones`, `pagos`).
6. Ejecutar app en emulador (API 34 recomendado) y validar flujo base.

---

## 8) Criterios de exito del proyecto

- Flujo HU 7.1 completo de punta a punta.
- Notificacion de HU 7.2 funcionando en dispositivo/emulador.
- Consulta HU 7.3 con filtros correctos y datos coherentes.
- Estructura MVVM mantenible y separacion clara de responsabilidades.
- Sin exposicion de credenciales (`local.properties` fuera del control de versiones).

## 9) Evidencia esperada por fase (academico + tecnico)

- **Modelado**: CRC/relaciones y justificacion de responsabilidades por modulo.
- **Implementacion**: commits por HU y PRs con alcance acotado.
- **Verificacion**: pruebas manuales por flujo HU 7.1, 7.2 y 7.3.
- **Integracion**: trazabilidad entre reglas del modelo y decisiones de codigo/datos.

---

## Referencias
- Documento base del proyecto: `ePark-Proyecto02.md`
- Insumo de modelado: actividad de clase "Modelado de Objetos" (CRC, roles de mensajeria, diagrama de clases y secuencia)

