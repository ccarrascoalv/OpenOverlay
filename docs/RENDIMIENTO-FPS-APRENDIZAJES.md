# Pérdida de FPS con OpenOverlay abierto: investigación

Estado al 2026-10-03, tras cuatro rondas de medición y las optimizaciones de UI (build v3). Documento de traspaso para retomar la investigación sin perder el hilo.

## Resumen del estado

- **Causa raíz: tener los widgets abiertos.**
  - Con todos los widgets abiertos, iRacing pierde un **13-18 %** de FPS de media y **~25-29 FPS** en los tramos rápidos, que es lo que se percibía.
  - OpenOverlay usa entonces ~0.7 núcleos de CPU y ~7 % del motor 3D de la GPU.
- **El bucle por frame (`CompositionTarget.Rendering`) era un coste real pero pequeño.**
  - Supone ~3 FPS con los widgets cerrados y ~5 FPS con los widgets abiertos.
  - Está quitado en `perf/frame-hook-test`, que debe pasar a la rama principal (sin el distintivo TEST).
- **Brújula de Weather animada: confirmada (ronda 4).** Cuesta −7.6 % de FPS y con 10 Hz baja a ≈0 %. A 144 Hz la animación redibuja 144 veces por segundo.
- **Pedal Trace y Cockpit: coste pequeño (ronda 4).** Quedan en −1 a −4 %, dentro del ruido, a 60 Hz y a 30 Hz. El −9.6 % de Pedal Trace en la ronda 3 fue ruido de la referencia.
- **Con todos los widgets habituales abiertos se sigue perdiendo un −12 %, aunque se bajen las frecuencias** (T-old ≈ T-new).
  - Además, con todos abiertos **los propios widgets se ven a tirones**; por separado se ven fluidos.
  - Apunta a que OpenOverlay se satura internamente: un único hilo de UI y un único hilo de render de WPF para todas las ventanas. Relative y Standings son los siguientes sospechosos.
- **Diagnóstico con todos los widgets (2026-10-03):**
  - GC muy alto: una colección gen2 cada ~4 s.
  - Tick de 100 ms de 9-10 ms, con picos de 25 ms.
  - Huecos de 67-79 ms en Cockpit y Pedal Trace, con un objetivo de 33 ms.
- **Causa: la capa WPF, no la lógica de negocio.** El log `PerfProbe` lo confirma: todos los builders juntos tardan <0.5 ms por tick.
  1. La plantilla de filas de Relative y Standings, y la de TireInfo, enganchaban suscripciones `ValueChanged` por cada enlace en cada objeto nuevo.
  2. Los colores eran strings convertidos a un pincel nuevo en cada tick, lo que obligaba a volver a formatear el texto.
  3. TireInfo se redibujaba en cada tick aunque solo cambia en el box.
- **Corregido en `perf/driver-table-bindings` (build v3: "TEST optimizaciones UI v3 + perf log"):**
  - enlaces `OneTime`;
  - huecos de fila fijos (`RowSlot`);
  - pinceles cacheados (`CachedBrushConverter`);
  - TireInfo sin redibujar si no cambia nada.
- **Resultado en carrera con todos los widgets:**
  - hilo de UI ocupado ~8 ms por tick (antes ~14 ms, al principio más);
  - layout y render ~3 ms (antes ~8);
  - espera del tick crítico de 9-18 ms en el peor caso típico (antes 18-38; al principio, huecos de 79 ms);
  - 8-10 MB/s de memoria asignada y sin gen2.
- **Sensación del usuario con v3:** "una gran mejora respecto a los FPS y los overlays; apenas noté lag en Cockpit, brújula ni Pedal Trace".
  - Ajustes: 30 Hz y brújula a 10 Hz.
  - Falta medirlo con CapFrameX.
- **Siguiente paso:** no tocar las filas de Standings y Relative que muestran lo mismo que antes (ver «Próximas pruebas»). Después, validar FPS.

## Problema

- Con OpenOverlay abierto y los widgets habituales, iRacing pierde muchos FPS en carrera. Al principio se estimaron 30-50 FPS.
- Se nota sobre todo en carreras con mucha carga.
- Widgets habituales: Standings, Relative, Incidents, Weather, Cockpit (tema Pit Wall), Flags y Delta. El dashboard también llegó a estar abierto.
- La observación inicial de que la pérdida seguía con todos los widgets cerrados **no se ha confirmado**. Con los widgets nunca abiertos, la versión nueva queda a la par que sin la app (ronda 2).
  - Probablemente se compararon FPS en zonas distintas de la pista: la app se cerraba en la recta de atrás.

## Hipótesis

### Bucle por frame: confirmado, pero coste menor

`CompositionTarget.Rendering += OnFrame` en `MainWindow` se suscribía en el constructor y solo se quitaba al cerrar la app.

- Con cualquier manejador ahí, WPF compone en **cada refresco del monitor** durante toda la vida del proceso y despierta el hilo de UI en cada frame.
- Medido en la ronda 2, con los widgets nunca abiertos:

  | Versión | FPS | CPU de OpenOverlay |
  |---|---|---|
  | Antigua | −3 % | 0.94 % |
  | Nueva | ≈ sin la app | 0.38 % |

### Ventanas transparentes y redibujado frecuente: confirmado en la ronda 3

- Todos los widgets son ventanas con `AllowsTransparency=true`. WPF las actualiza copiando cada frame de la GPU a memoria del sistema (`UpdateLayeredWindow`).
- El **Cockpit** se redibuja entero hasta 60 veces por segundo (*Critical refresh* en General › Performance). Relative, Standings y el resto se redibujan cada 100 ms.
- Con los widgets abiertos, el tiempo de CPU de iRacing por frame sube de 9.2 a 10.6-11.1 ms y OpenOverlay usa ~7 % del motor 3D.
- La ronda 3 lo confirma: solo cuestan los widgets que se redibujan a menudo, que son también los únicos que usan GPU. La brújula animada de Weather también cuenta: una animación WPF activa redibuja a la frecuencia del monitor.

### En carrera estamos limitados por CPU

- En replay hay más FPS con gráficos más altos que en carrera. La replay no simula los otros coches.
- En carrera, la CPU pasa ~9-11 ms en cada frame y la GPU ~5.5-6.5 ms. El cuello de botella es la **CPU**, así que lo que importa es cuánta CPU (y sincronización con la GPU) consume OpenOverlay.

