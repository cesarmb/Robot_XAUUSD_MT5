# CLAUDE.md — Proyecto Tradingview (Cesar)

Contexto permanente de este proyecto. Leer antes de arrancar cualquier
tarea acá para no volver a preguntar lo mismo.

## Idioma

Responder **en español**. El usuario escribe en español; toda la
documentación del proyecto (skills, referencias) está en español.

## De qué se trata el proyecto

Análisis cuantitativo de indicadores de TradingView (oro principalmente:
XAUUSD de OANDA y MGC1! de COMEX). El flujo típico es:

1. Leer del chart las señales que dibuja un indicador (labels/boxes).
2. Reconstruir la fecha/hora real de cada señal contra precio verdadero.
3. Simular qué pasó (TP / SL) con velas reales.
4. Agregar P&L por día, sesión, mes, símbolo, dirección; curvas de equity.
5. Probar optimizaciones **siempre con validación fuera de muestra**.

No es desarrollo de software: el entregable son análisis, tablas,
gráficos y conclusiones operativas.

## Estructura

```
C:/Cesar/Cursos/Tradingview/
├── .mcp.json              # MCP servers del proyecto (exa, notion, composio)
├── indicators/            # Pine Script propio (ICT_FVG_MultiTimeframe.pine)
└── tradingview-mcp/       # bridge CDP ↔ TradingView Desktop (repo clonado)
```

- Skills en `C:/Users/csrmt/.claude/skills/` — **`skill-FIBO`** tiene todo
  el pipeline de backtest ya empaquetado (scripts + metodología +
  hallazgos). Usarla para cualquier análisis de señales de indicadores,
  no solo del Fibo.
- Archivos temporales (JSON de barras/señales, scripts de análisis, HTML):
  siempre en el scratchpad de la sesión, nunca en el repo.

## Conexión con TradingView

El bridge habla con TradingView Desktop por Chrome DevTools Protocol
(puerto 9222).

```bash
cd C:/Cesar/Cursos/Tradingview/tradingview-mcp
node src/cli/index.js status          # ¿conectado?
node src/cli/index.js state           # símbolo, TF, indicadores del chart
```

- **La conexión se cae seguido entre turnos.** Verificar `status` antes de
  cualquier secuencia de comandos, no asumir que sigue viva.
- Para relanzar: `scripts/launch_tv_debug.bat`. **Ojo: esto mata el
  proceso de TradingView a la fuerza** — avisarle al usuario antes de
  correrlo, puede tener algo abierto.
- `bash scripts/ensure_connected.sh` (en skill-FIBO) hace check + relanzar.

## Reglas de operación en el chart (importante)

- **Nunca ejecutar operaciones reales** (comprar, vender, poner SL/TP en
  el broker) ni crear alertas. El proyecto es de análisis.
- Cuidado con clicks en el chart: ya pasó ocultar un indicador sin querer
  y clickear la escala de precio (que puede crear una alerta). Si se
  toca UI, verificar después que no quedó nada creado.
- **Modo Replay: siempre `node src/cli/index.js replay stop` al terminar.**
  Ya quedó el chart atrapado en el pasado más de una vez.

## Gotchas técnicos ya descubiertos

- **Labels de indicadores:** el CLI (`src/core/data.js`, `buildGraphicsJS()`)
  tiene hardcodeado `inner.get(false)`. Algunos indicadores guardan sus
  dibujos bajo la clave `true` (PROFIT MASTER 3.0) y entonces `tv data
  labels` devuelve vacío aunque haya etiquetas visibles. Iterar todas las
  claves:
  ```js
  var inner = pc.dwglabels.get('labels');
  Array.from(inner.keys()).forEach(function(k){ var c = inner.get(k);
    if (c && c._primitivesDataById) c._primitivesDataById.forEach(function(v){ out.push(v); }); });
  ```
- **El campo `x` de una label no es bar_index ni timestamp.** Es un
  contador creciente en orden de dibujo. Para ubicar una señal en el
  tiempo hay que emparejar por precio con **alineación monótona (DP)**,
  nunca vecino-más-cercano — eso ya generó una señal fantasma que el
  usuario detectó mirando el chart.
- **Vela que toca SL y TP a la vez:** asumir SL (conservador), nunca TP.
- **Match con diferencia de precio > $2 se descarta**, no se fuerza.
- **Antes de confiar en datos de Replay**, verificar que el indicador no
  repinte (comparar señales de Replay contra las mismas en vivo).
- **NUNCA usar `tv indicator set` en un script protegido.** El bridge hace
  `getInputValues()` → `setInputValues()`, y en scripts encriptados eso
  descarta el input `text` (el ILScript) y el indicador muere con
  "Runtime error: Can't parse pine". Ya pasó con PROFIT MASTER. Para
  cambiar un ajuste de estos, hacerlo por la UI. **Recuperación:** volver a
  aplicar los 52 inputs completos desde un snapshot de `tv values` (que
  incluye el blob `text`) y después `window.location.reload()`.
- **Guardar siempre un snapshot de `tv values` antes de tocar un indicador
  protegido** — es lo único que permite restaurarlo si se rompe.
- **Puntos ≠ precio, y la convención cambia por indicador:**
  Fibo AGO → 1 pt = 0.1 unidades de precio; PROFIT MASTER → 1 pt = 1.

## Cómo quiere el usuario que se trabaje

- **Rigor sobre rapidez.** Prefiere un número verificado a uno rápido.
- **Verificar contra la fuente real cuando algo no cuadra.** Si dice "no
  veo esa señal en el chart", tomarlo en serio: screenshot y confirmar.
  Ha detectado errores reales así más de una vez.
- **Listar los datos uno por uno**, no resumir "los relevantes" (pasó con
  un calendario económico: se transcribieron 2 de 4 líneas y lo notó).
