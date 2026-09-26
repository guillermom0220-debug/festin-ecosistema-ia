# Ecosistema de Automatización IA — Festín Café Bistró

Pipeline autónomo de generación y distribución de contenido para redes sociales, con un punto de validación humana antes de publicar.

**Entrega Final** · Diseño de Agentes y Automatización · Guillermo Manosalva · Septiembre 2026

---

## El problema

Festín es un café bistró de La Molina que publica en Instagram y Facebook. El cuello de botella no son las ideas: es el trecho entre tener una idea y publicarla. Alguien tiene que redactar, revisar y subir cada pieza, y eso se posterga.

Este sistema automatiza la redacción y la distribución, pero deja intacta la decisión de publicar.

## Stack

| Capa | Herramienta |
|---|---|
| Orquestación | Make |
| Base de datos | Airtable (2 bases, 3 tablas) |
| Procesamiento IA | Google Gemini 3.5 Flash |
| Canal de salida | Gmail |

## Cómo funciona

```
Cada 15 min
   │
   ├─ ESCENARIO 1 ──────────────────────────────────────────────┐
   │  Busca piezas en «Generando» con idea semilla escrita      │
   │  Consulta la base de conocimiento (registros activos)      │
   │  Agrega el contexto en un solo bloque                      │
   │  Gemini redacta el post                                    │
   │  Filtro: ¿hay contenido real?  ──── no ──▶ Log Error       │
   │  Estado → En revisión                                      │
   │  Log OK                                                     │
   └────────────────────────────────────────────────────────────┘
                              │
                    ⏸  EL SISTEMA SE DETIENE
                    Un editor marca ☑ Aprobado
                              │
   ┌─ ESCENARIO 2 ────────────────────────────────────────────── ┐
   │  Router                                                     │
   │   ├─ Aprobado existe  → Gmail → Publicado → Log OK          │
   │   └─ Estado Rechazado → vuelve a Generando → Log OK         │
   └─────────────────────────────────────────────────────────────┘
```

**Por qué dos escenarios y no uno.** Un flujo continuo no puede esperar a una persona sin quedarse colgado consumiendo operaciones. Al separarlos, el primero termina su trabajo y se apaga; el segundo corre en paralelo y solo encuentra trabajo cuando alguien ya aprobó algo. El estado en Airtable es lo que comunica a ambos.

## Enlaces

| Recurso | Enlace |
|---|---|
| Escenario 1 — Generación | https://us2.make.com/public/shared-scenario/pTeeVWUStNB/festin-1-generacion-de-contenido |
| Escenario 2 — Distribución con HITL | https://us2.make.com/public/shared-scenario/4jeHK0k5v8s/festin-2-distribucion-con-hitl |
| Dashboard de Control | https://airtable.com/app1KnLvAzHHtM5fP/pag6bIqvunOYUbRP6 |
| Vista pública — Logs | https://airtable.com/app1KnLvAzHHtM5fP/shrpOjQjwl5PjRlYN/tblEs5H4KXetAG5Jv/viwC6WdmO4mdYeIUV |
| Vista pública — Piezas | https://airtable.com/app1KnLvAzHHtM5fP/shrpOjQjwl5PjRlYN/tbl1aODnCVlOpLq0v/viwsKPOypw3AL44nF |

## Estructura del repositorio

```
├── docs/
│   └── Festin_EntregaFinal_Ecosistema_IA.pdf    Documentación completa (19 pág.)
├── blueprints/
│   ├── escenario-1-generacion.blueprint.json
│   └── escenario-2-distribucion-hitl.blueprint.json
└── evidencias/
    └── 14 capturas de ejecuciones reales
```

## Modelo de datos

**Base A — Conocimiento Festín** (memoria del negocio)

| Campo | Tipo |
|---|---|
| Tema | Texto |
| Categoría | Selección única |
| Contenido | Texto largo |
| Activo | Casilla |

**Base B — Centro de Comando**

`Piezas` guarda el estado de cada contenido; `Logs` registra cada ejecución. Están **vinculadas**: el campo `Pieza` en Logs es un link real a la tabla Piezas, y genera el campo inverso `Logs` en Piezas. Desde cualquier error se llega al contenido que lo provocó.

## Resiliencia

Los dos puntos frágiles son llamadas a terceros. Ambos llevan error handler con directiva **Resume**:

| Punto de fallo | Qué hace el sistema |
|---|---|
| Gemini no responde o devuelve error | Registra el mensaje literal en Logs y sigue con la siguiente pieza |
| Contenido vacío tras un fallo | Un filtro bloquea la escritura: la pieza no avanza contaminada |
| Gmail rechaza el envío | Registra el fallo sin marcar la pieza como publicada |
| Fila sin idea semilla | El trigger la descarta antes de llamar al modelo |

**Anti-bucle:** la condición del Router excluye `Estado ≠ Publicado`, el trigger del Escenario 2 solo busca estados no terminales, y ambos triggers tienen techo de 10 registros por ejecución.

**Anti-alucinación:** la instrucción de sistema obliga al modelo a usar solo el contexto recibido. El punto de validación humana funciona como segunda barrera.

## Test de estrés

Seis pruebas ejecutadas el 26 de septiembre de 2026, tres de ellas diseñadas para romper el sistema.

| # | Prueba | Resultado |
|---|---|---|
| 1 | Generación normal con tres piezas | 3 piezas generadas, 3 logs OK |
| 2 | Pieza con idea semilla vacía | Descartada por el trigger, sin logs |
| 3 | Modelo de IA inexistente | Log Error con mensaje 404, flujo no se detuvo |
| 4 | Escenario 2 sin ninguna aprobación | 0 bundles en ambas rutas, nada se publicó |
| 5 | Pieza aprobada | 1 bundle por la ruta HITL, correo enviado |
| 6 | Pieza rechazada con comentario | Devuelta a Generando con el feedback |

**Las pruebas encontraron dos defectos reales**, ambos corregidos y documentados en el PDF:

1. La directiva Resume marcaba el módulo como exitoso, así que una pieza sin contenido avanzaba igual y registraba un OK falso. Se resolvió con un filtro sobre el resultado del modelo.
2. La condición del Router comparaba un booleano contra la cadena `"Yes"`. Se resolvió pasando a una comprobación de existencia sobre el campo.

## Métricas medidas

| Indicador | Valor |
|---|---|
| Ejecuciones registradas | 11 |
| Tasa de éxito | 72,7% |
| Tasa de error | 27,3% |
| Errores por módulo | 3 en IA – Redacción Gemini |

La tasa de error refleja una sesión de pruebas donde el fallo se provocó tres veces a propósito. Lo relevante no es la magnitud sino que el sistema la haya medido solo.

## Replicar

1. Crear las dos bases de Airtable con el esquema descrito arriba.
2. Importar los blueprints en Make (*Create a new scenario* → menú ⋯ → *Import Blueprint*).
3. Reconectar las tres conexiones: Airtable, Gemini y Gmail. Los blueprints no incluyen credenciales, solo referencias numéricas.
4. Ajustar los IDs de base y tabla a los propios.
5. Activar ambos escenarios.

## Nota sobre credenciales

Los blueprints exportados **no contienen API keys**. Las conexiones aparecen como identificadores numéricos (`__IMTCONN__`) que apuntan al almacén de credenciales de Make y no son utilizables fuera de esa cuenta.