## Descartado tras revisar el código

- **Lector del SDK** (`IRacingConnection.RunConnectedLoop`): espera al evento de datos de iRacing (`WaitAny`) a 60 Hz y copia pocos KB por tick. Barato.
- **`_uiTimer` de 100 ms**:
  - Los trackers fijos (pit stops, cruces de línea, `EstTimeProfile`, lap log, penalizaciones) son CPU ligera.
  - Los `Feed` salen sin hacer nada si el widget es `null`.
- **Widgets ocultos**: hacen `Hide()`, y una ventana oculta no se compone. La ronda 2 lo respalda: con los widgets abiertos y después cerrados (D\*), OpenOverlay usa la misma CPU que sin haberlos abierto nunca, y nada de GPU.
- **Animaciones o efectos en bucle**: no hay ninguno.
- **El `_flashTimer` del Cockpit**: solo corre con el widget cargado.
- **Bandeja, watchdog y log**: triviales.

## Cambio ya probado: quitar el bucle por frame

- **Rama:** `perf/frame-hook-test`
- **Commit:** `d5bd4a2`
- **Base:** `relative_fixes` (`1320ab0`)

Qué cambia:
- Se quita `CompositionTarget.Rendering`.
- El tick crítico (Cockpit y Pedal Trace) se dispara con **cada tick nuevo de telemetría**: `_connection.TelemetryUpdated` → `PostCriticalTick()`.
- `PostCriticalTick()` agrupa los ticks con `Interlocked`, de modo que nunca hay más de una actualización en cola, y hace `Dispatcher.InvokeAsync(..., DispatcherPriority.Render)`.
- Se mantiene el límite de *Critical refresh*, con 3 ms de margen.

**Distintivo temporal:** la constante `MainWindow.FrameHookTestTag` añade "TEST sin bucle por frame" en tres sitios:
- la barra de título del panel de control;
- el tooltip de la bandeja;
- la línea de diagnóstico del panel.

**Hay que quitarlo** al pasar el cambio a la rama principal.

Compila sin avisos. Tests: App 546/546 y SDK 48/48 en verde. Probado con iRacing en vivo en la ronda 2, sin problemas funcionales.

## Método de medición

Es el que dio resultados repetibles en la ronda 2.

### Escena

- **No usar replays.** No envían todos los datos y no representan la carga de CPU de una carrera.
- Carrera offline contra la IA: **28 coches en Road Atlanta, salida lanzada**, cámara del coche y ajustes gráficos de carrera.
- **Reiniciar la sesión antes de cada prueba** y capturar **120 s desde antes de la bandera verde**.
  - Así cada prueba recorre la misma secuencia: final de la formación, salida y primeros tramos.
  - El ruido entre capturas equivalentes queda en ±4 FPS por tramo de 20 s.
- Hacer una pasada de calentamiento que no cuente.
- Repetir A (app cerrada) al principio y al final para detectar deriva.
- VSync y límite de FPS desactivados.
- CapFrameX sin el overlay de RTSS.

### Dos versiones a mano

```bash
git checkout relative_fixes && ./generar-openoverlay.sh
cp -r /mnt/c/Users/Carlos/Desktop/OpenOverlay /mnt/c/Users/Carlos/Desktop/OpenOverlay-ACTUAL
git checkout perf/frame-hook-test && ./generar-openoverlay.sh
cp -r /mnt/c/Users/Carlos/Desktop/OpenOverlay /mnt/c/Users/Carlos/Desktop/OpenOverlay-TEST
```

- Solo puede haber una instancia abierta a la vez (las dos comparten la configuración).
- La versión TEST se reconoce porque el título del panel pone "TEST sin bucle por frame".

### CapFrameX

- Proceso: `iRacingSim64DX11.exe`.
- Capturas de 120 s con tecla rápida.
- En el comentario de cada captura, la letra del escenario.
- Las capturas se guardan en `Documentos\CapFrameX\Captures`.
  - Son JSON con BOM: leerlos con `utf-8-sig`.
  - Traen `MsBetweenPresents`, `CpuActive`, `GpuActive`, `PresentMode`…
- `CreationDate` va en UTC (+2 h para la hora local) y marca el **final** de la captura.

### Registro de CPU y GPU por proceso

Script `Desktop\perf-procesos.ps1`. No hace falta mirar el Administrador de tareas.

1. Antes de empezar:
   ```powershell
   powershell -ExecutionPolicy Bypass -File "$env:USERPROFILE\Desktop\perf-procesos.ps1"
   ```
2. Dejarlo abierto toda la sesión y pararlo con Ctrl+C al terminar.
3. El resultado queda en `Documentos\CapFrameX\perf-procesos.csv` (`hora,proceso,cpu,gpu3d`, en hora local). Toma una muestra cada ~2 s.

Detalles:
- Usa las clases WMI `Win32_PerfFormattedData_PerfProc_Process` y `Win32_PerfFormattedData_GPUPerformanceCounters_GPUEngine`. No usa `Get-Counter` porque los nombres de sus contadores cambian con el idioma de Windows.
- **CPU:** % sobre la CPU entera (16 hilos del 5800X). 4.5 % ≈ 0.7 núcleos.
- **GPU:** solo el motor 3D.
- **Emparejar capturas con el CSV:** el tramo de cada captura va de `CreationDate` + 2 h − 120 s a `CreationDate` + 2 h. Comprobaciones:
  - en A no hay filas de `OpenOverlay`;
  - cada reinicio de sesión deja un hueco entre pruebas.

## Resultados

Equipo: Ryzen 7 5800X, RTX 4060 Ti, ventana sin bordes o pantalla completa optimizada. HAGS y Modo Juego desactivados.

### Ronda 1 (2026-10-02): no concluyente

- **Condiciones:** las de la escena de arriba, pero con capturas de 60 s y sin pasada de calentamiento, sin A al final y sin registro de CPU por proceso.
- Una pasada por escenario, salvo la versión actual con widgets, que tiene dos.

| Escenario | Avg | P1 | CPU activa/frame | GPU activa/frame |
|---|---|---|---|---|
| OpenOverlay cerrado | 98.7 | 74.4 | 10.1 ms | 6.0 ms |
| Actual, widgets cerrados | 92.9 | 72.3 | 10.6 ms | 6.1 ms |
| TEST, widgets cerrados | 91.9 | 71.4 | 10.8 ms | 6.0 ms |
| Actual, widgets habituales (2 pasadas) | 90.9 / 88.4 | 68.5 / 68.1 | 10.9 / 11.1 ms | 6.3 / 6.2 ms |
| TEST, widgets habituales | 90.1 | 67.1 | 10.9 ms | 6.1 ms |

