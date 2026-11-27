# Modelo 10 · Aerolínea

**Nivel:** avanzado (relación ternaria, varias FK hacia la misma tabla, restricciones múltiples)
**Diagrama:** [aerolinea.puml](aerolinea.puml)

## Contexto

Una aerolínea regional programa vuelos entre aeropuertos con su flota de aviones. Cada vuelo lleva una tripulación asignada. Los clientes hacen reservas que pueden incluir varios pasajeros y varios vuelos (ida y vuelta, escalas). Cada combinación de pasajero y vuelo es un billete, con su asiento, y a cada billete se le puede facturar equipaje.

## Entidades y atributos

### Aeropuerto → tabla `aeropuertos`

| Atributo | Tipo | Significado |
|---|---|---|
| id | Long | Identificador interno. |
| codigoIata | String | Código internacional de 3 letras (`SCQ`, `MAD`, `BCN`). Es único. |
| nombre | String | Nombre oficial ("Aeropuerto de Santiago-Rosalía de Castro"). |
| ciudad, pais | String | Ubicación. |

### ModeloAvion → tabla `modelos_avion`

| Atributo | Tipo | Significado |
|---|---|---|
| id | Long | Identificador interno. |
| fabricante | String | "Airbus", "Boeing", "ATR"… |
| nombre | String | "A320neo", "737-800", "ATR 72"… Es único. |
| numFilas, asientosPorFila | Integer | Distribución de la cabina. Sirven para validar que un asiento existe (fila 1..numFilas, letra A..F). |
| autonomiaKm | Integer | Distancia máxima que puede volar sin repostar. |

### Avion → tabla `aviones`

| Atributo | Tipo | Significado |
|---|---|---|
| id | Long | Identificador interno. |
| matricula | String | Registro del aparato (`EC-MXV`). Es único. |
| fechaFabricacion | LocalDate | Antigüedad del avión. |
| activo | boolean | `false` si está retirado o en revisión larga. |

### Vuelo → tabla `vuelos`
Un vuelo concreto en una fecha concreta.

| Atributo | Tipo | Significado |
|---|---|---|
| id | Long | Identificador interno. |
| codigoVuelo | String | Código comercial (`IB3456`). **Se repite cada día**, así que no es único por sí solo. |
| salidaProgramada | LocalDateTime | Fecha y hora prevista de despegue. |
| llegadaProgramada | LocalDateTime | Fecha y hora prevista de aterrizaje. |
| puertaEmbarque | String | Se asigna poco antes del embarque. Es null hasta entonces. |
| estado | EstadoVuelo (enum) | Situación operativa del vuelo. |

### Tripulante → tabla `tripulantes`

| Atributo | Tipo | Significado |
|---|---|---|
| id | Long | Identificador interno. |
| codigoEmpleado | String | Código interno de la plantilla. Es único. |
| nombre, apellidos | String | Nombre completo. |
| puesto | Puesto (enum) | `COMANDANTE`, `COPILOTO`, `SOBRECARGO` o `AUXILIAR`. |

### Pasajero → tabla `pasajeros`

| Atributo | Tipo | Significado |
|---|---|---|
| id | Long | Identificador interno. |
| numPasaporte | String | Documento de viaje. Es único. |
| nombre, apellidos | String | Tal y como figuran en el documento. |
| nacionalidad | String | País emisor del documento. |
| fechaNacimiento | LocalDate | Para distinguir bebés, niños y adultos. |
| email | String | Opcional, para avisos. |

### Reserva → tabla `reservas`
La compra que hace un cliente. Agrupa billetes.

| Atributo | Tipo | Significado |
|---|---|---|
| id | Long | Identificador interno. |
| localizador | String | Código de 6 caracteres ("K9QX2M"). Es único. |
| fechaReserva | LocalDateTime | Momento de la compra. |
| emailContacto | String | A dónde se envía la confirmación. |
| importeTotal | BigDecimal | Suma de los precios de sus billetes. |

### Billete → tabla `billetes`
Une **un pasajero** con **un vuelo** dentro de **una reserva**.

| Atributo | Tipo | Significado |
|---|---|---|
| id | Long | Identificador interno. |
| asiento | String | "12C". Es null hasta que se elige o se asigna en el check-in. |
| clase | ClaseBillete (enum) | `TURISTA`, `PREMIUM` o `BUSINESS`. |
| precio | BigDecimal | Lo que se pagó por este tramo. |
| facturado | boolean | Si ha hecho el check-in. |
| embarcado | boolean | Si ha pasado por la puerta de embarque. |