- **Validación fuera de muestra obligatoria** antes de recomendar
  cualquier filtro o estrategia. Cinco optimizaciones "obvias" ya fallaron
  esta prueba (ver `skill-FIBO/references/methodology.md` §5).
- **Decir cuando algo invalida trabajo previo**, aunque sea incómodo.
- Disclaimers de backtest (sin spread/comisión/slippage, no es asesoría
  financiera): una línea al final cuando se comparte P&L. No repetirlo en
  cada mensaje.

## Estado de los análisis

### Fibo AGO C.U T.V. — terminado
Pipeline completo en `skill-FIBO`. Números 2026 en
`skill-FIBO/references/findings.md` (referencia, no verdad fija).
Playbook publicado como artifact + PDF.

### PROFIT MASTER 3.0 ALUMNOS — en curso

**Setup actual (30-sep-2026): `VANTAGE:XAUUSD`, 5m, Heikin Ashi.** Antes era
OANDA 15m. El tipo de vela NO es cosmético: en 15m, velas normales daban 13
señales y HA 9, con solo 4 coincidencias. Verificar siempre `tv type` =
HeikinAshi.

**Versión actualizada** (sep-2026): 68 inputs (antes 45), **mismos 15
plots** — la estructura de lectura no cambió. Novedades: tipo de entrada
"Límite a % del SL", **"Medir también a mercado con precio real"**, score
A/B/C por señal, sección de Break Even, comparativas de cierre parcial.

**Regla de señal — validada 8/8 en la versión nueva** (y 9/9 en HA + 13/13
en velas normales en la anterior), siempre sin falsos positivos ni
faltantes. Sobrevivió cambio de versión, de bróker y de temporalidad:
dos líneas ATR, `plot_3` "Banda activa" (tendencia/trailing) y `plot_5`
"Banda opuesta" (gatillo). Sale señal cuando (a) el close cruza la banda
activa → giro de tendencia, o (b) dentro de una tendencia, el close cruza
la banda opuesta a favor → continuación. **Y** la vela cierra dentro de una
de las dos ventanas de sesión. Los 10 candidatos fuera de sesión dieron 0
señales.

**Niveles:** entrada = close de la vela de señal · **SL = 3.0000 × ATR(14)**
con "Amplitud del stop" en `Normal` (exacto en 6/6 señales limpias, en
ambas versiones y temporalidades) · TPs = exactamente los inputs `in_6/7/8`
(hoy **1.0 / 1.2 / 1.5 R**; antes 0.7/1.0/1.5) · BE 0.5 R.
$/punto/lote = 100.

**Score A/B/C — CONFIRMADO (30-sep-2026).** Cada señal trae una letra y un
valor en R (ej. "A 0.93R"). El valor es:

```
score = (R - δ) / (R + δ)      δ = desviación adversa entre el close REAL
                                   y el close Heikin Ashi de esa vela
```

O sea: **el R:R que realmente obtenés** si entrás a precio de mercado
verdadero mientras el SL y los TP quedan donde el indicador los dibujó.
Es el "impuesto HA" medido operación por operación — el autor conoce el
problema y lo reporta.

Verificado en 8 operaciones que cubren los tres grados: error medio
0.00228, máximo 0.00458, **8/8 dentro del redondeo de la etiqueta (0.005)**.
(Solo 8 de 63 porque hacen falta precios reales, y el chart expone ~300
velas.)

Fórmula general: **score = (TP1 × R − δ) / (R + δ)** — usa el input TP1, no
un 1.0R fijo (verificado cambiando TP1 de 1.0 a 0.7: los 62 scores cambiaron,
predichos con error medio 0.0025). **Las letras usan dos criterios distintos:
A/B es `δ/R ≤ 0.05` y NO depende de TP1; B/C es `score < 0.5` y SÍ depende.**
Por eso al bajar TP1 a 0.7, 24 operaciones pasaron de B a C y ninguna A se
movió. Interpretación: A mide calidad de ejecución, C mide si la operación
vale la pena contra el objetivo elegido. En términos de δ/R:
A = menos del 5% de R · B = 5% a 33% · C = más del 33%.

Distribución observada (63 señales): A 24% · B 71% · C 5%. Score medio
0.789 → **impuesto HA de 0.118R por operación**, consistente con el 0.14R
medido de forma independiente en la sesión anterior (OANDA 15m).

**Uso operativo:** el score ES el R:R real, así que el acierto mínimo para
no perder es `1/(1+score)`: score 1.0 → 50% · 0.8 → 55.6% · 0.7 → 58.8% ·
0.5 → 66.7% · 0.4 → 71.4%. Los inputs `in_19/20/21` (Operar señales A/B/C)
permiten filtrar por grado.