- Las dos versiones parecían iguales.
- La ronda 2, con un método más controlado, contradice esta ronda en parte. Hay que fiarse de la 2.
- No se sabe si en esta ronda los widgets cerrados se habían abierto antes.

### Ronda 2 (2026-10-02): concluyente

- **Condiciones:** las del método de arriba, con una pasada por escenario y sin Brave ni Medal.
- "Antigua" es `relative_fixes` y "nueva" es `perf/frame-hook-test`.
- "Nunca abiertos" quiere decir que los widgets ya estaban cerrados al arrancar la app y no se abrieron en toda la sesión.

| Esc | Versión y widgets | Avg | P1 | 0.1 % low | CPU iRacing/frame | GPU/frame | CPU OO | GPU 3D OO |
|---|---|---|---|---|---|---|---|---|
| A | App cerrada | 106.5 | 70.2 | 61.3 | 9.17 ms | 5.50 ms | – | – |
| B | Antigua, nunca abiertos | 103.2 | 71.5 | 63.2 | 9.47 ms | 5.58 ms | 0.94 % | 0 % |
| D | Nueva, nunca abiertos | 105.8 | 73.8 | 65.4 | 9.24 ms | 5.57 ms | 0.38 % | 0 % |
| D\* | Nueva, abiertos y después cerrados | 100.7 | 63.3 | 38.8 | 9.71 ms | 5.62 ms | 0.43 % | 0 % |
| C | Antigua, todos abiertos | 87.6 | 61.6 | 54.8 | 11.12 ms | 6.48 ms | 4.65 % | 6.6 % |
| E | Nueva, todos abiertos | 92.5 | 63.5 | 55.9 | 10.59 ms | 6.22 ms | 4.48 % | 6.8 % |

OO es OpenOverlay. iRacing usa ~33 % de CPU y ~58 % de motor 3D en todas.

Media de FPS por tramos de 20 s:

| Esc | 0-20 s | 20-40 s | 40-60 s | 60-80 s | 80-100 s |
|---|---|---|---|---|---|
| A | 86 | 86 | 94 | 124 | 122 |
| B | 88 | 87 | 97 | 113 | 118 |
| D | 90 | 89 | 99 | 118 | 123 |
| D\* | 80 | 86 | 92 | 115 | 116 |
| C | 77 | 77 | 83 | 95 | 99 |
| E | 80 | 79 | 84 | 99 | 108 |

Conclusiones:
1. **Los widgets abiertos son la causa raíz.**
   - Se pierde un −18 % de FPS con la versión antigua y un −13 % con la nueva.
   - En los tramos rápidos son ~25-29 FPS menos.
   - Los frames de más de 25 ms solo aparecen con widgets abiertos (C: 2, E: 4) o en D\*.
2. **Quitar el bucle por frame funciona.** Con los widgets nunca abiertos, la versión nueva gana ~3 FPS y gasta la mitad de CPU. Con los widgets abiertos gana ~5 FPS.
3. **PresentMode** es igual (3) en todas las capturas. Los widgets no cambian el modo de presentación del juego, aunque el tiempo hasta mostrarse el frame sube de 0.75 a 1.0-1.16 ms con los widgets abiertos.
4. **D\* no es concluyente.**
   - La caída del 0.1 % low viene de una ráfaga de frames de más de 25 ms entre los segundos 4 y 6. Coincide con un pico de CPU de iRacing (68.9 %), no de OpenOverlay.
   - OpenOverlay usa la misma CPU que en D.
   - La diferencia con D en el resto de la captura está cerca del ruido.
   - Repetir antes de sacar conclusiones.
5. **Brave y Medal:** no se probaron. La pérdida grande se reproduce sin ellos, así que no son necesarios para explicarla.

### Ronda 3 (2026-10-02): coste de cada widget por separado

**Condiciones:**
- Versión nueva, mismo método y una pasada por escenario.
- Antes de cada prueba se reiniciaba la sesión, se reabría OpenOverlay y se abría **un solo widget**.
- A al principio y al final.

**Ajustes del usuario durante la prueba** (en `%LOCALAPPDATA%\IRacingOverlay`):
- `critical-refresh.txt = 0`: *Critical refresh* a **16 ms (60 Hz)**, el máximo. Por defecto es 67 ms.
- `compass-refresh.txt = 0`: la brújula de Weather **sigue cada actualización con animación**. Por defecto se actualiza cada 500 ms, sin animar.

**Deriva:** A pasó de 101.2 a 111.2 FPS en 43 minutos (+0.23 FPS/min). Cada escenario se compara con una referencia interpolada entre las dos A.
- Haciendo el ajuste también con los widgets baratos, el error típico es de ±1.9 FPS.
- Así que diferencias de menos de ~3 % son ruido.
- E6B (Delta, segunda captura) se tomó en otra parte de la carrera (tramos planos de 110-128 FPS) y no es comparable.

| Widget | Avg | Ref. interpolada | Diferencia | CPU iRacing/frame vs ref. | CPU OO | GPU 3D OO |
|---|---|---|---|---|---|---|
| **Pedal Trace** | 96.7 | 106.9 | **−9.6 %** | +0.96 ms | 1.06 % | 1.3 % |
| **Weather** | 100.1 | 108.9 | **−8.0 %** | +0.78 ms | 2.21 % | 2.5 % |
| **Cockpit** | 96.7 | 103.2 | **−6.3 %** | +0.63 ms | 1.34 % | 3.5 % |
| **Relative** | 96.2 | 101.9 | **−5.6 %** | +0.57 ms | 1.36 % | 0.8 % |
| **Standings** | 97.7 | 102.5 | **−4.7 %** | +0.46 ms | 1.82 % | 1.6 % |
| Delta (A) | 101.0 | 105.1 | −3.9 % | +0.37 ms | 0.40 % | 0 % |
| Track Map | 105.4 | 109.5 | −3.7 % | +0.34 ms | 0.51 % | 0 % |
| Track & session | 104.6 | 108.2 | −3.3 % | +0.30 ms | 0.50 % | 0.1 % |
| Fuel Calc | 109.1 | 110.6 | −1.3 % | +0.11 ms | 0.58 % | 0 % |
| Tires | 103.3 | 104.5 | −1.2 % | +0.09 ms | 0.63 % | 0.1 % |
| Flags | 103.3 | 103.8 | −0.5 % | +0.03 ms | 0.51 % | 0 % |
| Incidents | 107.4 | 107.6 | −0.2 % | −0.02 ms | 0.37 % | 0 % |
| Fuel | 108.2 | 106.3 | +1.8 % | −0.19 ms | 0.54 % | 0 % |