### Equipaje → tabla `equipajes`

| Atributo | Tipo | Significado |
|---|---|---|
| id | Long | Identificador interno. |
| etiqueta | String | Código de barras de la etiqueta de la maleta. Es único. |
| pesoKg | Double | Peso medido en el mostrador. |
| tipo | TipoEquipaje (enum) | `CABINA`, `BODEGA` o `ESPECIAL` (bicicleta, instrumento…). |

## Relaciones

| Relación | Cardinalidad | Anotación JPA sugerida | Lado propietario (FK) |
|---|---|---|---|
| ModeloAvion – Avion | 1 : N | `@ManyToOne` en Avion | Avion (`modelo_id`) |
| Vuelo – Avion | N : 1 | `@ManyToOne` | Vuelo (`avion_id`) |
| Vuelo – Aeropuerto (**origen**) | N : 1 | `@ManyToOne @JoinColumn(name="origen_id")` | Vuelo |
| Vuelo – Aeropuerto (**destino**) | N : 1 | `@ManyToOne @JoinColumn(name="destino_id")` | Vuelo |
| Vuelo – Tripulante | N : M | `@ManyToMany @JoinTable(name="vuelo_tripulacion")` | Vuelo |
| Reserva – Billete | 1 : N composición | `@OneToMany(mappedBy, cascade=ALL, orphanRemoval=true)` | Billete (`reserva_id`) |
| Billete – Pasajero | N : 1 | `@ManyToOne(optional=false)` | Billete (`pasajero_id`) |
| Billete – Vuelo | N : 1 | `@ManyToOne(optional=false)` | Billete (`vuelo_id`) |
| Billete – Equipaje | 1 : N composición | `@OneToMany(cascade=ALL, orphanRemoval=true)` | Equipaje (`billete_id`) |

> **Fíjate:**
> - **Relación ternaria:** `Billete` relaciona tres entidades a la vez (Reserva, Pasajero y Vuelo). En JPA no existe una anotación "ternaria": se resuelve con **una entidad y tres `@ManyToOne`**.
> - **Varias restricciones únicas en la misma tabla:** un asiento no puede venderse dos veces en un vuelo, y un pasajero no puede tener dos billetes del mismo vuelo. Se declaran dos `@UniqueConstraint`. Ojo: varias filas con `asiento = NULL` no violan la restricción (en SQL, `NULL` no es igual a `NULL`).
> - **Origen y destino** son dos FK hacia `aeropuertos`. No olvides validar que no sean el mismo aeropuerto.
> - Vuelo–Tripulante es un `@ManyToMany` **sin atributos**, porque el puesto pertenece al tripulante. Si el mismo tripulante pudiera ser sobrecargo en un vuelo y auxiliar en otro, ¿cómo cambiarías el modelo?

## Conceptos JPA que se practican

- Relación ternaria mediante una entidad
- Varias `@ManyToOne` hacia la misma tabla (`@JoinColumn` con nombres distintos)
- Varias `@UniqueConstraint`, una de ellas con columnas que admiten null
- Composiciones encadenadas (Reserva → Billete → Equipaje)
- Consultas con varios JOIN y alias de la misma entidad

## Ejercicios propuestos

1. Implementa el modelo con todas las restricciones únicas indicadas en las notas del diagrama.
2. Consulta JPQL: vuelos de mañana entre dos ciudades (hace falta usar dos alias de Aeropuerto: `JOIN v.origen o JOIN v.destino d`).
3. Método `asignarAsiento(billete, "14F")`: valida que el asiento existe en el modelo de avión y que está libre.
4. Consulta JPQL: porcentaje de ocupación de cada vuelo (`billetes vendidos / (numFilas × asientosPorFila)`).
5. Consulta JPQL: pasajeros que han volado más de 5 veces a un destino concreto en el último año.
6. Valida que todo vuelo tenga como mínimo un comandante y un copiloto antes de pasar a `EMBARCANDO`.
7. Consulta JPQL: reservas con escala, es decir, con dos billetes del mismo pasajero cuyo destino del primero es el origen del segundo y con menos de 24 h entre ellos.
8. **Ampliación:** cambia Vuelo–Tripulante a una entidad `AsignacionTripulacion` con la función que desempeña el tripulante en ese vuelo.
