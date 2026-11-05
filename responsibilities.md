# Responsabilidades

Qué hace cada componente del sistema, qué no hace y un ejemplo. Se irá ampliando poco a poco.

Los ejemplos usan tres tareas inventadas:

- **JIRA-482:** en el resumen mensual, el total sale con cuatro decimales. Debe mostrar dos.
- **JIRA-497:** añadir un filtro por fecha al listado de movimientos.
- **JIRA-510:** avisar a los usuarios cuando se acerquen a su presupuesto mensual.

## Entrada

**Tipo:** disparador.

**Responsabilidad:** arrancar una tarea y entregar al Adaptador la petición en bruto, sin modificarla.

**Recibe:** un prompt escrito en una sesión, o la clave de un ticket de Jira con instrucciones opcionales.

**Produce:** la petición en bruto, con su origen (`source`) y quién la arrancó (`trigger`).

**Limitaciones:**

- No transforma ni interpreta la petición.
- No decide si la tarea es válida ni cómo se resuelve.
- Antes de entregar un ticket, comprueba que no está ya en curso, para no lanzar dos veces la misma tarea.

**Ejemplo:** escribes "haz JIRA-482" en una sesión. La Entrada lee el ticket de Jira y lo entrega al Adaptador con `source: jira` y `trigger: manual`.

**Pendiente:** el scheduler que consulta Jira está anotado como opción en [decisiones-aplazadas.md](decisiones-aplazadas.md), sin adoptar.

## Adaptador

**Tipo:** código determinista.

**Responsabilidad:** convertir cualquier petición al mismo formato, la solicitud normalizada. Hay un traductor por origen: uno para Jira y otro para el prompt.

**Recibe:** la petición en bruto de la Entrada.

**Produce:** la solicitud normalizada.

```yaml
schema: request/v1
source: jira
trigger: manual
ref: JIRA-482
author: ...
received_at: 2026-11-03T09:12:00Z
title: "Total del resumen mensual con cuatro decimales"
body: "En el resumen mensual, el total sale con cuatro decimales. Debe mostrar dos."
instructions: null
comments: []
hints: { type: Bug, priority: Medium, labels: [], components: [resumen] }
links: [...]
attachments: [...]
untrusted: true
```

**Limitaciones:**

- No resume, no reescribe y no descarta información: el texto llega tal cual.
- No clasifica la tarea. Los campos de `hints` son pistas copiadas de Jira, no decisiones.
- No descarga el contenido de los adjuntos, solo sus referencias.
- Separa el texto del ticket, que no es confiable, de las instrucciones de quien arranca la tarea.

**Ejemplo:** escribes "haz JIRA-482, pero solo en el backend". El traductor de Jira copia la descripción del ticket en `body` y tu frase en `instructions`. El Analista sabe así qué texto es una instrucción válida y cuál no.

**Patrón:** Normalizer, de Enterprise Integration Patterns.

## Analista

**Tipo:** agente.

**Responsabilidad:** entender la petición, explorar el código lo necesario para clasificarla y escribir el `intent.md`.

**Recibe:** la solicitud normalizada y, si hubo preguntas, las respuestas del Aclarador.

**Produce:** el `intent.md`, clasificado en cuatro ejes.

| Eje | Valores | Qué decide |
|---|---|---|
| Entrega | PR, investigación, propuesta, POC | El estado final |
| Claridad | Suficiente, con supuestos, bloqueada | Si hay que preguntar |
| Alcance | Trivial, acotado, amplio | Cuántas fases recorre |
| Riesgo | Bajo, medio, alto | Cuántas aprobaciones humanas exige |

**Limitaciones:**

- Solo lectura: no modifica código ni documentación.
- Explora el código antes de preguntar, para no hacer preguntas que puede responder solo.
- Solo pasa al Aclarador las dudas que cambian lo que se construye. Las demás quedan como supuestos en el `intent.md`.
- No decide la ruta. Eso le toca al Router.

**Ejemplo:** con JIRA-482, el Analista encuentra el formateo del total en el código y no ve dudas. Lo clasifica como PR, claridad suficiente, alcance trivial y riesgo bajo. Con JIRA-510, detecta dos dudas que cambian el resultado, el canal y el umbral, y las pasa al Aclarador.

## Aclarador

**Tipo:** agente, con pausa a la espera de una persona.

**Responsabilidad:** resolver las dudas bloqueantes preguntando a quien pidió la tarea, sin ningún agente esperando en directo.

**Recibe:** las dudas bloqueantes que le pasa el Analista.

**Produce:** las respuestas, que devuelve al Analista.

**Cómo funciona:**

1. Agrupa como mucho unas tres preguntas en un solo mensaje, en el canal de origen: un comentario en el ticket o una respuesta en el chat.
2. Cada pregunta incluye el contexto, la respuesta recomendada y qué cambia según la respuesta.
3. Guarda el estado y pausa la tarea. Nada se ejecuta mientras espera.
4. Cuando llega la respuesta, la tarea se retoma y el Analista la interpreta.

**Limitaciones:**