Conclusiones:
1. **Cinco widgets concentran el coste: Pedal Trace, Weather, Cockpit, Relative y Standings.** Son exactamente los que usan GPU en OpenOverlay. Los demás no usan GPU y no se distinguen del ruido.

   - **La pérdida de FPS no es proporcional a lo que consume OpenOverlay.** Restando la base de ~0.5 % de CPU que tiene cualquier widget, lo que añade cada uno es:

     | Widget | CPU extra | GPU | Pérdida de FPS |
     |---|---|---|---|
     | Pedal Trace | +0.55 % | 1.3 % | **−9.6 %** (la mayor) |
     | Weather | +1.7 % | 2.5 % | −8 % |
     | Cockpit | +0.85 % | 3.5 % | −6.3 % |
     | Relative | +0.9 % | 0.8 % | −5.6 % |
     | Standings | +1.3 % | 1.6 % | −4.7 % |

     Pedal Trace es el que menos consume y el que más FPS quita.
   - **La pérdida va mejor con la frecuencia de actualización.** Pedal Trace y Cockpit van a 60 Hz y la brújula animada va a la frecuencia del monitor: son los tres primeros. Relative y Standings van a 10 Hz, con más contenido.
   - **Hipótesis:** buena parte del coste no está dentro de OpenOverlay, sino en lo que provoca fuera. Cada actualización de una ventana transparente obliga a DWM (`dwm.exe`) a recomponer la pantalla, que compite con iRacing (su CPU por frame sube 0.5-1 ms). Pendiente de medir: el script registra `dwm` desde la ronda 4.
   - Los datos **no permiten separar** cuánta CPU de OpenOverlay es cálculo (`Feed`, builders) y cuánta es render de WPF. La línea de diagnóstico del panel (`UI x/y ms · critical x/y ms`) solo mide el cálculo y puede ayudar.
2. **Pedal Trace y Cockpit** se redibujan con el tick crítico, que con el ajuste del usuario va a 60 Hz:
   - `PedalTraceGraph` hace `InvalidateVisual()` en cada tick;
   - el Cockpit redibuja entero (`OnRender`).
3. **Weather** tiene la brújula en modo animado. `WindCompass.Point` lanza una `DoubleAnimation` de 350 ms en cada tick de 100 ms, porque el rumbo del coche cambia continuamente.
   - Una animación WPF siempre activa hace que el render corra a la frecuencia del monitor y que `OnRender` redibuje la brújula en cada frame.
   - En la práctica es el mismo bucle por frame que se quitó de `MainWindow`.
4. **Relative y Standings** se redibujan cada 100 ms, pero con muchas filas y celdas. Cuestan algo menos.
5. **Los costes no se suman.**
   - Por separado, los widgets habituales suman ~25-30 %, pero todos juntos cuestan −13 % (ronda 2, E).
   - Hay una parte fija: en cuanto alguna ventana transparente se redibuja a menudo, el render de WPF y DWM ya están trabajando, y cada ventana añade menos.
   - Por eso hay que bajar el redibujado de todas las caras, no solo de una.

### Ronda 4 (2026-10-02): precio de la fluidez, solo con ajustes

Mide cuánto rendimiento se gana bajando la frecuencia con los ajustes que ya existen y cuánta fluidez se pierde. Mismo método. En cada prueba, anotar también si se nota menos fluido.

**Dónde se cambia:** panel de control → **General** → sección **Performance**.
- **HIGH-RATE DISPLAYS › Refresh rate** (Cockpit, Pedal Trace, barras de proximidad y ABS):

  | Opción | Periodo |
  |---|---|
  | Fastest (~60 Hz) | 16 ms (ajuste actual del usuario) |
  | Fast (30 Hz) | 33 ms |
  | Normal (15 Hz) | 67 ms (valor por defecto) |
  | Slow (10 Hz) | 100 ms |
  | Slowest (5 Hz) | 200 ms |

- **WIND COMPASS › Refresh rate** (brújula de Weather):
  - Smooth (animated): ajuste actual del usuario;
  - 10 Hz, 5 Hz, **2 Hz** (valor por defecto) y 1 Hz.
  - Las opciones fijas saltan al ángulo nuevo, sin animar.

El cambio se aplica al momento. Se guarda en `%LOCALAPPDATA%\IRacingOverlay\critical-refresh.txt` y `compass-refresh.txt` (índices 0-4 en el orden de la lista).

| # | Situación |
|---|---|
| A | App cerrada (principio, mitad y final) |
| W-anim | Solo Weather, brújula Smooth (animated) |
| W-10 | Solo Weather, brújula a 10 Hz |
| P60 | Solo Pedal Trace, Fastest (~60 Hz) |
| P30 | Solo Pedal Trace, Fast (30 Hz) |
| K30 | Solo Cockpit, Fast (30 Hz) |
| T-old | Widgets habituales con los ajustes actuales (60 Hz y brújula animada) |
| T-new | Widgets habituales con 30 Hz y brújula a 10 Hz |

**Orden:** A → W-anim → W-10 → P60 → P30 → A → K30 → T-old → T-new → A.

Para comparar K30 basta con el Cockpit de la ronda 3 (60 Hz).

#### Resultados de la ronda 4


- **Referencia:** las tres A dieron 106.4, 112.9 y 107.1 FPS. No hay deriva lineal, así que se compara con su media (108.8). El ruido de la referencia es de ±3 %.
- **DWM** se registra desde esta ronda: CPU y motor 3D de `dwm.exe`.

