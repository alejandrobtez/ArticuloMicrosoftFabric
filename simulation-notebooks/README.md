# Cómo funciona el Notebook del simulador
 
El Notebook es el punto de entrada del proyecto. Vive dentro de Microsoft Fabric y se ejecuta sobre **Apache Spark**, aunque el código es Python puro.
 
---
 
## Estructura en 4 celdas
 
### Celda 0 — Instalación
```python
%pip install azure-eventhub
```
Instala la librería necesaria para conectarse al Event Hub. Solo hace falta ejecutarla una vez por sesión.
 
---
 
### Celda 1 — Configuración
```python
CONNECTION_STRING = "tu-connection-string"
EVENT_HUB_NAME    = "es_liga-futbol-stream"
JORNADA           = 1
DELAY_SEGUNDOS    = 0.5
```
Aquí se definen los parámetros principales. **Para simular una nueva jornada solo hay que cambiar `JORNADA` y re-ejecutar la Celda 4**, sin tocar nada más.
 
---
 
### Celda 2 — Datos sintéticos
Define los 20 equipos, los nombres y apellidos de los jugadores, y la función `generar_jugadores()` que crea 300 jugadores distribuidos por posición. También define el `dataclass EventoPartido`, que es la estructura de cada mensaje JSON que se enviará al Event Hub.
 
```python
random.seed(42)  # Garantiza que los jugadores son siempre los mismos
```
 
> **Por qué `random.seed(42)`:** Si el cluster de Spark se reinicia y volvemos a generar los jugadores, la semilla fija garantiza que los nombres saldrán exactamente igual. Así los datos históricos del Eventhouse siguen siendo consistentes.
 
---
 
### Celda 3 — Clase SimuladorLiga
Contiene toda la lógica de simulación:
 
- **`simular_partido(local, visitante)`** — simula un partido completo minuto a minuto. Por cada minuto sortea si ocurre un gol, una tarjeta o nada. Si hay gol, decide qué equipo marca según su **fuerza** (valor entre 0 y 1 configurable por equipo).
- **`simular_jornada()`** — genera los 10 partidos de la jornada y los simula en secuencia.
- **`_enviar(evento)`** — serializa el evento a JSON y lo envía al Event Hub usando el SDK `azure-eventhub`.
Cada evento enviado tiene este aspecto:
```json
{
  "tipo_evento":    "gol",
  "jornada":        1,
  "minuto":         67,
  "jugador_nombre": "Carlos García",
  "equipo_jugador": "Atlético Madrid",
  "goles_local":    2,
  "goles_visitante":0,
  "es_penalti":     false
}
```
 
---
 
### Celda 4 — Ejecutar jornada
```python
random.seed(42)
jugadores  = generar_jugadores()
simulador  = SimuladorLiga(JORNADA, jugadores, DELAY_SEGUNDOS)
resultados = simulador.simular_jornada()
```
Esta es la única celda que hay que re-ejecutar para cada jornada. El resto permanece en memoria mientras la sesión Spark esté activa.
 
---
 
## Flujo de un evento
 
```
Celda 4 llama a simular_jornada()
        ↓
Por cada partido, simula minuto a minuto
        ↓
Si ocurre un gol → construye el JSON del evento
        ↓
Lo envía al Event Hub (Connection String)
        ↓
Eventstream lo recoge y lo manda al Eventhouse
        ↓
KQL Database lo almacena en EventosPartido
        ↓
Las vistas materializadas se actualizan automáticamente
        ↓
El dashboard lo muestra en el siguiente refresco (30s)
```
 
---
 
## Parámetros configurables
 
| Parámetro | Descripción | Valor recomendado |
|---|---|---|
| `JORNADA` | Número de jornada a simular | 1, 2, 3... |
| `DELAY_SEGUNDOS` | Pausa entre eventos | `0.5` normal · `0.2` rápido |
| `FUERZA_EQUIPOS` | Probabilidad de marcar por equipo | Entre 0.25 y 0.90 |
