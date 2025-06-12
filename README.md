**Diseño de Base de Datos NoSQL para el Parque Nicole**

**Integrantes:**

- David Adolfo Gomez Uribe
- Daniel Cruz

---

## 1. Introducción

Este proyecto propone un modelo de datos NoSQL en MongoDB para el Parque Nicole, aprovechando la flexibilidad de documentos para representar atracciones, zonas, visitantes, tickets, empleados, eventos y registros de mantenimiento. Se combinan estrategias de incrustación (embedding) y referencia (referencing) según la frecuencia de acceso, tamaño de subdocumentos y necesidades de consistencia.

---

## 2. Entidades y Modelo de Datos

### 2.1 Colección `atracciones`

**Atributos:**

- `_id`: ObjectId
- `nombre`: String
- `tipo`: String\ (montaña rusa, carrusel, show, etc.)
- `descripcion`: String
- `altura_minima`: Number\  (altura mínima requerida en cm)
- `capacidad`: Number\  (personas simultáneas)
- `estado`: String\
  ("operativa", "mantenimiento")
- `espera_promedio`: Number\   (minutos)
- `zona_id`: ObjectId\
  (referencia a zona)

**Relación Atracción → Zona**

- Tipo: Uno a Muchos (una zona contiene muchas atracciones)
- Estrategia: **Referencia** (campo `zona_id` en documento de atracción)
- Justificación: Las atracciones son entidades con datos propios que pueden crecer (historial de mantenimiento, estadísticas), por lo que mantenerlas en su colección mejora performance en consultas específicas y evita documentos de zona muy grandes.

**Ejemplo de documento:**

```json
{
  _id: ObjectId("64f1a8..."),
  nombre: "Montaña Rusa Dragón",
  tipo: "montaña rusa",
  descripcion: "Vuelta de alta velocidad con loopings.",
  altura_minima: 120,
  capacidad: 24,
  estado: "operativa",
  espera_promedio: 15,
  zona_id: ObjectId("64f1a1...")
}
```

---

### 2.2 Colección `zonas`

**Atributos:**

- `_id`: ObjectId
- `nombre`: String
- `descripcion`: String
- `atracciones`: [ObjectId]\
  (lista de IDs de atracciones)

**Relación Zona → Atracciones**

- Tipo: Uno a Muchos
- Estrategia: **Referencia** (lista de `ObjectId`)
- Justificación: Se mantiene una lista de referencias para mostrar rápidamente qué atracciones hay, sin duplicar datos.

**Ejemplo de documento:**

```json
{
  _id: ObjectId("64f1a1..."),
  nombre: "Zona Infantil",
  descripcion: "Atracciones para niños de 3 a 10 años.",
  atracciones: [
    ObjectId("64f1a8..."),
    ObjectId("64f1b2...")
  ]
}
```

---

### 2.3 Colección `visitantes`

**Atributos:**

- `_id`: ObjectId
- `nombre`: String
- `apellido`: String
- `fecha_nacimiento`: Date
- `email`: String (único)
- `historial_visitas`: [Date]\
  (fechas de entradas al parque)
- `tickets`: [ObjectId]\
  (IDs de tickets comprados)

**Relaciones:**

- Visita → Historial: **Embedding** de fechas (subdocumentos pequeños y bounded)
- Visitante → Tickets: **Referencia** de IDs en array `tickets`

**Justificación:**

- El historial de visitas es un arreglo acotado (no excesivamente grande) y se consulta junto con el perfil.
- Los tickets tienen su propia colección porque contienen detalles y potencialmente vínculos a descuentos o validaciones, por lo que referenciarlos evita documentos de visitante demasiado pesados.

**Ejemplo de documento:**

```json
{
  _id: ObjectId("64f2c3..."),
  nombre: "María",
  apellido: "González",
  fecha_nacimiento: ISODate("1990-05-12"),
  email: "maria.gonzalez@example.com",
  historial_visitas: [
    ISODate("2025-06-01"),
    ISODate("2025-06-05")
  ],
  tickets: [
    ObjectId("64f2d5..."),
    ObjectId("64f2d8...")
  ]
}
```

---

### 2.4 Colección `tickets`

**Atributos:**

- `_id`: ObjectId
- `tipo`: String\
  ("diario", "anual", "VIP")