| # | Avg | vs media A | CPU iRacing/frame | CPU OO | GPU 3D OO | CPU DWM | GPU 3D DWM |
|---|---|---|---|---|---|---|---|
| A / A2 / A3 | 106.4 / 112.9 / 107.1 | – | 9.17 / 8.64 / 9.11 ms | – | – | ~0.3 % | ~0.1 % |
| W-anim | 100.5 | **−7.6 %** | 9.72 ms | 1.93 % | 2.6 % | 1.70 % | 3.7 % |
| W-10 | 108.7 | −0.1 % | 8.98 ms | 0.56 % | 0.1 % | 0.42 % | 0.3 % |
| P60 | 107.1 | −1.6 % | 9.10 ms | 1.07 % | 1.5 % | 1.08 % | 2.4 % |
| P30 | 104.9 | −3.6 % | 9.30 ms | 0.72 % | 1.0 % | 0.74 % | 1.2 % |
| K30 | 107.9 | −0.8 % | 9.04 ms | 0.93 % | 2.1 % | 0.68 % | 1.7 % |
| T-old | 95.9 | **−11.9 %** | 10.19 ms | 4.34 % | 6.7 % | 1.50 % | 3.7 % |
| T-new | 96.3 | **−11.5 %** | 10.16 ms | 3.46 % | 6.1 % | 1.23 % | 3.4 % |

Conclusiones:
1. **La brújula animada está confirmada:** −7.6 % → ≈0 % con 10 Hz. Por sí sola, el usuario no nota diferencia de fluidez a 10 Hz.
2. **Pedal Trace y Cockpit cuestan poco.** A 60 o 30 Hz quedan dentro del ruido, y el usuario apenas nota la diferencia de fluidez. El −9.6 % de Pedal Trace en la ronda 3 no se reproduce.
3. **Con todos los widgets, bajar las frecuencias no recupera FPS.**
   - T-new reduce el consumo de OpenOverlay (CPU de 4.34 % a 3.46 %), pero los FPS son los mismos (−12 %).
   - El coste que queda no viene de Cockpit, Pedal Trace ni la brújula.
   - Candidatos: Relative y Standings (−5 % cada uno en la ronda 3) y el coste combinado de muchas ventanas.
4. **Con todos abiertos, los widgets se ven a tirones** (T-old y T-new), cuando por separado se ven fluidos aunque vayan a la mitad de Hz.
   - Todas las ventanas comparten **un hilo de UI** (el `Dispatcher`, donde corren los `Feed` y el tick crítico) y **un hilo de render** de WPF.
   - Si un tick de Relative o Standings tarda decenas de ms, retrasa a los demás.
   - Es la hipótesis principal ahora. La línea de diagnóstico (`UI x/y ms · critical x/y ms (worst gap)`) lo puede confirmar.
5. **DWM sube con los widgets** (de 0.3 % a 1.2-1.7 % de CPU y a ~3.7 % de motor 3D), pero en valores absolutos es poco. No basta para explicar el −12 % por sí solo.
6. **Caída grande de FPS justo después de P60, con aviso de CPU:**
   - En el CSV, OpenOverlay y DWM estaban en ~0-1 %. iRacing subió su motor 3D de ~60 % a 90-96 % (23:16:30-23:17:30), mientras se hacía el reset a boxes.
   - No hay rastro de que fuera OpenOverlay.
   - El script solo registra los procesos de la lista, así que no puede descartar otro proceso de Windows.

### Lectura de la línea de diagnóstico (2026-10-03)

- **Condiciones:** versión nueva, todos los widgets habituales, *Critical refresh* a 30 Hz, brújula a 10 Hz y 5 minutos en carrera.
- **Cómo se lee la línea:**
  - `UI` y `critical` son la media y el máximo de la última ventana de ~1 s.
  - `GC` son colecciones acumuladas de las generaciones 0, 1 y 2 desde que arrancó la app.

| Momento | Memoria | GC gen0/1/2 | UI media/máx | critical media/máx | Peor hueco (objetivo 33 ms) |
|---|---|---|---|---|---|
| 0:30 | 228 MB | 6 / 4 / 2 | 1.0 / 8.8 ms | 0.0 / 0.0 ms | 67 ms |
| 2:00 | 301 MB | 189 / 66 / 23 | 9.1 / 17.8 ms | 0.1 / 0.2 ms | 78 ms |
| 3:19 | 310 MB | 365 / 124 / 41 | 10.1 / 25.5 ms | 0.1 / 0.3 ms | 79 ms |
| 5:09 | 297 MB | 613 / 207 / 69 | 9.0 / 18.0 ms | 0.1 / 0.1 ms | 79 ms |

Conclusiones:
1. **Recolección de basura muy alta en carrera.**
   - Entre 2:00 y 5:09 hay ~2.2 colecciones de gen0 por segundo, ~0.75 de gen1 y **una de gen2 cada ~4 s**.
   - A los 0:30, aún sin carrera en marcha, casi no había.
   - Una gen2 es una colección completa de un heap de ~300 MB: gasta CPU en varios hilos y puede pausar el hilo de UI.
   - Significa que OpenOverlay **crea muchísimos objetos temporales** en cada tick con 28 coches. Es probablemente el mayor desperdicio de CPU y el sospechoso de los tirones.
2. **El tick de 100 ms cuesta 9-10 ms de media**, unas 10 veces más que antes de empezar la carrera, con picos de 18-25 ms.
   - Ocupa ~10 % del hilo de UI: no lo satura, pero cada pico bloquea a los demás widgets.
   - El tiempo de layout y render de WPF que viene después de los `Feed` **no** se cuenta aquí.
3. **El cálculo del tick crítico es despreciable** (0.1 ms), pero el **peor hueco entre ticks es de 67-79 ms** con un objetivo de 33 ms.
   - Cockpit y Pedal Trace se saltan 1-2 actualizaciones, bloqueados por el tick de 100 ms, el layout y render que lo sigue y las pausas de GC.
   - Esto encaja con los tirones que el usuario ve con todos los widgets abiertos.
4. **Siguiente paso:** buscar en código las asignaciones por tick (builders y paneles de Relative y Standings, strings, LINQ, `FormattedText`, brushes, filas recreadas). Reducirlas debería bajar el GC, el tiempo del tick y los huecos.
   - Para medir la tasa de asignación exacta: `dotnet-counters monitor -n OpenOverlay System.Runtime`, mirando *Allocation Rate* y *% Time in GC*.

### Revisión de código: Relative y Standings (2026-10-03)

Medido fuera de la app con un programa de prueba (`%TEMP%\oo-probe`):
- usa el `RelativePanel` real y su plantilla de filas (`Themes/DriverTable.xaml`), con 9 filas;
- cada tick crea filas nuevas, como hace la app;
- mide la memoria asignada y el tiempo de `SetRows` + layout.