- Como máximo dos rondas de preguntas. Si hacen falta más, la tarea necesita una sesión en directo.
- Solo acepta respuestas del autor o de alguien autorizado.
- Si no hay respuesta, envía un recordatorio y, pasado un plazo, cierra la tarea como bloqueada. El silencio no cuenta como aprobación.

**Ejemplo:** para JIRA-510 publica en el ticket dos preguntas. Recomienda enviar el aviso por push, porque la app ya lo usa para otras alertas, y avisar al 80 % del presupuesto. La persona responde "ok" y la tarea sigue.

**Pendiente:** el modo profundo con la skill `grill-with-docs` está anotado como posible mejora en [decisiones-aplazadas.md](decisiones-aplazadas.md).

## Router

**Tipo:** código determinista.

**Responsabilidad:** leer los campos del `intent.md` y elegir la ruta según una tabla de reglas fija.

**Recibe:** el `intent.md`.

**Produce:** la ruta de la tarea.

| Campos del intent | Recorrido | Termina en |
|---|---|---|
| PR · trivial | Implementar → verificar → publicar | PR draft |
| PR · acotado | Planificar → implementar → verificar → revisar → publicar | PR draft |
| PR · amplio o riesgo alto | Spec → aprobación → planificar → implementar → verificar → revisar → publicar | PR draft |
| Investigación o propuesta | Investigar → redactar | Informe o propuesta |
| POC | Implementar en rama desechable → redactar | Rama e informe de conclusiones |

**Limitaciones:**

- No usa el modelo: las mismas entradas dan siempre la misma ruta.
- No interpreta la petición. Si la ruta es incorrecta, el fallo está en la clasificación del Analista o en la tabla de reglas.

**Ejemplo:** JIRA-497 llega como PR y acotado, así que el Router lo manda al Planificador sin pasar por la spec.

## Especificador

**Tipo:** agente. Solo en la ruta "amplio" o con riesgo alto.

**Responsabilidad:** escribir el `spec.md`, que cierra todas las decisiones de comportamiento antes de construir.

**Recibe:** el `intent.md`.

**Produce:** el `spec.md`, con:

- El comportamiento: el flujo normal y los casos límite.
- Los criterios de aceptación que usará después el Verificador.
- Lo que queda fuera de alcance.
- Las decisiones con alternativas y por qué se tomaron.
- Las políticas aplicables: seguridad, cumplimiento y UX.

**Limitaciones:**

- No toca código.
- No repite el `intent.md`: lo enlaza y añade solo lo que decide.
- Describe qué verá el usuario, no cómo se construye. Dos planes distintos deberían poder cumplir la misma spec.
- Una persona debe aprobar el `spec.md` antes de que la tarea avance.

**Ejemplo:** el intent de JIRA-510 dice "aviso push al 80 %". La spec decide lo que nadie había dicho: se avisa una vez al mes por categoría, se recalcula si el presupuesto cambia a mitad de mes, no se avisa si el usuario desactivó las notificaciones, y el aviso por email queda fuera de esta versión.

## Planificador

**Tipo:** agente.

**Responsabilidad:** escribir el `plan.md`, que fija qué hay que cambiar en el sistema y dónde.

**Recibe:** el `intent.md` y, en la ruta "amplio", el `spec.md` aprobado.

**Produce:** el `plan.md`, con los componentes que cambian, el orden de los pasos, los criterios de hecho de cada paso y qué criterios de aceptación cubre cada uno.

**Limitaciones:**

- Solo lectura.
- No incluye código: si lo hiciera, el Implementador se limitaría a copiar.
- Tampoco puede ser vago: si no dice qué componentes cambian, el Implementador acabaría diseñando la solución.
- No repite la spec: relaciona cada paso con los criterios que cubre.

**Ejemplo:** para JIRA-510, el plan dice que el servicio de presupuestos calcula el porcentaje y publica un evento, y que el servicio de notificaciones lo envía. No dice qué clases ni qué consultas se escriben.

**Pendiente:** el grafo no tiene un paso propio de investigación antes del plan. Hay que decidir si se reutiliza el Investigador o si este tramo se apoya en el workflow RPI ya existente.

## Implementador

**Tipo:** agente que reparte en workers lo que sea independiente.

**Responsabilidad:** escribir el código y sus tests cumpliendo el plan. Decide cómo se escribe cada cambio.

**Recibe:** el `plan.md`, o el `intent.md` en la ruta "trivial", que no tiene plan.

**Produce:** el cambio en una rama, con sus tests.

**Limitaciones:**

- Trabaja en un worktree propio y no puede hacer push a ramas protegidas.
- No se sale del plan. Si necesita tocar un componente que el plan no incluye, vuelve al Planificador.
- No decide si el cambio es correcto: eso le toca al Verificador.
- Si el Verificador o el Revisor le devuelven el cambio, lo corrige dentro de un límite de iteraciones.

**Ejemplo:** para JIRA-497, el plan dice que el endpoint del listado acepta un rango de fechas y que la pantalla muestra el filtro. El Implementador decide cómo validar las fechas y qué tests escribir.

## Pendientes de documentar

Aparecen en el grafo, pero aún no se han trabajado en detalle: Aprobación, Verificador, Revisor, Publicador, Investigador y Redactor.