**Para leer scores hace falta `in_50` ("Dejar dibujado el SL/TP de
operaciones anteriores") en true**, si no solo dibuja la última operación.

**El "impuesto HA"** (medido en la versión anterior, 15m): la entrada que
marca el indicador es el close HA, que no es precio operable. No es sesgo
general del HA (sobre 278 velas normales la desviación media es −0.01 pts):
aparece **solo en velas de ruptura**, y el indicador dispara justo ahí.
Crece monótono con lo que se movió la vela a favor: −2.27 pts si fue en
contra, +2.51 entre 0.3 y 0.6 ATR, **+7.55 por encima de 1.0 ATR**. Las
velas de señal se mueven 0.95 ATR contra 0.00 del resto. Costo ≈ 4.7 pts ≈
**0.14R por operación**, y no se promedia a cero con más datos.

**LAS BANDAS — RESUELTAS (30-sep-2026).** No son dos líneas: son **un canal
simétrico**; "Banda activa" / "Banda opuesta" son etiquetas que se
intercambian al girar la tendencia (U y L son continuas a través de los
giros).

```
centro     = SMA(hl2, 8)                                 <- EXACTO
semiancho  = 0.496*SMA(TR,8) + (2 - nivel/100)*ATR(120)  <- 0.023 pts
U = centro + semiancho          L = centro - semiancho
señal = el close sale del canal (arriba BUY / abajo SELL), dentro de sesión
```

ATR = Wilder (RMA de TR) sobre las velas **Heikin Ashi** del chart.

- **Centro exacto**: error 0.00000000 sobre 1978 velas, en 5 capturas
  independientes y en los 3 niveles de sensibilidad. La única vela con
  error es la que está viva.
- **Escala de sensibilidad, lineal y medida**: `b = 2 - nivel/100`.
  **Tabla de presets, medida y completa:**

  | Preset | nivel | b = 2 - nivel/100 |
  |---|---|---|
  | Muy baja | 60 | 1.4 |
  | Baja | 80 | 1.2 |
  | Media | 100 | 1.0 |
  | Alta | **130** | 0.7 |
  | Muy alta | 150 | 0.5 |

  **CORRECCIÓN (30-sep):** durante un rato se creyó que "Alta" daba la misma
  banda que "Muy alta" (284/285 velas bit-idénticas). **Era falso.** La
  captura de "Alta" se tomó antes de que el estudio terminara de recalcular,
  así que devolvió las bandas del ajuste anterior. El usuario lo detectó
  mirando el chart: las bandas no coincidían solo en ese preset.

  **El control de calidad del centro NO detecta esto**, porque el centro no
  depende de la sensibilidad: una captura enteramente vieja lo pasa igual.
  **Chequeo adicional obligatorio tras cambiar un input:** comparar el
  semi-ancho contra la captura anterior y verificar que **cambió**. Si sale
  idéntico, es captura vieja, no un hallazgo.

  **Método de medición sin sesgo** (el bueno, usar este): poner la réplica en
  dos niveles conocidos (100 y 150) y despejar `S = 2*(hw100 - hw150)` y
  `F = (hw150 - 0.5*S)/0.496`. Eso da el ATR(120) tal como lo calcula
  TradingView con toda su historia, sin el sesgo de warmup que aparece al
  reconstruirlo desde ~300 velas. Con este método Alta dio b=0.7028,
  cuartiles 129.6-130.1. La réplica sirve de control: su b es conocido, así
  que si la medición no lo recupera, la medición está mal. "Personalizado 100" dio b=0.9421 vs Media 0.9454 → confirmado
  que el dial y los presets son la misma escala.
- **Componente lento identificado**: al despejarlo empíricamente da
  `ATR(130) × 1.0005` — coeficiente 1.0. El despeje es consistente entre
  capturas de distinto nivel (diferencia 0.009-0.065), lo que confirma la
  estructura multiplicativa.
- **Residuo: CERRADO (30-sep-2026).** El componente rápido **no es ATR**:
  es `SMA(TR, 8)` — media simple, no Wilder, y con el mismo largo 8 que el
  centro. Usar RMA(TR,6) daba 0.179; SMA(TR,8) da **0.023**, ocho veces
  menos. El lento es `ATR(120)` (Wilder) con coeficiente exactamente
  `2 - nivel/100`. Cómo se encontró: el residuo tenía autocorrelación 0.78 en
  lag 1 pero **−0.38 en lag 5**, con un solo lóbulo negativo que muere en
  lag 8 — firma de una diferencia entre dos medias de largo parecido, no de
  ruido. Eso descartó "falta un horizonte de volatilidad" y apuntó al tipo
  de suavizado. Descartados por el camino: un solo ATR (10%), tercer ATR
  (coeficiente negativo = sobreajuste), volatilidad de velas reales (10.5%),
  desviación/stdev respecto del centro (41%), percentil (12%), término
  constante (no ayuda), ATR de temporalidad mayor (descartado: el ancho
  nunca se repite entre velas consecutivas, 0/1599).

- **Control de calidad obligatorio en cada medición:** verificar que
  `centro = SMA(hl2,8)` dé error 0.00000000. Si da otra cosa, la captura no
  sirve (pasó al medir con el chart en 15m contra ATRs de 5m: dio 17.75 y
  el resultado era basura). Ese chequeo detecta desalineación de
  temporalidad, de layout o capturas a mitad de recálculo.

**Validación de la réplica**: 29 señales reales del original -> **28 exactas
en la misma vela, 0 inventadas**. Error de banda 0.023 pts sobre 4681
observaciones de 7 niveles de sensibilidad. La única que falla es un cruce
que el original hizo por 0.018 pts.

**Gotcha de captura:** al cambiar un input el chart recarga y vuelve a
~300 velas. Para los ATR largos hay que usar una serie de precio larga
guardada antes y unir por timestamp — el precio no depende del nivel.
Scrollear hacia atrás carga ~2000 velas.

**Regla de señal, versión simplificada y exacta:** `señal = el close sale
del canal` (por arriba → BUY, por abajo → SELL), con la vela dentro de una
ventana de sesión. La tendencia es simplemente el último lado por el que
salió. Reproduce 8/8 sin falsos positivos. Esto reemplaza la descripción
anterior de dos casos (giro / cruce de banda opuesta): son lo mismo.

**Estado de la réplica:** el canal y la regla de señal están resueltos y
validados. Falta: medir los presets Alta y Baja, re-medir Muy alta con
historia completa (se midió con 301 velas y es el punto más flojo), y
escribir el Pine. Después queda el score A/B/C.


### Backtest de 2 meses (30-sep-2026) — SUPERADO por el de 6 meses, ver abajo

**Datos:** 112 señales, VANTAGE:XAUUSD 15m HA, 28-jul a 30-sep-2026,
sensibilidad Muy alta (150). Pipeline: precio real (4305 velas, cargadas
fijando el rango visible con `tv range`) → HA derivado → fórmula de la
réplica → señales. **Validado 8/8 contra el indicador** en la ventana
solapada, con sesiones 302/302 exactas.

**El impuesto HA decide el resultado:** entrando al close HA da +$12.022;
entrando a precio real da **−$28.682**. Diferencia −$40.704 = **−$367 por
operación = −0.147R**, idéntica en los tres objetivos porque es un
corrimiento puro del precio de entrada. Coincide con el 0.14R medido en
septiembre y con el 0.118R que implican los scores.

**El grado A es el único filtro del proyecto que sobrevive fuera de
muestra.** Definición operativa: **grado A = δ/R ≤ 0.05** (dato clave: es
**independiente de TP1**, mientras que la frontera B/C sí se mueve con TP1).
P&L a 1.0R: A +$11.792 (n=26) · B −$35.781 (n=78) · C −$4.693 (n=7).
Por mitades: A +$10.510 y +$1.282 · B −$17.853 y −$17.928 · C −$827 y
−$3.866. Filtro derivado de la 1ª mitad aplicado a la 2ª: +$1.282 contra
−$20.512 sin filtrar. **Funciona porque el score ES el R:R real**, no es una
correlación histórica como los filtros por día o dirección que fallaron.

**El TP no es determinante y NO hay que optimizarlo.** A gana en los 8
objetivos probados (0.5R a 3.0R), B pierde en los 8, C pierde en los 8.
Pero el TP "óptimo" no generaliza: el mejor de la 1ª mitad (3.0R, +$19.057)
rinde −$8.560 en la 2ª. Rango robusto para grado A: **0.7R a 1.5R**, positivo
en ambas mitades; a 2.0R se rompe.

**La entrada limitada NO arregla el sistema.** Limitada al close HA reduce la
pérdida de −$30.092 a −$10.299 (TP1 0.7R, validez 2 velas), pero **fuera de
muestra la 2ª mitad da negativo en todos los objetivos** (−$8k a −$15k). Y
**empeora si ya filtrás por grado A** (+$15.917 a mercado vs +$11.895 con
límite, a 1.2R). Razón mecánica: en grado A el δ/R medio es **−0.0164** y
**12 de 25 tuvieron precio real MEJOR que la etiqueta**; poner límite al
close HA renuncia a ese deslizamiento favorable. En B y C el δ/R medio es
+0.1927 y **0 de 87 fueron favorables**. Los dos tratamientos son
sustitutos, no complementos.

**Mejor combinación robusta: grado A + entrada a mercado, TP entre 0.7R y
1.5R.** Reservas: 26 operaciones en 2 meses (~13/mes), la magnitud de la
ventaja es inestable (+$876/op en la 1ª mitad vs +$92 en la 2ª), un solo
régimen de mercado, y sin slippage ni comisión más allá del spread (que no
es el problema: harían falta 5.35 pts para anular la ventaja del grado A).


### Backtest de 6 meses — SUPERSEDE al de 2 meses (30-sep-2026)

**Datos:** 299 señales, abr-sep 2026, 11.918 velas de 15m cargadas con
`tv range` sobre un rango de un año (el feed de VANTAGE llega hasta
2026-03-31). Validado **16/16** contra el indicador en la ventana solapada.
Config: Muy alta (150), TP1 0.7R, solo precio real.

**Lo que se confirmó:** el impuesto HA da **−0.125R por operación**
(−$93.676 sobre 299), consistente con −0.147R de los 2 meses y 0.118R de los
scores. Es estructural.

**Lo que CAMBIÓ:** con 2 meses el grado B parecía perder (−$35.781). Con 6
meses **B gana +$24.259**. No es contradicción: el TP1 mueve la frontera B/C,
y los dos backtests usaron TP1 distinto (1.0 vs 0.7). La variable de fondo es
δ/R, no la letra.

**P&L por δ/R (independiente del TP, el corte real está en ~0.15):**

| δ/R | n | $/op (TP 0.7R) | $/op (TP 1.2R) |
|---|---|---|---|
| < 0 | 54 | +$458 | +$464 |
| 0.00–0.05 | 37 | +$102 | +$260 |
| **0.05–0.10** | 48 | **+$412** | **+$697** |
| 0.10–0.15 | 52 | +$14 | +$147 |
| 0.15–0.25 | 67 | −$316 | −$250 |
| 0.25–0.40 | 30 | −$395 | −$116 |
| > 0.40 | 11 | −$234 | −$490 |

**El grado A (δ/R ≤ 0.05) es demasiado estricto:** tira la banda 0.05–0.15,
que fue la más rentable por operación. **Con TP1 = 0.7 la frontera B/C cae en
δ/R = 0.1333** — casi exactamente el umbral empírico. O sea que con esa
configuración, **operar A+B ≈ operar δ/R ≤ 0.133**, que es lo correcto.
Con TP1 = 1.0 la frontera cae en 0.333, demasiado permisiva — por eso el
backtest de 2 meses vio perder a B.

**Fuera de muestra (abr-jun entrena, jul-sep prueba):**
A +$13.361 → +$15.147 · B +$11.896 → +$12.363 · C −$4.218 → −$35.125.
Filtro A+B aplicado a jul-sep: **+$27.510** contra −$7.614 sin filtrar y
+$15.147 con solo A.

**Consistencia mensual (solo grado A): 6 de 6 meses positivo.** B: 3/6.
C: 2/6. En los dos meses malos (ago −$7.547 y sep −$22.431 sin filtrar), el
grado A aguantó: +$2.829 y +$631. El filtro se gana el sueldo justo cuando
el mercado se pone feo.

**Recomendación revisada:** con TP1 = 0.7, operar **A y B**, dejar C apagado.
A solo es más consistente (6/6 meses) pero deja sobre la mesa la banda más
rentable.


**Spread (hallazgo 30-sep):** el original **NO suma el spread al lotaje**.
Verificado con spread=0.2 sobre 3 operaciones: mostró 0.98 / 1.22 / 0.70,
que son exactamente `2500/(SL×100)` sin spread (con spread darían
0.97 / 1.21 / 0.70). Dónde sí lo use no se pudo determinar — probablemente
en las tablas de estadísticas internas, que están apagadas. La réplica copia
ese comportamiento (mismo lotaje) y usa el input solo para mostrar el costo
estimado en dólares en la etiqueta.

**Impacto medido del spread sobre el backtest de 6 meses (TP1 0.7R):**

| Selección | sin spread | spread 0.2 | spread 0.5 |
|---|---|---|---|
| A (n=91) | +$28.508 | +$27.111 | +$25.016 |
| B (n=80) | +$24.259 | +$22.966 | +$21.028 |
| **A+B (n=171)** | **+$52.767** | **+$50.078** | **+$46.044** |
| C (n=128) | −$39.343 | −$41.385 | −$44.448 |
| todas (n=299) | +$13.424 | +$8.693 | +$1.595 |

236.6 lotes operados en total → cada 0.1 de spread cuesta $2.366. El spread
se lleva el **35% del resultado si no filtrás** y solo el **5% con A+B**:
otra razón para filtrar, además de la obvia.


### ¿Conviene el filtro horario? — SÍ, y por bastante (30-sep-2026)

Probado sobre los mismos 6 meses, grados A+B, TP1 0.7R, spread 0.2. Sin
filtro horario salen **710 señales** (299 en sesión, 411 fuera).

| | n | acierto | P&L | $/op | drawdown | P&L/DD |
|---|---|---|---|---|---|---|
| **Con horario** | 184 | **64.1%** | **+$47.858** | **+$260** | **−$16.747** | **2.86** |
| Sin horario | 501 | 54.1% | +$43.059 | +$86 | −$23.476 | 1.83 |
| Solo fuera de sesión | 317 | 53.6% | +$26.194 | +$83 | | |

Gana en **todas** las dimensiones: más total con 2.7× menos operaciones,
3× más por operación, 10 puntos más de acierto y 40% menos drawdown.

**Fuera de muestra (abr-jun → jul-sep) es donde se decide:**
con horario +$22.680 → **+$25.178** (mejora) · sin horario +$24.234 →
+$18.825 (cae) · **solo fuera de sesión +$22.737 → +$3.457 (se derrumba)**.
Las operaciones fuera de sesión funcionaron en la primera mitad y
desaparecieron en la segunda.

**Elegir "las mejores horas" de los datos NO funciona** (otra vez): las 15
horas rentables de abr-jun rindieron **+$5.273** en jul-sep, peor que operar
todas las horas (+$18.825) y mucho peor que las sesiones del indicador
(+$25.178). El desglose por hora tiene signos que se dan vuelta entre
mitades (16:00: +$8.143 → −$9.151; 19:00: +$11.493 → −$5.178). Es ruido.

**Conclusión:** dejar `Filtrar por horario` en true. Las ventanas del
indicador no son un parámetro ajustable más — son lo único del horario que
sobrevive fuera de muestra.


### ¿5m para scalping? — conclusión de 7 semanas, CORREGIDA abajo (1-oct)

El feed de VANTAGE solo guarda **10.656 velas de 5m** (9-ago a 1-oct, ~7
semanas), contra 6 meses en 15m. Comparación sobre el **mismo período**
(10-ago a 1-oct), grados A+B, spread 0.2, sesiones 0700-1130 y 2000-2300.
Generación validada 33/34 contra el indicador.

| | n | acierto | P&L | $/op | duración mediana | drawdown | σ |
|---|---|---|---|---|---|---|---|
| **5 min** | 202 | 60.9% | +$44.379 | +$220 | **55 min** | −$19.470 | 1.75 |
| **15 min** | 58 | 63.8% | +$7.636 | +$132 | **180 min** | −$13.162 | 0.51 |

**Sí es más scalping:** duración mediana 55 min contra 180 (p25 25 vs 75,
p75 95 vs 405).

**Pero la ventaja no es real.** Todo el resultado de 5m viene de un mes:
agosto +$38.199 (87 ops) · **septiembre +$2.622 (110 ops)** · octubre
+$3.559 (5 ops). Y fuera de muestra (corte 3-sep) se derrumba:
**A+B primera mitad +$43.687 (n=105), segunda +$692 (n=97)**.

**Otra señal de alerta:** en 5m el grado **C fue rentable** (+$11.096,
+$150/op, positivo en las dos mitades), cuando en 15m sobre 6 meses pierde
−$613/op. El filtro por score no se comporta igual. Con 7 semanas no se
puede saber si es una propiedad del marco de 5m o ruido.

**El 15m rindió mal en esa misma ventana** (+$132/op, 0.51σ) contra +$283/op
y 2.07σ sobre los 6 meses — o sea que ago-sep fue un período flojo para 15m,
lo que infla la comparación a favor de 5m.

**Conclusión: quedarse en 15m.** Es el único de los dos que supera 2σ, y lo
hace sobre tres veces más historia. El 5m puede tener ventaja, pero con 7
semanas y el resultado concentrado en un mes no hay cómo demostrarlo. Para
decidirlo haría falta historial de 5m que el feed no da.


### Historia larga de 5m vía Replay + CORRECCIÓN sobre 5m vs 15m (1-oct-2026)

**Técnica para extender historia (funciona, usar esta):** `replay start
--date X` y después `tv range --from --to` con una ventana ancha. Devuelve
~10.000 velas de 5m por checkpoint, muy anteriores al límite en vivo. Tres
checkpoints (15-may, 1-jul, 15-ago) + la data viva dieron **37.853 velas de
5m, 22-mar a 1-oct, 6.3 meses**.

**Validación de los solapes: 4.464 velas comunes, 100% idénticas, diferencia
máxima 0.0000.** Los datos de Replay coinciden exactamente con los de vivo —
no hay repintado ni discrepancia de feed. Yahoo Finance NO sirve para esto
(límite de 60 días en intradía de 5/15m).

**CORRECCIÓN IMPORTANTE:** con solo 7 semanas se concluyó que el 5m no tenía
ventaja demostrable. **Era un artefacto del tamaño de muestra.** Con 6 meses
(mismo período que 15m, abr-sep, cuenta $5.000 al 1%, spread 0.2):

| | n | acierto | P&L | % cuenta | DD | P&L/DD | σ | duración |
|---|---|---|---|---|---|---|---|---|
| **5 min** | 637 | 60.4% | **+$2.245** | **+44.9%** | 9.1% | **4.92** | **2.44** | 50 min |
| 15 min | 194 | 64.9% | +$1.001 | +20.0% | 5.2% | 3.81 | 1.84 | 195 min |

**5m gana en total, en ajustado por riesgo y en significancia.** Fuera de
muestra las dos mejoran: 5m +19.3% → +25.6%; 15m +6.0% → +14.1%. Mes a mes,
5m solo fue negativo en abril (−4.3%); 15m solo en junio (−1.4%).

**El grado C sigue comportándose distinto por marco:** en 5m da +$697 sobre
6 meses (4/6 meses positivo) y en 15m pierde −$473. Pero **se está
debilitando** (1ª mitad +$555, 2ª +$142), así que no conviene activarlo
pese a que A+B+C da los mejores números en papel (+60.4%, 2.87σ, P&L/DD
6.70).

**Lo que hay que esperar operando 5m:** el primer mes fue **negativo**. La
cuenta cayó de $5.000 a $4.524 (−9.5%) entre el 2 y el 21 de abril antes de
recuperar. 637 operaciones en 6 meses son ~25 por semana.


### MES1! (Micro E-mini S&P 500) — NO funciona (1-oct-2026)

Probado con el pipeline completo en 5m y 15m. Datos: **31.246 velas de 5m**
(26-abr a 1-oct, 2 checkpoints de Replay + vivo, solapes 1.608/1.608
idénticos) y **10.048 velas de 15m** (30-abr a 1-oct).

**La fórmula del indicador SÍ transfiere** — centro = SMA(hl2,8) con error
0.00000000 sobre 1.537 velas de MES. Es agnóstico al instrumento. **Lo que
no transfiere es la ventaja.**

| | n | acierto | R/op | σ |
|---|---|---|---|---|
| MES 5m A+B en sesión | 665 | 43.5% | −0.0115 | **−0.37** |
| MES 15m A+B en sesión | 215 | 38.1% | −0.1357 | **−2.42** |
| *oro 5m (referencia)* | *869* | *51.4%* | *+0.088* | *+2.71* |

En 15m **pierde de forma significativa** (−2.42σ, negativo 5 de 6 meses).
En 5m queda plano y no se sostiene (1ª mitad +4.8R, 2ª −12.5R). Ningún
objetivo rescata: barrido 0.5R a 3.0R, en 15m todos negativos; en 5m solo
0.5R/0.7R dan +2.9R sobre 696 ops (ruido).

**No es culpa del impuesto HA**, que es igual al del oro (δ/R medio 0.102 en
5m y 0.129 en 15m, contra ~0.09 y ~0.13 del oro). Es la señal: el acierto
cae de 51% a 39-44% con el mismo objetivo. Hipótesis: el canal detecta
rupturas de volatilidad; el oro rompe y sigue, el índice rompe y revierte.

**Problema aparte, de tamaño:** MES vale **$5 por punto por contrato**
(tick 0.25 × $1.25) y no se fracciona. Con SL típico de 14.5 puntos, 1
contrato arriesga **$72 = 1.45%** de una cuenta de $5.000. Solo el **14%** de
las señales entran en un presupuesto de $50. Harían falta ~$25.000 para
operar al 1% todas las señales.

**Lección metodológica:** cada instrumento se mide por separado, igual que
cada temporalidad. Ya había pasado con 5m vs 15m y con el grado C.


### Configuración operativa vigente (1-oct-2026)

Original **y** réplica en el chart, idénticos: `VANTAGE:XAUUSD` · **5
minutos** · Heikin Ashi · sensibilidad **Muy alta** (150) · amplitud del stop
**Normal** (3.0×ATR14) · **TP 1.0 / 1.2 / 1.5** · dirección Ambas · horarios
**0700-1130** y 2000-2300 New York · entrada a **Mercado** · **grados A y B
activos, C apagado** · balance **$5.000** al **1%** ($50/op) · spread **0.2**
· tick 0.01 / $1.

**Validada como óptima robusta el 7-oct** (ver "Optimización completa" más
abajo). Lo único que se agrega, a nivel robot: **máximo 2 posiciones
simultáneas** y cierre por señal contraria en sesión.

**Control cruzado (1-oct, tras pasar TP1 de 0.7 a 1.0):** sobre 1.588 velas
de 5m, **47 señales operables en los dos, 46 coincidencias exactas**. La
única diferencia es 29-sep 01:20 vs 01:25 — misma dirección, una vela — por
el residuo de banda en un cruce ajustado. Bandas con 0.04 pts de diferencia.

**Por qué TP1 = 1.0** (medido sobre 6 meses de 5m): sube de +46.4% a +68.8%
de la cuenta, con la mejor significancia del proyecto (**2.71σ**), y gana en
**las dos mitades** fuera de muestra (+32.0% y +36.8%, contra +18.1% y
+28.3% con 0.7). Dos tercios de la mejora vienen de que la frontera B/C se
corre de δ/R 0.145 a 0.347 y entran 222 operaciones más (+$690, positivas en
ambas mitades); un tercio del objetivo más ancho. **No estirar más allá de
1.2R:** a 1.5R el resultado se desbalancea entre mitades (+23.0% vs +48.7%).

**Advertencia que resultó equivocada:** se avisó que subir TP1 metería
operaciones malas, extrapolando del 15m donde δ/R > 0.15 pierde. **En 5m esa
banda es rentable** — misma diferencia por marco que ya mostraba el grado C.
No trasladar conclusiones de un marco al otro sin re-medir.

**Costo de TP1 = 1.0:** el acierto baja de 60.4% a 51.4% y el drawdown sube
de 11% a 13.5% de la cuenta. Se gana con ganadoras más grandes, no
acertando más.

**Umbrales de grado en la réplica:** A: `δ/R ≤ 0.05` (exacto, 16/16) ·
B/C: `score < 0.485` (aproximado; el original corta entre 0.480 y 0.490).

**Gotchas de la réplica:** (1) `umbralBC` está declarado fuera del bloque de
inputs, así que Pine le asigna un índice al final — si se agregan inputs,
revisar que los valores guardados no se desalineen. (2) Tras un
`indicator set` puede quedar colgada en `status.type = 0` sin datos y sin
error; se destraba con `indicator toggle <id> --hidden` y después
`--visible`. (3) Al reiniciar TradingView las bandas difieren más de lo
normal (0.15 vs 0.04) hasta que carga historia suficiente para el ATR(120).


**Horario 1 = 0700-1130 (decidido 30-sep).** La media hora extra respecto de
0730 agrega 10 operaciones A+B en 6 meses: +$7.052 con error estándar de
$5.695 = **1.24σ, ruido puro**. El detalle lo confirma: las 2 pérdidas en
abr/jun y las 8 ganancias concentradas de ago en adelante — una racha. Las
dos opciones son equivalentes; se dejó 0700 y no hay que volver a tocarlo.

**Fuerza estadística, para no sobre-confiar:** el resultado principal (184
ops en sesión, +$260/op, desvío $1.908) está a **1.85σ** de cero —
por debajo del umbral convencional de 2. Lo que sostiene la conclusión es la
consistencia (grado A positivo 6/6 meses, ambas mitades positivas, aguantó
ago y sep), no el t-stat. **Probable, no seguro.**

Separando lo que es medición de lo que es inferencia:
certeza → fórmula del canal/señales (40/40) y del score (8/8) ·
muy firme → impuesto HA ~0.13R (3 mediciones independientes) ·
firme → el grado C pierde, las sesiones ganan ·
probable → que A+B rinda lo que rindió ·
nada → que 0700 sea mejor que 0730.

**Gotcha:** tras un `indicator set` la réplica puede quedar colgada en
`status.type = 0` (cargando) sin datos y sin error — `tv values` deja de
listarla. Se destraba con `indicator toggle <id> --hidden` y después
`--visible`. No es que esté rota; hay que forzarle el recálculo.


### Backtest con la cuenta real: $5.000 al 1% (1-oct-2026)

5m, A+B, sesiones 0700-1130 y 2000-2300, spread 0.2, 10-ago a 1-oct.

| | n | acierto | P&L | % cuenta |
|---|---|---|---|---|
| **A+B** | 202 | 60.9% | **+$929** | **+18.6%** |
| solo A | 88 | 59.1% | +$470 | +9.4% |
| solo B | 114 | 62.3% | +$459 | +9.2% |
| solo C | 74 | 70.3% | +$232 | +4.6% |

Drawdown máximo **$381 = 7.6% de la cuenta**, P&L/DD 2.44. Peor operación
−$63 (−1.3%). Peor racha: **5 pérdidas seguidas**.

**Mismo problema de concentración:** agosto +14.4% · septiembre **+2.7%** ·
octubre +1.5%. Fuera de muestra: 1ª mitad +15.8%, 2ª mitad **+2.8%**.

**Redondeo de lotes — importante con cuenta chica.** Con $50 de riesgo los
lotes van de **0.01 a 0.07** (mediana 0.03), y al redondear a 2 decimales el
riesgo real por operación queda entre **$33 y $66** (objetivo $50), con
**10% de desvío medio**. Distribución: 0.01→14 ops · 0.02→73 · 0.03→71 ·
0.04→39 · 0.05→4 · 0.07→1. El efecto sobre el P&L fue +$41 (azar, no sesgo).
Con 14 operaciones en el lote mínimo, si la cuenta fuera más chica varias
señales quedarían sin poder dimensionarse.


### Fijo vs compuesto — proyección Monte Carlo (1-oct-2026)

**Todos los backtests del proyecto usan riesgo FIJO** sobre el balance
inicial, sin recalcular tras cada operación. El indicador hace lo mismo: su
"Balance de cuenta" es un número manual.

Sobre las 7 semanas reales de 5m, componer no cambió nada: $5.922 contra
$5.929 del fijo, y **peor drawdown** (7.7% vs 6.4%) porque el pico de equity
cayó justo antes del tramo plano. Con lotes exactos el compuesto gana $12
(0.2%): el redondeo a 0.01 se come la ventaja.

**Proyección a 12 meses** (remuestreo de las operaciones reales, 1.416
ops/año — el ritmo observado —, 4.000 simulaciones, cuenta $5.000):

| Universo | Sizing | Mediana | p10 | p90 | Prob. pérdida | DD mediano |
|---|---|---|---|---|---|---|
| Todo el período (R=0.088) | fijo | $11.527 | $9.787 | $13.228 | 0.0% | 5.0% |
| Todo el período | **compuesto** | **$16.698** | $11.880 | $23.553 | 0.0% | 7.9% |
| 2ª mitad (R=0.013) | **fijo** | **$6.975** | $5.171 | $8.742 | 8.1% | 13.8% |
| 2ª mitad | compuesto | $5.827 | $4.127 | $8.366 | **26.4%** | 21.7% |

**La regla que sale de esto:** componer amplifica en las dos direcciones.
Con ventaja fuerte (R=0.088) suma 45% sobre el fijo; con ventaja marginal
(R=0.013, que es la mitad fuera de muestra y la más honesta) **resta**, y
triplica la probabilidad de terminar perdiendo (26.4% vs 8.1%) con un
drawdown mediano que pasa de 13.8% a 21.7%.

**Limitación grande del método:** el remuestreo trata las operaciones como
independientes, y en la realidad las rachas se agrupan por régimen (agosto
bueno, septiembre plano). Eso significa que **los drawdowns reales van a ser
peores** que estos percentiles. La proyección sirve para comparar fijo vs
compuesto, no para predecir el saldo.


### Optimización completa de parámetros — configuración final (7-oct-2026)

**Pregunta:** cuál es la mejor combinación de TP, grados, horarios,
posiciones simultáneas, sensibilidad, SL, cierre por señal contraria, BE y
tope diario. **Respuesta corta: la configuración vigente ya era la óptima
robusta; el único cambio que sobrevive es limitar a 2 posiciones abiertas.**

**Método:** grilla de 21.600 combinaciones en 5m y 20.925 en 15m (abr-sep),
después walk-forward, análisis marginal por tramos (abr-may, jun-jul,
ago-sep), vecindario de la config vigente, y **holdout real 1-7 oct** (datos
nuevos que no existían cuando se diseñó nada). Simulador `final_sim.js` con
orden intravela conservador: **SL → TP → cierre por contraria al close**.

**El campeón in-sample es sobreajuste.** Mejor de la grilla en 5m: nivel 150,
SL 3×ATR, TP 2.0R, δ/R≤0.35, **24h**, posiciones ilimitadas → +187.7% con DD
31.8%. Walk-forward: optimizar en julio y aplicar a ago-sep dio **+25.2%**
contra **+26.6%** de la config vigente sin tocar. Correlación entre ranking
in-sample y out-of-sample: 0.36-0.40. En la semana nueva de octubre el
campeón dio +7.9% (70 ops, 43% acierto) y el campeón "peor tramo" +6.8%,
contra **+9.9%** de la recomendada.

**Lo que es robusto (gana en todos o casi todos los tramos):**
- Cerrar al aparecer una señal contraria **dentro de sesión** (sin importar
  su grado): gana en todos los tramos, en 5m y 15m.
- Filtro de sesión: 15m 3/3 tramos, 5m 2/3, y DD 14.0% vs 21.6% sin él.
- Sensibilidad 130-150. En 5m los niveles 60-100 pierden.

**Lo que NO es robusto (el ganador cambia de tramo en tramo):** tamaño del
TP, multiplicador del SL, umbral δ/R, número de posiciones. Por eso no se
optimizan: se dejan en el centro de la meseta.

**Vecindario de la vigente:** meseta ancha, casi todo positivo en 3/3
tramos. Único cambio que domina en todas las métricas: **máximo 2
posiciones** (+74.3% vs +68.0%, DD 12.3% vs 13.5%, 2.99σ vs 2.68σ; misma
semana de octubre +9.9%). TP 1.2 da +83.9% / 3.12σ / oct +9.6% —
**equivalente dentro del ruido**, no justifica cambiar. TP 2.0 da más total
pero desbalanceado entre tramos.

**Secundarios:** BE 0.5R +75.2% (oct +8.5%, no aporta) · BE 0.7R +82.7%
pero mejora solo 2/3 tramos (ruido) · tope 10 ops/día sin efecto (+73.6%,
dejarlo como seguro) · **tope 5/día destruye** (+40.7%).

**CONFIGURACIÓN FINAL (para el robot):**

| Parámetro | Valor | Robustez |
|---|---|---|
| Símbolo / marco | VANTAGE:XAUUSD 5m, Heikin Ashi | 5m > 15m (2.44σ vs 1.84σ) |
| Sensibilidad | Muy alta (150) | robusto (130-150) |
| SL | 3.0 × ATR(14) (Normal) | meseta |
| TP | 1.0R (1.2R equivalente) | meseta, no optimizar |
| Grados | A+B = **δ/R ≤ 0.35** | meseta |
| Horarios | 0700-1130 y 2000-2300 NY | robusto |
| Cierre por contraria | sí, señales en sesión, cualquier grado | robusto |
| Posiciones simultáneas | **máx 2** | único cambio |
| Break even | no | ruido |
| Tope diario | 10 (seguro, no filtro) | sin efecto |
| Entrada | mercado | ya medido |
| Riesgo | 1% fijo sobre balance | deliberadamente no optimizado |

Resultado (abr-2 a oct-7, $5.000): 871 ops, **$9.212 final (+84%)**, DD
máximo $616 (12.3%). Holdout 1-7 oct: +9.9%.

**Implicaciones para el robot:** (1) filtrar por δ/R directamente, no por la
letra (la letra depende de TP1); (2) en la misma vela, primero cerrar por
contraria y después abrir la nueva; (3) SL/TP se evalúan intravela antes del
cierre por contraria, que solo se conoce al close; (4) el límite de 2
posiciones es del robot — el indicador no limita concurrencia.

**Advertencia:** la semana de holdout son ~33 operaciones — es una
confirmación de que nada se rompió, no una prueba estadística.


### Calendario macro (oro)
- Fuente base **ForexFactory**, + Retail Sales de **Investing.com**,
  Investing como segunda opinión.
- **ForexFactory muestra hora fija UTC-6 sin DST.** Para pasar a ET:
  +1h antes del 2026-03-08, +2h después. (Validado: los 7 NFP caen
  exactos en 08:30 ET.)
- 191 eventos USD de alto impacto en 2026 ya recolectados.
