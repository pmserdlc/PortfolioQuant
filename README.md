# PortfolioQuant — Sistema de Análisis y Optimización de Cartera

Sistema cuantitativo de análisis, optimización y recomendación de carteras de inversión para el inversor particular español. Genera un dashboard HTML completamente auto-contenido con todos los datos embebidos, sin necesidad de servidor ni base de datos.

> **Entorno:** Python 3.14 · Windows · `.venv`
> **Última actualización:** 21 de abril de 2026
> **Estado:** ✅ Producción · ✅ Multiperfil · ✅ Dashboard interactivo

---

## Ejemplo de salida (perfil demo)

El sistema genera automáticamente las siguientes métricas al ejecutar el pipeline completo:

| Métrica | Ejemplo (demo) |
|---|---|
| Capital analizado | 184.270 € |
| Sharpe actual | 0.610 |
| Sharpe óptimo (Máx Sharpe) | 0.705 (+0.095) |
| Rendimiento esperado actual | 14.65% |
| Rendimiento óptimo | 15.96% |
| Volatilidad actual | 18.85% |
| Plusvalía neta latente | 32.470 € |
| IRPF estimado (si se realiza todo) | ~6.699 € |

**Escenarios de estrés incluidos:**

| Escenario | Impacto estimado |
|---|---|
| COVID Crash (Mar 2020, 1 mes) | ~-20% a -26% según escenario |
| Bear 2022 — tipos al alza (año completo) | ~-17% a -19% según escenario |
| Corrección Tech 30% (hipotético) | ~-13% a -15% según escenario |

> Los datos del perfil demo son ficticios y sirven únicamente como referencia de las capacidades del sistema. Cada usuario trabaja con sus propios datos en `datos/cartera/`.

---

## Índice