**Hallazgos:**
1. **Cada tick creaba ~1.45 MB de basura y tardaba ~16 ms (pico de 42 ms)**, solo con Relative y 9 filas.
   - Con valores idénticos costaba lo mismo, así que el coste no está en los datos que cambian, sino en el mecanismo de actualizar las filas.
   - Standings tiene más filas y usa la misma plantilla. Esto explica el GC muy alto y los ticks de 10-25 ms de la línea de diagnóstico.
2. **Causa principal: las filas no implementan `INotifyPropertyChanged`.**
   - Para cada uno de los ~45 enlaces de cada fila, WPF engancha un `PropertyDescriptor.AddValueChanged` (`ValueChangedEventManager`) y lo rehace en cada fila nueva.
   - En el perfil de asignaciones (`GCAllocationTick`) eran la mayoría: `EventHandler<ValueChangedEventArgs>`, `MoreEventInfo`, `EventKey`, `WeakReference`, `ValueChangedRecord`…
   - Probar con un `INotifyPropertyChanged` vacío solo baja un 20 %, porque el gestor de eventos débiles sigue suscribiéndose.
3. **Causa secundaria: `Rows[i] = fila` (Replace)** vuelve a enganchar el contenedor de cada línea y reevalúa todos sus enlaces, también los de `Options.*` con `RelativeSource`.

**Cambios** (rama `perf/driver-table-bindings`, sobre `perf/frame-hook-test`). No tocan la lógica de negocio: los builders y los datos de las filas son los mismos.

| Cambio | Archivo | Efecto |
|---|---|---|
| Enlaces a datos de la fila en `Mode=OneTime` (41, incluidos los de los `DataTrigger`) | `Themes/DriverTable.xaml` | Las filas son inmutables; `OneTime` se reevalúa al cambiar el `DataContext` y no se suscribe a cambios |
| Huecos fijos (`RowSlot`): la lista no cambia de elementos, solo el `Value` de cada hueco; `ItemTemplate` con `ContentPresenter Content="{Binding Value}"` | `Widgets/RowSlot.cs`, `RelativePanel`, `StandingsPanel` (.xaml y .cs) | El árbol visual de cada línea se conserva; solo cambia su contexto de datos |

Medido con el `RelativePanel` real y 9 filas:

| Versión | Memoria/tick | Tiempo medio | Pico |
|---|---|---|---|
| Antes | 1446 KB | ~16 ms | 42 ms |
| Solo `OneTime` | 891 KB | ~15 ms | 22 ms |
| `OneTime` + huecos fijos | **594 KB (−59 %)** | **~4-7 ms** | **12-14 ms** |

- Lo que queda es sobre todo trabajo de texto de WPF (`TextFormatting`, `GlyphRun`), inevitable cuando cambian los números.
- **Comprobado en la prueba:**
  - los textos muestran los valores de la fila nueva;
  - los `DataTrigger` (fila del jugador, boxes, doblados) se aplican y se quitan bien al cambiar el contenido de un hueco.
- Tests: App 546/546.
- Distintivo de la build: `FrameHookTestTag = "TEST filas fijas"`.

**Otros hallazgos sin tocar:**
- Aparecen `FrameworkElementAutomationPeer` en las asignaciones: algún cliente de UI Automation está activo en el equipo. Son pocos KB por tick. Se podría desactivar en las ventanas de overlay.
- Los builders (`StandingsBuilder`) no se han perfilado: son lógica de negocio y faltan datos de carrera reproducibles.

### Primera prueba de "filas fijas" (2026-10-03): no concluyente para FPS

- **Condiciones:** dos capturas de 120 s, primero T (todos los widgets, build "filas fijas") y luego A. Una pasada de cada una, sin la build anterior en la misma sesión.

| | Avg | P1 | 0.1 % low | Frames > 25 ms | CPU iRacing/frame | CPU OO | GPU 3D OO | CPU / GPU DWM |
|---|---|---|---|---|---|---|---|---|
| T, filas fijas | 86.0 | 57.3 | 39.5 | 11 | 11.38 ms | **2.26 %** | 7.7 % | 2.40 % / 3.9 % |
| A | 95.0 | 68.5 | 57.4 | 2 | 10.29 ms | – | – | 0.63 % / 0.0 % |
| *Ronda 4, T-new (build anterior)* | *96.3* | *65.6* | *54.8* | *4* | *10.16 ms* | *3.46 %* | *6.1 %* | *1.23 % / 3.4 %* |

- **CPU de OpenOverlay:** baja un **35 %** respecto a la ronda 4 con los mismos widgets y ajustes (3.46 % → 2.26 %). Encaja con lo medido en la prueba aislada.
- **FPS:**
  - −9.5 % frente a A de la misma sesión, contra −11.5 % antes. La diferencia está dentro del ruido de una sola pasada.
  - La referencia A de esta sesión (95.0) está muy por debajo de las anteriores (106-113), y DWM sin la app también está más alto (0.63 % frente a ~0.3 %). Algo era distinto en el equipo o en la sesión.
- **Grabación de pantalla:** durante la prueba el usuario grababa la pantalla, lo que invalida la comparación de FPS y DWM con rondas anteriores. Falta saber si grababa también durante A.
- **Línea de diagnóstico al final de T:** `378 MB · GC 885/363/20 · UI 6,7/14,3 ms · critical 0,3/6,5 ms (target 33, worst gap 47)`.
  - Antes, con la build anterior a los 5:09: `GC 613/207/69 · UI 9,0/18,0 · worst gap 79`.
  - **Gen2 en proporción a gen0: del 11 % al 2.3 %.** Las colecciones completas casi desaparecen. No se sabe cuánto duró esta sesión, así que no se pueden comparar tasas por segundo.
  - **Tick de UI ~30 % más rápido** (media 9-10 → 6.7 ms, máx. 18-25 → 14 ms).
  - **Peor hueco del tick crítico: de 79 a 47 ms.**
- **Percepción del usuario:** el juego en general algo más fluido; Cockpit y Pedal Trace a 30 Hz se ven más o menos igual.
- **Frames de más de 25 ms en T:** 11, repartidos. Los de los segundos ~103-106 coinciden con un pico de GPU de OpenOverlay (26 % a las 00:47:35). Posible animación (¿alerta de Incidents?), sin confirmar.

### Medición dentro de la app (`PerfProbe`, temporal)

