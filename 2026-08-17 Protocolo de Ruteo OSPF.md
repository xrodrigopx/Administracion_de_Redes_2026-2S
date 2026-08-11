# 17-08-2026 — Protocolo de Ruteo OSPF

> **Nota:** este es un borrador de la clase en curso — todavía no terminamos de desarrollarlo (nos quedó pendiente cerrar el ejemplo del paquete Hello). Se retoma y se completa en la próxima clase.

## 1. OSPF en una frase

OSPF también es un **IGP** (corre dentro de un mismo sistema autónomo, [[igual que RIP]]), pero cambia completamente el enfoque: en vez de **vector distancia** (donde cada router solo sabe "a qué distancia" está un destino, según le contó un vecino), OSPF es **estado de enlace (link-state)**: cada router informa **el estado de sus propios links directamente conectados** a *todos* los demás routers del área, y con toda esa información cada uno arma, por su cuenta, el mapa completo de la red.

Es la diferencia entre "che, para llegar ahí tenés que ir 3 saltos" (RIP) y "yo estoy conectado a estos routers, por estos links, con estas condiciones — armate vos el mapa" (OSPF).

## 2. Cómo baja la carga de la red

- Los updates son **incrementales**: no se manda la base completa cada vez, solo lo que cambió.
- Se envían a una dirección **multicast 224.0.0.5**, y son **triggered** (ante un cambio, no hay que esperar un timer para avisar).
- Cada tanto igual hay un **full update**, que sirve para resincronizar todo por las dudas algo se haya perdido o desincronizado.

Esto es justo lo contrario de RIP, que manda la tabla completa entera cada 30 segundos exista o no un cambio — por eso OSPF escala mucho mejor.

## 3. Escalabilidad

OSPF soporta redes bastante más grandes que RIP. Como referencia (valores recomendados por Cisco, no un límite duro del protocolo):

- Hasta ~50 routers por área.
- Hasta ~60 vecinos por router.

## 4. El algoritmo: Dijkstra (SPF)

OSPF usa el algoritmo **Shortest Path First (SPF)**, que es el algoritmo de **Dijkstra** aplicado a la topología de red. Con la base de datos de estado de enlace, cada router calcula **su propio árbol de caminos más cortos** hacia todos los destinos, garantizando una topología **libre de loops**.

## 5. La métrica: el costo

La métrica de OSPF es el **costo**, y se calcula así:

$$\text{costo} = \frac{10^8}{\text{ancho de banda del enlace (bps)}}$$

Donde el ancho de banda es el que está **configurado en la interfaz del router** (no necesariamente el real). Por ejemplo, si la interfaz es de 10 Mbps, el costo queda en 10 (10^8 / 10^7 = 10).

### ¿Qué pasa con interfaces de más de 100 Mbps?

Acá está el problema con la fórmula: $10^8$ = 100.000.000, que corresponde a 100 Mbps. Si tengo una interfaz de 1 Gbps o 10 Gbps, el resultado da menor a 1, y OSPF **redondea el costo mínimo a 1** — con lo cual una interfaz de 100 Mbps, una de 1 Gbps y una de 10 Gbps terminan teniendo **el mismo costo (1)**, y el protocolo pierde la capacidad de diferenciarlas.

La solución es cambiar el **ancho de banda de referencia** (*reference bandwidth*) que usa la fórmula — en vez de $10^8$, se configura un valor más alto (por ejemplo $10^{10}$ para poder diferenciar hasta 10 Gbps). Es una configuración manual que hay que aplicar **de forma consistente en todos los routers del dominio OSPF**, porque si cada uno usa una referencia distinta, los costos dejan de ser comparables entre sí y el cálculo de mejor camino se rompe.

### Entonces, ¿gana el camino con más ancho de banda?

No necesariamente el que tiene un solo enlace más rápido, sino el de **menor costo acumulado** a lo largo de todo el camino (SPF suma los costos de cada link en el camino). Por eso puede pasar algo un poco contra-intuitivo: un camino **más largo** (más saltos) pero compuesto por enlaces de mayor ancho de banda puede terminar teniendo **menor costo total** que un camino más corto con enlaces lentos, y en ese caso OSPF va a preferir el camino largo-pero-rápido. La fórmula está pensada justamente para **priorizar el ancho de banda por sobre la cantidad de saltos**, al revés de lo que hace RIP.

## 6. En la práctica: la base de datos de estado de enlace

