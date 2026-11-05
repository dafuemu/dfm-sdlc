# Decisiones aplazadas

## Medición de tiempos y almacenamiento del progreso

**Estado:** aplazada, sin decidir (noviembre de 2026).

**Motivo:** es pronto para añadir complejidad. Primero hay que probar y madurar el flujo.

**Cuándo retomarla:** cuando el wf esté probado y suficientemente maduro.

**Hasta entonces:** no se añade nada de esto al sistema ni al grafo. Mientras tanto no se acumulan datos históricos para estimar.

### Objetivo

Medir cuánto tarda una tarea de cualquier tipo desde que empieza hasta que queda desplegada y lista para probar, y usar ese histórico para hacer estimaciones más precisas. El progreso de cada tarea debería poder compartirse y versionarse.

### Ideas sobre la medición

- Registrar un evento por cada transición del grafo, escrito por el orquestador y no por los agentes.
- Separar tres tiempos: agentes, personas y total.
- Arrancar el reloj cuando la tarea entra en el sistema, y guardar aparte la fecha de creación del ticket.
- Parar el reloj cuando el cambio está desplegado y comprobado en un entorno de pruebas. Esto requiere un nodo Desplegador, porque el flujo actual acaba en el PR draft. Las tareas sin despliegue, como una investigación, se paran al entregar el informe y se miden aparte.
- Estimar con rangos a partir de tareas parecidas, y guardar cada predicción para compararla con el resultado real.
- Conservar la clasificación inicial y la final de cada tarea.
- Acompañar el tiempo con métricas de calidad: retrabajo tras la revisión, PRs rechazados y fallos en el entorno de pruebas.
- Guardar la versión del sistema en cada evento, porque los cambios de modelo, prompts o grafo dejan los tiempos antiguos sin ser comparables.

### Opciones de almacenamiento

| Opción | Versionar y compartir | Consultar tiempos | Encaje |
|---|---|---|---|
| JSONL en git + SQLite regenerable | Muy bueno: texto, diffs y merges | Muy bueno | Candidata preferida por ahora |
| Dolt | Muy bueno: ramas y merges de datos | Bueno, con SQL | Alternativa si se necesita todo en SQL |
| Beads | Bueno, con Dolt y `push`/`pull` | No documenta marcas de tiempo por cambio | Se solapa con el grafo |
| Postgres | Requiere exportar para versionar | Muy bueno | Si hay muchos workers o un panel en tiempo real |
| SQLite solo | Malo: git no puede fusionar un binario | Muy bueno | Solo como índice |
| Engram (open source) | Git, con fragmentos comprimidos | Sin métricas de tiempo | Complemento para la memoria de los agentes |
| LangGraph (checkpointers) | No es su objetivo | No es su objetivo | Solo para reanudar la ejecución |
| engram.com | No documentado | No documentado | Descartada: resuelve otro problema |

Ninguna se ha probado: la comparación se basa en su documentación. Temporal queda sin evaluar.

### Si se elige JSONL + SQLite

```text
sdlc-runs/
  tasks/JIRA-482/
    events.jsonl     # fuente de verdad, solo añadir
    intent.md
    plan.md
  metrics.db         # en .gitignore, se regenera
```

- Usar un repo separado del código, para no añadir ruido a los PRs.
- Hacer commit al terminar cada nodo o al pausar la tarea, no por cada evento.
- Guardar la marca de tiempo dentro del evento, no depender de la del commit.
- Guardar referencias y metadatos, no el contenido de tickets ni de logs.

## Posibles mejoras

### Modo profundo del Aclarador con `grill-with-docs`

**Estado:** posibilidad de mejora, sin adoptar (noviembre de 2026).

