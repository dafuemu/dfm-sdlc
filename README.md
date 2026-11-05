# dfm-sdlc

Diseño de un flujo de agentes que lleva una tarea desde el ticket (o un prompt) hasta un PR draft. Cada tarea sigue el camino que necesita según su tamaño y alcance: un fix pequeño va directo al cambio, y una feature nueva pasa antes por especificación y aprobación.

Las personas siguen aprobando la spec, revisando el PR y decidiendo si se despliega. No se busca una fábrica que construya y despliegue sin revisión humana.

## Contenido

| Fichero | Qué es |
|---|---|
| [`grafo-agentes.html`](grafo-agentes.html) | Mapa visual del grafo, con los recorridos de cada tipo de tarea (fix pequeño, feature pequeña, feature amplia, investigación) y cómo elige ruta el Router. Ábrelo en el navegador. |
| [`responsibilities.md`](responsibilities.md) | Qué hace cada componente, qué no hace y un ejemplo con tareas inventadas. |
| [`decisiones-aplazadas.md`](decisiones-aplazadas.md) | Decisiones que se han dejado para más adelante, posibles mejoras y lo que falta para una software factory completa. |

## Componentes

| Componente | Tipo | Responsabilidad |
|---|---|---|
| Entrada | Disparador | Arranca la tarea y entrega la petición en bruto. |
| Adaptador | Código determinista | Normaliza cualquier petición al mismo formato. |
| Analista | Agente | Clasifica la petición y escribe el `intent.md`. |
| Aclarador | Agente con pausa humana | Resuelve las dudas bloqueantes con quien pidió la tarea. |
| Router | Código determinista | Elige la ruta según los campos del `intent.md`. |
| Especificador | Agente | Escribe el `spec.md` (solo en la ruta "amplio" o con riesgo alto). |
| Planificador | Agente | Escribe el `plan.md`: qué cambiar y dónde. |
| Implementador | Agente con workers | Escribe el código y sus tests según el plan. |

El detalle de cada uno está en [`responsibilities.md`](responsibilities.md). Aprobación, Verificador, Revisor, Publicador, Investigador y Redactor aparecen en el grafo, pero aún están sin documentar.

## Estado

En diseño. El flujo acaba en un PR draft. Faltan el despliegue, la medición de tiempos y la entrada automática de trabajo, anotados en [`decisiones-aplazadas.md`](decisiones-aplazadas.md).
