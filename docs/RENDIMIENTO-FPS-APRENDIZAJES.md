# Pérdida de FPS con OpenOverlay abierto: investigación

Estado al 2026-10-02, tras dos rondas de medición. Documento de traspaso para retomar la investigación sin perder el hilo.

## Resumen del estado

- **Causa raíz: tener los widgets abiertos.**
  - Con todos los widgets abiertos, iRacing pierde un **13-18 %** de FPS de media y **~25-29 FPS** en los tramos rápidos, que es lo que se percibía.
  - OpenOverlay usa entonces ~0.7 núcleos de CPU y ~7 % del motor 3D de la GPU.
- **El bucle por frame (`CompositionTarget.Rendering`) era un coste real pero pequeño.**
  - Supone ~3 FPS con los widgets cerrados y ~5 FPS con los widgets abiertos.
  - Está quitado en `perf/frame-hook-test`, que debe pasar a la rama principal (sin el distintivo TEST).
- **Pendiente:** averiguar qué widget concentra el coste (ronda 3) y optimizarlo.

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

### Ventanas transparentes y redibujado: lo más probable, pendiente de aislar

- Todos los widgets son ventanas con `AllowsTransparency=true`. WPF las actualiza copiando cada frame de la GPU a memoria del sistema (`UpdateLayeredWindow`).
- El **Cockpit** se redibuja entero hasta 60 veces por segundo (*Critical refresh* en General › Performance). Relative, Standings y el resto se redibujan cada 100 ms.
- Con los widgets abiertos, el tiempo de CPU de iRacing por frame sube de 9.2 a 10.6-11.1 ms y OpenOverlay usa ~7 % del motor 3D. Encaja con esta hipótesis, sumada al trabajo de los `Feed`.

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

## Próximas pruebas

### Ronda 3: aislar qué widget cuesta

- Con la versión nueva y el mismo método.
- Hay que **reiniciar la app antes de cada escenario** y abrir solo los widgets de ese escenario. Así los demás quedan "nunca abiertos".
- Orden: A → E → E1 → E2 → E3 → E4 → D\* → A.

| # | Situación |
|---|---|
| A | App cerrada (referencia, al principio y al final) |
| E | Todos los widgets habituales (referencia con widgets) |
| E1 | Todos menos el Cockpit |
| E2 | Solo el Cockpit |
| E3 | Solo Relative y Standings |
| E4 | Todos, con *Critical refresh* a 200 ms |
| D\* | Repetir: abrir los widgets, cerrarlos y luego capturar |

Cómo interpretar los resultados:
- **E1 se acerca a A y E2 cuesta casi tanto como E:** el problema es el Cockpit. Hay que atacar su redibujado completo a alta frecuencia: dibujar solo lo que cambia, bajar la frecuencia por defecto o evitar la ventana transparente.
- **E4 alivia mucho:** el coste depende de la frecuencia de redibujado. Hay que revisar el valor por defecto de *Critical refresh*.
- **E3 cuesta mucho:** el problema está en las tablas de 100 ms, en el trabajo de los `Feed` con 28-40 coches o en su redibujado. Hay que perfilar `UiTimer_TickCore`.
- **El coste se reparte entre todos los widgets:** apunta al coste fijo de cada ventana transparente (`UpdateLayeredWindow`). Hay que valorar reducir el número de ventanas o su tamaño.

### Después de la ronda 3

1. Perfilar con PerfView o dotnet-trace el escenario que más cueste, en carrera.
2. Pasar `perf/frame-hook-test` a la rama principal sin `FrameHookTestTag`. Se puede hacer ya, porque es independiente de la ronda 3.
3. Opcional: repetir E con Brave y Medal abiertos para ver si se suman costes.