- Rama `perf/driver-table-bindings`, distintivo "TEST filas fijas + perf log". Clase `Diagnostics/PerfProbe.cs`.
- Cada 5 s escribe una línea `"source":"Perf"` en `%LOCALAPPDATA%\IRacingOverlay\logs\openoverlay-AAAAMMDD.log` con:
  - **GC del proceso:** colecciones por segundo (gen0/gen1/gen2), MB asignados por segundo, ms de pausa de GC por segundo y tamaño del heap;
  - **por sección:** llamadas, media y máximo en ms, KB asignados por llamada en el hilo de UI y llamadas de más de 50 ms. Ordenadas de más a menos tiempo total.
- **Secciones:**

  | Sección | Qué mide |
  |---|---|
  | `ui.tick` | Tick de 100 ms completo |
  | `trackers` | Pit stops, cruces de línea, `EstTimeProfile`, lap log y penalizaciones |
  | `standings.order` | `BuildStandings` compartido |
  | `<widget>.build` / `.apply` / `.dashboard` | Cada `Feed` |
  | `ui.after-tick (layout+render)` | Del final del tick hasta `ContextIdle`: layout, render y lo que haya en cola |
  | `critical.tick` | Tick crítico (Cockpit y Pedal Trace) |
  | `critical.queue-wait` | Cuánto espera un tick nuevo de telemetría a que el hilo de UI lo atienda: origen de los huecos |

- **Para quitarla:** borrar `PerfProbe.cs` y sus llamadas en `MainWindow.xaml.cs` (todas llevan `PerfProbe.`), o poner `Enabled = false`.

### Resultado del log `PerfProbe` en carrera (2026-10-03)

- **Condiciones:** build "filas fijas + perf log", todos los widgets, 30 Hz y brújula a 10 Hz, dos vueltas a Road Atlanta, sin grabar y con el panel de control minimizado.
- El log deduplicó las líneas iguales y solo dejó una por minuto, así que hay 3 ventanas útiles. Corregido después numerando las líneas.

**Proceso:**
- GC: **1.0 gen0/s, 0.2-0.4 gen1/s y 0 gen2**;
- 15-18 MB/s asignados;
- 6-8 ms de pausa de GC por segundo (<1 %);
- heap de ~25 MB.

En la prueba anterior había ~5 gen0/s. La diferencia probablemente se debía al panel de control visible mientras se grababa.

**Hilo de UI por tick de 100 ms** (media/máx.):

| Sección | ms | KB/llamada |
|---|---|---|
| `ui.after-tick (layout+render)` | 7.5-9.5 / 15-48 | 850-970 |
| `ui.tick` completo | 5.7-6.6 / 13-20 | ~660 |
| ↳ `Standings.apply` (×2 por tick: filas y progreso) | 1.4 / 4.5-9.8 | 138 |
| ↳ `Relative.apply` (×2) | 0.6-0.8 / 1.6-9.2 | 73 |
| ↳ `TireInfo.apply` | 0.5-0.9 / 1-8 | **165** |
| ↳ Todos los `*.build` (lógica de negocio), `trackers`, `standings.order` | ≤0.1 | ≤20 |
| `critical.tick` (30 Hz) | 0.1 | 7 |
| `critical.queue-wait` | 0.8-1.1 / **18-38** | – |

Conclusiones:
1. **La lógica de negocio es barata:** todos los builders juntos tardan menos de 0.5 ms por tick. El coste está en la capa WPF: aplicar los datos a la UI y, sobre todo, el layout y render posterior (~8 ms y ~900 KB por tick).
2. **El hilo de UI está ocupado ~14 % del tiempo**, sin saturarse. Pero mientras dura un tick con su layout (~15 ms, a veces 48), el tick crítico espera hasta 18-38 ms. Son los saltos que se ven en Cockpit y Pedal Trace.
3. **TireInfo** tenía el mismo problema que las filas: enlaces sin `OneTime` sobre objetos nuevos en cada tick.

### Pinceles cacheados y TireInfo (2026-10-03)

**Hallazgo:** los colores de filas y neumáticos son strings (`"#F2F5F8"`) enlazados directamente a `Foreground`, `Fill` o `Background`.
- El conversor por defecto crea un `SolidColorBrush` nuevo en cada evaluación. Como es otra instancia, WPF lo trata como un valor cambiado y vuelve a formatear y pintar el texto.
- Eso explica gran parte de `TextFormatting`, `GlyphRun` y `FullTextLine` en las asignaciones.

**Cambios** (sin tocar lógica de negocio):
- `Converters/CachedBrushConverter.cs`: string → pincel congelado, cacheado por string, con la misma conversión que WPF. Registrado en `DesignTokens.xaml`.
- Aplicado a 7 enlaces de color en `DriverTable.xaml` y 6 en `TireInfoPanel.xaml`.
- `TireInfoPanel.xaml` (`CornerTemplate`): 19 enlaces a `Mode=OneTime`. `TireCornerInfo` es inmutable.

**Medido con el programa de prueba** (Relative con 9 filas y TireInfo):

| Caso | Original | Filas fijas + OneTime | + pinceles cacheados |
|---|---|---|---|
| Relative, solo cambian los gaps (caso realista) | 1405 KB | 554 KB | **290 KB** |
| Relative, valores idénticos | 1372 KB | 521 KB | **189 KB** |
| TireInfo | 306 KB | 181 KB (OneTime) | **91 KB** |

- Textos y `DataTrigger` comprobados de nuevo.
- Tests: App 546/546.
- Build: "TEST filas fijas + pinceles + perf log".

### TireInfo sin redibujar si no cambia nada (2026-10-03)

- **Idea del usuario:** iRacing solo actualiza presiones, temperaturas y desgaste en el box, pero TireInfo recibía un objeto nuevo en cada tick.
- **Cambio:** `TireInfoPanel.UpdateState` compara cada rueda con la que ya muestra, usando exactamente las propiedades que pinta `CornerTemplate`. Si son iguales, no la toca. No supone nada sobre cuándo cambian los datos, así que nunca queda un valor desactualizado.
- **Medido con el programa de prueba:**
  - con valores sin cambios, **5 KB y 0.04 ms por tick** (antes 306 KB);
  - al cambiar la presión se actualiza al momento.
- Build: "TEST optimizaciones UI v3 + perf log". Tests 546/546.

