# Visor SSA Uruguay

Imagen Situacional Espacial (SSA) para Uruguay: una consola web que muestra el estado del entorno orbital sobre territorio nacional, identifica activos críticos, y detecta amenazas (conjunciones, sobrevuelos, reentradas, anomalías) a partir de datos públicos revalidados.

Es una aplicación **autocontenida en un único archivo HTML**, sin backend ni instalación: toda la propagación orbital (SGP4) corre en el navegador. Arranca al instante con un catálogo de muestra embebido y puede conectarse a datos en vivo de CelesTrak.

> **Estado del proyecto:** prototipo / MVP de conciencia situacional (Fase 0 de la escalera de madurez SSA). Es una herramienta de **triaje y visualización**, no un sistema operacional de evasión de colisiones. Ver [Limitaciones](#limitaciones).

---

## Tabla de contenidos

- [Funcionalidades](#funcionalidades)
- [Cómo funciona (técnico)](#cómo-funciona-técnico)
- [Fuentes de datos](#fuentes-de-datos)
- [Índice de confianza](#índice-de-confianza)
- [Limitaciones](#limitaciones)
- [Uso](#uso)
- [Configuración](#configuración)
- [Hoja de ruta](#hoja-de-ruta)
- [Stack y atribuciones](#stack-y-atribuciones)

---

## Funcionalidades

### Visualización
- **Mapa mundial** equirectangular con costas de alta resolución (Natural Earth 50m) y **fronteras nacionales**.
- **Zoom y desplazamiento**: rueda del mouse (hacia el cursor), arrastre, doble clic, botones, centrado rápido en Uruguay/estación.
- **Terminador día/noche** dinámico calculado desde la posición solar.
- **Territorio uruguayo** resaltado (terrestre + área marítima aproximada).
- **Domo celeste polar (az/el)**: el cielo local de la estación con el cenit al centro y barrido tipo radar.
- **Estelas (ground tracks)** recientes de cada objeto.
- **Marcadores diferenciados**: estación terrena (cruz dorada), activos del registro (color por tipo + bandera uruguaya para activos soberanos), y objetos coloreados por estado o por nivel de confianza.

### Imagen situacional
- Tabla de **objetos sobre el horizonte** en tiempo real (azimut, elevación, rango, iluminación solar).
- Estado **día/noche y fase de crepúsculo** en la estación (clave para planificar observación óptica).
- Estaciones predefinidas: **OALM (Los Molinos), Montevideo, Salto**.
- Reloj de simulación con velocidad ajustable (1x a 900x) para adelantar y ver eventos futuros.

### Registro nacional de activos
- Lista editable de **activos críticos** clasificados como soberano, GNSS/PNT, comunicaciones u observación.
- **Estado en vivo** por activo: nominal, en vista, sobrevolando, alerta, decaimiento.
- Bandera uruguaya en el mapa para los activos soberanos.

### Motor de análisis
Ejecuta sobre una ventana temporal configurable, **en tramos para no bloquear la interfaz**, y produce un feed de eventos con severidad:
- **Conjunciones** (cribado por distancia mínima) entre activos del registro y el resto del catálogo, con prefiltro por solape de capa orbital.
- **Sobrevuelos de territorio** nacional (terrestre y marítimo) con horarios de entrada/salida.
- **Candidatos a reentrada** detectados por perigeo bajo.
- **Anomalías / posible maniobra**, inferidas de la discrepancia entre fuentes.
- **Sobrevuelos de zonas de interés** (conflicto / interferencia GNSS) por satélites de observación/reconocimiento.
- **Severidades elevadas** para satélites recon, activos soberanos y reentradas con traza sobre territorio.
- Feed **filtrable** por tipo (colisión, sobrevuelo, reentrada, anomalía, zona de interés).

### Multi-fuente y validación cruzada
- Soporta una **fuente primaria** y una **fuente de cruce**.
- Empareja objetos por NORAD ID y calcula el **delta de posición entre fuentes** en kilómetros, base de la validación cruzada y de la detección de anomalías.

### Índice de confianza
- Puntaje **auditable (0–100)** por objeto, con bandas Alta / Media / Baja. Ver [metodología](#índice-de-confianza).

### Capa de zonas de interés
- Capa de **referencia editable** (zonas de conflicto e interferencia GNSS) que se cruza con el catálogo.

### Auto-actualización
- Refresco automático del catálogo **mientras la página está abierta** (intervalos de 1h / 3h / 6h) y botón de refresco manual.
- Indicador de antigüedad de los datos en la barra de estado.

### Drill-down de objeto
Al seleccionar un objeto (clic en el mapa, el registro, el horizonte o una alerta):
- Elementos orbitales (NORAD, clase, inclinación, período, perigeo/apogeo).
- Altitud y sub-punto actuales, ángulos desde la estación, **próximo pasaje**.
- **Antigüedad del TLE** (marca en rojo los > 14 días).
- **Desglose del índice de confianza** con barras por factor.
- Eventos asociados y procedencia de los datos.
- **Traza orbital predicha** y **huella de cobertura** dibujadas en el mapa.

### Otras herramientas
- **Búsqueda** de objeto por nombre.
- **Filtros de capa**: LEO / MEO / GEO / Debris / Solo registro / Zonas de interés.
- **Modo de color**: por estado o por nivel de confianza.
- **Barra de estado de sistema**: fuentes en línea, confianza media, antigüedad media de TLE, antigüedad de los datos, índice Kp de clima espacial, hora de última sincronización.
- **Reporte exportable (JSON)** con metadatos, parámetros, registro nacional, eventos, metodología y advertencias.

---

## Cómo funciona (técnico)

Toda la mecánica orbital corre en el cliente:

- **Propagación**: SGP4 vía [satellite.js](https://github.com/shashwatak/satellite-js) sobre elementos TLE.
- **Marcos de referencia**: conversión TEME → ECEF por rotación de GMST (simplificada, sin nutación fina), y topocéntrica a coordenadas ENU (azimut/elevación/rango).
- **Iluminación**: posición solar de baja precisión (almanaque) y prueba de sombra cilíndrica de la Tierra para determinar si un objeto está iluminado.
- **Observabilidad óptica**: un pasaje es observable cuando el cielo de la estación está oscuro (Sol bajo el umbral de crepúsculo) y el objeto está iluminado por el Sol.
- **Huella de cobertura**: círculo de horizonte calculado por ángulo central `acos(Re / (Re + h))`.
- **Terminador**: curva calculada desde el punto subsolar.
- **Territorio y zonas**: detección punto-en-polígono sobre el contorno de Uruguay y cajas de zonas de interés.

El análisis de ventana propaga todos los objetos una vez por paso temporal y reutiliza las posiciones para detectar conjunciones, sobrevuelos y zonas, ejecutándose en tramos (`setTimeout`) para mantener la interfaz fluida.

---

## Fuentes de datos

| Fuente | Uso | Acceso |
|--------|-----|--------|
| **CelesTrak** (grupos `visual`, `active`) | Catálogo de objetos (TLE) en vivo | Requiere que el navegador alcance `celestrak.org` |
| Catálogo de **muestra embebido** | Funcionamiento offline / demostración | Incluido en el archivo |
| Fuente **independiente (simulada)** | Validación cruzada en modo demo | Incluida; en producción se reemplaza por una fuente real independiente |
| **NOAA SWPC** | Índice Kp de clima espacial | Best-effort; muestra `n/d` si no hay acceso |
| **Natural Earth** | Costas y fronteras del mapa | Embebido (dominio público) |

El catálogo público de objetos en órbita proviene en última instancia del **catálogo de la US Space Force** (distribuido por Space-Track y reempaquetado por CelesTrak). Por eso CelesTrak y Space-Track **no son fuentes independientes entre sí**.

---

## Índice de confianza

El índice es una **heurística transparente y auditable**, no una covarianza formal. Se calcula por objeto como:

```
confianza (0–100) = 100 × ( 0.30 · frescura
                           + 0.32 · concordancia
                           + 0.20 · independencia
                           + 0.18 · estabilidad )
```

| Factor | Qué mide |
|--------|----------|
| **Frescura** | Antigüedad del TLE (decae hacia 0 a los 14 días). |
| **Concordancia** | Acuerdo de posición entre fuentes (delta en km, escalado por régimen orbital). |
| **Independencia** | Si existe una segunda fuente y si es genuinamente independiente del origen USSF. |
| **Estabilidad** | Estabilidad dinámica a partir del término de arrastre B\* y la excentricidad. |

Bandas: **Alta ≥ 70**, **Media ≥ 45**, **Baja < 45**.

Un objeto de **fuente única** no puede alcanzar banda Alta por diseño: sin validación independiente la confianza queda acotada, lo cual es el comportamiento epistémicamente correcto.

---

## Limitaciones

Documentadas deliberadamente, porque definen el alcance honesto del sistema:

- **Precisión de los TLE**: nivel de kilómetros en la época, degradándose en días. Los TLE **no traen covarianza**.
- **Conjunciones**: el sistema hace **triaje por distancia mínima**, no cálculo operacional de probabilidad de colisión (Pc). No reemplaza a un servicio de COLA.
- **Reentradas**: se marcan **candidatos por perigeo bajo**; SGP4 no permite afirmar lugar ni hora de reingreso.
- **Anomalía / maniobra**: se **infiere** de la discrepancia entre fuentes; requiere verificación humana.
- **Validación cruzada**: solo aporta confianza real si las fuentes son **independientes** del origen USSF.
- **Zonas de interés**: capa de **referencia curada y editable**, NO un feed en vivo. Verificar vigencia.
- **Auto-actualización**: ocurre **mientras la página está abierta**. Una actualización diaria programada (sin la página abierta) requiere un backend con tareas programadas.
- **Datos en vivo**: dependen de que el navegador pueda alcanzar CelesTrak (restricciones CORS según el entorno).
- **Marcos simplificados**: la conversión TEME→ECEF y el modelo de sombra son aptos para predicción de pasajes y planificación, no para determinación de órbita de precisión.

---

## Uso

No requiere build ni dependencias locales.

1. Abrir `ssa_visor_uruguay.html` en un navegador moderno, **o** servirlo con cualquier servidor estático:
   ```bash
   python -m http.server 8000
   # luego abrir http://localhost:8000/ssa_visor_uruguay.html
   ```
2. Arranca con el catálogo de muestra. Para datos reales, usar **"Datos en vivo"** o el selector de catálogo (requiere acceso a `celestrak.org`).
3. Seleccionar la estación, ajustar la ventana y el umbral de conjunción, y pulsar **"Analizar"**.
4. Hacer clic en cualquier objeto para el drill-down. Exportar la situación con **"Reporte"**.

---

## Configuración

Se edita directamente en el código del archivo HTML:

- **Registro nacional** (`REGISTRY`): activos por NORAD ID o nombre, y su tipo (`soberano`, `gnss`, `comms`, `eo`).
- **Zonas de interés** (`ZONES`): polígonos de zonas de conflicto e interferencia GNSS.
- **Estaciones** (`STATIONS`): coordenadas de las estaciones terrenas.
- **Parámetros de análisis**: umbral de conjunción (`conjKm`), umbral de reentrada (`decayKm`), ventana (`windowHrs`), máscara de elevación (`maskDeg`).

---

## Hoja de ruta

| Fase | Capacidad | Estado |
|------|-----------|--------|
| **0** | Software y catálogo: propagación, cribado, imagen situacional | ✅ Este visor |
| **1** | Primera estación óptica + determinación de órbita propia | Pendiente (acceso OALM) |
| **2** | Catálogo propio, red de estaciones, fusión de datos | Pendiente |
| **3** | Integración, intercambio con redes y nicho del hemisferio sur | Pendiente |

**Próximos pasos hacia un producto robusto:**
- Re-plataforma a backend: ingesta multi-fuente, persistencia histórica de elementos (habilita detección real de maniobra), API versionada y autenticada, propagación numérica de alta fidelidad (Orekit), actualización programada del lado servidor, y feeds externos (reentradas, clima espacial).
- SDLC seguro: modelado de amenazas (STRIDE), SAST/SCA/secrets y SBOM en CI, RBAC y registro de auditoría con cadena de custodia de alertas.
- **Independencia del dato**: estación óptica propia (astrometría + determinación de órbita) para validar contra un sensor del hemisferio sur, el verdadero diferencial de soberanía.

---

## Stack y atribuciones

- **HTML / CSS / JavaScript vanilla**, Canvas 2D, sin framework ni build.
- [satellite.js](https://github.com/shashwatak/satellite-js) — propagación SGP4 (vía cdnjs).
- [Natural Earth](https://www.naturalearthdata.com/) — datos de costas y fronteras (dominio público).
- Catálogo de objetos: US Space Force vía [CelesTrak](https://celestrak.org/) / [Space-Track](https://www.space-track.org/).
- Clima espacial: [NOAA SWPC](https://www.swpc.noaa.gov/).

---

## Aviso

Este software se provee como herramienta de **conciencia situacional y triaje**. No constituye un servicio operacional de evasión de colisiones ni una garantía de detección. Las decisiones críticas deben apoyarse en fuentes y servicios validados.

## Licencia

MIT