Cada router mantiene una **base de datos (LSDB - Link State Database)** con la topología completa de la red: describe todos los routers y todos los links del área. Todos los routers de una misma área tienen, en teoría, **exactamente la misma base de datos** — es a partir de ahí que cada uno corre Dijkstra de forma local para armar su propia tabla de ruteo.

## 7. Definiciones básicas

- **Router ID**: identifica al router dentro del dominio OSPF. Por defecto se usa la IP más alta configurada (o depende del fabricante); si existe una interfaz **loopback**, se prefiere esa IP por sobre las físicas (porque una loopback nunca se cae, entonces el Router ID es estable).
- **Link**: una conexión entre dos routers, con ciertos atributos asociados (costo, tipo, etc.).
- **Router vecino (neighbor)**: un router que tiene una interfaz en el **mismo segmento de red** que el propio router. Los vecinos se descubren enviando **multicast**, y a partir de ahí se establece una relación de vecindad. En una LAN puede haber varios routers que son todos vecinos entre sí, simplemente porque comparten el mismo segmento.
- **Área OSPF**: un conjunto de routers de la red que corren la misma instancia del protocolo OSPF. Sirve para **segmentar la red** y, sobre todo, para **limitar el flooding** de información (cada área tiene su propia LSDB, no hace falta que toda la red conozca el detalle de todas las demás áreas).
  - Aclaración importante: **broadcast** y **flooding** no son lo mismo. Broadcast es un mensaje que mando una vez para que lo reciban todos en el segmento. Flooding es un mensaje que cada router que lo recibe **lo retransmite** a sus otros vecinos, para que se propague por toda el área.
- **Router interno**: un router donde **todas** sus interfaces pertenecen a una misma área.
- **ABR (Area Border Router)**: un router que pertenece a **más de un área** — hace de frontera entre áreas.
- **ASBR (AS Boundary Router)**: un router que intercambia información de ruteo con **otro sistema autónomo** (por ejemplo, redistribuyendo rutas de RIP o de BGP hacia OSPF).

## 8. LSAs (Link State Advertisement)

Un **LSA** es una porción de la base de datos de estado de enlace: representa la información de un link puntual. Cuando hay un cambio de topología, los routers se intercambian LSAs referidos a ese link específico — eso es lo que decíamos antes, el **update incremental**.

Hay varios tipos de LSA, cada uno con un rol distinto:

| Tipo | Nombre | Quién lo genera | Qué describe |
|---|---|---|---|
| 1 | Router | Cada router, uno por router | Sus propios links directamente conectados |
| 2 | Network | El **DR** (router designado) de una red multiacceso | Todos los routers vecinos conectados a esa red donde él es el designado |
| 3 | Network Summary | Los **ABR** | Sumariza las redes IP de un área hacia otra área |
| 4 | ASBR Summary | Los **ABR** | Informa cómo llegar hasta el ASBR (se envía del ABR hacia donde está el ASBR) |
| 5 | AS External | Los **ASBR** | Redes que están en **otro sistema autónomo** |
| 7 | NSSA External | Los **ASBR** dentro de un área NSSA | Rutas externas, pero solo dentro de esa área (ver más abajo) |

## 9. Áreas OSPF

OSPF puede usarse en una configuración de **una sola área** o **multi-área**.

- Si uso una única área, se la llama **área de backbone o área 0**.
- Si agrego más áreas, **todas tienen que conectarse al backbone** (directa o indirectamente) — el área 0 es el punto de tránsito obligado entre áreas.

Tipos de área:

- **Área estándar**: todos los routers conocen las redes del área y comparten la misma base topológica (LSDB) completa.
- **Área stub**: los routers conocen las redes del área y las de otras áreas del sistema autónomo, pero **no aceptan LSAs de tipo 5** (rutas externas al AS). Como no tienen esa información detallada, para salir hacia otros sistemas autónomos usan una **ruta por defecto**.
- **Totally stubby area**: un paso más restrictivo — tampoco aceptan LSAs de **tipo 3**. O sea que para salir del área, ya sea hacia otras áreas internas o hacia otros sistemas autónomos, usan una **ruta por defecto** en ambos casos.
- **NSSA (Not-So-Stubby Area)**: se puede pensar como una variante de área stub. Las rutas externas **no se propagan** hacia adentro ni hacia afuera del área (LSAs tipo 4 y tipo 5 no están permitidos), **pero sí** se pueden distribuir, dentro del área, redes aprendidas por otros protocolos — por ejemplo, un router que también habla RIP, o que tiene una relación BGP con otro sistema autónomo. Para eso existe el **LSA tipo 7**: propaga esas rutas *dentro* del área NSSA, pero no hacia otras áreas. Si esa información necesita propagarse a otras áreas, el **ABR** de esa área NSSA la convierte de tipo 7 a **tipo 5** antes de mandarla para afuera.
- **Área 0 / backbone**: se conecta contra todas las demás áreas y propaga todos los tipos de LSA, **excepto el tipo 7**, que justamente se traduce a tipo 5 antes de entrar al backbone (por eso el tipo 7 nunca "cruza" hacia el área 0 tal cual).

