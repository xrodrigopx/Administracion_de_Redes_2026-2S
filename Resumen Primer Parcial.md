# Resumen para el Primer Parcial — Administración de Redes 2026-2S

Cubre las clases del 03-08 al 07-09: repaso, RIP, OSPF, Laboratorio 1 (OSPF/BGP/NAT) y QoS.
Cada tema tiene: **resumen → ejemplos → preguntas**. Las respuestas están en bloques desplegables al final de cada tema: intentá responder antes de abrirlas.

## Índice

1. [Repaso: cómo funciona el ruteo](#1-repaso-cómo-funciona-el-ruteo)
2. [RIP](#2-rip)
3. [OSPF](#3-ospf)
4. [Laboratorio 1: OSPF multi-área + BGP + NAT](#4-laboratorio-1-ospf-multi-área--bgp--nat)
5. [QoS](#5-qos)
6. [Tabla de datos para memorizar](#6-tabla-de-datos-para-memorizar)

---

## 1. Repaso: cómo funciona el ruteo

### Resumen

- El ruteo vive en la **capa de red (3)**. Cada router decide **de forma independiente** mirando solo su tabla; nadie tiene el mapa completo.
- Tabla de ruteo = entradas "para llegar a la red X, next hop Y por la interfaz Z". Puede haber **default gateway** (0.0.0.0/0) para "todo lo que no conozco".
- **Cómo reenvía un router**: (1) AND bit a bit entre IP destino y la máscara de cada entrada; (2) si varias matchean, gana la **más específica (máscara más larga)**; (3) obtiene interfaz y next hop; (4) usa **ARP** para conseguir la MAC del next hop y arma la trama Ethernet. Para entregar al siguiente salto hay que compartir un **medio físico/LAN**: a nivel de enlace se entrega por MAC, no por IP.
- **Ruteo estático vs. dinámico**: a mano no escala; un **protocolo de ruteo** hace que los routers intercambien información y las tablas se armen y actualicen solas (reaccionan a cambios de topología).
- **Sistema Autónomo (AS)**: conjunto de routers bajo la misma administración y políticas de ruteo.

| | **IGP** | **EGP** |
|---|---|---|
| Dónde corre | Dentro de un AS | Entre ASes |
| Decide por | **Métrica** (número calculado por el protocolo) | **Política** (yo elijo qué acepto/anuncio) |
| Ejemplos | RIP, OSPF | BGP |
| Escalabilidad | Peor (reacciona a todo cambio) | Mejor (ignora cambios chicos) |

- **Distancia administrativa (AD)**: "confianza" en la fuente de la ruta; **no es la métrica**. Sirve para elegir **entre protocolos distintos** para la misma red. Menor = mejor: conectada 0, estática 1, **OSPF 110, RIP 120**. Si es el **mismo protocolo**, se compara la métrica propia.

### Ejemplo

El router tiene estas entradas y llega un paquete a 10.1.2.77:

| Red | Next hop |
|---|---|
| 10.0.0.0/8 | R-A |
| 10.1.0.0/16 | R-B |
| 10.1.2.64/26 | R-C |
| 0.0.0.0/0 | R-D |

Las tres primeras matchean; gana **10.1.2.64/26** (máscara más larga) → sale por R-C. Si la destino fuera 8.8.8.8, solo matchea la default → R-D.

### Preguntas

1. ¿Por qué un router necesita ARP para reenviar un paquete si ya tiene el next hop en la tabla?
2. Dos entradas matchean el mismo destino: 172.16.0.0/16 y 172.16.5.0/24. ¿Cuál se usa y por qué?
3. OSPF y RIP anuncian la misma red. ¿Cuál entra a la tabla y por qué, aunque RIP diga "1 salto"?
4. Diferenciá IGP y EGP con dos ejemplos y explicá por qué el EGP escala mejor.
5. ¿Qué es un AS y por qué existe el concepto?

<details><summary>Respuestas</summary>

1. La tabla da la **IP** del next hop, pero la trama Ethernet necesita su **MAC** de destino. ARP resuelve IP→MAC dentro del segmento compartido.
2. La **/24** (más específica, prefijo más largo), si el destino está dentro de esa /24.
3. **OSPF (AD 110 < 120)**. La AD se compara primero y es independiente de la métrica interna de cada protocolo (saltos vs. costo).
4. IGP dentro de un AS (RIP, OSPF) decide por métrica. EGP entre ASes (BGP) decide por política. Escala mejor porque no recalcula por cambios pequeños internos y permite aplicar criterios administrativos.
5. Conjunto de routers bajo una misma administración y política de ruteo. Permite que cada organización elija libremente su IGP y defina políticas hacia afuera.

</details>

---

## 2. RIP

### Resumen

- **Vector distancia**: ningún nodo conoce la topología; solo sabe "a la red X llego por el vecino Y a N saltos". Métrica = **cantidad de saltos**.
- **Algoritmo**:
  1. Cada **30 s** manda **toda su tabla por todas las interfaces**, haya o no cambios ni vecinos escuchando.
  2. Al anunciar una red, **suma 1 salto** a la métrica.
  3. Al recibir: si es **mejor** (menos saltos) → actualiza; si viene del **mismo vecino** que ya usaba pero **peor** → acepta el empeoramiento; si no → descarta.
- **Límite**: máximo **15 saltos**; **16 = infinito** (inalcanzable).
- **Mensaje**: sobre **UDP puerto 520** (IP debe estar funcionando antes). Multicast **224.0.0.9** en v2 (broadcast en v1). Hasta **25 entradas** por mensaje. Cada entrada: familia, route tag, IP red, máscara, next hop, métrica. Comando: request/response. Especificado en **RFC 2453**.
- **Timers**:
  - **Update ~30 s**: envío periódico.
  - **Invalid/timeout ~180 s**: sin update de una red → se marca inalcanzable (16).
  - **Flush ~240 s**: se borra la entrada.
- **Anti-loop**:
  - **Split horizon**: no anuncio a un vecino, por la misma interfaz, una ruta que aprendí de él.
  - **Poison reverse**: la anuncio igual pero con métrica **16**.
  - **Triggered updates**: ante un cambio, update inmediato sin esperar los 30 s (acelera convergencia).
- **Ventajas**: simple, poco consumo, compatible con todo. **Desventajas**: 15 saltos, convergencia lenta ("cuenta hasta infinito"), métrica solo saltos (ignora ancho de banda/latencia), manda tabla completa siempre.

| | RIPv1 | RIPv2 | RIPng |
|---|---|---|---|
| Direccionamiento | Classful (sin máscara) | Classless (VLSM/CIDR) | IPv6 |
| Envío | Broadcast 255.255.255.255 | Multicast 224.0.0.9 | Multicast IPv6 |
| Autenticación | No | Opcional | — |

### Ejemplo resuelto (Ejercicio 1 de clase)

Router A: E0 → 10.0.4.0/24, E1 → 10.0.3.0/24. Router B: E0 → 10.0.3.0/24 (mismo segmento que A-E1), E1 → 10.0.1.0/24, E2 → 10.0.2.0/24. Convención: red conectada = métrica 0.

- A anuncia a B: (10.0.4.0, 0) y (10.0.3.0, 0). B suma 1: 10.0.4.0 → 1 vía A (nueva, la agrega); 10.0.3.0 → 1 pero ya la tiene con 0 → descarta.
- B anuncia a A: 10.0.1.0 y 10.0.2.0 llegan con 1 → A las agrega vía B.
- **Tabla convergida de A**: 10.0.4.0 (0, directa E0), 10.0.3.0 (0, directa E1), 10.0.1.0 (1, vía B), 10.0.2.0 (1, vía B).
- **En régimen**: siguen los updates de 30 s. Por **split horizon**, A no le reanuncia a B por E1 lo que aprendió de B (10.0.1.0 y 10.0.2.0).

### Ejemplo resuelto (Ejercicio 2)

Se agrega un cable entre B-E3 y la LAN 10.0.4.0/24 de A.

- B ahora tiene 10.0.4.0 **directa con métrica 0**, mejor que "1 vía A" → la reemplaza y dispara **triggered update**.
- A queda con **dos rutas de igual costo (1)** hacia 10.0.1.0 y 10.0.2.0 (por E1 y por E0).
- Camino de 10.0.1.0/24 → 10.0.4.0/24: antes **2 saltos** (B y A); ahora **1 salto** (solo B).

### Preguntas

1. ¿Por qué RIP suma 1 al anunciar una ruta a un vecino?
2. Un router con 60 redes en su tabla: ¿cuántos mensajes RIP manda por update y por qué?
3. Una red deja de existir. Sin triggered updates, ¿cuánto tarda el router en marcarla inalcanzable y cuánto en borrarla?
4. Explicá con un ejemplo de dos routers qué loop evita el split horizon. ¿En qué se diferencia el poison reverse?
5. ¿Por qué RIP puede elegir un camino de 3 saltos por enlaces lentos en lugar de uno de 4 por fibra?
6. ¿Qué significa métrica 16? ¿Qué implica para el tamaño máximo de la red?
7. Diferencias entre RIPv1, RIPv2 y RIPng (mínimo 3 aspectos).
8. Recibís de un vecino "red X con 3 saltos" y ya tenías "red X con 5 saltos vía otro vecino". ¿Qué hacés? ¿Y si el que empeora es el mismo vecino que ya usabas?
9. ¿Por qué RIP es ineficiente en redes grandes, aunque nada cambie?

<details><summary>Respuestas</summary>

1. Porque para el vecino, a través de mí, la red queda un salto más lejos (él tiene que pasar además por mí).
2. **3 mensajes** (25 + 25 + 10): el máximo es 25 entradas por mensaje.
3. ~**180 s** para marcarla como inalcanzable (16) y ~**240 s** para flush (borrado de la tabla).
4. A aprende la red X de B y se la devuelve a B; si X cae, B podría creer que A tiene un camino alternativo, cuando ese camino pasa por B mismo → loop. Split horizon no la reanuncia; **poison reverse** sí la anuncia pero con 16, dejando explícito que no es alcanzable por ahí.
5. Porque la métrica es **solo cantidad de saltos**; no considera ancho de banda ni latencia.
6. Infinito/inalcanzable. La red no puede tener más de **15 saltos** de diámetro.
7. v1: classful, broadcast, sin autenticación. v2: classless (manda máscara), multicast 224.0.0.9, autenticación opcional, next hop y route tags. ng: IPv6, multicast IPv6, mismo vector distancia y límite 15.
8. Se queda con la de **3 saltos** (mejor). Si el que empeora es el mismo vecino que ya usaba como next hop, **acepta el empeoramiento** y aumenta la métrica.
9. Manda la **tabla completa cada 30 s** por todas las interfaces siempre, generando mucho tráfico de control y convergencia lenta.

</details>

---

## 3. OSPF

### Resumen

- **IGP de estado de enlace (link-state)**: cada router informa el estado de **sus links directamente conectados** a todos los del área, y cada uno arma **el grafo completo** de la red. Luego corre **Dijkstra (SPF)** con él mismo como raíz → árbol de caminos más cortos, libre de loops.
- **Menos carga que RIP**: updates **incrementales** y **triggered**, por multicast **224.0.0.5**, más un full update periódico de resincronización. Escala hasta ~50 routers/área, ~60 vecinos/router (recomendación Cisco).
- **Métrica = costo** = `10^8 / ancho de banda (bps)`, con el BW **configurado** en la interfaz (`bandwidth <kbps>`) o fijado directo con `ip ospf cost`.
  - Costo mínimo **1**: 100 Mbps, 1 Gbps y 10 Gbps dan todos 1. Solución: cambiar **reference bandwidth** (`auto-cost reference-bandwidth <Mbps>`), **igual en todos los routers**.
  - Gana el **menor costo acumulado**, no el de menos saltos (al revés que RIP).
- **LSDB**: base con la topología completa del área; es **igual en todos los routers del área**.

### Definiciones

- **Router ID (RID)**, 32 bits. Orden: (1) `router-id` manual, (2) IP más alta de **loopback**, (3) IP más alta de interfaz física activa. Se prefiere loopback porque es **virtual**: no se cae por problemas físicos → RID estable. El RID se fija al **iniciar el proceso**, no cambia con solo un `clear ip ospf process` en algunos casos.
- **Vecino**: router con interfaz en el mismo segmento; se descubre por multicast.
- **Área**: conjunto de routers con la misma instancia; **limita el flooding** (cada área tiene su LSDB).
- **Broadcast ≠ flooding**: broadcast se manda una vez; en flooding cada receptor **retransmite** a sus vecinos.
- **Router interno**: todas sus interfaces en la misma área. **ABR**: pertenece a varias áreas, frontera entre ellas. **ASBR**: intercambia rutas con otro AS (ej.: redistribuye BGP/RIP en OSPF).
- **DR/BDR**: router designado / de respaldo en redes multiacceso.

### Tipos de LSA

| Tipo | Nombre | Lo genera | Qué describe | En la tabla |
|---|---|---|---|---|
| 1 | Router | Cada router | Sus links directos, estado y costo | `O` |
| 2 | Network | **DR** | Routers conectados a esa red multiacceso | `O` |
| 3 | Network Summary | **ABR** | Redes de un área hacia otra | `O IA` |
| 4 | ASBR Summary | **ABR** | Cómo llegar al ASBR | `O IA` |
| 5 | AS External | **ASBR** | Redes de otro AS | `O E1` / `O E2` |
| 7 | NSSA External | ASBR en NSSA | Externas solo dentro del NSSA | — |

- Header de LSA: **LS Age**, **LS Type**, **Link State ID** (tipo 1: RID del originador; tipo 2: IP del DR; tipo 3 y 5: IP de la red destino; tipo 4: RID del ASBR), y **número de secuencia** (mayor = más nuevo).
- **E1** suma el costo interno hasta el ASBR al costo externo; **E2** (por defecto) mantiene solo el costo externo fijo.

### Áreas

- Un área única = **área 0 / backbone**. Si hay varias, **todas deben conectarse al área 0** (directa o indirectamente).

| Tipo de área | LSA 1 | LSA 2 | LSA 3 | LSA 4 | LSA 5 | LSA 7 |
|---|---|---|---|---|---|---|
| Backbone / no-backbone | Sí | Sí | Sí | Sí | Sí | No |
| **Stub** | Sí | Sí | Sí | No | No | No |
| **Totally stubby** | Sí | Sí | **No** | No | No | No |
| **NSSA** | Sí | Sí | Sí | No | No | **Sí** |

- **Stub**: sin externas (5) → usa **ruta por defecto** para salir del AS. **Totally stubby**: además sin tipo 3 → default para todo lo que esté fuera del área. **NSSA**: stub que permite redistribuir externas dentro del área con **LSA 7**; el **ABR lo convierte a tipo 5** al salir.
- **Entre áreas**: los tipo 1 y 2 al cruzar un ABR se **traducen a tipo 3**. Orden de preferencia: rutas **intra-área** > **inter-área** (3/4) > **externas** (5).

### Paquetes OSPF (encabezado común de 24 B: versión, tipo, longitud, Router ID, Area ID, checksum, AuType, autenticación)

| Tipo | Nombre | Función |
|---|---|---|
| 1 | **Hello** | Descubrir/mantener vecinos y elegir DR/BDR |
| 2 | **DBD** | Resumen (solo headers) de la LSDB |
| 3 | **LSR** | Pedir LSAs que faltan o están viejos |
| 4 | **LSU** | Enviar LSAs completos |
| 5 | **LSAck** | Confirmar recepción |

- **Hello**: HelloInterval (10 s típico), **RouterDeadInterval = 4× Hello** (40 s), prioridad, DR/BDR, lista de vecinos, máscara. **HelloInterval y dead deben coincidir entre vecinos.** Es local (no se reenvía fuera del segmento) por 224.0.0.5.
- **DBD**: bits **I** (primero), **M** (vienen más), **MS** (master/slave) + número de secuencia.

### Secuencia al aparecer un vecino nuevo

1. **Hello** (descubrimiento) → 2. **DBD** (intercambio de headers) → **LSR / LSU / LSAck** (se pide, envía y confirma lo que falta) → 3. **Dijkstra** → 4. **Tabla de ruteo** → 5. **Flooding** de los LSAs nuevos al resto.

### En régimen

- Hellos periódicos; si no llegan por **RouterDeadInterval** → vecino caído.
- Ante un cambio: el router recalcula, **incrementa el número de secuencia** del LSA y hace flooding.
- Al recibir un LSA: **nuevo o más nuevo** → ACK, recalcula, flooding. **Más viejo** → ACK, lo descarta y le manda al vecino **su versión más nueva**.
- **LSRefreshTime = 30 min** (el originador reenvía el LSA aunque no cambie). **MaxAge = 1 h** (se elimina de la LSDB).

### DR y BDR

- Sin DR, en una LAN con N routers habría **full mesh** de adyacencias (N(N−1)/2) y flooding redundante. Con DR/BDR, cada router forma adyacencia **solo con DR y BDR**.
- Elección por **Router Priority** (mayor gana; **0 = nunca DR**), desempate por **Router ID más alto**. Multicast **224.0.0.6** (AllDRouters) para hablar con ellos. El DR genera el **LSA tipo 2**.

### Ejemplos

**Costo y camino.** Referencia por defecto 100 Mbps. Camino 1: R1→R2→R3, dos enlaces de 100 Mbps: costo 1+1 = **2**. Camino 2: R1→R3 directo por un enlace de 10 Mbps: costo 10. **Gana el camino 1 (más largo pero rápido).** En RIP habría ganado el directo (1 salto).

**Referencia.** Enlaces de 1 Gbps y 10 Gbps con referencia 100 Mbps → ambos costo 1 (no se distinguen). Con `auto-cost reference-bandwidth 10000`: 1 Gbps = 10, 10 Gbps = 1.

**Elección de DR.** Routers en una LAN: R1 (prio 1, RID 1.1.1.1), R2 (prio 1, RID 2.2.2.2), R3 (prio 100, RID 0.0.0.3), R4 (prio 0). DR = **R3** (prioridad más alta); BDR = **R2** (empate de prioridad 1 con R1, desempata RID más alto); R4 **nunca** puede serlo.

**Configuración básica.**
```
router ospf 1
 router-id 1.1.1.1
 network 192.168.1.0 0.0.0.255 area 0
 network 10.0.0.0 0.0.0.3 area 0
 passive-interface GigabitEthernet0/1
```
La wildcard es la máscara invertida (/24 → 0.0.0.255, /30 → 0.0.0.3). El ID de proceso es **local**: no tiene que coincidir entre routers.

### Preguntas

1. Diferencia fundamental entre vector distancia y estado de enlace, con una frase para cada uno.
2. ¿Por qué OSPF escala mejor que RIP? Nombrá al menos tres razones.
3. Calculá el costo OSPF de un enlace de 10 Mbps, 100 Mbps y 1 Gbps con referencia por defecto. ¿Cuál es el problema y cómo se soluciona?
4. ¿Por qué es peligroso cambiar la reference bandwidth solo en un router?
5. ¿Cómo se elige el Router ID? ¿Por qué se recomienda una loopback?
6. Diferencia entre broadcast y flooding.
7. Un router tiene interfaces en las áreas 0 y 1. ¿Qué rol cumple? ¿Y uno que redistribuye rutas de BGP?
8. ¿Quién genera cada LSA (1, 2, 3, 4, 5, 7) y qué describe?
9. Completá: ¿qué LSAs acepta un área stub? ¿Y una totally stubby? ¿Por qué usan ruta por defecto?
10. ¿Para qué existe un NSSA y qué pasa con un LSA 7 al llegar al ABR?
11. Ordená los 5 tipos de paquete OSPF y explicá para qué sirve cada uno en la formación de una adyacencia.
12. ¿Cuánto vale por defecto el RouterDeadInterval si el Hello es 10 s? ¿Qué pasa si dos vecinos tienen HelloInterval distinto?
13. Recibís un LSA con número de secuencia menor al que tenés. ¿Qué hacés?
14. ¿Para qué sirven LSRefreshTime y MaxAge? ¿Cuánto valen?
15. En una LAN con 5 routers, ¿cuántas adyacencias habría sin DR? ¿Y con DR/BDR?
16. ¿Cómo se elige el DR? ¿Qué hace la prioridad 0?
17. En un área con el orden de preferencia intra-área / inter-área / externa, ¿qué ruta gana si hay tres caminos a la misma red, uno de cada tipo?
18. ¿Qué diferencia hay entre `O`, `O IA`, `O E1` y `O E2`?
19. ¿Para qué sirve `passive-interface` y en qué interfaces se usa?
20. Un LSA tipo 1 (Link State ID) y uno tipo 3: ¿qué contiene ese campo en cada caso?

<details><summary>Respuestas</summary>

1. Vector distancia: "para llegar ahí tenés que ir N saltos, según me contó mi vecino". Estado de enlace: "estos son mis links y su estado; armate vos el mapa".
2. Updates **incrementales** y triggered (no tabla completa cada 30 s); **áreas** limitan el flooding; métrica por ancho de banda sin límite de 15 saltos; multicast; Dijkstra local sin loops.
3. 10 Mbps → **10**; 100 Mbps → **1**; 1 Gbps → 0,1 → redondea a **1**. No diferencia 100M/1G/10G. Se soluciona subiendo la reference bandwidth (`auto-cost reference-bandwidth`) en todos los routers.
4. Porque cada router calcularía costos con distinta referencia, los costos dejan de ser comparables y el SPF elige mal (o inconsistente).
5. (1) Manual `router-id`; (2) IP más alta de loopback; (3) IP más alta de interfaz física activa. La loopback es virtual y no se cae por problemas físicos → RID estable.
6. Broadcast: se envía una vez y lo reciben todos en el segmento. Flooding: cada router que lo recibe lo **retransmite** a sus otros vecinos para que se propague por toda el área.
7. Área 0 y 1 → **ABR**. Redistribuye BGP en OSPF → **ASBR**.
8. 1: cada router (sus links). 2: el DR (routers de la red multiacceso). 3: ABR (redes de un área hacia otra). 4: ABR (cómo llegar al ASBR). 5: ASBR (redes externas al AS). 7: ASBR en un NSSA (externas dentro de ese área).
9. Stub: 1, 2 y 3 (no 4 ni 5). Totally stubby: solo 1 y 2. Usan ruta por defecto porque no tienen el detalle de esas rutas y así se reducen LSDB y tabla.
10. Permite un área tipo stub que igual pueda **redistribuir rutas externas** (ej. otro protocolo/BGP) dentro de ella con LSA 7. El ABR lo **convierte a tipo 5** para propagarlo al resto.
11. Hello (descubrir vecinos) → DBD (resumen de headers de LSDB) → LSR (pido los que me faltan/viejos) → LSU (me mandan los LSAs completos) → LSAck (confirmo).
12. **40 s** (4×Hello). Si el HelloInterval no coincide, **no se forma la adyacencia**.
13. Lo **reconozco (ACK)**, lo **descarto** y le envío al vecino **mi versión más nueva**.
14. Evitar desincronización. **LSRefreshTime 30 min**: el originador reenvía el LSA aunque no haya cambios. **MaxAge 1 h**: si no se refrescó, se elimina de la LSDB.
15. Sin DR: 5·4/2 = **10**. Con DR/BDR: cada router con DR y BDR → **7** (3 routers × 2 + 1 entre DR y BDR).
16. Mayor **Router Priority** gana; empate → mayor **Router ID**. Prioridad **0** = el router **nunca** puede ser DR ni BDR.
17. La **intra-área**, luego inter-área y por último externa.
18. `O`: intra-área. `O IA`: inter-área (vía ABR, LSA 3/4). `O E1`/`O E2`: externas (LSA 5); E1 suma el costo interno hasta el ASBR, E2 usa solo el costo externo.
19. No manda Hellos/OSPF por esa interfaz. Se usa en **LANs de usuarios** donde no hay otros routers: se anuncia la red pero no se forman vecindades.
20. Tipo 1: el **Router ID** del router que lo originó. Tipo 3: la **IP de la red destino**.

</details>

---

## 4. Laboratorio 1: OSPF multi-área + BGP + NAT

### Topología

- **Área 0**: R1, R2, R3, R5. **Área 1**: R2–R4 (Sucursal 1). **Área 2**: R3–R6 (Sucursal 2).
- **R2 y R3 son ABR**. **R1 es ASBR** (conecta por 192.0.2.0/24 con R7, del AS 64497 = ISP). R7 sale a Internet vía nodo NAT (DHCP).
- LANs: 172.16.0.0/24 (Casa Central, R1), 172.17.0.0/24 (Suc. 1, R4), 172.18.0.0/24 (Suc. 2, R6). Enlaces entre routers: /30 en 10.0.x.x.

### Pasos y qué se aprendió

| Paso | Qué se hizo | Qué observar / por qué |
|---|---|---|
| 2 | IP en routers (`ip address`, `no shutdown`), R7 con `ip address dhcp`; en los Debnet se edita `/etc/network/interfaces` | `show ip interface brief` → todas "up/up" |
| 3 | `show ip route` antes de OSPF | Solo aparecen **C** (conectadas) y **L** (locales) |
| 4 | Ping Suc. 1 → Casa Central **antes de OSPF** | **Falla**: R4 solo conoce sus redes directas; no hay ruta a 172.16.0.0/24 |
| 5 | Configurar OSPF multi-área. Las LAN con `network` + `passive-interface`. El enlace R1-R7 **no** entra en OSPF (es el peering BGP) | R2 y R3 tienen interfaces en dos áreas |
| 6 | `show ip ospf neighbor`; loopbacks 192.168.1.X/32 | Sin loopback, el RID de R1 sería 192.0.2.1 (IP más alta activa, aunque no esté en OSPF). El RID **no cambia solo con `clear ip ospf process`**: hay que reiniciar el router |
| 7 | `show ip route` después de OSPF | Aparecen `O` (intra-área) y `O IA` (inter-área vía ABR). El ping ahora **funciona** |
| 8 | BGP entre R1 (AS 64496) y R7 (AS 64497); prefix-lists + route-maps; `default-information originate` y `redistribute bgp ... subnets` en OSPF | R1 pasa a ser ASBR → LSA tipo 5 |
| 9 | Traceroute a 8.8.8.8 antes y después del NAT en R7 | Antes: no llega (origen privado no enrutable en Internet, la respuesta no vuelve). Después: llega. NAT **overload (PAT)** traduce origen a la IP pública de R7 |
| 10 | Tabla de R1 y R4 | Nuevas: **O\*E2** (default 0.0.0.0/0, métrica 1) y **O E2** (192.168.122.0/24 desde BGP), porque son LSA 5 |
| 11 | Wireshark filtrando `ospf` + `clear ip ospf process`; `show ip ospf database` | LSA 3 (ABR R2/R3), LSA 4 (ubica al ASBR R1), LSA 5 (originado por R1) |
| 12 | Métrica de R4 a 172.18.0.0 con `show ip ospf database` y `show ip route`; luego `ip ospf cost 10` en R2-G2/0 | Al encarecer R2-R1 el tráfico prefiere R2-R5-R3 si es más barato |
| 13 | `shutdown` en R2-G2/0 y traceroute desde Debnet-1 | R2 origina un **nuevo Router-LSA** y hace flooding. El traceroute de Debnet-1 a 8.8.8.8 **no cambia** (sale por R1→R7, no pasa por R2). Lo que cambia: R2 llega a R1/R3 vía R5 |

### Comandos clave del lab

```
show ip route            show ip route ospf
show ip ospf neighbor    show ip ospf database
show ip ospf interface   show ip protocols
clear ip ospf process
interface loopback 0 / ip address 192.168.1.2 255.255.255.255
ip ospf cost 10          auto-cost reference-bandwidth 1000
access-list 100 permit ip any any
ip nat inside source list 100 interface FastEthernet 0/0 overload
ip nat outside / ip nat inside     (según la interfaz)
```

### Preguntas

1. ¿Por qué falla el ping Suc. 1 → Casa Central antes de configurar OSPF?
2. ¿Qué tipos de entradas aparecen en `show ip route` antes de OSPF? ¿Y después?
3. ¿Qué routers son ABR y cuál es ASBR en la maqueta? Justificá.
4. ¿Por qué el enlace R1–R7 no se incluye en OSPF?
5. ¿Por qué el Router ID de R1 podía ser 192.0.2.1 sin loopback? ¿Qué hay que hacer para que tome la loopback?
6. ¿Para qué se pone `passive-interface` en las interfaces LAN?
7. ¿Por qué el traceroute a 8.8.8.8 no llegaba antes de configurar NAT? ¿Qué hace `overload`?
8. ¿Qué significan las entradas `O*E2` y `O E2` en R4? ¿Qué LSA las origina?
9. ¿Qué LSAs se ven al reiniciar OSPF en Wireshark y quién genera cada uno en esta maqueta?
10. Si se fija `ip ospf cost 10` en R2-G2/0, ¿cómo cambia el camino de R4 a 172.18.0.0/24? ¿Qué error común hay con `auto-cost reference-bandwidth`?
11. Al dar `shutdown` en R2-G2/0, ¿qué mensaje OSPF se genera? ¿Por qué el traceroute de Debnet-1 hacia Internet no cambia?
12. ¿Por qué R2 sigue alcanzando a R1 después de bajar ese enlace?
13. ¿Cuál es la diferencia de configuración entre `network ... area` y `ip ospf 1 area 0` bajo la interfaz?

<details><summary>Respuestas</summary>

1. R4 solo conoce sus redes conectadas (172.17.0.0/24 y 10.0.10.0/30); no tiene ruta a 172.16.0.0/24 ni default, así que el paquete no sale.
2. Antes: **C** y **L**. Después: además **O** (intra-área) y **O IA** (inter-área), y con BGP/redistribución **O\*E2** y **O E2**.
3. **ABR**: R2 (áreas 0 y 1) y R3 (áreas 0 y 2). **ASBR**: R1, porque redistribuye rutas de BGP (otro AS) en OSPF.
4. Es el peering BGP con otro AS (el ISP); no debe formarse vecindad OSPF con el proveedor.
5. Sin loopback usa la **IP más alta entre interfaces activas**, aunque no participen en OSPF: 192.0.2.1 > 172.16.x.x y 10.x.x.x. Hay que configurar la loopback y **reiniciar el proceso/router** (el RID se fija al iniciar el proceso).
6. Para **no enviar Hellos/OSPF hacia las LAN de usuarios**, pero seguir anunciando esa red al resto.
7. Los paquetes salen con **IP origen privada** (172.17.x.x / 172.18.x.x), no enrutable en Internet: la respuesta no vuelve. **Overload (PAT)** traduce muchos orígenes privados a **una** IP pública usando puertos.
8. Rutas **externas tipo 2**: la default 0.0.0.0/0 (`default-information originate`) y 192.168.122.0/24 (`redistribute bgp`). Las origina el ASBR R1 como **LSA tipo 5**.
9. **Tipo 3**: los ABR (R2, R3) resumen redes de un área hacia otra. **Tipo 4**: los ABR informan cómo llegar al ASBR R1. **Tipo 5**: R1 (ASBR) para default y 192.168.122.0/24.
10. R2-R1 cuesta más → si hay un camino más barato (R2–R5–R3), OSPF lo prefiere. Error común: cambiar la reference bandwidth solo en un router → costos inconsistentes entre routers.
11. **LS Update con nuevo Router-LSA** de R2 (perdió un link), flooded a sus vecinos. El tráfico de Debnet-1 va Debnet-1→R1→R7→Internet y nunca pasa por R2.
12. Por **R5**: R2 → R5 → R3 → R1 (camino alternativo dentro del área 0). La tabla de R2 muestra el nuevo next hop.
13. Es lo mismo en efecto (asocia la interfaz al proceso y área OSPF). `network` lo hace por **rango de IP con wildcard**; el otro directamente **en la interfaz**.

</details>

---

## 5. QoS

### Resumen

- **QoS**: medida de la **calidad de transmisión y disponibilidad**, con variables medibles: **retardo, jitter, pérdida, ancho de banda, disponibilidad**.
- **QoE**: cómo **percibe el usuario** el servicio (subjetivo). QoS son números de la red; QoE es la percepción, y depende de la aplicación (un correo con 1 s de retardo no molesta; una videollamada sí). QoE se puede medir experimentalmente con grupos de usuarios.
- **Por qué QoS**: la capacidad es finita y cuando el tráfico la supera hay que decidir quién pasa, espera o se descarta. Sin QoS todo es **best effort**.
- **Hardware**: los routers/switches dedicados usan **ASIC** (reenvío en hardware). Los servidores mejoran con acceso directo a la NIC (DPDK, SR-IOV), pero la capacidad sigue siendo finita.

| Aplicación | Sensible a | Tolera |
|---|---|---|
| Correo / transferencia de archivos | **Pérdida** (necesita todo) | Retardo y jitter |
| Videollamada / VoIP | **Retardo y jitter** | Cierta pérdida |
| Streaming | **Ancho de banda** sostenido | Retardo inicial y jitter (buffer) |
| Gaming online | **Retardo** | Casi nada |

### Retardo

- Se mide con **RTT (ping)** porque no requiere relojes sincronizados. **RTT/2 no es el retardo de ida**: el ruteo puede ser **asimétrico** (ida 5 ms, vuelta 45 ms → RTT 50). Para medir en un sentido: **OWAMP (RFC 4656)**, con relojes sincronizados (NTP/PTP).
- **Retardo total = Σ por salto (procesamiento + encolamiento + serialización + propagación)**.

| Componente | Qué es | Fórmula / dato |
|---|---|---|
| **Serialización** | Poner los bits del paquete en el medio | `tamaño (bits) / velocidad (bps)`. Pesa en enlaces **lentos** |
| **Propagación** | Viaje físico de la señal | `distancia / velocidad`; fibra ≈ 2×10⁸ m/s. Solo baja acortando distancia |
| **Encolamiento** | Espera en la cola de salida | **Único que depende de la congestión**; causa principal del **jitter** |
| **Procesamiento** | Decidir qué hacer (lookup, clasificación, ACL, DPI) | µs en ASIC; mayor con firewall/DPI |

### Ejemplos de retardo

- **Serialización**: paquete de 1500 B = 12.000 bits. En 1,5 Mbps → **8 ms**. En 1 Gbps → **0,012 ms**.
- **Propagación**: fibra Montevideo–Buenos Aires ~250 km → 250.000 / 2×10⁸ = **1,25 ms**. Satélite geoestacionario (~36.000 km): ~120 ms de ida, **~240 ms de RTT** solo por propagación.
- **Total con 3 saltos** (clase): serialización 1,2 + 1,2 + 1,2 ms, propagación 0,5 + 2 + 0,5, procesamiento 0,05 + 0,05, sin congestión → **≈ 5,7 ms**. Con 20 ms de cola en el salto 2 → **≈ 25,7 ms**.

### Jitter

- **Variación del retardo** entre paquetes consecutivos de un flujo (ej.: voz cada 20 ms que llega a 18, 25, 15, 30 ms). Causa principal: **encolamiento variable**.
- Solución: **jitter buffer** en el receptor → **reduce el jitter pero aumenta el retardo**. Buffer chico = poco retardo pero cortes si hay mucho jitter; buffer grande = sin cortes pero retardo notorio (> ~150 ms totales en una conversación se siente).

### Integridad, ancho de banda, disponibilidad

- **Pérdida** (ej. *tail drop* por cola llena) y **errores** (checksum falla → descarte). Transferencia de archivos no tolera pérdida (TCP retransmite); la voz tolera algo (mejor perder un paquete que esperarlo tarde).
- **Ancho de banda nominal ≠ throughput real** (se mide con `iperf`, NetFlow/SNMP; ISPs usan **percentil 95**).
- **CIR**: tasa **garantizada** por contrato. **PIR**: tasa **máxima** en ráfagas, **sin garantía** (se descarta primero ante congestión); lo que supera el PIR lo descarta el **policing**.
  - Ejemplo: enlace físico 100 Mbps, CIR 50, PIR 80 → siempre 50 garantizados; entre 50 y 80 puede descartarse si la red está congestionada; **nunca** más de 80 aunque el enlace sea de 100.
- **Disponibilidad**: % del tiempo en que el servicio está **utilizable** (no solo "encendido"; hay que definir umbrales de qué es "caído").

| Disponibilidad | Caída por año | Caída por mes |
|---|---|---|
| 99% | ~3,65 días (87,6 h) | ~7,2 h |
| 99,9% | ~8,76 h | ~43,2 min |
| 99,99% | ~52,6 min | ~4,3 min |
| 99,999% ("cinco nueves") | ~5,3 min | ~26 s |
| 99,9999% | ~31,5 s | ~2,6 s |

Cinco nueves (estándar telecom) exige **redundancia real** (enlaces, equipos y rutas físicas). Cálculo: `(1 − disponibilidad) × tiempo`. Ej.: 99,9% de un año = 0,001 × 8760 h = 8,76 h.

### Mecanismos (vista previa)

- **Clasificación y marcado** (ej. **DSCP** en el header IP): identificar el tipo de tráfico y etiquetarlo para tratarlo sin reinspeccionar en cada salto.
- **Policing**: si excede la tasa contratada, **descarta** el exceso.
- **Shaping**: en lugar de descartar el exceso, lo **guarda en un buffer** y lo envía más tarde (suaviza ráfagas).
- **Queueing**: define el **orden de atención** de las colas ante congestión (ej.: prioridad a voz sobre best-effort).

### Preguntas

1. Definí QoS y QoE. ¿Qué relación y diferencia hay? Dá un ejemplo donde el mismo retardo afecte distinto a dos aplicaciones.
2. ¿Por qué se mide con ping (RTT) y qué problema tiene asumir retardo de ida = RTT/2? ¿Cómo se mide realmente en un solo sentido?
3. Nombrá las cuatro componentes del retardo. ¿Cuál depende de la congestión y cuál solo de la distancia?
4. Calculá la serialización de un paquete de 1000 bytes en un enlace de 2 Mbps.
5. Calculá la propagación en una fibra de 1000 km (velocidad 2×10⁸ m/s).
6. ¿Por qué un enlace satelital geoestacionario "se siente lento" aunque tenga buen ancho de banda?
7. Un flujo de voz sale con paquetes cada 20 ms y llegan cada 12, 30, 18, 25 ms. ¿Cómo se llama ese fenómeno? ¿Cuál es su causa principal y cómo se mitiga? ¿Qué costo tiene la mitigación?
8. ¿Qué aplicaciones son más sensibles a retardo, cuáles a pérdida y cuáles a ancho de banda? Justificá con la tabla.
9. Explicá CIR y PIR con el ejemplo: enlace de 100 Mbps, CIR 30, PIR 60. ¿Qué pasa si intentás mandar 80 Mbps?
10. Calculá cuántos minutos por mes puede caerse un servicio con 99,95% de disponibilidad (mes de 30 días).
11. ¿Por qué "router encendido" no equivale a "servicio disponible"?
12. ¿Qué diferencia hay entre policing y shaping? ¿Cuál agrega retardo?
13. ¿Qué es best effort y por qué se necesita QoS?
14. ¿Qué es marcar tráfico y por qué se hace en el borde de la red?
15. Un paquete de 1500 B viaja 3 saltos de 10 Mbps, cada uno con propagación 1 ms, procesamiento 0,1 ms y sin cola. ¿Retardo total? Si un salto tiene 15 ms de cola, ¿cuánto pasa a ser?

<details><summary>Respuestas</summary>

1. **QoS**: medidas objetivas de la red (retardo, jitter, pérdida, BW, disponibilidad). **QoE**: percepción subjetiva del usuario. QoS influye en QoE pero depende de la aplicación: 1 s de retardo en un correo es irrelevante; en una videollamada o juego arruina la experiencia.
2. Ping mide RTT con un solo reloj (sin sincronizar extremos). RTT/2 asume simetría, pero el ruteo IP puede ser **asimétrico** (distinto camino/congestión de ida y vuelta). Para el retardo de un sentido se usa **OWAMP (RFC 4656)** con relojes sincronizados.
3. Procesamiento, encolamiento, serialización, propagación. Depende de la congestión: **encolamiento**. Solo de la distancia: **propagación**.
4. 1000 B = 8000 bits; 8000 / 2.000.000 = **4 ms**.
5. 1.000.000 m / 2×10⁸ = **5 ms**.
6. Por el **retardo de propagación**: ~36.000 km de subida y bajada → ~240 ms de RTT solo por propagación. Es física, no se arregla con configuración ni más ancho de banda.
7. **Jitter** (variación del retardo). Causa: **retardo de encolamiento variable**. Mitigación: **jitter buffer**; costo: **más retardo** (trade-off con el tamaño del buffer).
8. Retardo: voz/video en tiempo real y gaming. Pérdida: correo y transferencia de archivos. Ancho de banda: streaming.
9. CIR 30: siempre garantizados. PIR 60: ráfagas hasta 60 sin garantía (descartables si hay congestión). Si mandás 80 Mbps, lo que pasa de 60 lo **descarta el policing**; el enlace físico soporta 100 pero el contrato no.
10. 0,05% de 43.200 min (30 días) = **21,6 min**.
11. Un router puede estar encendido y el enlace "up" pero con tanta pérdida/retardo que el servicio no es usable. Hay que definir umbrales de qué se considera "caído".
12. **Policing descarta** el exceso; **shaping lo encola** y lo envía después. El **shaping** agrega retardo.
13. Best effort: todo el tráfico compite igual, sin garantías. En congestión el tráfico crítico (voz) puede sufrir por tráfico que podía esperar (descarga). QoS diferencia el trato según necesidades.
14. Es etiquetar el paquete (ej. **DSCP**) según su clase para que el resto de la red lo trate sin reclasificar. Se hace en el borde para no repetir la inspección en cada salto.
15. Serialización: 12.000 bits / 10 Mbps = 1,2 ms por salto ×3 = 3,6 ms. Propagación 3 ms. Procesamiento 0,3 ms. Total ≈ **6,9 ms**. Con 15 ms de cola → **≈ 21,9 ms**.

</details>

---

## 6. Tabla de datos para memorizar

| Dato | Valor |
|---|---|
| RIP: update / invalid / flush | 30 s / 180 s / 240 s |
| RIP: máx. saltos / infinito | 15 / 16 |
| RIP: puerto, multicast v2, entradas por mensaje | UDP 520, 224.0.0.9, 25 |
| RIP: RFC | 2453 |
| AD: conectada / estática / OSPF / RIP | 0 / 1 / 110 / 120 |
| OSPF: multicast todos / DR-BDR | 224.0.0.5 / 224.0.0.6 |
| OSPF: Hello / Dead | 10 s / 40 s (4×) |
| OSPF: LSRefreshTime / MaxAge | 30 min / 1 h |
| OSPF: costo | 10⁸ / BW(bps); mínimo 1 |
| OSPF: paquetes | 1 Hello, 2 DBD, 3 LSR, 4 LSU, 5 LSAck |
| OSPF: tablas | `O`, `O IA`, `O E1`, `O E2` |
| OSPF: DR | mayor prioridad; empate mayor RID; prio 0 = nunca |
| QoS: retardo | Procesamiento + encolamiento + serialización + propagación |
| QoS: retardo one-way | OWAMP, RFC 4656 |
| Fibra / satélite geoestacionario | 2×10⁸ m/s / ~240 ms RTT |
| Disponibilidad 5 nueves | ~5,3 min/año |

### Errores típicos a evitar

- Confundir **AD** (entre protocolos) con **métrica** (dentro de un protocolo).
- Decir que OSPF elige el camino de "menos saltos": elige el de **menor costo acumulado**.
- Olvidar que RIP **suma 1** al anunciar, y que los updates son **periódicos aunque no haya cambios**.
- Confundir **broadcast** con **flooding**, y **LSA 3** (ABR, inter-área) con **LSA 5** (ASBR, externas).
- Dividir el RTT por 2 y llamarlo "retardo de ida".
- Creer que shaping y policing son lo mismo: uno **descarta**, el otro **retiene**.
- Cambiar la reference bandwidth en un solo router.