1. [Arquitectura del sistema](#1-arquitectura-del-sistema)
2. [Datos de entrada](#2-datos-de-entrada)
3. [Pipeline de módulos](#3-pipeline-de-módulos)
4. [Dashboard HTML](#4-dashboard-html)
5. [Motor de alertas de cartera](#5-motor-de-alertas-de-cartera)
6. [Perfiles múltiples](#6-perfiles-múltiples)
7. [Asistente de IA integrado](#7-asistente-de-ia-integrado)
8. [Flujo de trabajo](#8-flujo-de-trabajo)
9. [Dependencias e instalación](#9-dependencias-e-instalación)
10. [Limitaciones conocidas](#10-limitaciones-conocidas)
11. [Roadmap](#11-roadmap)

---

## 1. Arquitectura del sistema

```
optimizacion-inversiones/
├── modelos/                         → pipeline Python (módulos 01-04)
├── dashboard/
│   ├── generar_dashboard.py         → generador HTML (fuente de verdad)
│   ├── index.html                   → dashboard perfil principal
│   ├── demo.html                    → dashboard perfil demo
│   └── maa3.html                    → dashboard perfil maa3
├── datos/
│   ├── cartera/                     → posiciones, estrategia DCA, movimientos
│   ├── historico/                   → precios, retornos, sentiment, noticias
│   ├── perfil/                      → perfil_inversor.json, objetivos_financieros.json
│   ├── perfiles/                    → datos por perfil alternativo (demo, maa3)
│   └── productos/                   → universo_productos.json, carteras_referencia.json
├── resultados/                      → pesos_optimos.csv, hrp_pesos.csv, informes, gráficos
├── scripts/                         → utilidades de mantenimiento
├── notebooks/                       → análisis interactivo Jupyter
└── .github/                         → instrucciones y skills para GitHub Copilot
    ├── copilot-instructions.md
    ├── skills/                      → 5 dominios especializados
    ├── agents/                      → 4 agentes especializados
    ├── instructions/                → reglas por tipo de archivo
    └── prompts/                     → workflows reutilizables
```

---

## 2. Datos de entrada

### `posiciones_actuales.csv`

Posiciones vivas con coste histórico y valoración actual.

| Campo | Tipo | Descripción | Ejemplo |
|---|---|---|---|
| `producto` | texto | Nombre del fondo/activo | `MSCI World Index P ACC EUR` |
| `isin` | texto | ISIN o identificador | `LU1737652583` |
| `tipo` | enum | fondo / etf / acciones / roboadvisor / crypto / efectivo | `fondo` |
| `subtipo` | enum | indexado / activo / monetario / pension / btc / liquido... | `indexado` |
| `entidad` | texto | Broker o entidad custodio | `MyInvestor` |
| `riesgo_1_10` | 1-10 | Nivel de riesgo subjetivo | `7` |
| `coste_eur` | decimal | Precio de coste total en EUR | `25000.00` |
| `valor_mercado_eur` | decimal | Valor de mercado actual en EUR | `31500.00` |
| `ganancia_latente_eur` | decimal | Plusvalía/pérdida latente en EUR | `6500.00` |
| `ganancia_pct` | decimal | Rentabilidad (0.26 = 26%) | `0.26` |
| `notas` | texto | Comentarios libres (entre comillas si hay comas) | |
| `fecha_compra` | YYYY-MM-DD | Fecha de primera aportación (opcional) | `2021-03-15` |
| `divisa_base` | texto | Divisa de cotización (opcional, vacío = EUR) | `USD` |
| `distribucion` | enum | acumulacion / distribucion (opcional) | `acumulacion` |
| `dividendo_pendiente_eur` | decimal | Dividendos sin reinvertir en EUR (opcional) | `0` |
| `anios_minusvalia` | entero | Antigüedad de pérdidas para compensar (opcional) | `2` |

### `estrategia_activa.csv`

Estrategias DCA o traspasos programados activos.

| Campo | Descripción |
|---|---|
| `id` | Identificador único |
| `origen` / `destino` | Producto de origen y destino |
| `importe_mensual_eur` | Aportación periódica |
| `meses` | Duración total en meses |
| `estado` | activa / completada / pausada |
| `fecha_inicio` / `fecha_ultima_ejecucion` | Fechas de control (YYYY-MM-DD) |

### `posiciones_bolsa_americana.csv`

Posiciones en acciones USA cotizadas en USD.

| Campo | Descripción |
|---|---|
| `ticker` | Símbolo de cotización |
| `nombre` / `sector` | Nombre y sector |
| `posicion` | Número de acciones |
| `precio_usd` / `valor_mercado_usd` | Precio y valor actual |
| `base_coste_usd` / `precio_medio_usd` | Coste total y precio medio |
| `pyg_no_realizada_usd` / `pyg_no_realizada_pct` | Plusvalía/pérdida latente |

### Archivos de perfil

| Archivo | Contenido clave |
|---|---|
| `datos/perfil/perfil_inversor.json` | Horizonte, perfil riesgo, hipoteca, plataformas disponibles |
| `datos/perfil/objetivos_financieros.json` | Capital objetivo, aportación mensual, horizonte en años |
| `datos/productos/universo_productos.json` | 24 ETFs/fondos UCITS con TER, Sharpe 5a y plataformas |
| `datos/productos/carteras_referencia.json` | 7 filosofías: Dalio, Browne, Bogle, Swensen, Buffett, ARK, España |

---

## 3. Pipeline de módulos

```
01_descarga_datos.py  →  02_analisis_cartera.py  →  03_optimizador.py
                                                  →  03b_optimizador_hrp.py
                                                  →  04_recomendador.py
                                                             ↓
                                         dashboard/generar_dashboard.py
```

### `01_descarga_datos.py` — Descarga y contexto macro

Pipeline completo en 10 pasos (~60 segundos):

| Paso | Función | Output |
|---|---|---|
| 1-4 | Precios históricos + indicadores macro + score geopolítico + alertas | `precios_historicos.csv`, `parametros_mercado.json` |
| 5 | Métricas reales del universo de 24 ETFs/fondos | Actualiza `universo_productos.json` |
| 6 | Autodescubrimiento de sectores emergentes (EMA baseline α=0.3) | `autodescubrimiento.json` |
| 7 | Scanner de momentum en 35+ sectores (Sharpe, SMA50, benchmark) | `tendencias_sector.json` |
| 8 | Google News RSS — buzz score 0-100 por tema (15 topics) | `noticias_sectores.json` |
| 9 | Cruce precio × noticias → EMERGENTE / CONFIRMADO / AGOTADO | `oportunidades_detectadas.json` |
| 10 | Alertas accionables con productos concretos y pesos por perfil | `alertas_tendencias.txt` + `.json` |

```bash
python modelos/01_descarga_datos.py   # ejecutar 1 vez/semana
```

### `02_analisis_cartera.py` — Análisis completo

Análisis de distribución, plusvalías fiscales, cartera USA y estrategia DCA. Genera 6 gráficos y un informe de texto. Usa caché de `parametros_mercado.json` — no requiere conexión a internet.

```bash
python modelos/02_analisis_cartera.py   # ~5s
```

### `03_optimizador.py` — Markowitz Media-Varianza

Optimización con 10 clases de activos, correlaciones empíricas reales (5 años) y restricciones de peso (2%-30% por activo). Genera 3 escenarios: Máximo Sharpe, Mínima Volatilidad, Máximo Rendimiento con vol ≤ 18%.

Métricas incluidas: Sharpe, Sortino, CVaR 95%, escenarios de estrés históricos.

```bash
python modelos/03_optimizador.py   # ~45s
```

### `03b_optimizador_hrp.py` — Hierarchical Risk Parity

Implementación de HRP (López de Prado 2016). Pipeline de 3 pasos: clustering jerárquico Ward → quasi-diagonalización → bisección recursiva. Genera HRP puro, HRP acotado y contribución al riesgo por clase.

```bash
python modelos/03b_optimizador_hrp.py   # ~10s
```

### `04_recomendador.py` — Cartera recomendada por filosofía

Combina HRP con el universo de 24 productos reales disponibles en las plataformas del inversor. Filtra por plataformas disponibles, prioriza fondos traspasables (IRPF), y proyecta el capital a N años.

Compara la cartera recomendada contra 7 filosofías: All Weather, Permanent Portfolio, Bogle 3-Fund, Yale Endowment, Buffett Value, ARK Disruptive, Clásica España.

```bash
python modelos/04_recomendador.py   # ~5s
```

---

## 4. Dashboard HTML

El dashboard se genera como un único archivo HTML auto-contenido con todos los datos embebidos (sin servidor, sin llamadas externas).

### Estructura de pestañas

| Pestaña | Contenido |
|---|---|
| **Perfil** | Situación hipoteca, motor de alertas (23 checks), tabla de posiciones |
| **Optimización** | Pesos Markowitz (3 escenarios), HRP, comparativa y rebalanceo |
| **Recomendaciones** | Cartera recomendada, 7 filosofías de inversión, proyección de capital |
| **Mercado** | Sentiment Reddit, noticias por sector, tendencias y oportunidades |
| **Estrategia** | DCA activos, movimientos MyInvestor, plusvalías USA |

### Generación

```bash
python dashboard/generar_dashboard.py                  # → dashboard/index.html
python dashboard/generar_dashboard.py --perfil demo    # → dashboard/demo.html
python dashboard/generar_dashboard.py --perfil maa3    # → dashboard/maa3.html
```

---

## 5. Motor de alertas de cartera

La pestaña **Perfil** incluye 23 checks automáticos agrupados en 6 bloques temáticos:

| Bloque | N | Ejemplos |
|---|---|---|
| **K** — Concentración | 3 | Magnificent 7 >25%, RV+Crypto >85%, USD no cubierto >40% |
| **F** — Fiscal | 4 | Escalonamiento IRPF, minusvalías a punto de caducar, ETF+fondo duplicado, dividendos sin reinvertir |
| **T** — Temporal | 3 | Rebalanceo pendiente (SAD >15pp), liquidez <3 meses gastos, DCA sin ejecutar >45d |
| **O** — Diversificación | 2 | Más de 12 posiciones, posiciones zombi <1.500€ |
| **C** — Contraparte | 3 | Sin diversificación jurisdicciones, suma entidad >FOGAIN 100k€, brokers omnibus |
| **M** — Macro | 2 | Sin activos anti-inflación (oro/TIPS/REITs), sin exposición materias primas |

Severidades: 🔴 Urgente · 🟠 Atención · 🟡 Informativo · 🔵 Oportunidad

---

## 6. Perfiles múltiples

| Perfil | Datos | Dashboard | Descripción |
|---|---|---|---|
| `principal` | `datos/` | `dashboard/index.html` | Cartera real del inversor |
| `demo` | `datos/perfiles/demo/` | `dashboard/demo.html` | Demostración con datos ficticios |
| `maa3` | `datos/perfiles/maa3/` | `dashboard/maa3.html` | Perfil alternativo |

Crear nuevo perfil:
```bash
python scripts/nuevo_perfil.py --nombre <nombre> --desde demo
```

---

## 7. Asistente de IA integrado

El proyecto incluye una arquitectura completa de skills y agentes para GitHub Copilot en `.github/`:

### Agentes especializados (`@nombre-agente`)

| Agente | Capacidades | Acceso |
|---|---|---|
| `@quant-analyst` | Markowitz, HRP, métricas riesgo, backtesting | Solo lectura |
| `@tax-optimizer` | IRPF, traspasos fondos, minusvalías, FOGAIN | Solo lectura |
| `@dashboard-builder` | Modifica generar_dashboard.py y regenera | Lectura + edición + ejecución |
| `@data-updater` | Actualiza CSVs y ejecuta el pipeline | Lectura + edición + ejecución |

### Prompts reutilizables (`/nombre-prompt`)

| Prompt | Descripción |
|---|---|
| `/analizar-cartera` | Análisis ejecutivo completo de la cartera |
| `/nueva-alerta` | Guía para añadir una alerta al motor |
| `/nuevo-perfil` | Crea un nuevo perfil de inversor completo |

---

## 8. Flujo de trabajo

### Actualización semanal completa

```bash
# 1. Actualizar datos de cartera si hubo operaciones
#    editar: datos/cartera/posiciones_actuales.csv

# 2. Descargar datos de mercado (~60s)
python modelos/01_descarga_datos.py

# 3. Ejecutar análisis y optimización (~65s total)
python modelos/02_analisis_cartera.py
python modelos/03_optimizador.py
python modelos/03b_optimizador_hrp.py
python modelos/04_recomendador.py

# 4. Regenerar dashboard
python dashboard/generar_dashboard.py

# — o todo de una vez —
actualizar_todo.bat
```

### Cuándo re-ejecutar cada módulo

| Evento | Módulos |
|---|---|
| Nuevas posiciones / cambio de valor | `02` + dashboard |
| Compra o venta de activos | `02` + `03` + `03b` + dashboard |
| Cambio de perfil de riesgo | `03` (ajustar restricciones) |
| Ejecución mensual de DCA | Actualizar `estrategia_activa.csv` + `02` |
| Cambio de tipo hipoteca / Euribor | Actualizar `perfil_inversor.json` |
| Actualización semanal de mercado | `01` → todos → dashboard |

### Publicar en GitHub

```bash
subir_github.bat   # fetch + rebase + commit + push automático
```

---

## 9. Dependencias e instalación

```bash
# Crear entorno virtual
py -3 -m venv .venv
.venv\Scripts\activate

# Instalar dependencias
pip install -r requirements.txt
```

**Dependencias principales:**

```
yfinance>=0.2.50      → descarga de precios históricos
pandas>=2.2.0         → manipulación de datos
numpy>=1.26.0         → álgebra lineal y optimización
scipy>=1.13.0         → optimización numérica (SLSQP)
matplotlib>=3.8.0     → generación de gráficos
seaborn>=0.13.0       → estilos estadísticos
requests>=2.31.0      → API calls (ApeWisdom sentiment)
jupyter>=1.0.0        → notebooks interactivos
```

---

## 10. Limitaciones conocidas

| Limitación | Causa | Solución actual |
|---|---|---|
| Tickers delisted sin cotización | Posiciones OTC o suspendidas | `valor_mercado_usd = 0`; no contabilizados |
| TC EUR/USD fijo en módulo 02 | Sin API de divisas activa | `01_descarga_datos.py` descarga el tipo real |
| CVaR paramétrico (asume normalidad) | Colas gordas subestimadas | Shocks históricos calibrados como proxy |
| Correlaciones en régimen normal ≠ crisis | Régimen de estrés eleva correlaciones | Escenarios de estrés históricos como corrección |
| Sin ejecución de órdenes automática | Sin integración con API de broker | Alertas de rebalanceo manuales |

---

## 11. Roadmap

### Mejoras de modelos cuantitativos

| Estado | Módulo / Mejora | Descripción |
|---|---|---|
| ✅ Implementado | `03_optimizador.py` — Markowitz (MaxSharpe / MinVol / MaxRend) | Optimización clásica con 50 arranques multi-inicio |
| ✅ Implementado | `03b_optimizador_hrp.py` — HRP (Hierarchical Risk Parity) | Bisección recursiva sobre clúster jerárquico, acotamiento de pesos |
| ✅ Implementado | Ledoit-Wolf shrinkage | Regularización de la covarianza muestral en `01_descarga_datos.py`; coeficiente guardado en `parametros_mercado.json` |
| 📅 P1 | `05_rebalanceo_bandas.py` | Alertas de rebalanceo con bandas de tolerancia ±X% y priorización fiscal |
| 📅 P1 | `07_tax_harvesting.py` | Cosecha fiscal automatizada: pérdidas compensables con ganancia del ejercicio |
| 📅 P1.5 | `03c_black_litterman.py` | Prior bayesiano sobre equilibrio de mercado + views macro/sentiment; fórmula: $E[R] = [(\tau\Sigma)^{-1} + P^\top\Omega^{-1}P]^{-1}[(\tau\Sigma)^{-1}\pi + P^\top\Omega^{-1}q]$ |
| 📅 P2 | `03d_risk_parity.py` | Equal Risk Contribution (ERC): cada activo aporta $\sigma_p/N$ de riesgo total |
| 📅 P2 | `06_montecarlo.py` | Simulaciones Monte Carlo con percentiles P10/P50/P90 usando Cholesky de la covarianza |
| 📅 P2 | `04_auditoria_costes.py` | Scanner de solapamientos entre fondos + TER total de la cartera |
| 📅 P3 | `03e_optimizacion_robusta.py` | Optimización robusta: incertidumbre en $\hat{\mu}$ modelada como elipsoide |
| 📅 P3 | `08_sentimiento.py` | Radar de sentimiento social ampliado (Reddit, Google Trends, RSI) |

### Arquitectura IA (GitHub Copilot)

| Estado | Componente | Descripción |
|---|---|---|
| ✅ Implementado | `copilot-instructions.md` | Contexto global del proyecto (esquemas CSV, convenciones, fiscalidad) |
| ✅ Implementado | 5 Skills (`quant-finance`, `financial-advisor-spain`, `dashboard-builder`, `portfolio-alerts`, `data-pipeline`) | Conocimiento especializado por dominio |
| ✅ Implementado | 4 Agentes (`Quant Analyst`, `Tax Optimizer`, `Dashboard Builder`, `Data Updater`) | Roles con scope de herramientas delimitado |
| ✅ Implementado | 3 Instructions files + 3 Prompts | Contexto automático por tipo de archivo, tareas predefinidas |