## 10. El paquete OSPF: encabezado común

*(pendiente de desarrollar en clase — dejo la estructura general, RFC 2328)*

Todos los paquetes OSPF (sea cual sea su tipo) comparten un encabezado común de 24 bytes:

```
┌──────────────┬──────────────┬────────────────────────┐
│ Version (1B) │  Type (1B)   │   Packet Length (2B)    │
├──────────────┴──────────────┴────────────────────────┤
│                    Router ID (4B)                      │
├─────────────────────────────────────────────────────────┤
│                     Area ID (4B)                        │
├──────────────┬──────────────┬────────────────────────┤
│ Checksum (2B)│ AuType (2B)  │                          │
├─────────────────────────────────────────────────────────┤
│              Authentication (8B, según AuType)           │
└─────────────────────────────────────────────────────────┘
```

- **Version**: versión de OSPF (2 = OSPFv2, para IPv4).
- **Type**: qué tipo de paquete es — 1: Hello, 2: Database Description, 3: Link State Request, 4: Link State Update, 5: Link State Acknowledgment.
- **Packet Length**: longitud total del paquete OSPF, en bytes.
- **Router ID**: el ID del router que originó el paquete.
- **Area ID**: el área a la que pertenece la interfaz por la que se envía.
- **Checksum**: para detectar errores en el paquete.
- **AuType**: tipo de autenticación (0 = ninguna, 1 = password simple, 2 = criptográfica).
- **Authentication**: los datos de autenticación en sí, según lo que indique AuType.

## 11. Funcionamiento del protocolo, paso a paso

Cuando conecto un router nuevo a una topología, lo que aparece del otro lado es, ni más ni menos, un **vecino nuevo**. La secuencia es:

1. **Descubrimiento del vecino**: se manda un paquete **Hello** (ver más abajo) para encontrar routers vecinos en el segmento.
2. **Intercambio de información / sincronización**: una vez reconocidos como vecinos, se sincronizan las bases de datos de estado de enlace.
3. **Cálculo del nuevo grafo**: con la LSDB actualizada, corre Dijkstra (SPF) de nuevo.
4. **Actualización de la tabla de ruteo**, en base a las mejores rutas que salieron del cálculo.
5. **Flooding de los LSAs nuevos** hacia el resto de los vecinos, para que todos terminen sincronizados.

### El paquete Hello

Para descubrir vecinos, cada router manda periódicamente unos paquetes pequeños llamados **Hello**. Además de servir para el descubrimiento inicial, estos paquetes se siguen mandando después para **mantener viva la relación de vecindad** (si dejo de recibir Hellos de un vecino, eventualmente asumo que se cayó).

- El **Hello timer** es configurable, aunque los equipos vienen con un valor por defecto (típicamente 10 segundos en redes broadcast/punto a punto).
- El paquete Hello **no se reenvía fuera del segmento de red** — es estrictamente local, a diferencia de los LSAs que sí se floodean.
- Con los Hellos recibidos se arma una **tabla de vecinos**.

*(pendiente de desarrollar en clase)* — estructura del paquete Hello, más allá del encabezado común de OSPF, incluye entre otros estos campos:

| Campo | Qué indica |
|---|---|
| Network Mask | Máscara de red de la interfaz por la que se manda |
| HelloInterval | Cada cuánto se mandan los Hello (tiene que coincidir entre vecinos) |
| Options | Capacidades que soporta el router (por ejemplo, si maneja áreas stub) |
| Router Priority | Usado para la elección de DR/BDR en redes multiacceso |
| RouterDeadInterval | Tiempo sin recibir Hello de un vecino antes de darlo por caído (normalmente 4x el HelloInterval) |
| Designated Router | IP del DR conocido en ese segmento |
| Backup Designated Router | IP del BDR conocido en ese segmento |
| Lista de vecinos | Router IDs de los vecinos de los que recibió Hello recientemente |

---

*Borrador incompleto — seguimos la próxima clase con el detalle del paquete Hello y lo que quede pendiente.*
