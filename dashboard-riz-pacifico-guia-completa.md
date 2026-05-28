# 📊 DASHBOARD RAÍZ PACÍFICO — GUÍA COMPLETA PARA METABASE
## Sistema de Encuestas Dinámicas · MongoDB + Metabase
### Proyecto SENA — Pacífico Colombiano

**Generado:** 24 de mayo de 2026
**Base de datos:** `pacifico_db_2026-05-16`

---

## 📑 ÍNDICE

1. [Configuración de conexión en Metabase](#configuración-de-conexión-en-metabase)
2. [Índices obligatorios — ejecutar ANTES](#índices-obligatorios--ejecutar-antes)
3. [Inventario de colecciones](#inventario-de-colecciones)
4. [Mapa de relaciones (JOINs)](#mapa-de-relaciones-joins)
5. [KPIs Ejecutivos — Cards Numéricas](#kpis-ejecutivos--cards-numéricas)
6. [Consultas de Avance General](#consultas-de-avance-general)
7. [Consultas Geográficas Comparativas](#consultas-geográficas-comparativas)
8. [Consultas de Productividad](#consultas-de-productividad)
9. [Consultas de Calidad del Dato](#consultas-de-calidad-del-dato)
10. [Consultas de Análisis Temporal](#consultas-de-análisis-temporal)
11. [Consultas de Análisis Cualitativo](#consultas-de-análisis-cualitativo)
12. [Consultas de Soporte — Catálogos](#consultas-de-soporte--catálogos)
13. [Arquitectura analítica recomendada (Fase 2)](#arquitectura-analítica-recomendada-fase-2)
14. [Anexo: Visualizaciones Metabase disponibles](#anexo-visualizaciones-metabase-disponibles)

---

## Configuración de conexión en Metabase

| Parámetro | Valor |
|---|---|
| **Host** | `mongito` (o `localhost` si accedes fuera del Docker) |
| **Puerto** | `27018` |
| **Base de datos** | `pacifico_db_2026-05-16` |
| **Usuario** | `redash` |
| **Contraseña** | `redash` |
| **Auth DB** | `admin` |

> ⚠️ En Metabase, las consultas nativas MongoDB se escriben como **array JSON** (pipeline de agregación). La colección se selecciona en el campo "Collection".

---

## Índices obligatorios — Ejecutar ANTES

Sin estos índices, **todas las consultas harán COLLSCAN** (escaneo completo) sobre 112,431 documentos. El sistema se volverá lento con más de 2-3 usuarios concurrentes.

```javascript
// ============================================
// PRIORIDAD ALTA — Ejecutar inmediatamente
// ============================================
db.respuestas.createIndex({ formulario_id: 1, user_id: 1 })
db.respuestas.createIndex({ vivienda_id: 1 })
db.respuestas.createIndex({ comenzado_en: 1 })
db.respuestas.createIndex({ id_respuesta: 1 })
db.detalles_respuestas.createIndex({ respuesta_id: 1 })
db.detalles_respuestas.createIndex({ pregunta_id: 1, opcion_respuesta_id: 1 })

// ============================================
// PRIORIDAD MEDIA — Para dashboards geográficos
// ============================================
db.viviendas.createIndex({ zona_id: 1 })
db.viviendas.createIndex({ demografia_id: 1 })
db.viviendas.createIndex({ id_vivienda: 1 })
db.zonas.createIndex({ id_zona: 1 })
db.zonas.createIndex({ centro_poblado_id: 1 })
db.centros_poblados.createIndex({ id_centro_poblado: 1 })
db.centros_poblados.createIndex({ id_municipio: 1 })
db.municipios.createIndex({ id_municipio: 1 })
db.municipios.createIndex({ id_departamento: 1 })
db.departamentos.createIndex({ id_departamento: 1 })
db.preguntas.createIndex({ id_pregunta: 1 })
db.preguntas.createIndex({ tipo_pregunta_id: 1 })
db.opciones_respuestas.createIndex({ id_opcion_respuesta: 1 })
db.opciones_respuestas.createIndex({ pregunta_id: 1 })
db.formularios_preguntas.createIndex({ formulario_id: 1, pregunta_id: 1 })
db.formularios.createIndex({ modulo_id: 1, tipo_formulario_id: 1 })
```

---

## Inventario de colecciones

| Colección | Documentos | Rol en el sistema |
|---|---|---|
| `detalles_respuestas` | **112,431** | Núcleo analítico — cada respuesta individual |
| `respuestas` | **5,052** | Sesiones de encuesta completadas |
| `preguntas` | **989** | Catálogo de preguntas |
| `opciones_respuestas` | **3,343** | Opciones disponibles |
| `formularios_preguntas` | **928** | Relación formulario ↔ pregunta |
| `viviendas` | **969** | Unidades de observación |
| `centros_poblados` | **9,212** | Geografía nivel 3 |
| `municipios` | **1,122** | Geografía nivel 2 |
| `departamentos` | **33** | Geografía nivel 1 |
| `zonas` | **20** | Unidad operativa de campo |
| `formularios` | **36** | Instrumentos distribuidos en 8 módulos |
| `modulos` | **8** | Agrupadores temáticos SENA |
| `demografias` | **8** | Tipos de comunidad |
| `users` | **50** | Encuestadores y administradores |

### Módulos temáticos

| ID | Módulo |
|---|---|
| 1 | Social/Comunitario/Económico |
| 2 | Comunicaciones |
| 3 | Arquitectura |
| 4 | Energético |
| 5 | Ambiental |
| 6 | Agroproductivo |
| 7 | Hidrosanitario |
| 8 | Construcción y Estructuras |

### Tipos de formulario

| ID | Tipo |
|---|---|
| 1 | Caracterización |
| 2 | Observación |

### Tipos de comunidad (demografía)

| ID | Nombre |
|---|---|
| 1 | Resguardo indígena |
| 2 | Consejo comunitario |
| 3 | Veredal |
| 4 | Mestizos y blancos |
| 5 | Afrocolombianos |
| 6 | Rom |
| 7 | Junta Comunal |
| 8 | Ninguno |

### Tipos de pregunta

| ID | Nombre |
|---|---|
| 1 | Abierta |
| 2 | Selección múltiple |
| 3 | Selección única |
| 4 | Cuadrícula selección única |
| 5 | File |
| 6 | Cuadrícula selección múltiple |

---

## Mapa de relaciones (JOINs)

```
departamentos                    centros_poblados               respuestas
┌──────────────────┐             ┌──────────────────┐           ┌──────────────────┐
│ id_departamento  │◄────┐       │ id_centro_poblado│◄────┐     │ id_respuesta     │◄────┐
│ nombre           │     │       │ id_municipio     │     │     │ formulario_id    │     │
└──────────────────┘     │       │ nombre           │     │     │ vivienda_id      │     │
                         │       └──────────────────┘     │     │ user_id          │     │
                         │           ▲                     │     │ comenzado_en     │     │
                    ┌────┘           │                     │     │ finalizado_en    │     │
                    │                 │                     │     └──────────────────┘     │
               municipios            │                     │         ▲                     │
          ┌──────────────────┐       │                     │         │                     │
          │ id_municipio     │───────┘                     │         │                     │
          │ id_departamento  │────┐                        │    viviendas                 │
          │ nombre           │    │                        │    ┌──────────────────┐       │
          └──────────────────┘    │                        │    │ id_vivienda      │───────┘
                                  │                        │    │ zona_id          │───┐
                                  │                   zonas│    │ demografia_id    │──┐ │
                                  │                   ┌────┴──┐  │ apellidos_familia│  │ │
                                  │                   │id_zona│  │ numero_habitantes│  │ │
                                  └──────────────────►│centro_│  └──────────────────┘  │ │
                                                       │poblado│          ▲             │ │
                                                       │id     │          │             │ │
                                                       │codigo_│          │             │ │
                                                       │zona   │          │             │ │
                                                       │des-   │    demografias         │ │
                                                       │crip-  │    ┌──────────────────┐ │ │
                                                       │cion   │    │ id_demografia    │─┘ │
                                                       └───────┘    │ nombre           │   │
                                                                    └──────────────────┘   │
                                                                                           │
detalles_respuestas          preguntas                    opciones_respuestas             │
┌──────────────────────┐    ┌──────────────────┐         ┌──────────────────────┐         │
│ id_detalle_respuesta │    │ id_pregunta      │◄────────│ pregunta_id          │         │
│ respuesta_id         │──┐ │ tipo_pregunta_id │         │ id_opcion_respuesta  │         │
│ pregunta_id          │──┤ │ titulo_pregunta  │         │ texto_opcion         │         │
│ opcion_respuesta_id  │──┤ └──────────────────┘         └──────────────────────┘         │
│ respuesta_texto      │  │                                                                 │
│ grid_fila_id         │  │                                                                 │
└──────────────────────┘  │                                                                 │
                          │                                                                 │
                     ┌────┘                                                                 │
                     │                                                                      │
                     │   formularios                    formularios_preguntas                │
                     │   ┌──────────────────────┐      ┌──────────────────────────┐         │
                     │   │ id_formulario        │      │ formulario_id            │         │
                     │   │ modulo_id            │      │ pregunta_id              │─────────┘
                     │   │ tipo_formulario_id   │      │ orden                    │
                     │   │ titulo               │      └──────────────────────────┘
                     │   │ versionFormulario    │
                     │   └──────────────────────┘
                     │           ▲
                     │      modulos
                     │      ┌──────────┐
                     │      │ id_modulo│
                     └──────│ nombre   │
                            └──────────┘
```

---

## KPIs Ejecutivos — Cards Numéricas

Cada consulta va en una **Card separada** en Metabase (Scalar / Number).

### Card 1 — Total Encuestas Completadas
**Colección:** `respuestas`
```json
[
  { "$count": "total_encuestas" }
]
```

### Card 2 — Viviendas con al menos una Encuesta
**Colección:** `respuestas`
```json
[
  { "$group": { "_id": "$vivienda_id" } },
  { "$count": "viviendas_encuestadas" }
]
```

### Card 3 — Total Personas Alcanzadas
**Colección:** `viviendas`
```json
[
  {
    "$lookup": {
      "from": "respuestas",
      "localField": "id_vivienda",
      "foreignField": "vivienda_id",
      "as": "encuestas"
    }
  },
  { "$match": { "encuestas": { "$not": { "$size": 0 } } } },
  { "$group": { "_id": null, "total_personas": { "$sum": "$numero_habitantes" } } },
  { "$project": { "_id": 0, "total_personas": 1 } }
]
```

### Card 4 — Encuestadores Activos
**Colección:** `respuestas`
```json
[
  { "$group": { "_id": "$user_id" } },
  { "$count": "encuestadores_activos" }
]
```

### Card 5 — Total Datos Recolectados
**Colección:** `detalles_respuestas`
```json
[
  { "$count": "total_datos_recolectados" }
]
```

### Card 6 — Formularios Disponibles
**Colección:** `formularios`
```json
[
  { "$count": "total_formularios" }
]
```

### Card 7 — Zonas Activas con Datos
**Colección:** `viviendas`
```json
[
  { "$group": { "_id": "$zona_id" } },
  { "$count": "zonas_con_viviendas" }
]
```

---

## Consultas de Avance General

### AG-01 — Avance Total de Encuestas por Formulario
**📊 Visualización:** Barra horizontal
**📋 Decisión:** Reasignar encuestadores a módulos rezagados
**📁 Colección:** `respuestas`
```json
[
  { "$group": { "_id": "$formulario_id", "total_respuestas": { "$sum": 1 } } },
  {
    "$lookup": {
      "from": "formularios",
      "localField": "_id",
      "foreignField": "id_formulario",
      "as": "formulario"
    }
  },
  { "$unwind": "$formulario" },
  {
    "$lookup": {
      "from": "modulos",
      "localField": "formulario.modulo_id",
      "foreignField": "id_modulo",
      "as": "modulo"
    }
  },
  { "$unwind": { "path": "$modulo", "preserveNullAndEmptyArrays": true } },
  {
    "$project": {
      "_id": 0,
      "nombre_formulario": "$formulario.titulo",
      "modulo": "$modulo.nombre",
      "total_respuestas": 1
    }
  },
  { "$sort": { "total_respuestas": -1 } }
]
```

### AG-02 — Encuestas Completadas por Módulo Temático
**📊 Visualización:** Barra vertical o Circular
**📋 Decisión:** Vista ejecutiva de cobertura por área temática
**📁 Colección:** `respuestas`
```json
[
  {
    "$lookup": {
      "from": "formularios",
      "localField": "formulario_id",
      "foreignField": "id_formulario",
      "as": "formulario"
    }
  },
  { "$unwind": "$formulario" },
  {
    "$lookup": {
      "from": "modulos",
      "localField": "formulario.modulo_id",
      "foreignField": "id_modulo",
      "as": "modulo"
    }
  },
  { "$unwind": { "path": "$modulo", "preserveNullAndEmptyArrays": true } },
  {
    "$group": { "_id": "$modulo.nombre", "total_encuestas": { "$sum": 1 } }
  },
  {
    "$project": { "_id": 0, "modulo": "$_id", "total_encuestas": 1 }
  },
  { "$sort": { "total_encuestas": -1 } }
]
```

### AG-03 — Versiones de Formularios V1 vs V2
**📊 Visualización:** Barra apilada (Stacked bar)
**📋 Decisión:** Verificar adopción de formularios actualizados por módulo
**📁 Colección:** `respuestas`
```json
[
  {
    "$lookup": {
      "from": "formularios",
      "localField": "formulario_id",
      "foreignField": "id_formulario",
      "as": "formulario"
    }
  },
  { "$unwind": "$formulario" },
  {
    "$lookup": {
      "from": "modulos",
      "localField": "formulario.modulo_id",
      "foreignField": "id_modulo",
      "as": "modulo"
    }
  },
  { "$unwind": { "path": "$modulo", "preserveNullAndEmptyArrays": true } },
  {
    "$addFields": {
      "version_formulario": {
        "$cond": [
          { "$ifNull": ["$formulario.versionFormulario", false] },
          { "$concat": ["V", { "$toString": "$formulario.versionFormulario" }] },
          "V1 (Original)"
        ]
      }
    }
  },
  {
    "$group": {
      "_id": { "modulo": "$modulo.nombre", "version": "$version_formulario" },
      "total_encuestas": { "$sum": 1 }
    }
  },
  {
    "$project": {
      "_id": 0,
      "modulo": "$_id.modulo",
      "version": "$_id.version",
      "total_encuestas": 1
    }
  },
  { "$sort": { "modulo": 1, "version": 1 } }
]
```

### AG-04 — Cobertura Multidimensional por Vivienda
**📊 Visualización:** Tabla con progreso o Histograma
**📋 Decisión:** Identificar viviendas con módulos faltantes para priorizar visitas
**📁 Colección:** `respuestas`
```json
[
  {
    "$group": {
      "_id": "$vivienda_id",
      "formularios_respondidos": { "$addToSet": "$formulario_id" },
      "total_sesiones": { "$sum": 1 }
    }
  },
  {
    "$addFields": {
      "cantidad_formularios_unicos": { "$size": "$formularios_respondidos" }
    }
  },
  {
    "$lookup": {
      "from": "viviendas",
      "localField": "_id",
      "foreignField": "id_vivienda",
      "as": "vivienda"
    }
  },
  { "$unwind": { "path": "$vivienda", "preserveNullAndEmptyArrays": true } },
  {
    "$project": {
      "_id": 0,
      "apellidos_familia": "$vivienda.apellidos_familia",
      "numero_vivienda": "$vivienda.numero_vivienda",
      "numero_habitantes": "$vivienda.numero_habitantes",
      "total_sesiones": 1,
      "cantidad_formularios_unicos": 1,
      "cobertura_pct": {
        "$round": [
          { "$multiply": [{ "$divide": ["$cantidad_formularios_unicos", 8] }, 100] },
          1
        ]
      }
    }
  },
  { "$sort": { "cobertura_pct": 1 } }
]
```

### AG-05 — Complejidad de Formularios (Preguntas por Formulario)
**📊 Visualización:** Barra
**📋 Decisión:** Detectar formularios demasiado extensos vs muy cortos
**📁 Colección:** `formularios_preguntas`
```json
[
  {
    "$group": { "_id": "$formulario_id", "total_preguntas": { "$sum": 1 } }
  },
  {
    "$lookup": {
      "from": "formularios",
      "localField": "_id",
      "foreignField": "id_formulario",
      "as": "formulario"
    }
  },
  { "$unwind": "$formulario" },
  {
    "$lookup": {
      "from": "modulos",
      "localField": "formulario.modulo_id",
      "foreignField": "id_modulo",
      "as": "modulo"
    }
  },
  { "$unwind": { "path": "$modulo", "preserveNullAndEmptyArrays": true } },
  {
    "$project": {
      "_id": 0,
      "formulario": "$formulario.titulo",
      "modulo": "$modulo.nombre",
      "total_preguntas": 1
    }
  },
  { "$sort": { "total_preguntas": -1 } }
]
```

### AG-06 — Duración Promedio de Diligenciamiento por Formulario
**📊 Visualización:** Barra con rango min/max
**📋 Decisión:** Identificar formularios demasiado largos que fatigan al encuestador
**📁 Colección:** `respuestas`
```json
[
  {
    "$addFields": {
      "inicio": {
        "$dateFromString": {
          "dateString": "$comenzado_en",
          "format": "%Y-%m-%d %H:%M:%S"
        }
      },
      "fin": {
        "$dateFromString": {
          "dateString": "$finalizado_en",
          "format": "%Y-%m-%d %H:%M:%S"
        }
      }
    }
  },
  {
    "$addFields": {
      "duracion_minutos": {
        "$divide": [{ "$subtract": ["$fin", "$inicio"] }, 60000]
      }
    }
  },
  { "$match": { "duracion_minutos": { "$gt": 0, "$lt": 120 } } },
  {
    "$group": {
      "_id": "$formulario_id",
      "promedio_minutos": { "$avg": "$duracion_minutos" },
      "minimo_minutos": { "$min": "$duracion_minutos" },
      "maximo_minutos": { "$max": "$duracion_minutos" },
      "total_diligenciados": { "$sum": 1 }
    }
  },
  {
    "$lookup": {
      "from": "formularios",
      "localField": "_id",
      "foreignField": "id_formulario",
      "as": "formulario"
    }
  },
  { "$unwind": "$formulario" },
  {
    "$project": {
      "_id": 0,
      "formulario": "$formulario.titulo",
      "promedio_minutos": { "$round": ["$promedio_minutos", 1] },
      "minimo_minutos": { "$round": ["$minimo_minutos", 1] },
      "maximo_minutos": { "$round": ["$maximo_minutos", 1] },
      "total_diligenciados": 1
    }
  },
  { "$sort": { "promedio_minutos": -1 } }
]
```

---

## Consultas Geográficas Comparativas

### GEO-01 — Distribución de Respuestas por Zona (una pregunta)
**📊 Visualización:** Tabla Dinámica o Barra agrupada
**📋 Decisión:** Comparar cómo responde cada zona a una misma pregunta
**📁 Colección:** `detalles_respuestas`
> 🔧 Cambiar `pregunta_id: 187` por el ID de la pregunta a analizar
```json
[
  { "$match": { "pregunta_id": 187 } },
  {
    "$lookup": {
      "from": "respuestas",
      "localField": "respuesta_id",
      "foreignField": "id_respuesta",
      "as": "r"
    }
  },
  { "$unwind": "$r" },
  {
    "$lookup": {
      "from": "viviendas",
      "localField": "r.vivienda_id",
      "foreignField": "id_vivienda",
      "as": "v"
    }
  },
  { "$unwind": "$v" },
  {
    "$lookup": {
      "from": "zonas",
      "localField": "v.zona_id",
      "foreignField": "id_zona",
      "as": "z"
    }
  },
  { "$unwind": "$z" },
  {
    "$lookup": {
      "from": "opciones_respuestas",
      "localField": "opcion_respuesta_id",
      "foreignField": "id_opcion_respuesta",
      "as": "op"
    }
  },
  { "$unwind": { "path": "$op", "preserveNullAndEmptyArrays": true } },
  {
    "$group": {
      "_id": {
        "zona": "$z.descripcion",
        "opcion": "$op.texto_opcion"
      },
      "total": { "$sum": 1 }
    }
  },
  {
    "$project": {
      "_id": 0,
      "zona": "$_id.zona",
      "opcion": { "$ifNull": ["$_id.opcion", "Sin respuesta"] },
      "total": 1
    }
  },
  { "$sort": { "zona": 1, "total": -1 } }
]
```

### GEO-02 — Distribución de Respuestas por Municipio
**📊 Visualización:** Barra apilada o Tabla Dinámica
**📋 Decisión:** Focalizar intervenciones en municipios con mayor frecuencia de respuestas negativas
**📁 Colección:** `detalles_respuestas`
```json
[
  { "$match": { "pregunta_id": 187 } },
  {
    "$lookup": {
      "from": "respuestas",
      "localField": "respuesta_id",
      "foreignField": "id_respuesta",
      "as": "r"
    }
  },
  { "$unwind": "$r" },
  {
    "$lookup": {
      "from": "viviendas",
      "localField": "r.vivienda_id",
      "foreignField": "id_vivienda",
      "as": "v"
    }
  },
  { "$unwind": "$v" },
  {
    "$lookup": {
      "from": "zonas",
      "localField": "v.zona_id",
      "foreignField": "id_zona",
      "as": "z"
    }
  },
  { "$unwind": "$z" },
  {
    "$lookup": {
      "from": "centros_poblados",
      "localField": "z.centro_poblado_id",
      "foreignField": "id_centro_poblado",
      "as": "cp"
    }
  },
  { "$unwind": "$cp" },
  {
    "$lookup": {
      "from": "municipios",
      "localField": "cp.id_municipio",
      "foreignField": "id_municipio",
      "as": "m"
    }
  },
  { "$unwind": "$m" },
  {
    "$lookup": {
      "from": "opciones_respuestas",
      "localField": "opcion_respuesta_id",
      "foreignField": "id_opcion_respuesta",
      "as": "op"
    }
  },
  { "$unwind": { "path": "$op", "preserveNullAndEmptyArrays": true } },
  {
    "$group": {
      "_id": { "municipio": "$m.nombre", "opcion": "$op.texto_opcion" },
      "total": { "$sum": 1 }
    }
  },
  {
    "$project": {
      "_id": 0,
      "municipio": "$_id.municipio",
      "opcion": { "$ifNull": ["$_id.opcion", "Sin respuesta"] },
      "total": 1
    }
  },
  { "$sort": { "municipio": 1, "total": -1 } }
]
```

### GEO-03 — Distribución de Respuestas por Departamento
**📊 Visualización:** Barra o Circular
**📋 Decisión:** Informe ejecutivo para dirección SENA — priorización regional
**📁 Colección:** `detalles_respuestas`
```json
[
  { "$match": { "pregunta_id": 187 } },
  {
    "$lookup": {
      "from": "respuestas",
      "localField": "respuesta_id",
      "foreignField": "id_respuesta",
      "as": "r"
    }
  },
  { "$unwind": "$r" },
  {
    "$lookup": {
      "from": "viviendas",
      "localField": "r.vivienda_id",
      "foreignField": "id_vivienda",
      "as": "v"
    }
  },
  { "$unwind": "$v" },
  {
    "$lookup": {
      "from": "zonas",
      "localField": "v.zona_id",
      "foreignField": "id_zona",
      "as": "z"
    }
  },
  { "$unwind": "$z" },
  {
    "$lookup": {
      "from": "centros_poblados",
      "localField": "z.centro_poblado_id",
      "foreignField": "id_centro_poblado",
      "as": "cp"
    }
  },
  { "$unwind": "$cp" },
  {
    "$lookup": {
      "from": "municipios",
      "localField": "cp.id_municipio",
      "foreignField": "id_municipio",
      "as": "m"
    }
  },
  { "$unwind": "$m" },
  {
    "$lookup": {
      "from": "departamentos",
      "localField": "m.id_departamento",
      "foreignField": "id_departamento",
      "as": "dep"
    }
  },
  { "$unwind": "$dep" },
  {
    "$lookup": {
      "from": "opciones_respuestas",
      "localField": "opcion_respuesta_id",
      "foreignField": "id_opcion_respuesta",
      "as": "op"
    }
  },
  { "$unwind": { "path": "$op", "preserveNullAndEmptyArrays": true } },
  {
    "$group": {
      "_id": { "departamento": "$dep.nombre", "opcion": "$op.texto_opcion" },
      "total": { "$sum": 1 }
    }
  },
  {
    "$project": {
      "_id": 0,
      "departamento": "$_id.departamento",
      "opcion": { "$ifNull": ["$_id.opcion", "Sin respuesta"] },
      "total": 1
    }
  },
  { "$sort": { "departamento": 1, "total": -1 } }
]
```

### GEO-04 — Respuesta Dominante por Zona (Top 1)
**📊 Visualización:** Barra horizontal con etiqueta de opción ganadora
**📋 Decisión:** Ver de un vistazo el "perfil" de cada zona según su respuesta más frecuente
**📁 Colección:** `detalles_respuestas`
```json
[
  { "$match": { "pregunta_id": 187, "opcion_respuesta_id": { "$ne": null } } },
  {
    "$lookup": {
      "from": "respuestas",
      "localField": "respuesta_id",
      "foreignField": "id_respuesta",
      "as": "r"
    }
  },
  { "$unwind": "$r" },
  {
    "$lookup": {
      "from": "viviendas",
      "localField": "r.vivienda_id",
      "foreignField": "id_vivienda",
      "as": "v"
    }
  },
  { "$unwind": "$v" },
  {
    "$lookup": {
      "from": "zonas",
      "localField": "v.zona_id",
      "foreignField": "id_zona",
      "as": "z"
    }
  },
  { "$unwind": "$z" },
  {
    "$lookup": {
      "from": "opciones_respuestas",
      "localField": "opcion_respuesta_id",
      "foreignField": "id_opcion_respuesta",
      "as": "op"
    }
  },
  { "$unwind": "$op" },
  {
    "$group": {
      "_id": {
        "zona": "$z.descripcion",
        "opcion": "$op.texto_opcion",
        "opcion_id": "$opcion_respuesta_id"
      },
      "frecuencia": { "$sum": 1 }
    }
  },
  { "$sort": { "frecuencia": -1 } },
  {
    "$group": {
      "_id": "$_id.zona",
      "opcion_dominante": { "$first": "$_id.opcion" },
      "frecuencia_dominante": { "$first": "$frecuencia" },
      "total_respuestas_zona": { "$sum": "$frecuencia" }
    }
  },
  {
    "$project": {
      "_id": 0,
      "zona": "$_id",
      "opcion_dominante": 1,
      "frecuencia_dominante": 1,
      "total_respuestas_zona": 1,
      "porcentaje": {
        "$round": [
          { "$multiply": [{ "$divide": ["$frecuencia_dominante", "$total_respuestas_zona"] }, 100] },
          1
        ]
      }
    }
  },
  { "$sort": { "frecuencia_dominante": -1 } }
]
```

### GEO-05 — Tabla Maestra: Todo un Formulario × Zona
**📊 Visualización:** Tabla Dinámica (filas = pregunta, columnas = zona)
**📋 Decisión:** Dashboard navegable con drill-down por formulario y zona
**📁 Colección:** `detalles_respuestas`
> 🔧 Cambiar `"r.formulario_id": 10` por el ID del formulario deseado
```json
[
  {
    "$lookup": {
      "from": "respuestas",
      "localField": "respuesta_id",
      "foreignField": "id_respuesta",
      "as": "r"
    }
  },
  { "$unwind": "$r" },
  { "$match": { "r.formulario_id": 10 } },
  {
    "$lookup": {
      "from": "viviendas",
      "localField": "r.vivienda_id",
      "foreignField": "id_vivienda",
      "as": "v"
    }
  },
  { "$unwind": "$v" },
  {
    "$lookup": {
      "from": "zonas",
      "localField": "v.zona_id",
      "foreignField": "id_zona",
      "as": "z"
    }
  },
  { "$unwind": "$z" },
  {
    "$lookup": {
      "from": "preguntas",
      "localField": "pregunta_id",
      "foreignField": "id_pregunta",
      "as": "preg"
    }
  },
  { "$unwind": "$preg" },
  {
    "$lookup": {
      "from": "opciones_respuestas",
      "localField": "opcion_respuesta_id",
      "foreignField": "id_opcion_respuesta",
      "as": "op"
    }
  },
  { "$unwind": { "path": "$op", "preserveNullAndEmptyArrays": true } },
  {
    "$group": {
      "_id": {
        "zona": "$z.descripcion",
        "pregunta": "$preg.titulo_pregunta",
        "opcion": "$op.texto_opcion"
      },
      "total": { "$sum": 1 }
    }
  },
  {
    "$project": {
      "_id": 0,
      "zona": "$_id.zona",
      "pregunta": { "$substr": ["$_id.pregunta", 0, 80] },
      "opcion": { "$ifNull": ["$_id.opcion", "Texto libre"] },
      "total": 1
    }
  },
  { "$sort": { "pregunta": 1, "total": -1 } }
]
```

### GEO-06 — Comparativo por Tipo de Comunidad (Demografía)
**📊 Visualización:** Barra agrupada o Circular
**📋 Decisión:** Detectar diferencias étnico-territoriales en condiciones de vida
**📁 Colección:** `detalles_respuestas`
```json
[
  { "$match": { "pregunta_id": 187, "opcion_respuesta_id": { "$ne": null } } },
  {
    "$lookup": {
      "from": "respuestas",
      "localField": "respuesta_id",
      "foreignField": "id_respuesta",
      "as": "r"
    }
  },
  { "$unwind": "$r" },
  {
    "$lookup": {
      "from": "viviendas",
      "localField": "r.vivienda_id",
      "foreignField": "id_vivienda",
      "as": "v"
    }
  },
  { "$unwind": "$v" },
  {
    "$lookup": {
      "from": "demografias",
      "localField": "v.demografia_id",
      "foreignField": "id_demografia",
      "as": "dem"
    }
  },
  { "$unwind": { "path": "$dem", "preserveNullAndEmptyArrays": true } },
  {
    "$lookup": {
      "from": "opciones_respuestas",
      "localField": "opcion_respuesta_id",
      "foreignField": "id_opcion_respuesta",
      "as": "op"
    }
  },
  { "$unwind": "$op" },
  {
    "$group": {
      "_id": { "demografia": "$dem.nombre", "opcion": "$op.texto_opcion" },
      "total": { "$sum": 1 }
    }
  },
  {
    "$project": {
      "_id": 0,
      "tipo_comunidad": { "$ifNull": ["$_id.demografia", "Sin clasificar"] },
      "opcion": "$_id.opcion",
      "total": 1
    }
  },
  { "$sort": { "tipo_comunidad": 1, "total": -1 } }
]
```

### GEO-07 — Densidad Poblacional por Zona
**📊 Visualización:** Barra o Mapa
**📋 Decisión:** Priorizar zonas con mayor concentración de personas para maximizar impacto
**📁 Colección:** `viviendas`
```json
[
  {
    "$lookup": {
      "from": "zonas",
      "localField": "zona_id",
      "foreignField": "id_zona",
      "as": "zona"
    }
  },
  { "$unwind": { "path": "$zona", "preserveNullAndEmptyArrays": true } },
  {
    "$group": {
      "_id": "$zona_id",
      "descripcion_zona": { "$first": "$zona.descripcion" },
      "sector": { "$first": "$zona.sector_geografico" },
      "total_viviendas": { "$sum": 1 },
      "total_personas": { "$sum": "$numero_habitantes" },
      "promedio_personas": { "$avg": "$numero_habitantes" }
    }
  },
  {
    "$project": {
      "_id": 0,
      "descripcion_zona": 1,
      "sector": 1,
      "total_viviendas": 1,
      "total_personas": 1,
      "promedio_personas_vivienda": { "$round": ["$promedio_personas", 1] }
    }
  },
  { "$sort": { "total_personas": -1 } }
]
```

### GEO-08 — Distribución por Tipo de Comunidad (Viviendas)
**📊 Visualización:** Barra vertical con % o Circular
**📋 Decisión:** Garantizar representatividad de todos los grupos étnicos
**📁 Colección:** `viviendas`
```json
[
  {
    "$lookup": {
      "from": "demografias",
      "localField": "demografia_id",
      "foreignField": "id_demografia",
      "as": "demografia"
    }
  },
  { "$unwind": { "path": "$demografia", "preserveNullAndEmptyArrays": true } },
  {
    "$group": {
      "_id": "$demografia.nombre",
      "total_viviendas": { "$sum": 1 },
      "total_personas": { "$sum": "$numero_habitantes" },
      "promedio_personas": { "$avg": "$numero_habitantes" }
    }
  },
  {
    "$project": {
      "_id": 0,
      "tipo_comunidad": { "$ifNull": ["$_id", "Sin clasificar"] },
      "total_viviendas": 1,
      "total_personas": 1,
      "promedio_personas": { "$round": ["$promedio_personas", 1] }
    }
  },
  { "$sort": { "total_viviendas": -1 } }
]
```

---

## Consultas de Productividad

### PROD-01 — Ranking de Encuestadores
**📊 Visualización:** Barra horizontal
**📋 Decisión:** Reconocimiento de alto desempeño, detectar inactividad
**📁 Colección:** `respuestas`
```json
[
  { "$group": { "_id": "$user_id", "total_encuestas": { "$sum": 1 } } },
  {
    "$lookup": {
      "from": "users",
      "localField": "_id",
      "foreignField": "id_user",
      "as": "usuario"
    }
  },
  { "$unwind": "$usuario" },
  {
    "$project": {
      "_id": 0,
      "encuestador": {
        "$concat": ["$usuario.nombres", " ", "$usuario.apellidos"]
      },
      "total_encuestas": 1,
      "estado_usuario": "$usuario.estado"
    }
  },
  { "$sort": { "total_encuestas": -1 } }
]
```

### PROD-02 — Matriz Encuestador × Módulo
**📊 Visualización:** Tabla Dinámica (filas = encuestador, columnas = módulo)
**📋 Decisión:** Redistribuir carga, detectar encuestadores sin formación en ciertos módulos
**📁 Colección:** `respuestas`
```json
[
  {
    "$lookup": {
      "from": "formularios",
      "localField": "formulario_id",
      "foreignField": "id_formulario",
      "as": "formulario"
    }
  },
  { "$unwind": "$formulario" },
  {
    "$lookup": {
      "from": "modulos",
      "localField": "formulario.modulo_id",
      "foreignField": "id_modulo",
      "as": "modulo"
    }
  },
  { "$unwind": { "path": "$modulo", "preserveNullAndEmptyArrays": true } },
  {
    "$lookup": {
      "from": "users",
      "localField": "user_id",
      "foreignField": "id_user",
      "as": "usuario"
    }
  },
  { "$unwind": "$usuario" },
  {
    "$group": {
      "_id": {
        "encuestador": {
          "$concat": ["$usuario.nombres", " ", "$usuario.apellidos"]
        },
        "modulo": "$modulo.nombre"
      },
      "total": { "$sum": 1 }
    }
  },
  {
    "$project": {
      "_id": 0,
      "encuestador": "$_id.encuestador",
      "modulo": "$_id.modulo",
      "total_encuestas": "$total"
    }
  },
  { "$sort": { "encuestador": 1, "modulo": 1 } }
]
```

### PROD-03 — Heatmap de Actividad (Día × Franja Horaria)
**📊 Visualización:** Tabla Dinámica o progreso
**📋 Decisión:** Optimizar agendas de campo, detectar trabajo nocturno inusual
**📁 Colección:** `respuestas`
```json
[
  {
    "$addFields": {
      "fecha_dt": {
        "$dateFromString": {
          "dateString": "$comenzado_en",
          "format": "%Y-%m-%d %H:%M:%S"
        }
      }
    }
  },
  {
    "$addFields": {
      "dia_semana": { "$dayOfWeek": "$fecha_dt" },
      "hora_dia": { "$hour": "$fecha_dt" }
    }
  },
  {
    "$addFields": {
      "nombre_dia": {
        "$switch": {
          "branches": [
            { "case": { "$eq": ["$dia_semana", 1] }, "then": "Domingo" },
            { "case": { "$eq": ["$dia_semana", 2] }, "then": "Lunes" },
            { "case": { "$eq": ["$dia_semana", 3] }, "then": "Martes" },
            { "case": { "$eq": ["$dia_semana", 4] }, "then": "Miércoles" },
            { "case": { "$eq": ["$dia_semana", 5] }, "then": "Jueves" },
            { "case": { "$eq": ["$dia_semana", 6] }, "then": "Viernes" },
            { "case": { "$eq": ["$dia_semana", 7] }, "then": "Sábado" }
          ],
          "default": "Desconocido"
        }
      },
      "franja_horaria": {
        "$switch": {
          "branches": [
            { "case": { "$and": [{ "$gte": ["$hora_dia", 6] }, { "$lt": ["$hora_dia", 12] }] }, "then": "Mañana (6-12)" },
            { "case": { "$and": [{ "$gte": ["$hora_dia", 12] }, { "$lt": ["$hora_dia", 18] }] }, "then": "Tarde (12-18)" },
            { "case": { "$and": [{ "$gte": ["$hora_dia", 18] }, { "$lt": ["$hora_dia", 22] }] }, "then": "Noche (18-22)" }
          ],
          "default": "Madrugada (0-6)"
        }
      }
    }
  },
  {
    "$group": {
      "_id": { "dia": "$nombre_dia", "franja": "$franja_horaria", "num_dia": "$dia_semana" },
      "total_encuestas": { "$sum": 1 }
    }
  },
  {
    "$project": {
      "_id": 0,
      "dia_semana": "$_id.dia",
      "franja_horaria": "$_id.franja",
      "orden_dia": "$_id.num_dia",
      "total_encuestas": 1
    }
  },
  { "$sort": { "orden_dia": 1 } }
]
```

---

## Consultas de Calidad del Dato

### CAL-01 — Preguntas con Mayor Tasa de Omisión (>20%)
**📊 Visualización:** Barra horizontal con % de vacíos
**📋 Decisión:** Rediseñar preguntas problemáticas para la siguiente versión
**📁 Colección:** `detalles_respuestas`
```json
[
  {
    "$addFields": {
      "es_vacia": {
        "$cond": [
          {
            "$and": [
              { "$eq": ["$respuesta_texto", null] },
              { "$eq": ["$opcion_respuesta_id", null] }
            ]
          },
          1,
          0
        ]
      }
    }
  },
  {
    "$group": {
      "_id": "$pregunta_id",
      "total_respuestas": { "$sum": 1 },
      "respuestas_vacias": { "$sum": "$es_vacia" }
    }
  },
  {
    "$addFields": {
      "pct_vacias": {
        "$round": [
          { "$multiply": [{ "$divide": ["$respuestas_vacias", "$total_respuestas"] }, 100] },
          1
        ]
      }
    }
  },
  { "$match": { "pct_vacias": { "$gt": 20 } } },
  {
    "$lookup": {
      "from": "preguntas",
      "localField": "_id",
      "foreignField": "id_pregunta",
      "as": "pregunta"
    }
  },
  { "$unwind": "$pregunta" },
  {
    "$project": {
      "_id": 0,
      "pregunta": { "$substr": ["$pregunta.titulo_pregunta", 0, 80] },
      "total_respuestas": 1,
      "respuestas_vacias": 1,
      "pct_vacias": 1
    }
  },
  { "$sort": { "pct_vacias": -1 } },
  { "$limit": 20 }
]
```

### CAL-02 — Composición del Tipo de Respuesta (Texto vs Opción vs Vacío)
**📊 Visualización:** Circular (Donut)
**📋 Decisión:** Detectar problemas de diseño del instrumento
**📁 Colección:** `detalles_respuestas`
```json
[
  {
    "$addFields": {
      "tipo_respuesta": {
        "$switch": {
          "branches": [
            {
              "case": {
                "$and": [
                  { "$ne": ["$respuesta_texto", null] },
                  { "$ne": ["$opcion_respuesta_id", null] }
                ]
              },
              "then": "Texto + Opción"
            },
            {
              "case": { "$ne": ["$opcion_respuesta_id", null] },
              "then": "Solo Opción"
            },
            {
              "case": { "$ne": ["$respuesta_texto", null] },
              "then": "Solo Texto"
            }
          ],
          "default": "Sin Respuesta"
        }
      }
    }
  },
  {
    "$group": { "_id": "$tipo_respuesta", "total": { "$sum": 1 } }
  },
  { "$project": { "_id": 0, "tipo_respuesta": "$_id", "total": 1 } },
  { "$sort": { "total": -1 } }
]
```

### CAL-03 — Tipos de Pregunta más Usados (Volumen de Respuestas)
**📊 Visualización:** Circular (Donut)
**📋 Decisión:** Evaluar el balance entre preguntas cualitativas y cuantitativas
**📁 Colección:** `detalles_respuestas`
```json
[
  {
    "$lookup": {
      "from": "preguntas",
      "localField": "pregunta_id",
      "foreignField": "id_pregunta",
      "as": "pregunta"
    }
  },
  { "$unwind": "$pregunta" },
  {
    "$lookup": {
      "from": "tipos_preguntas",
      "localField": "pregunta.tipo_pregunta_id",
      "foreignField": "id_tipo_pregunta",
      "as": "tipo"
    }
  },
  { "$unwind": { "path": "$tipo", "preserveNullAndEmptyArrays": true } },
  {
    "$group": {
      "_id": "$tipo.nombre",
      "total_respuestas": { "$sum": 1 }
    }
  },
  {
    "$project": {
      "_id": 0,
      "tipo_pregunta": { "$ifNull": ["$_id", "Sin clasificar"] },
      "total_respuestas": 1
    }
  },
  { "$sort": { "total_respuestas": -1 } }
]
```

### CAL-04 — Top 20 Opciones Más Seleccionadas en el Sistema
**📊 Visualización:** Barra horizontal
**📋 Decisión:** Identificar problemáticas dominantes en las comunidades
**📁 Colección:** `detalles_respuestas`
```json
[
  { "$match": { "opcion_respuesta_id": { "$ne": null } } },
  { "$group": { "_id": "$opcion_respuesta_id", "frecuencia": { "$sum": 1 } } },
  { "$sort": { "frecuencia": -1 } },
  { "$limit": 20 },
  {
    "$lookup": {
      "from": "opciones_respuestas",
      "localField": "_id",
      "foreignField": "id_opcion_respuesta",
      "as": "opcion"
    }
  },
  { "$unwind": "$opcion" },
  {
    "$lookup": {
      "from": "preguntas",
      "localField": "opcion.pregunta_id",
      "foreignField": "id_pregunta",
      "as": "pregunta"
    }
  },
  { "$unwind": { "path": "$pregunta", "preserveNullAndEmptyArrays": true } },
  {
    "$project": {
      "_id": 0,
      "opcion_texto": "$opcion.texto_opcion",
      "pregunta": "$pregunta.titulo_pregunta",
      "frecuencia": 1
    }
  }
]
```

### CAL-05 — Distribución de Habitantes por Vivienda
**📊 Visualización:** Barra (histograma por rangos)
**📋 Decisión:** Dimensionar correctamente programas de intervención social
**📁 Colección:** `viviendas`
```json
[
  {
    "$addFields": {
      "rango_habitantes": {
        "$switch": {
          "branches": [
            { "case": { "$lte": ["$numero_habitantes", 2] }, "then": "1-2 personas" },
            { "case": { "$lte": ["$numero_habitantes", 4] }, "then": "3-4 personas" },
            { "case": { "$lte": ["$numero_habitantes", 6] }, "then": "5-6 personas" },
            { "case": { "$lte": ["$numero_habitantes", 9] }, "then": "7-9 personas" }
          ],
          "default": "10+ personas"
        }
      }
    }
  },
  {
    "$group": {
      "_id": "$rango_habitantes",
      "total_viviendas": { "$sum": 1 },
      "total_personas": { "$sum": "$numero_habitantes" }
    }
  },
  { "$project": { "_id": 0, "rango_habitantes": "$_id", "total_viviendas": 1, "total_personas": 1 } },
  { "$sort": { "rango_habitantes": 1 } }
]
```

---

## Consultas de Análisis Temporal

### TEMP-01 — Tendencia Mensual de Encuestas
**📊 Visualización:** Línea
**📋 Decisión:** Velocidad del trabajo de campo, detectar picos/caídas
**📁 Colección:** `respuestas`
```json
[
  {
    "$addFields": {
      "fecha_dt": {
        "$dateFromString": {
          "dateString": "$comenzado_en",
          "format": "%Y-%m-%d %H:%M:%S"
        }
      }
    }
  },
  {
    "$group": {
      "_id": {
        "anio": { "$year": "$fecha_dt" },
        "mes": { "$month": "$fecha_dt" }
      },
      "total_encuestas": { "$sum": 1 }
    }
  },
  {
    "$addFields": {
      "periodo": {
        "$concat": [
          { "$toString": "$_id.anio" }, "-",
          {
            "$cond": [
              { "$lt": ["$_id.mes", 10] },
              { "$concat": ["0", { "$toString": "$_id.mes" }] },
              { "$toString": "$_id.mes" }
            ]
          }
        ]
      }
    }
  },
  {
    "$project": {
      "_id": 0,
      "periodo": 1,
      "anio": "$_id.anio",
      "mes": "$_id.mes",
      "total_encuestas": 1
    }
  },
  { "$sort": { "anio": 1, "mes": 1 } }
]
```

### TEMP-02 — Tendencia de una Opción Específica por Zona en el Tiempo
**📊 Visualización:** Línea (eje X = mes, series = zonas)
**📋 Decisión:** Ver si una intervención tuvo impacto en el tiempo
**📁 Colección:** `detalles_respuestas`
> 🔧 Cambiar `pregunta_id` y `opcion_respuesta_id`
```json
[
  { "$match": { "pregunta_id": 187, "opcion_respuesta_id": 887 } },
  {
    "$lookup": {
      "from": "respuestas",
      "localField": "respuesta_id",
      "foreignField": "id_respuesta",
      "as": "r"
    }
  },
  { "$unwind": "$r" },
  {
    "$addFields": {
      "fecha_dt": {
        "$dateFromString": {
          "dateString": "$r.comenzado_en",
          "format": "%Y-%m-%d %H:%M:%S"
        }
      }
    }
  },
  {
    "$lookup": {
      "from": "viviendas",
      "localField": "r.vivienda_id",
      "foreignField": "id_vivienda",
      "as": "v"
    }
  },
  { "$unwind": "$v" },
  {
    "$lookup": {
      "from": "zonas",
      "localField": "v.zona_id",
      "foreignField": "id_zona",
      "as": "z"
    }
  },
  { "$unwind": "$z" },
  {
    "$group": {
      "_id": {
        "zona": "$z.descripcion",
        "anio": { "$year": "$fecha_dt" },
        "mes": { "$month": "$fecha_dt" }
      },
      "total": { "$sum": 1 }
    }
  },
  {
    "$addFields": {
      "periodo": {
        "$concat": [
          { "$toString": "$_id.anio" }, "-",
          { "$cond": [{ "$lt": ["$_id.mes", 10] }, { "$concat": ["0", { "$toString": "$_id.mes" }] }, { "$toString": "$_id.mes" }] }
        ]
      }
    }
  },
  {
    "$project": {
      "_id": 0,
      "zona": "$_id.zona",
      "periodo": 1,
      "total": 1
    }
  },
  { "$sort": { "periodo": 1, "zona": 1 } }
]
```

---

## Consultas de Análisis Cualitativo

### CUAL-01 — Respuestas Textuales Exportables
**📊 Visualización:** Tabla exportable
**📋 Decisión:** Análisis cualitativo, nubes de palabras, NLP
**📁 Colección:** `detalles_respuestas`
```json
[
  {
    "$match": {
      "respuesta_texto": { "$ne": null },
      "$expr": { "$gt": [{ "$strLenCP": "$respuesta_texto" }, 5] }
    }
  },
  {
    "$lookup": {
      "from": "preguntas",
      "localField": "pregunta_id",
      "foreignField": "id_pregunta",
      "as": "pregunta"
    }
  },
  { "$unwind": { "path": "$pregunta", "preserveNullAndEmptyArrays": true } },
  {
    "$lookup": {
      "from": "respuestas",
      "localField": "respuesta_id",
      "foreignField": "id_respuesta",
      "as": "sesion"
    }
  },
  { "$unwind": { "path": "$sesion", "preserveNullAndEmptyArrays": true } },
  {
    "$lookup": {
      "from": "formularios",
      "localField": "sesion.formulario_id",
      "foreignField": "id_formulario",
      "as": "formulario"
    }
  },
  { "$unwind": { "path": "$formulario", "preserveNullAndEmptyArrays": true } },
  {
    "$project": {
      "_id": 0,
      "formulario": "$formulario.titulo",
      "pregunta": { "$substr": ["$pregunta.titulo_pregunta", 0, 100] },
      "respuesta_texto": 1,
      "vivienda_id": "$sesion.vivienda_id",
      "fecha": "$sesion.comenzado_en"
    }
  },
  { "$sort": { "fecha": -1 } },
  { "$limit": 500 }
]
```

---

## Consultas de Soporte — Catálogos

### CAT-01 — Catálogo de Preguntas por Formulario y Módulo
**📊 Visualización:** Tabla
**📋 Decisión:** Encontrar `pregunta_id` para usar en filtros de otras consultas
**📁 Colección:** `formularios_preguntas`
```json
[
  {
    "$lookup": {
      "from": "preguntas",
      "localField": "pregunta_id",
      "foreignField": "id_pregunta",
      "as": "pregunta"
    }
  },
  { "$unwind": "$pregunta" },
  {
    "$lookup": {
      "from": "formularios",
      "localField": "formulario_id",
      "foreignField": "id_formulario",
      "as": "formulario"
    }
  },
  { "$unwind": "$formulario" },
  {
    "$lookup": {
      "from": "modulos",
      "localField": "formulario.modulo_id",
      "foreignField": "id_modulo",
      "as": "modulo"
    }
  },
  { "$unwind": { "path": "$modulo", "preserveNullAndEmptyArrays": true } },
  {
    "$lookup": {
      "from": "tipos_preguntas",
      "localField": "pregunta.tipo_pregunta_id",
      "foreignField": "id_tipo_pregunta",
      "as": "tipo"
    }
  },
  { "$unwind": { "path": "$tipo", "preserveNullAndEmptyArrays": true } },
  {
    "$project": {
      "_id": 0,
      "pregunta_id": "$pregunta_id",
      "formulario_id": "$formulario_id",
      "formulario": "$formulario.titulo",
      "modulo": "$modulo.nombre",
      "tipo_pregunta": "$tipo.nombre",
      "pregunta": { "$substr": ["$pregunta.titulo_pregunta", 0, 100] },
      "orden": "$orden"
    }
  },
  { "$sort": { "formulario_id": 1, "orden": 1 } }
]
```

### CAT-02 — Catálogo de Opciones por Pregunta
**📊 Visualización:** Tabla
**📋 Decisión:** Ver qué valores puede tomar cada pregunta de selección
**📁 Colección:** `opciones_respuestas`
```json
[
  {
    "$lookup": {
      "from": "preguntas",
      "localField": "pregunta_id",
      "foreignField": "id_pregunta",
      "as": "pregunta"
    }
  },
  { "$unwind": "$pregunta" },
  { "$match": { "pregunta.tipo_pregunta_id": { "$in": [2, 3] } } },
  {
    "$project": {
      "_id": 0,
      "pregunta_id": 1,
      "id_opcion": "$id_opcion_respuesta",
      "pregunta": { "$substr": ["$pregunta.titulo_pregunta", 0, 80] },
      "texto_opcion": 1
    }
  },
  { "$sort": { "pregunta_id": 1, "id_opcion": 1 } }
]
```

### CAT-03 — Viviendas y Zonas Geográficas
**📊 Visualización:** Tabla
**📋 Decisión:** Conocer la distribución territorial
**📁 Colección:** `viviendas`
```json
[
  {
    "$lookup": {
      "from": "zonas",
      "localField": "zona_id",
      "foreignField": "id_zona",
      "as": "zona"
    }
  },
  { "$unwind": { "path": "$zona", "preserveNullAndEmptyArrays": true } },
  {
    "$project": {
      "_id": 0,
      "id_vivienda": 1,
      "apellidos_familia": 1,
      "numero_vivienda": 1,
      "numero_habitantes": 1,
      "zona": "$zona.descripcion",
      "sector": "$zona.sector_geografico",
      "demografia_id": 1
    }
  },
  { "$sort": { "zona": 1, "apellidos_familia": 1 } }
]
```

### CAT-04 — Formularios por Módulo e ID
**📊 Visualización:** Tabla
**📋 Decisión:** Conocer a qué módulo pertenece cada formulario
**📁 Colección:** `formularios`
```json
[
  {
    "$lookup": {
      "from": "modulos",
      "localField": "modulo_id",
      "foreignField": "id_modulo",
      "as": "modulo"
    }
  },
  { "$unwind": { "path": "$modulo", "preserveNullAndEmptyArrays": true } },
  {
    "$lookup": {
      "from": "tipos_formulario",
      "localField": "tipo_formulario_id",
      "foreignField": "id_tipo_formulario",
      "as": "tipo"
    }
  },
  { "$unwind": { "path": "$tipo", "preserveNullAndEmptyArrays": true } },
  {
    "$project": {
      "_id": 0,
      "id_formulario": 1,
      "titulo": 1,
      "modulo": "$modulo.nombre",
      "tipo_formulario": "$tipo.tipo_formulario",
      "version": "$versionFormulario",
      "estado": 1
    }
  },
  { "$sort": { "modulo": 1, "id_formulario": 1 } }
]
```

---

## Arquitectura analítica recomendada (Fase 2)

### Problema crítico detectado
Con **112,431 documentos** en `detalles_respuestas` y **5,052** en `respuestas`, y los pipelines que requieren 4-6 lookups, **cada consulta escaneará toda la colección**. En producción con varios usuarios concurrentes, el rendimiento será deficiente.

### Solución: Crear colección analítica pre-unida

```javascript
// analytics_respuestas_geo — se recalcula nocturnamente por ETL
// Esta colección elimina TODOS los lookups en tiempo real
{
  detalle_id: 1,
  respuesta_id: 1,
  pregunta_id: 187,
  pregunta_titulo: "¿Qué problemas energéticos tiene su vivienda?",
  tipo_pregunta_id: 2,
  tipo_pregunta: "seleccion_multiple",
  opcion_respuesta_id: 887,
  opcion_texto: "No tiene conexión eléctrica",
  respuesta_texto: null,
  formulario_id: 10,
  formulario_titulo: "Módulo 1 – Componente Energético",
  modulo_id: 4,
  modulo_nombre: "Energético",
  tipo_formulario: "Caracterización",
  vivienda_id: 43,
  apellidos_familia: "Yusela Banguera",
  numero_habitantes: 5,
  demografia_id: 1,
  demografia_nombre: "Resguardo indígena",
  zona_id: 1,
  zona_descripcion: "CAUCA-TIMBIQUÍ-COTEJE-CENTRO",
  sector: "CENTRO",
  centro_poblado_id: 2719,
  centro_poblado_nombre: "COTEJE",
  municipio_id: 123,
  municipio_nombre: "TIMBIQUÍ",
  departamento_id: 8,
  departamento_nombre: "CAUCA",
  fecha_comenzado: ISODate("2025-11-25T11:03:38Z"),
  fecha_finalizado: ISODate("2025-11-25T11:04:21Z"),
  user_id: 7,
  encuestador: "Juan Pérez",
}
```

Con esta colección, **todos los dashboards son un simple `$match` + `$group`** sin ningún lookup.

### Otras colecciones analíticas sugeridas

| Colección | Propósito | Frecuencia |
|---|---|---|
| `analytics_resumen_formulario` | Avance + tiempos por formulario | Diaria |
| `analytics_cobertura_zona` | Viviendas + personas + encuestas por zona | Diaria |
| `analytics_productividad_usuario` | Encuestas por usuario/módulo/semana | Semanal |
| `analytics_opciones_frecuencia` | Top opciones por pregunta (snapshot) | Nocturna |

---

## Anexo: Visualizaciones Metabase disponibles

| Visualización | Cuándo usarla |
|---|---|
| **Barra** | Comparar categorías (opciones, zonas, módulos) |
| **Barra apilada** | Mostrar composición (V1 vs V2, opciones por zona) |
| **Circular / Donut** | Pocas categorías con proporciones (tipos de pregunta, demografía) |
| **Línea** | Tendencias temporales (encuestas por mes) |
| **Área** | Volumen acumulado en el tiempo |
| **Tabla** | Datos detallados exportables |
| **Tabla Dinámica** | Cruzar dos dimensiones (zona × opción, encuestador × módulo) |
| **Número / Scalar** | KPIs ejecutivos individuales |
| **Mapa** | Distribución geográfica (si tienes coordenadas) |
| **Diagrama de Caja** | Distribución de variables numéricas (habitantes, duración) |
| **Medidor / Progreso** | Avance contra meta (80% de cobertura, formularios completados) |
| **Embudo** | Procesos secuenciales (no aplica directamente aquí) |
| **Cascada** | Cambios incrementales |
| **Combo** | Múltiples métricas combinadas |
| **Dispersión** | Correlaciones entre variables numéricas |
| **Sankey** | Flujos entre categorías (origen ↔ destino) |
| **Tendencia** | Evolución con línea de tendencia |
| **Detalle** | Información detallada de un punto específico |

---

> **🔒 Nota de seguridad:** La colección `users` contiene `pass_pub` (contraseñas en texto plano). Se recomienda **no exponer** esta colección en Metabase con permisos públicos. Crear un usuario MongoDB de solo lectura que no tenga acceso a `users`.

---

*Documento generado el 24 de mayo de 2026 para el proyecto **Raíz Pacífico — SENA***