### Log `PerfProbe` con la build v3 (2026-10-03)

- **Condiciones:** las de la prueba anterior, con 66 ventanas de 5 s.
  - Hasta las 23:20:44, la app abierta sin carrera.
  - De 23:21:40 a 23:25:07, carrera; la primera ventana incluye el arranque, con un tick de 636 ms que no cuenta.
- Sin capturas de FPS.

Valores típicos en carrera (media/máx.), comparados con la build "filas fijas + perf log":

| Sección | Antes | v3 |
|---|---|---|
| `ui.after-tick (layout+render)` | 7.5-9.5 / 15-48 ms, 850-970 KB | **~3.0 / 5-14 ms, ~210 KB** (picos aislados de 46-50 ms) |
| `ui.tick` | 5.7-6.6 / 13-20 ms, ~660 KB | **~5.3 / 13-14 ms, ~460 KB** |
| ↳ `Standings.apply` (×2) | 1.4 ms, 138 KB | 1.3 ms, 125 KB |
| ↳ `Relative.apply` (×2) | 0.6-0.8 ms, 73 KB | 0.6 ms, 66 KB |
| ↳ `TireInfo.apply` | 0.5-0.9 ms, 165 KB | **0.0 ms, 3 KB** |
| `critical.queue-wait` | 0.8-1.1 / 18-38 ms | **0.4-0.5 / 9-18 ms** (picos aislados de 34-41 ms) |
| Hilo de UI ocupado por tick | ~14 ms | **~8 ms** |
| Memoria asignada | 15-18 MB/s | **8-10 MB/s** |
| GC | 1.0 / 0.2-0.4 / 0 por s | 0.6 / 0.6 / 0 por s |

Conclusiones:
1. **El layout y el render después del tick bajan a un tercio** (~8 → ~3 ms). Son los colores cacheados: el texto ya no se vuelve a formatear si su color no cambia.
2. **La espera del tick crítico se reduce a la mitad.** El peor caso típico baja de 18-38 a 9-18 ms, que a 30 Hz casi nunca llega a saltarse una actualización.
3. **Lo que más pesa ahora es `Standings.apply` + `Relative.apply`** (~4 ms y ~380 KB por tick).
   - Es la reevaluación de los enlaces `OneTime` y los getters de formato de cada fila nueva.
   - Siguiente posible mejora: no tocar una fila si muestra exactamente lo mismo que antes, como en TireInfo. En Standings, probablemente la mayoría de filas no cambian en cada tick.

## Próximas pruebas

### Paso siguiente: filas de Standings y Relative que no cambian

**Situación con v3:** lo que más pesa del hilo de UI es `Standings.apply` + `Relative.apply`, ~4 ms y ~380 KB por tick.
- Cada tick llegan filas nuevas y cada hueco reevalúa sus ~45 enlaces y getters de formato, aunque la fila muestre exactamente lo mismo que antes.
- En Standings, probablemente la mayoría de filas no cambian de un tick a otro: posición, nombre, vueltas y tiempos solo cambian al cruzar la línea, salvo que el gap sea en vivo.
- En Relative cambian los gaps de casi todas las filas, así que se espera menos ganancia.

**Plan** (sin tocar la lógica de negocio):
1. **Medir antes de cambiar.** Añadir al log `PerfProbe` cuántas filas llegan idénticas en cada tabla (`Standings.rows-unchanged`, `Relative.rows-unchanged`). Así se sabe si merece la pena.
2. **Si merece la pena:** en `RowSlot.Sync`, no cambiar el `Value` de un hueco si la fila nueva muestra lo mismo que la que ya tiene.
   - La comparación tiene que cubrir **todo lo que pinta la plantilla**, también el tipo de fila, el `DataTrigger` (`IsPlayer`, `OnPitRoad`, `LapRelation`, `IsMultiClass`, `IRatingTrend`…) y los colores.
   - Opción segura: comparar todas las propiedades públicas legibles del tipo, con getters cacheados. Las de tipo referencia que no tengan igualdad por valor cuentan como distintas, así que en caso de duda se actualiza.
   - Comprobar con el programa de prueba que una fila que cambia se actualiza y una igual no hace trabajo.
3. **Validar** con el log `PerfProbe` en carrera y, después, con una ronda de FPS en la misma sesión: A → v3 → v4 → A → v4 → v3 → A. Sin grabar pantalla y con el panel minimizado.

**También pendiente:**
- Probar si con v3 se pueden volver a poner 60 Hz y la brújula animada sin perder fluidez ni FPS.
- Pasar todo a la rama principal sin el distintivo TEST. Decidir si `PerfProbe` se queda desactivado (`Enabled = false`) o se elimina.

### Ideas para mantener la fluidez y ganar rendimiento (sin probar)

Según la ronda 3, la pérdida va con la frecuencia de actualización de las ventanas más que con lo que consume OpenOverlay. Cada redibujado de una ventana con `AllowsTransparency=true` obliga a WPF a renderizar la ventana entera y copiarla de la GPU a la memoria del sistema (`UpdateLayeredWindow`), y a DWM a recomponer. Por eso las ideas atacan el coste de cada redibujado, de más a menos impacto previsto:

1. **Transparencia sin ventana en capas.** Usar `AllowsTransparency=false` con transparencia compuesta por DWM, por ejemplo con `WindowChrome` y el cristal extendido a toda la ventana. Así se ahorra la copia GPU→CPU de cada frame y la fluidez es la misma. Hay que comprobar que siguen funcionando la transparencia por píxel, el clic a través y el modo siempre encima sobre iRacing en ventana sin bordes. Prototipo con Pedal Trace y medir.
2. **No redibujar si no cambia nada visible.**
   - El Cockpit hace `InvalidateVisual()` en cada tick de telemetría, aunque los valores mostrados sean idénticos (parado o en recta a fondo). Habría que comparar con el estado anterior ya redondeado.
   - Lo mismo para Relative y Standings: solo tocar las celdas que cambian.
3. **Separar lo estático de lo dinámico.** Fondo, marcos y etiquetas en una capa con `BitmapCache`, y redibujar solo agujas, barras y números. Para la brújula, girar una capa cacheada con `RotateTransform` en vez de volver a ejecutar `OnRender` en cada frame de la animación.
4. **Ventanas más pequeñas.** El coste de la copia es proporcional al área de la ventana, así que conviene ajustar el tamaño al contenido y evitar márgenes transparentes grandes.

