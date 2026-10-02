# Relative y Standings: cronometraje — aprendizajes y estado

Documento de trabajo para retomar el hilo sobre cómo OpenOverlay calcula el **Relative** (gaps alrededor del piloto) y el **Standings** (clasificación). Recoge las cuatro iteraciones hechas entre el 1 y el 2 de octubre de 2026, lo que salió mal en la prueba en vivo, lo que se aprendió de la telemetría de iRacing y el diseño vigente.

**Para retomar:** lee primero [Estado actual](#1-estado-actual), luego [Hipótesis abiertas](#6-hipótesis-abiertas-y-cómo-verificarlas) y [Próximos pasos](#7-próximos-pasos). La [historia](#2-historia-de-las-iteraciones) explica por qué no hay que volver a ciertas ideas.

---

## 1. Estado actual

Rama `bugs_fixes_0.8.0`. El commit `4e8aca0` ("Refactor lap time calculations and introduce ReferencePace class") contiene la **iteración 2**, ya descartada. La **iteración 3** (la vigente) está **sin commitear** encima de ese commit.

| Pieza | Fichero | Qué hace hoy |
|---|---|---|
| Relative | `IRacingOverlay.App/ViewModels/StandingsBuilder.cs` → `BuildRelative` | Gaps en segundos a partir de `CarIdxEstTime`, en el reloj del coche del jugador; en carrera, POS en vivo por distancia de carrera |
| Curva de tiempos estimados | `IRacingOverlay.App/ViewModels/EstTimeProfile.cs` | Aprende en vivo `LapDistPct → EstTime` del coche del jugador |
| Standings (carrera) | `StandingsBuilder.cs` → `BuildStandings` | Orden oficial `CarIdxPosition`; gap medido al cruzar meta; `+nL` para doblados |
| Pasos por meta | `IRacingOverlay.App/ViewModels/LineCrossingTracker.cs` | Instante en que cada coche completa cada vuelta, interpolado entre ticks |
| Posición en pista | `IRacingOverlay.App/ViewModels/TrackPosition.cs` | `CarIdxLap + CarIdxLapDistPct`; distancia de carrera y distancia plegada a ±½ vuelta |
| Cableado | `IRacingOverlay.App/MainWindow.xaml.cs` | Trackers actualizados en cada tick (10 Hz); Standings reconstruido en cada tick |
| Formato de gaps | `RelativeRow.cs`, `StandingsRow.cs` | `NaN` → "—"; `LapsDown > 0` → `+1L` |
| Eliminado | `ReferencePace.cs` | Ritmo de clase por mediana de vueltas: descartado tras la prueba en vivo |

Tests: 543 de la app y 48 del SDK en verde, 0 warnings. **Sin probar todavía en una sesión real.**

---

## 2. Historia de las iteraciones

### Iteración 0 — el código original (antes del 1 de octubre)

- **Relative:** gap = diferencia de `CarIdxEstTime / CarClassEstLapTime` de cada coche (fracción de vuelta en tiempo), plegada a ±½ vuelta y multiplicada por el `CarClassEstLapTime` del jugador. Si faltaba el tiempo estimado, usaba la diferencia de `EstTime` en bruto plegada con un tiempo de referencia medido.
- **Standings:** orden continuo por `(CarIdxLap, EstTime/EstLapTime)` con desempate por `CarIdxPosition`. Gap al líder de clase = diferencia × reloj estimado de la clase. Reconstrucción una vez por segundo.
- **Queja del usuario:** desviaciones respecto al relativo oficial de iRacing, sobre todo en multiclase.
- **Nota histórica:** una versión aún anterior del Standings usaba `CarIdxPosition`/`CarIdxF2Time` oficiales. Se cambió a orden continuo tras un informe de "la clasificación solo se actualiza al acabar la vuelta". En la iteración 3 ese comportamiento en meta se ha recuperado **a propósito**.

### Iteración 1 — posición física y ritmo medido (primer encargo)

Encargo: que la posición salga solo de `CarIdxLap + CarIdxLapDistPct`, y que los segundos sean una capa posterior basada en un ritmo de referencia común, sin depender de `CarClassEstLapTime`.

- Se creó `TrackPosition`. El orden del Relative pasó a la distancia plegada.
- Gap = distancia × ritmo reciente de la clase del jugador: media de la última vuelta de cada coche, descartando vueltas por encima del 107 % de la mejor. `CarClassEstLapTime` solo como semilla antes de la primera vuelta.

### Iteración 2 — desacoplar y unificar (segundo encargo, commit `4e8aca0`)

- `TrackPosition.RaceDistance` (absoluta) separada de `OnTrackGapTo` (plegada).
- `ReferencePace`: **mediana** de las últimas vueltas válidas por clase. Respaldo: mediana de mejores vueltas de la sesión anterior del fin de semana. Sin nada: gap "—". `CarClassEstLapTime` eliminado.
- `RaceOrder` compartido: Standings ordenado continuamente por distancia de carrera. El Relative calculaba su POS en cada tick con el mismo orden, para evitar el desfase de ~1 s respecto al Standings.
- El Cockpit pasó a usar `TrackPosition.FoldToHalfLap`.

### Prueba en vivo de la iteración 2 → **regresión** (informe del usuario)

1. Gaps vacíos durante la vuelta de formación y la primera vuelta de carrera.
2. Tras un incidente y una entrada a boxes en las primeras vueltas, los gaps se alejaron mucho de los de iRacing, **también entre coches de la misma clase**.
3. El usuario pidió revisar qué aportaba `CarIdxEstTime` antes de seguir cambiando nada, y además cambiar la filosofía del Standings (ver iteración 3).

### Iteración 3 — la vigente (sin commitear)

- **Relative:** vuelve a `CarIdxEstTime` para los segundos, leído en el reloj del jugador mediante una curva aprendida (`EstTimeProfile`). Detalle en §5.1.
- **Standings:** clasificación tradicional. Orden oficial, gaps medidos en meta y fijos hasta el siguiente paso, indicadores en vivo refrescados en cada tick. Detalle en §5.3.

---

## 3. Prueba en vivo: análisis con datos

Fichero analizado: `C:\Users\Carlos\Documents\iRacing\telemetry\porsche992rgt3_roadatlanta full 2026-10-02 16-02-47.ibt` (`/mnt/c/...` desde WSL).

**La sesión:**
- Road Atlanta, 4,0569 km. Carrera **sin práctica ni clasificación previa** en el YAML: una sola sesión, `SessionNum 0`, tipo Race.
- 3 clases y 32 pilotos. `CarClassEstLapTime` por coche:
  - prototipos: 65,33–66,67 s (6 valores distintos);
  - GT3: 75,09–75,82 s (**10 valores distintos, uno por modelo con BoP**);
  - tercera clase: 79,28 y 81,42 s.
- Jugador: CarIdx 0, Porsche 911 GT3 R (992), `CarClassEstLapTime` 75,3978 s, CarClassID 4091.

**Cronología del jugador** (`SessionTime`):
- `SessionState` 3 (vuelta de formación) desde el principio de la grabación.
- Bandera verde a t = 333,2 s: `SessionState` pasa a 4 y `Lap` pasa de 0 a 1 en ese mismo instante, así que el contador de vueltas sube al cruzar la línea en la salida.
- Vuelta 1: 6 puntos de incidente.
- Boxes entre t ≈ 506 y 534 s, al final de la vuelta 2.
- Duraciones de vuelta: vuelta 3 ≈ 98,6 s (parada) y vuelta 4 = **80,23 s** (limpia).
- Rareza sin explicar: `LapLastLapTime` leído en el primer tick de cada vuelta no coincide con la vuelta que acaba de terminar (por ejemplo, 98,952 al empezar la vuelta 3, cuando la vuelta 2 duró ~90,3 s). Parece actualizarse con retraso.

**Por qué salieron vacíos los gaps:** la capa de ritmo necesitaba tiempos de vuelta medidos y no había ninguna sesión previa. Fallo del diseño de la iteración 2.

**Por qué no cuadraban en la misma clase (medido):** en la vuelta 4, si un coche va exactamente 1,0 s por detrás en la misma trazada, convertir su distancia a segundos con el ritmo medio daría:

| Estadístico | Lectura de un gap real de 1,00 s |
|---|---|
| mínimo | 0,47 s |
| p10 | 0,68 s |
| mediana | 1,02 s |
| p90 | 1,31 s |
| máximo | 1,40 s |

| Punto de la vuelta | Velocidad | 1,00 s se lee como |
|---|---|---|
| 0,05 | 242 km/h | 1,32 s |
| 0,35 | 138 km/h | 0,78 s |
| 0,55 | 141 km/h | 0,72 s |
| 0,75 | 242 km/h | 1,32 s |
| 0,85 | 121 km/h | 0,80 s |

**Conclusión:** la distancia no se convierte linealmente en tiempo. La velocidad local cambia la lectura entre −50 % y +40 %. `CarIdxEstTime` contiene ese perfil de velocidad; la posición sola, no. Además, tus vueltas reales (80–98 s) estaban muy lejos de los 75,4 s del tiempo estimado, y es razonable pensar que iRacing usa el estimado.

**Límite del análisis:** los `.ibt` **no guardan arrays `CarIdx*`** (solo la telemetría del coche propio), así que no se pudo leer `CarIdxEstTime` directamente. Ver §6.

---

## 4. Telemetría de iRacing: lo que sabemos

Nivel de confianza: **[V]** verificado (datos, código o comportamiento observado), **[D]** deducido, **[?]** sin verificar.

| Variable | Semántica y rarezas |
|---|---|
| `CarIdxLap` | Vueltas *empezadas*. **-1** = coche fuera del mundo (garaje, grúa, desconectado). Sube al cruzar la línea, también en la salida [V]. **Cambia en el mismo tick (60 Hz) en que el pct da la vuelta, también dentro del pit lane** [V con `Lap` del jugador, 2026-10-02 16:57]. Cuenta desde el inicio de la sesión, así que en práctica o clasificación no indica "cuánto de cerca están" [V]. |
| `CarIdxLapCompleted` | Vueltas completadas; sube al cruzar meta, en el mismo tick que `Lap` [V jugador]. **-1 en parrilla y en la vuelta de formación; pasa a 0 al cruzar la línea con la bandera verde** [V jugador]. El tracker trata ese -1 como "fuera del mundo", así que el paso de la salida no se cronometra. Si no existe, el tracker usa `CarIdxLap`. |
| `CarIdxLapDistPct` | 0–1 desde la línea de meta. -1 fuera del mundo [V]. En el pit lane se proyecta sobre la pista [D]. |
| `CarIdxEstTime` | "Tiempo estimado para llegar a la posición actual" desde la línea, **en el reloj `CarClassEstLapTime` de cada coche** [V por definición]. Sigue el perfil de velocidad [D]. Es función solo de la posición [D]. Valor en el pit lane [?]. |
| `CarClassEstLapTime` (YAML) | **Por coche, no por clase**: los modelos con BoP de una misma clase tienen valores distintos [V, 10 valores en GT3]. Disponible desde el principio [V]. |
| `CarIdxPosition` / `CarIdxClassPosition` | Oficiales. Cambian cuando *cualquier* coche cruza meta (por ejemplo, coches que te adelantan en el pit lane y cruzan antes que tú), pero **con 1,3–2,3 s de retraso respecto al cruce** (el tuyo se actualizó entre pct 0,020 y 0,037 de la vuelta siguiente) [V con `PlayerCarPosition`]. **0 durante la parrilla y la vuelta de formación, hasta la bandera verde**, al menos para el jugador en una carrera offline [V]; contradice un comentario antiguo del código que decía que se asignan en parrilla. **0 en sesiones no puntuables** (práctica, test) [V]. Placeholders de IA en sesiones de test: posición asignada con `Lap` -1 [V]. |
| `CarIdxF2Time` | "Tiempo de carrera tras el líder o vuelta más rápida". Semántica exacta para doblados y clases [?]. **No se usa.** |
| `CarIdxLastLapTime` / `CarIdxBestLapTime` | Por eventos: -1 hasta que este cliente ve cruzar al coche. Al conectar a mitad de sesión hay que leer `ResultsPositions` del YAML [V]. Un coche aparcado en boxes puede volver a 0 [V] (para eso existe `SessionBestLapTracker`). |
| `SessionInfo.CurrentSessionNum` | iRacing **no lo escribe**. Usar `SessionNum` de la telemetría (`CurrentSession.Number`) [V]. |
| `SessionState` | 1 GetInCar, 2 Warmup, 3 ParadeLaps, 4 Racing, 5 Checkered, 6 CoolDown [V en el .ibt]. |
| `CarIdxTrackSurface` | -1 = fuera del mundo (lo usa `PitStopTracker`) [V]. |
| `CarDistAhead` / `CarDistBehind` | Solo del jugador. Metros al coche más cercano delante y detrás en pista, de cualquier clase; detrás no se pliega a media vuelta (hasta 2182 m vistos). **500000 = sin dato, y así está durante todo el paso por pit road** [V]. Están en los `.ibt`. |
| Pit lane (Road Atlanta) | Pit road de pct 0,957 a 0,088 (~530 m). A 72 km/h son 25,3 s, frente a 8,4 s del mismo tramo en pista: cada paso sin parar cuesta ~16,9 s. El pct avanza al ritmo del coche, salvo en la entrada (0,957–0,97), donde avanza ~35 % más rápido [V]. |
| `.ibt` | Formato de cabecera igual que la memoria compartida, a 60 Hz. **Sin arrays CarIdx.** YAML de sesión incluido [V]. |

---

## 5. Diseño vigente en detalle

### 5.1 Relative (`BuildRelative`)

1. **Qué coches:**
   - el jugador siempre;
   - del resto, se excluyen los que tienen `CarIdxLap` -1 y los que no han salido (vuelta 0 y pct 0);
   - solo se incluyen coches que se puedan situar con `TrackPosition.Read` (vuelta y pct ≥ 0);
   - si el jugador no está en el mundo, solo aparece su fila.
2. **Tiempo de cada coche en el reloj del jugador** (`OnPlayersClock`):
   - **Mismo modelo que el jugador** (`EstTimeProfile.SharesClock`: mismo `CarID` y mismo `CarClassEstLapTime`): su propio `CarIdxEstTime`, que es exacto.
   - **Otro modelo o clase:** `EstTimeProfile.EstTimeAt(su pct)`, si ese tramo está aprendido y la curva corresponde al reloj actual del jugador.
   - **Si no:** escalado proporcional, `EstTime_otro / EstLap_otro × EstLap_jugador`. Es el comportamiento de la iteración 0.
3. **Gap:**

   ```
   ahead = t(otro) − t(jugador)
   gap   = −(ahead − L · floor(ahead / L + 0.5))
   ```

   con L = `CarClassEstLapTime` del jugador. Negativo = delante. Sin `CarIdxEstTime`: `−OnTrackGapTo × L`. Sin L: `NaN`, que se muestra como "—".
4. **Orden:** por el gap mostrado, con la distancia plegada como desempate. Así el signo nunca contradice el orden.
5. **Doblado o doblando:** por **distancia de carrera** (`RaceGapTo`), y solo en sesiones Race:
   - más de +0,5 vueltas → `Lapping`;
   - menos de −0,5 vueltas → `Lapped`, o `BeingLapped` si está delante a 5 s o menos (`BeingLappedWithinSeconds`).
6. **POS, posición de clase e iRΔ:** de las filas del Standings, que ahora se reconstruye en el mismo tick. Respaldo: `CarIdxPosition` y luego `ResultsPositions`.

### 5.2 `EstTimeProfile`

- 1000 bins sobre el pct. Cada bin guarda la última muestra `(pct, EstTime)`.
- **Muestras:** el coche del jugador y los del mismo modelo, en cada tick. Se descartan pct fuera de [0, 1), `EstTime` ≤ 0 o mayor que la vuelta estimada, y coches en pit road.
- **Lectura:** interpolación lineal entre la muestra anterior y la siguiente, cruzando la línea si hace falta (±1 vuelta en pct, ±L en tiempo). Si están separadas más del 2 % de la vuelta, devuelve `null`. Cerca de la línea puede salir ligeramente negativo; el plegado del gap lo absorbe.
- **Clave** `(SubSessionID, CarID, CarClassEstLapTime)`: si cambia, la curva se vacía.
- Se llena con la vuelta de formación. A 10 Hz y 300 km/h un coche avanza ~8 m por tick; con el 2 % de margen sobra.

### 5.3 Standings (`BuildStandings`, ruta de carrera)

Práctica y clasificación siguen ordenando por vuelta rápida (`BuildFastestLapStandings`, sin cambios).

- **Quién aparece:**
  - el jugador siempre;
  - con `CarIdxLap` -1, solo si tiene posición oficial **y** ha corrido (vueltas en `ResultsPositions` o algún paso por meta visto). Así un coche en grúa o desconectado mantiene su puesto y los placeholders de IA quedan fuera;
  - en el resto de casos, se excluyen los coches sin posición oficial, en vuelta 0 y con pct ≤ 0.
- **Orden:** posición oficial; los coches sin ella van detrás, por vueltas completadas (de más a menos) y luego por el instante de su último paso por meta. `OrderBy` es estable, así que los empates conservan el orden del roster.
- **Gap al líder de clase**, medido en meta:
  - L = última vuelta completada que se ha visto al coche, y T = instante de ese paso;
  - gap = T − (primer coche de su clase en completar L);
  - `LapsDown` = número de vueltas L+1, L+2… que algún coche de la clase ya había completado antes de T;
  - si falta algún dato, `NaN` ("—").
- **En vivo en cada reconstrucción:** pit road, banderas y penalizaciones, neumático y última parada.
- **Formato:** líder de clase → "Leader"; `LapsDown > 0` → `+1L`; si no, `+x.x`.

### 5.4 `LineCrossingTracker`

- **Detección:** un paso por meta es cuando `LapsCompleted` sube **exactamente 1** entre dos ticks consecutivos. Fuente: `CarIdxLapCompleted`, o `CarIdxLap` si no existe.
- **Interpolación:** si el pct saltó de > 0,5 a < 0,5, el instante se coloca entre los dos ticks en proporción a lo que faltaba hasta la línea. Si no, se toma el instante del tick.
- **Fuera del mundo** (vueltas < 0): se borra el tick anterior, así que la vuelta siguiente al reaparecer no se cronometra.
- **Confianza:** `FirstCrossing(coches, L)` devuelve `null` si algún coche ya había completado L cuando se le vio por primera vez. Ese paso por meta pudo ocurrir sin que lo viéramos (overlay abierto a mitad de carrera).
- **Reinicio:** si cambia `SessionNum` o `SessionTime` retrocede (repetición).
- Debe actualizarse en **cada tick**: un paso por meta perdido no se recupera.

### 5.5 Cableado (`MainWindow.xaml.cs`)

- El tick de UI va a 100 ms (`_uiTimer`). Hay un bucle "crítico" aparte a 60 Hz sincronizado con el frame.
- En cada tick se actualizan `_pitStopTracker`, `_lineCrossings` y `_estTimeProfile`, cada uno con su `ComponentGuard` ("Line crossing timing", "Relative time curve").
- El Standings se reconstruye **en cada tick** (antes, cada 10). `StandingsUpdateEveryNTicks` se ha renombrado a `DiagnosticsUpdateEveryNTicks`, porque solo lo usan los diagnósticos.
- Con un evento nuevo se reinician `_lineCrossings` y `_estTimeProfile`.
- A `BuildStandings(..., crossings: _lineCrossings)` y `BuildRelative(..., estTimeProfile: _estTimeProfile)` se les pasan los trackers.

### 5.6 Otros

- `TrackPosition`:
  - `RaceDistance` (vuelta + pct) y `RaceGapTo`: absolutos, solo para doblados;
  - `OnTrackGapTo` y `FoldToHalfLap`: plegados, para la ventana del Relative y el Cockpit;
  - `Read` devuelve `null` si la vuelta o el pct son -1.
- `CockpitBuilder` (barras de proximidad) usa `TrackPosition.FoldToHalfLap` sobre el pct y la longitud de pista. Nunca usó `EstTime`, porque en multiclase dejaba las barras apagadas.
- `TelemetryVarNames.CarIdxEstTime` se ha restaurado con documentación y se ha añadido `CarIdxLapCompleted`.

---

## 6. Hipótesis abiertas y cómo verificarlas

1. **El relativo oficial de iRacing usa `CarIdxEstTime`** [D]. Encaja con la definición, con los datos y con que la iteración 0 "se parecía más".
2. **En coches de otra clase, ¿con qué reloj mide iRacing?** [?] Lo implementado mide con el **reloj del jugador** ("cuánto tardaría mi coche en llegar a su posición"). La alternativa es el **reloj del otro coche**, que cambiaría los gaps entre clases en la proporción de sus vueltas estimadas (65 frente a 75 s ≈ 15 %).
   - Cómo comprobarlo: en vivo, comparar contra un prototipo. Si la desviación es sistemática y proporcional, cambiar `OnPlayersClock` para leer al rival en su propio reloj, escalado o con su curva.
3. **Modelos BoP de la misma clase:** la forma de la curva es casi idéntica [D]. Con la curva aprendida, el error debería ser despreciable.
4. **`EstTime` depende solo de la posición** [D]; si no, la curva aprendida sería ruidosa. **Comportamiento en pit lane** [?]: por eso se ignoran esas muestras.
5. ~~Sincronía en meta entre `CarIdxLap`/`LapCompleted` y `LapDistPct`~~ **Verificada** para el jugador: mismo tick, también en el pit lane. Falta confirmarlo en los arrays `CarIdx*`.
6. **`CarIdxLapCompleted` en parrilla y en la salida:** verificado para el jugador: -1 hasta la verde y 0 tras cruzar. El paso de la salida **no** se cronometra, así que el Standings muestra "—" durante toda la vuelta 1.
7. **Posiciones oficiales con penalizaciones o drive-through** [?]: se usan tal cual.

**Para verificar con datos reales** (los `.ibt` no sirven porque no tienen arrays `CarIdx`), lo más útil sería un **grabador de diagnóstico**. Escribiría a disco, durante unos minutos y a 10 Hz:
- `SessionTime`;
- por coche: `CarIdxEstTime`, `CarIdxLapDistPct`, `CarIdxLap`, `CarIdxLapCompleted`, `CarIdxPosition`, `CarIdxF2Time` y `CarIdxOnPitRoad`;
- el YAML de pilotos.

Con eso se pueden contrastar las hipótesis 1, 2, 4, 5 y 6 sin ir de memoria.

---

## 7. Próximos pasos

- [ ] **Prueba en vivo** de la iteración 3, comparando con la black box de iRacing:
  - Relative contra un coche de tu mismo modelo (debería clavarse);
  - contra otro GT3 de distinto modelo;
  - contra un coche de otra clase (hipótesis 2);
  - durante la formación y la vuelta 1 (deben verse gaps);
  - tras una parada en boxes.
- [ ] Standings: comprobar que solo cambia al cruzar meta; que los gaps coinciden con los de iRacing; `+1L`; el comportamiento al abrir el overlay a mitad de carrera ("—" durante ~1 vuelta); y que no hay coste de rendimiento al reconstruir en cada tick (dashboard con todo el campo).
- [ ] Decidir el formato `+1L` (cambio visible no pedido explícitamente).
- [ ] Indicador de conexión/desconexión: **no existe** en la tabla; el usuario lo mencionó como ejemplo de dato en vivo.
- [ ] `SessionProgress.ReferenceLapSeconds` sigue usando `CarClassEstLapTime` como último recurso para estimar el total de vueltas en carreras por tiempo. No afecta a posiciones.
- [ ] `CHANGELOG.md` no actualizado (define la versión de la build y las notas de release; ver `.github/skills/changelog`).
- [ ] Commit de la iteración 3 cuando se valide en vivo.
- [ ] Opcional: grabador de diagnóstico (§6).
- [x] **Posición en vivo en el Relative — hecho (sin commitear):** `LiveRaceRanks` en `StandingsBuilder`. Solo en sesiones Race con `SessionState` = 4 (en carrera); en formación, con bandera a cuadros, en práctica y en clasificación sigue la posición oficial del Standings. El Standings no cambia. Contexto: la black box de iRacing actualiza la posición del relativo al cruzar meta, igual que nosotros ahora; RaceLab la actualiza al instante. Coste: un `RaceOrder` por tick, despreciable; no afecta a gaps ni a banderas. Feedback de la prueba del 2026-10-02 (iteración 3): la mejor hasta ahora; gaps grandes solo contra otras clases y como mucho una vuelta (probablemente la curva aún sin aprender: el overlay se abrió al final de la formación); Standings con "—" en la vuelta 1 y posición oficial con 1–2 s de retraso, ambos aceptados por el usuario.
- [ ] **Standings en parrilla y formación:** con `CarIdxPosition` = 0 y sin pasos por meta, el orden cae al orden del roster. Propuesta: desempatar por distancia en pista (orden de formación).
- [ ] **Desfase de 1,3–2,3 s entre nuestro gap (al cruzar) y la posición oficial:** en ese intervalo, un coche que acaba de ponerse primero de su clase puede mostrar `+0.0` sin ser aún "Leader". Opciones: ordenar con nuestro propio cronometraje y usar el oficial solo como respaldo, o esperar a que cambie la posición oficial para publicar el gap.
- [ ] **Vuelta 1 sin gaps en Standings:** se podría cronometrar el paso de -1 a 0 en la salida.

---

## 8. Lecciones aprendidas

1. **No sustituir una fuente de tiempo por distancia sin datos.** Convertir una distancia a segundos con un ritmo medio ignora la velocidad local, y el error medido llega a −50 %/+40 %. "Conceptualmente más limpio" no es "más fiel a iRacing".
2. **Hay carreras sin sesiones previas**, y la vuelta de formación existe. Un diseño que necesita vueltas medidas para mostrar algo deja pantallas vacías justo cuando más se miran.
3. **Los tiempos medidos se contaminan:** incidentes, paradas, vuelta 1, tráfico. El estimado de iRacing no.
4. **`CarClassEstLapTime` no es el problema, sino comparar relojes distintos.** La solución es llevar todos los coches al mismo reloj (el del jugador), no eliminar el tiempo estimado.
5. **Clasificación ≠ orden en pista.** El usuario quiere una tabla estable que cambie al cruzar meta, como iRacing y los overlays conocidos, con los indicadores de estado en vivo. Las dos cosas se calculan por separado.
6. **Antes de rediseñar tras una regresión, analizar con datos reales.** Los `.ibt` en `Documents\iRacing\telemetry` sirven para la telemetría del coche propio y el YAML.
7. **Las especificaciones pegadas** (en español, con tono de propuesta) se implementaron al pie de la letra en las iteraciones 1 y 2, y la iteración 2 empeoró el resultado en vivo. Conviene contrastar cada propuesta con datos y señalar riesgos antes de implementarla.

---

## 9. Entorno de desarrollo (WSL)

### Compilar y ejecutar tests

En WSL hay un SDK de .NET en `~/.dotnet` (8.0.424 y 10.0.400). Los proyectos son `net8.0-windows`:
- **Compilar:** en Linux, con `-p:EnableWindowsTargeting=true`.
- **Ejecutar tests:** hace falta `Microsoft.WindowsDesktop.App`, que solo está en el `dotnet.exe` de Windows (sin SDK, solo runtimes). Se ejecutan con el runner de consola de xunit sobre ese host:

```bash
#!/usr/bin/env bash
set -euo pipefail
cd /home/carlos/openoverlay
~/.dotnet/dotnet build IRacingOverlay.App.Tests/IRacingOverlay.App.Tests.csproj -p:EnableWindowsTargeting=true --nologo -v q
cd IRacingOverlay.App.Tests/bin/Debug/net8.0-windows
XC=$(wslpath -w ~/.nuget/packages/xunit.runner.console/2.5.3/tools/netcoreapp2.0/xunit.console.dll)
"/mnt/c/Program Files/dotnet/dotnet.exe" exec --runtimeconfig IRacingOverlay.App.Tests.runtimeconfig.json \
  --depsfile IRacingOverlay.App.Tests.deps.json "$XC" IRacingOverlay.App.Tests.dll "$@"
# Filtrar por clase: ... IRacingOverlay.App.Tests.dll -class IRacingOverlay.App.Tests.StandingsBuilderTests
# SDK: igual en IRacingOverlay.Sdk.Tests/bin/Debug/net8.0-windows con IRacingOverlay.Sdk.Tests.*
```

- El runner muestra una excepción inofensiva al buscar "reporters" en la ruta `\\wsl.localhost`.
- `dotnet test` en Linux falla por falta de `WindowsDesktop.App`.
- Ejecutar el `vstest.console.dll` del SDK de Linux con el `dotnet.exe` de Windows da `BadImageFormatException`.

### Lector mínimo de `.ibt` (Python)

```python
"""Minimal iRacing .ibt reader: header, var table, session YAML and samples."""
import struct, mmap

TYPES = {0: ('c', 1), 1: ('?', 1), 2: ('i', 4), 3: ('I', 4), 4: ('f', 4), 5: ('d', 8)}

class Ibt:
    def __init__(self, path):
        self.f = open(path, 'rb')
        self.m = mmap.mmap(self.f.fileno(), 0, access=mmap.ACCESS_READ)
        h = struct.unpack_from('<10i2i', self.m, 0)
        (self.ver, self.status, self.tick_rate, _, self.si_len, self.si_off,
         self.num_vars, self.var_off, self.num_buf, self.buf_len) = h[:10]
        self.buf_off = struct.unpack_from('<i', self.m, 48 + 4)[0]          # varBuf[0].bufOffset
        _, self.start_time, self.end_time, self.lap_count, self.records = struct.unpack_from('<qddii', self.m, 112)
        self.vars = {}
        for i in range(self.num_vars):                                       # 144 bytes por cabecera
            o = self.var_off + i * 144
            vtype, voff, count = struct.unpack_from('<3i', self.m, o)
            name = self.m[o + 16:o + 48].split(b'\0')[0].decode()
            self.vars[name] = (vtype, voff, count)
        self.session_yaml = self.m[self.si_off:self.si_off + self.si_len].split(b'\0')[0].decode('latin-1')

    def get(self, rec, name):
        vtype, voff, count = self.vars[name]
        fmt, size = TYPES[vtype]
        base = self.buf_off + rec * self.buf_len + voff
        vals = struct.unpack_from('<' + fmt * count, self.m, base)
        return vals if count > 1 else vals[0]
```

Medición de la no linealidad (§3): dentro de una vuelta limpia, para cada tick *i*, `dpct = pct[i] − pct[i − 60]` (60 ticks = 1 s a 60 Hz, sumando 1 si cruza la línea). Entonces `dpct × tiempo_de_vuelta` es lo que el modelo lineal mostraría para un gap real de 1 s. Variables usadas: `SessionTime`, `LapDistPct`, `Lap`, `Speed`, `OnPitRoad`, `PlayerCarMyIncidentCount`, `SessionState`.

---

## 10. Referencias de tests

- `IRacingOverlay.App.Tests/StandingsBuilderTests.cs`:
  - **Relative:** `BuildRelative_SameCar_GapIsTheDifferenceInEstimatedTime_NotTheDistance`, `_FormationLap_ShowsGapsBeforeAnyoneHasALap`, `_AcrossTheLine_FoldsToTheNearestHalfLap`, `_OtherModel_BeforeOurCurveIsKnown_ScalesItsEstimateToOurLap`, `_OtherModel_IsReadOnTheCurveWeHaveDriven`, `_CarsOfDifferentClassesBehind_StayInTrackOrder`, `_LappedCarRightBehind_...`, `_PlayerNotInTheWorld_ShowsOnlyThemselves`, `_NoEstimatedLapForOurCar_LeavesTheGapBlank`.
  - **Standings:** el simulador `Race` (coches a velocidad constante, ticks cada 0,097 s), `BuildStandings_InARace_HoldsPositionsAndGapsUntilEachCarCrossesTheLine`, `_LiveStateUpdatesBetweenCrossings`, `_LappedCar_ShowsLapsDown`, `_AttachedMidRace_GapStaysBlankUntilALapIsSeenWhole`, `_Multiclass_GapIsMeasuredAgainstTheClassLeader`, `_NoOfficialPosition_RanksByLapsThenWhoCompletedTheLatestOneFirst`, `_ClassifiedCarOutOfTheWorld_KeepsItsPlace`, `_BeforeAnyoneCrossesTheLine_GapIsBlankRatherThanZero`.
- `IRacingOverlay.App.Tests/EstTimeProfileTests.cs`: interpolación, muestras del mismo modelo, tramos desconocidos, cruce de meta, pit road, cambio de coche.
- `IRacingOverlay.App.Tests/LineCrossingTrackerTests.cs`: interpolación entre ticks, primer paso de un grupo, confianza al conectar a mitad de carrera, cambio de sesión, vuelta desde fuera del mundo.
- `IRacingOverlay.App.Tests/CockpitBuilderTests.cs`: pruebas de regresión multiclase que confirman que la proximidad ignora `EstTime`.