- `precio`: Number
- `fecha_compra`: Date
- `fecha_validez`: Date
- `visitante_id`: ObjectId\
  (referencia)

**Relación Ticket → Visitante**

- Tipo: Muchos a Uno
- Estrategia: **Referencia** de `visitante_id`
- Justificación: Permite consultas desde ticket a visitante y mantiene tickets independientes para registros, devolviendo su precio histórico incluso si el perfil cambia.

**Ejemplo de documento:**

```json
{
  _id: ObjectId("64f2d5..."),
  tipo: "diario",
  precio: 50,
  fecha_compra: ISODate("2025-06-05"),
  fecha_validez: ISODate("2025-06-05"),
  visitante_id: ObjectId("64f2c3...")
}
```

---

### 2.5 Colección `empleados`

**Atributos:**

- `_id`: ObjectId
- `nombre`: String
- `apellido`: String
- `cargo`: String
- `horario`: { `entrada`: String, `salida`: String }
- `atracciones_asignadas`: [ObjectId]

**Relación Empleado → Atracciones**

- Tipo: Muchos a Muchos (un empleado puede cubrir varias atracciones y viceversa)
- Estrategia: **Referencia** (array de `ObjectId` de atracciones)
- Justificación: Evita duplicación y permite reasignar sin operaciones costosas.

**Ejemplo de documento:**

```json
{
  _id: ObjectId("64f3e9..."),
  nombre: "Carlos",
  apellido: "Ramírez",
  cargo: "Operador",
  horario: { entrada: "08:00", salida: "16:00" },
  atracciones_asignadas: [
    ObjectId("64f1a8..."),
    ObjectId("64f1b2...")
  ]
}
```

---

### 2.6 Colección `eventos`

**Atributos:**

- `_id`: ObjectId
- `nombre`: String
- `descripcion`: String
- `horario`: { `inicio`: Date, `fin`: Date }
- `ubicacion`: { `tipo`: String, `ref_id`: ObjectId }
- `empleados_responsables`: [ObjectId]

**Relaciones:**

- Evento → Ubicación: **Referencia polimórfica** (puede referenciar a atracción o zona)
- Evento → Empleados: **Referencia** de IDs en array

**Justificación:**

- La localización polimórfica mantiene flexibilidad para futuros tipos de ubicación.
- Los empleados son independientes y referenciados para evitar duplicación.

**Ejemplo de documento:**

```json
{
  _id: ObjectId("64f4a1..."),
  nombre: "Concierto Nocturno",
  descripcion: "Show de luces y música.",
  horario: { inicio: ISODate("2025-06-20T20:00:00Z"), fin: ISODate("2025-06-20T22:00:00Z") },
  ubicacion: { tipo: "zona", ref_id: ObjectId("64f1a1...") },
  empleados_responsables: [ ObjectId("64f3e9...") ]
}
```

---

### 2.7 Colección `mantenimientos`

**Atributos:**

- `_id`: ObjectId
- `atraccion_id`: ObjectId
- `fecha`: Date
- `descripcion_trabajo`: String
- `empleados`: [ObjectId]
- `costo`: Number

**Relación Mantenimiento → Atracción/Empleado**

- Ambos por **Referencia**
- Justificación: Los registros de mantenimiento pueden crecer con el tiempo; referenciarlos mantiene la colección manejable y permite auditoría.

**Ejemplo de documento:**

```json
{
  _id: ObjectId("64f5b2..."),
  atraccion_id: ObjectId("64f1a8..."),
  fecha: ISODate("2025-06-01"),
  descripcion_trabajo: "Reemplazo de frenos en carril.",
  empleados: [ ObjectId("64f3e9..."), ObjectId("64f3f1...") ],
  costo: 1200
}
```

---

## 3. Conclusiones y Desafíos

- **Decisiones clave:**
  - Se utilizó **referencias** para relaciones con crecimiento indefinido o entornos de muchos a muchos.
  - Se utilizó **incrustación** en historial de visitas por ser bounded y consultado junto al visitante.
- **Desafíos:**
  - Manejo de ubicación polimórfica en `eventos` requiere lógica adicional en la aplicación.
  - Balance entre tamaño de documento y número de joins: MongoDB limita joins, pero las referencias permiten `$lookup` en agregaciones.

Este diseño proporciona un equilibrio entre rendimiento, flexibilidad y escalabilidad, aprovechando las fortalezas de MongoDB.

