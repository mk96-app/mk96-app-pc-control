# mk96-app-pc-control

**MK96 PC Control**: centro de control personal para Windows que vive en la bandeja del sistema. Reúne en una sola ventana un monitor de hardware en tiempo real, perfiles de rendimiento con automatización, un gestor de programas de arranque y grupos de aplicaciones que se abren y se cierran de un clic.

Está hecha en C++ con [Dear ImGui](https://github.com/ocornut/imgui) y DirectX 11, más un pequeño proceso auxiliar en .NET que lee los sensores que las APIs nativas de Windows no alcanzan de forma fiable.

> **Solo Windows.** Usa APIs de Win32, DirectX y el registro de Windows. Se compila y se ejecuta en Windows, y **siempre pide permisos de administrador**.

## Qué hace

La navegación es una barra lateral de iconos, con una sección por herramienta.

### Monitor de hardware

- Uso de CPU, RAM y GPU; VRAM usada y total; temperatura de CPU y de GPU; velocidad de los ventiladores (si el hardware los expone); velocidad de red de subida y bajada; espacio libre por disco; tiempo de actividad.
- Gráficas de los últimos 60 segundos (CPU y RAM, temperaturas, GPU y VRAM, red).
- Si un sensor no está disponible, lo dice (por ejemplo "N/A") en vez de mostrar ceros falsos.

### Perfiles

Agrupa varios ajustes de Windows en perfiles que se aplican con un clic.

- **Qué puede fijar un perfil:** el plan de energía (Ahorro, Equilibrado, Alto rendimiento, Máximo rendimiento), los tiempos de apagado de pantalla, suspensión e hibernación (con corriente y con batería), el brillo (panel interno o monitores externos compatibles con DDC/CI) y, opcionalmente, el volumen.
- **5 perfiles predefinidos**, bloqueados: Rendimiento, Equilibrado, Ahorro de batería, Silencio/Noche y Presentación/Multimedia.
- **Perfiles personalizados** ilimitados, con su editor.
- **Automatización:** reglas por franja horaria (con opción "solo entre semana"), por batería bajo un umbral o por cambio de fuente de alimentación. Se pueden reordenar arrastrando. Si varias coinciden, gana la primera de la lista.
- **Botón "Apagar pantalla ahora"**, siempre visible.
- Al arrancar, detecta qué perfil coincide con el estado real de Windows. Nunca aplica nada que no hayas pedido.

### Inicio

Gestiona los programas de terceros que arrancan con Windows.

- Lee el registro (`Run` de usuario y de equipo) y las carpetas "Inicio" del usuario y de todos los usuarios.
- **Activar o desactivar** de forma reversible, sin borrar nada.
- **Eliminar** cualquier entrada, siempre con confirmación. Nunca borra el programa, solo lo que lo hace arrancar. Los accesos directos van a la Papelera.
- **Información** de cada programa: ruta, firmante, tamaño, versión y descripción.
- **Abrir ubicación** en el Explorador de Windows.
- **Añadir** un programa a mano.
- Oculta los programas firmados por Microsoft.

### Grupos

Define conjuntos de aplicaciones asociadas (por ejemplo, un juego, Discord y un overlay) y ábrelos o ciérralos todos de un clic.

- Grupos ilimitados. Admite `.exe`, accesos directos `.lnk` y argumentos de línea de comandos.
- **Abrir grupo** lanza cada aplicación que no esté ya en marcha. **Cerrar grupo** cierra solo lo que ese grupo abrió.
- Si dos grupos abiertos comparten una aplicación, no se cierra hasta que se cierre el último. Una aplicación que ya tenías abierta nunca se toca.
- El cierre es amable: pide cerrar la ventana y espera unos segundos antes de forzarlo.

### La aplicación

- **Bandeja del sistema:** cerrar la ventana la minimiza a la bandeja. Doble clic la reabre. El menú tiene "Abrir", "Iniciar con Windows" y "Salir".
- **Instancia única:** si ya está abierta, abrirla otra vez trae la ventana existente al frente.
- **Eficiente:** sin bucle de dibujo mientras está oculta en la bandeja.

## Requisitos

**Para ejecutar:** Windows. No hace falta instalar .NET aparte, porque el proceso auxiliar se publica autocontenido.

**Para compilar:**

- Visual Studio 2022 (carga de trabajo "Desarrollo para el escritorio con C++") o MSVC Build Tools con el Windows SDK.
- CMake 3.21 o superior.
- .NET SDK 8.0.
- Git.

No necesita claves ni configurar ninguna cuenta.

## Compilar

Clona el repositorio con sus submódulos (`imgui` e `implot`):

```powershell
git clone --recurse-submodules https://github.com/mk96-app/mk96-app-pc-control.git
cd mk96-app-pc-control
./scripts/build.ps1
```

Si ya lo clonaste sin `--recurse-submodules`, ejecuta antes `git submodule update --init --recursive`.

El resultado queda en `build/bin/`:

- `mk96-app-pc-control.exe`: la aplicación.
- `sensor-bridge/mk96-app-pc-control-sensor-bridge.exe`: el proceso auxiliar de sensores.

Si cambias de generador de CMake, mueves carpetas o subes la versión de LibreHardwareMonitorLib, borra `build/bin` y compila desde cero. Una compilación incremental puede dejar desactualizado el `.exe` del proceso auxiliar.

## Ejecutar

Abre `build/bin/mk96-app-pc-control.exe`. Windows mostrará el aviso de control de cuentas (UAC) en cada arranque, porque el ejecutable exige permisos de administrador. Sin ellos no se pueden leer la temperatura de la CPU ni los ventiladores.

Si activas "Iniciar con Windows", es probable que Windows muestre el aviso de UAC (o no la lance, según tu configuración) en cada inicio de sesión. Es una limitación conocida.

## Qué guarda y dónde

| Dato | Dónde |
|---|---|
| Perfiles personalizados y reglas de automatización | `%LOCALAPPDATA%\mk96-app-pc-control\profiles.json` |
| Grupos de aplicaciones | `%LOCALAPPDATA%\mk96-app-pc-control\groups.json` |
| Plan de energía propio de la aplicación | `HKCU\Software\mk96-app-pc-control\PowerPlan` |
| Inicio automático propio | Valor `mk96-app-pc-control` en `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` |

La aplicación no formatea discos, no borra archivos ni instala servicios o tareas programadas. Casi todo lo que escribe va a `HKCU` y es reversible. La única excepción es el gestor de Inicio, que al desactivar una entrada de todos los usuarios escribe en `HKLM\...\Explorer\StartupApproved\Run`, el mismo indicador que usa el Administrador de tareas. Nunca toca el valor `Run` en sí.

## Cómo funciona por dentro

Son dos procesos:

- **`mk96-app-pc-control.exe`** (C++): la ventana, DirectX 11 y Dear ImGui, la bandeja, y la lectura directa de CPU, RAM, red, disco y tiempo de actividad por Win32. Lanza el proceso auxiliar dentro de un Job Object de Windows, para que nunca quede huérfano aunque la aplicación se cierre de forma anómala.
- **`mk96-app-pc-control-sensor-bridge.exe`** (.NET 8): lee temperaturas, uso y VRAM de la GPU y ventiladores con LibreHardwareMonitorLib, y publica una instantánea por segundo en memoria compartida. Si un sensor falla, lo marca como no disponible en vez de caerse.

El diseño completo está en [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) y el estado detallado de cada sección en [`docs/PROJECT_STATUS.md`](docs/PROJECT_STATUS.md).

### LibreHardwareMonitorLib fijada a la versión 0.9.4

No la actualices sin más. A partir de la 0.9.5 la librería necesita un driver aparte, [PawnIO](https://github.com/namazso/PawnIO), para leer la temperatura de la CPU y los ventiladores. Ese driver no viene en el paquete NuGet y esta aplicación no lo instala, así que esos sensores fallarían en silencio aunque se ejecute como administrador. La 0.9.4 usa un driver embebido que solo necesita permisos de administrador.

## Estructura del repositorio

```
.
├── CMakeLists.txt                   Compila la aplicación nativa y publica el proceso auxiliar de .NET
├── README.md
├── .gitignore
├── .gitmodules                      Declara los submódulos imgui e implot
├── docs/
│   ├── ARCHITECTURE.md              Diseño de la aplicación
│   ├── BACKLOG.md                   Ideas pendientes
│   ├── POWER_GUIDS.md               Identificadores de energía de Windows (volcado de powercfg)
│   ├── PROJECT_STATUS.md            Estado, requisitos y limitaciones de cada sección
│   └── superpowers/
│       ├── plans/
│       │   ├── 2026-07-13-inicio-lista-horizontal-plan.md
│       │   └── 2026-07-18-grupos-lanzamiento.md
│       └── specs/
│           ├── 2026-07-13-inicio-lista-horizontal-design.md
│           └── 2026-07-18-grupos-lanzamiento-design.md
├── resources/
│   ├── AppIcon.ico                  Icono de la aplicación
│   ├── AppIcon.rc
│   └── resource.h
├── scripts/
│   └── build.ps1                    Compila todo de principio a fin
├── sensor-bridge/
│   └── Mk96AppPcControl.SensorBridge/    Proceso auxiliar de sensores (.NET 8)
│       ├── LhmSensorReader.cs
│       ├── Mk96AppPcControl.SensorBridge.csproj
│       ├── Program.cs
│       ├── SharedMemoryWriter.cs
│       └── SharedSensorData.cs
├── src/
│   ├── main.cpp                     Punto de entrada e instancia única
│   ├── app/                         Aplicación, bandeja, inicio automático y proceso auxiliar
│   │   ├── Application.cpp
│   │   ├── Application.h
│   │   ├── AutoStart.cpp
│   │   ├── AutoStart.h
│   │   ├── BridgeProcess.cpp
│   │   ├── BridgeProcess.h
│   │   ├── TrayIcon.cpp
│   │   └── TrayIcon.h
│   ├── groups/                      Sección Grupos
│   │   ├── GroupJson.cpp
│   │   ├── GroupJson.h
│   │   ├── GroupLauncher.cpp
│   │   ├── GroupLauncher.h
│   │   ├── GroupManager.cpp
│   │   ├── GroupManager.h
│   │   ├── GroupProcessTracker.cpp
│   │   ├── GroupProcessTracker.h
│   │   ├── GroupStore.cpp
│   │   ├── GroupStore.h
│   │   ├── GroupTypes.h
│   │   ├── ProcessLifecycle.cpp
│   │   └── ProcessLifecycle.h
│   ├── ipc/                         Memoria compartida con el proceso auxiliar
│   │   ├── SharedMemoryChannel.h
│   │   └── SharedSensorData.h
│   ├── platform/                    Utilidades de Windows (DirectX, ventana, JSON, iconos)
│   │   ├── ComScope.h
│   │   ├── DX11Renderer.cpp
│   │   ├── DX11Renderer.h
│   │   ├── IconTextureCache.cpp
│   │   ├── IconTextureCache.h
│   │   ├── MiniJson.cpp
│   │   ├── MiniJson.h
│   │   ├── StringConvert.h
│   │   ├── WndProc.cpp
│   │   └── WndProc.h
│   ├── profiles/                    Sección Perfiles
│   │   ├── SystemControl/           Control de energía, brillo, pantalla y volumen
│   │   │   ├── DisplayBrightness.cpp
│   │   │   ├── DisplayBrightness.h
│   │   │   ├── PowerPlan.cpp
│   │   │   ├── PowerPlan.h
│   │   │   ├── PowerTimeouts.cpp
│   │   │   ├── PowerTimeouts.h
│   │   │   ├── ScreenControl.cpp
│   │   │   ├── ScreenControl.h
│   │   │   ├── Volume.cpp
│   │   │   └── Volume.h
│   │   ├── AutomationEngine.cpp
│   │   ├── AutomationEngine.h
│   │   ├── ProfileJson.cpp
│   │   ├── ProfileJson.h
│   │   ├── ProfileManager.cpp
│   │   ├── ProfileManager.h
│   │   ├── ProfileStore.cpp
│   │   ├── ProfileStore.h
│   │   └── ProfileTypes.h
│   ├── sensors/                     Sensores nativos de Windows
│   │   ├── NativeSensors.cpp
│   │   ├── NativeSensors.h
│   │   └── SensorAggregator.h
│   ├── startup/                     Sección Inicio
│   │   ├── AppInfoReader.cpp
│   │   ├── AppInfoReader.h
│   │   ├── IconExtractor.cpp
│   │   ├── IconExtractor.h
│   │   ├── RegistryStartupControl.cpp
│   │   ├── RegistryStartupControl.h
│   │   ├── ShortcutStartupControl.cpp
│   │   ├── ShortcutStartupControl.h
│   │   ├── SignatureVerifier.cpp
│   │   ├── SignatureVerifier.h
│   │   ├── StartupScanner.cpp
│   │   ├── StartupScanner.h
│   │   └── StartupTypes.h
│   └── ui/                          Interfaz: secciones, diálogos y tema visual
│       ├── AddStartupEntryDialog.cpp
│       ├── AddStartupEntryDialog.h
│       ├── CardWidgets.cpp
│       ├── CardWidgets.h
│       ├── ConfirmDeleteDialog.cpp
│       ├── ConfirmDeleteDialog.h
│       ├── ConfirmDeleteGroupDialog.cpp
│       ├── ConfirmDeleteGroupDialog.h
│       ├── GroupEditorDialog.cpp
│       ├── GroupEditorDialog.h
│       ├── GroupsTab.cpp
│       ├── GroupsTab.h
│       ├── HardwareMonitorTab.cpp
│       ├── HardwareMonitorTab.h
│       ├── ITab.h
│       ├── PerfilesTab.cpp
│       ├── PerfilesTab.h
│       ├── ProfileEditorDialog.cpp
│       ├── ProfileEditorDialog.h
│       ├── RingBuffer.h
│       ├── RuleEditorDialog.cpp
│       ├── RuleEditorDialog.h
│       ├── StartupTab.cpp
│       ├── StartupTab.h
│       ├── TabManager.cpp
│       ├── TabManager.h
│       ├── Theme.cpp
│       └── Theme.h
└── third_party/
    ├── imgui/                       Submódulo: Dear ImGui
    └── implot/                      Submódulo: ImPlot
```

## Estado y verificación

La aplicación se compila y se prueba en la máquina Windows de su autor. Todas las secciones están implementadas. La documentación de `docs/PROJECT_STATUS.md` recoge, sección por sección, qué se ha comprobado en una máquina real y qué queda pendiente.

## Limitaciones conocidas

- **Luz nocturna y Asistente de enfoque no están soportados**, a propósito: Windows no tiene API pública para ellos.
- El brillo de un monitor externo requiere que soporte DDC/CI. Es más fiable por DisplayPort que por HDMI.
- "Máximo rendimiento" está oculto en la mayoría de instalaciones de Windows. La aplicación intenta duplicarlo y, si falla, aplica "Alto rendimiento".
- El estado "abierto" de un grupo solo vive en memoria: si reinicias la aplicación con un grupo abierto, sus aplicaciones siguen en marcha, pero hay que pulsar "Abrir grupo" otra vez para que el recuento sea correcto.
- Los grupos no admiten tareas programadas ni direcciones de lanzadores (`steam://…`), solo rutas de archivo.