**Idea:** usar la skill pública [`grill-with-docs`](https://github.com/mattpocock/skills/blob/main/docs/engineering/grill-with-docs.md) como modo profundo del Aclarador en tareas amplias, de riesgo alto o con un plan difuso. No sustituye al modo rápido y asíncrono para dudas bloqueantes en tickets.

| Situación | Modo del Aclarador | Cómo funciona |
|---|---|---|
| Sin dudas bloqueantes | No interviene | Los supuestos van al `intent.md` |
| Dudas bloqueantes en un ticket | Rápido y asíncrono | Agrupa pocas preguntas en un comentario, cada una con recomendación, y pausa la tarea |
| Tarea amplia, de riesgo alto o plan difuso | Profundo, con la skill | Sesión interactiva con una persona, con un límite de preguntas |

**Qué aporta:**

- Pregunta de una en una y da su respuesta recomendada.
- Explora el código antes de preguntar y contrasta lo que dice la persona con lo que hace el código.
- Detecta términos ambiguos o que chocan con el glosario.
- Crea ADR solo si la decisión es difícil de revertir, sorprendente sin contexto y fruto de un compromiso real.

**Qué habría que resolver:**

- Asume una conversación en directo, así que una persona debe aceptar la sesión.
- Recorre todas las ramas del diseño y no distingue dudas bloqueantes, por lo que necesita un límite de preguntas.
- Su documentación avisa de que puede no generar ficheros dentro de otra capa de orquestación, y lleva `disable-model-invocation: true`.
- No produce un `intent.md`: las decisiones de la sesión tendrían que pasar a ese fichero.
- Escribe en el repo durante la entrevista: el glosario y los ADR deberían ir en el PR o en los artefactos de la tarea.
- Depende de las skills `grilling` y `domain-modeling`, y conviene fijar una versión. La copia local es más antigua y usa `CONTEXT.md` en lugar de `GLOSSARY.md`.

**Primera prueba si se retoma:** invocarla desde el orquestador con un caso real. Antes hay que revisar las reglas de la versión actual de `grilling`, que no se han podido leer.

**Reutilizable sin adoptar la skill:** dar una recomendación con cada pregunta, explorar el código primero y el criterio para crear ADR.

### Disparadores de Entrada: scheduler y manual

**Estado:** opción, sin adoptar (noviembre de 2026).

**Idea:** dentro de Entrada, separar el origen de los datos (`source`) de quién arranca la tarea (`trigger`). Un ticket de Jira puede entrar por dos disparadores, y los dos usan el mismo traductor de Jira en el Adaptador.

```text
Entrada
 ├─ Scheduler: consulta Jira cada X minutos
 ├─ Manual: le das el ticket desde una sesión
 └─ Prompt manual
        ↓
    Adaptador
```

| | Scheduler | Manual desde una sesión |
|---|---|---|
| Necesita la etiqueta `agente` | Sí, es la forma de dar permiso | No: pedirlo ya es dar permiso |
| Instrucciones añadidas | Ninguna | Las que escribas junto al ticket |
| Límite por ejecución | Sí | No hace falta |

**Scheduler:**

- Lanza una JQL fija con un límite de resultados, por ejemplo:

  ```text
  project = FINPFM AND assignee = currentUser()
    AND status = "To Do" AND labels = agente
  ```

- Descarta tickets ya en curso o sin cambios, reserva cada ticket para que no se procese dos veces y lo entrega al Adaptador.
- Procesa pocos tickets por ejecución, para no generar más PRs de los que se pueden revisar.
- Empezar con polling en lugar de webhook: es más simple y la siguiente ejecución recoge lo pendiente si algo falla.
- La misma ejecución puede comprobar si hay respuestas nuevas a las preguntas del Aclarador y reanudar esas tareas.

**Disparador manual con instrucciones:** si se escribe "haz JIRA-482, pero solo en el backend", el ticket sigue siendo no confiable y tu frase son instrucciones válidas. Van en campos separados:

```yaml
source: jira
trigger: manual
ref: JIRA-482
body: "..."                               # del ticket, untrusted
instructions: "Solo en el backend"        # tuyas, de confianza
```

**Imprescindible si se adopta:** los dos disparadores comprueban y reservan el ticket en el mismo sitio, para no lanzar dos veces la misma tarea. Esto requiere guardar al menos el ID de cada ticket procesado y su última actualización, aunque la medición siga aplazada.

## Pendiente para una software factory completa

**Estado:** pendiente, a tener en cuenta (noviembre de 2026).

El sistema ya estandariza el proceso, define entradas y salidas, integra el control de calidad y produce de forma repetible. Para funcionar como una software factory completa le faltan tres piezas:

| Pieza | Situación actual | Dónde está anotada |
|---|---|---|
| Llegar hasta el final | Acaba en un PR draft, no en algo desplegado | Nodo Desplegador, en "Medición de tiempos y almacenamiento del progreso" |
| Medir su producción | No hay métricas de ritmo, calidad ni tiempos | "Medición de tiempos y almacenamiento del progreso" |
| Recibir trabajo sola | Toda tarea entra a mano | "Disparadores de Entrada: scheduler y manual" |

**Lo que no se busca:** una fábrica que construya y despliegue sin revisión humana. Las personas siguen aprobando la spec, revisando el PR y decidiendo si se despliega.
