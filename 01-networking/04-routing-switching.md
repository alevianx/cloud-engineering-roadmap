# 🔀 Routing y switching

> **Fase:** 01 · Networking **Estado:** 🟡 En progreso
> **Anterior:** [`03-subnetting.md`](03-subnetting.md) · **Siguiente:** [`05-dns-dhcp-nat.md`](05-dns-dhcp-nat.md)
> **Base:** capas 2 y 3 de [`00-osi-model.md`](00-osi-model.md)

---

## 📑 Contenido

1. [¿Por qué hacen falta dos capas?](#1-por-qué-hacen-falta-dos-capas)
2. [Switching (capa 2)](#2-switching-capa-2)
3. [Broadcast, flooding y ARP](#3-broadcast-flooding-y-arp)
4. [Routing (capa 3)](#4-routing-capa-3)
5. [Ejemplo completo: PC-A → router → servidor](#5-ejemplo-completo-pc-a--router--servidor)
6. [La MAC cambia en cada salto](#6-la-mac-cambia-en-cada-salto)
7. [Problemas típicos](#7-problemas-típicos)
8. [Diagnóstico](#8-diagnóstico)
9. [En AWS](#9-en-aws)
10. [BGP y ASN](#10-bgp-y-asn)
11. [Autoevaluación](#11-autoevaluación)

---

## 1. ¿Por qué hacen falta dos capas?

La capa 2 funciona muy bien **dentro de una red local**, pero no escala más allá:

| Problema de usar solo MAC                                     | Por qué                                                                  |
| ------------------------------------------------------------- | ------------------------------------------------------------------------ |
| La MAC no dice **dónde** está el equipo                       | Es plana, como una cédula: el OUI indica el fabricante, no la ubicación. |
| Los switches tendrían que conocer **todas las MAC del mundo** | Tablas con miles de millones de entradas.                                |
| Los broadcast y el flooding **inundarían el planeta**         | Cada ARP llegaría a todos los equipos del mundo.                         |

La **IP** resuelve esto porque es **jerárquica**, como una dirección postal: una parte dice **en qué red** está el equipo y otra **qué equipo es**.

```
192.168.1.10 /24
└───┬────┘ └┬┘
  red       host
```

Un router no necesita conocer cada equipo: solo **redes** (“todo lo que empiece por 192.168.2 va por este lado”).

### División del trabajo

|                       | Capa 3 · IP                                     | Capa 2 · MAC                                      |
| --------------------- | ----------------------------------------------- | ------------------------------------------------- |
| Pregunta              | ¿Cuál es el **destino final** y por dónde sigo? | ¿Cómo llego al **siguiente** aparato en esta red? |
| Alcance               | Extremo a extremo                               | Un solo enlace                                    |
| Dirección             | Jerárquica (red + host)                         | Plana                                             |
| ¿Cambia en el camino? | No (salvo NAT)                                  | Sí, en cada router                                |
| Dispositivo           | **Router**                                      | **Switch**                                        |

---

## 2. Switching (capa 2)

Un **switch** conecta equipos de la **misma red local** y reenvía tramas según la **MAC de destino**, usando su **tabla MAC** (o _CAM table_).

| MAC               | Puerto |
| ----------------- | :----: |
| AA:AA:AA:AA:AA:01 |   1    |
| BB:BB:BB:BB:BB:02 |   2    |

Por cada trama que recibe:

1. **Aprende:** guarda la **MAC de origen** y el puerto por el que llegó.
2. **Busca:** consulta la **MAC de destino** en su tabla.
3. **Decide:**
   - La conoce → la envía **solo** por ese puerto.
   - No la conoce → **flooding**: la envía por todos los puertos (menos el de llegada).
   - Es broadcast (`FF:FF:FF:FF:FF:FF`) → la envía por todos los puertos.

> [!NOTE]
> El switch **no lee la IP ni cambia nada** de la trama: solo la reenvía. Por eso **no cuenta como un salto**.

### Dominio de broadcast y VLANs

- **Dominio de broadcast:** hasta dónde llega un broadcast. El switch **no** lo corta; el router **sí**.
- **VLAN (802.1Q):** divide un switch en varias redes lógicas, cada una con su propio dominio de broadcast.

> [!IMPORTANT]
> En la práctica: **1 subred IP = 1 dominio de broadcast = (normalmente) 1 VLAN**. Las VLANs separan en capa 2; el router las une en capa 3.

---

## 3. Broadcast, flooding y ARP

### Broadcast vs. flooding

|                               | Broadcast                  | Flooding                                            |
| ----------------------------- | -------------------------- | --------------------------------------------------- |
| ¿Quién decide enviar a todos? | **El emisor**, a propósito | **El switch**, porque no sabe dónde está el destino |
| MAC de destino                | `FF:FF:FF:FF:FF:FF`        | La MAC de un equipo concreto                        |
| ¿Quién la acepta?             | **Todos**                  | **Solo el destinatario**; los demás la descartan    |
| ¿Es permanente?               | Sí                         | No: desaparece cuando el switch aprende la MAC      |
| Usos                          | ARP, DHCP                  | — (es un efecto secundario)                         |

> [!TIP]
> **Analogía:** _broadcast_ = el profesor grita en el salón “¿quién es Juan Pérez?” (todos escuchan, solo Juan responde). _Flooding_ = un mensajero con una carta para Juan pasa por todas las filas; cada uno mira el nombre y, si no es el suyo, la ignora.

### ARP (Address Resolution Protocol)

Responde a: **“conozco la IP de un equipo de mi red, ¿cuál es su MAC?”**. Es el puente entre la capa 3 (IP) y la capa 2 (MAC). EtherType `0x0806`.

| Paso            | Tipo                    | Contenido                                                         |
| --------------- | ----------------------- | ----------------------------------------------------------------- |
| **ARP Request** | _Broadcast_ (`FF:FF:…`) | “¿Quién tiene 192.168.1.20? Respóndele a 192.168.1.10 (AA:…:01)”  |
| **ARP Reply**   | _Unicast_               | “192.168.1.20 soy yo, mi MAC es BB:…:02”                          |
| **Caché ARP**   | —                       | La respuesta se guarda un tiempo para no preguntar en cada trama. |

> [!WARNING]
> ARP **solo funciona dentro de la red local** (el broadcast no cruza el router). Si el destino está en otra red, se pregunta por la MAC del **gateway**, nunca por la del destino.

### Los tres juntos (switch recién encendido)

```
PC-A (puerto 1)          Impresora (puerto 2)          PC-C (puerto 3)
192.168.1.10 · AA:..:01  192.168.1.20 · BB:..:02       192.168.1.30 · CC:..:03
```

1. PC-A no tiene la MAC de 192.168.1.20 en su caché → envía un **ARP Request (broadcast)**.
2. El switch **aprende** que AA:..:01 está en el puerto 1 → como es broadcast, lo envía a los puertos 2 y 3.
3. PC-C ve que no es su IP → **ignora**. La impresora responde → y guarda en su caché la MAC de PC-A.
4. **ARP Reply (unicast)** hacia AA:..:01.
5. El switch **aprende** que BB:..:02 está en el puerto 2 → ya conoce AA:..:01 → lo envía **solo** al puerto 1 (sin flooding).
6. PC-A guarda la MAC de la impresora → los datos van en **unicast** directo al puerto 2. PC-C no se entera.

> [!NOTE]
> **¿Cuándo hay flooding?** Si la impresora ya tenía la MAC de PC-A en su caché, pero el switch se reinició: su primera trama hacia AA:..:01 se inunda a los puertos 1 y 3. PC-C la descarta; el switch aprende y la siguiente va directa.

---

## 4. Routing (capa 3)

Un **router** conecta **redes distintas**. Tiene **una interfaz por red**, y cada interfaz tiene su propia **IP y MAC**.

### ¿Local o remoto?

Antes de enviar, el equipo compara la IP de destino con **su propia red** (usando la máscara):

| Resultado             | Qué hace                                                 |
| --------------------- | -------------------------------------------------------- |
| **Local** (misma red) | ARP por la IP **del destino** → entrega directa.         |
| **Remoto** (otra red) | ARP por la IP **del gateway** → se lo entrega al router. |

### Gateway por defecto

La IP del router en **mi** red. Es a quien le envío todo lo que **no** está en mi red.

### Tabla de rutas

Cada router (y cada equipo) tiene una tabla que dice por dónde enviar cada red:

| Red destino    | Salida                                  |
| -------------- | --------------------------------------- |
| 192.168.1.0/24 | eth0 (conectada directamente)           |
| 192.168.2.0/24 | eth1 (conectada directamente)           |
| 0.0.0.0/0      | 181.50.20.1 (ruta por defecto → el ISP) |

- **`0.0.0.0/0`** = “todo lo demás”: la **ruta por defecto**.
- Si varias rutas coinciden, gana **la más específica** (el prefijo más largo, _longest prefix match_): `/24` gana sobre `/0`.

### TTL

Cada router le **resta 1** al TTL del paquete. Si llega a **0**, lo descarta (y avisa con ICMP). Así un error de rutas no deja paquetes dando vueltas para siempre.

---

## 5. Ejemplo completo: PC-A → router → servidor

```
     RED 1: 192.168.1.0/24                     RED 2: 192.168.2.0/24

  PC-A ──── Switch 1 ──── [eth0  ROUTER  eth1] ──── Switch 2 ──── Servidor
```

| Equipo      | IP               | MAC               | Gateway     |
| ----------- | ---------------- | ----------------- | ----------- |
| PC-A        | 192.168.1.10 /24 | AA:AA:AA:AA:AA:01 | 192.168.1.1 |
| Router eth0 | 192.168.1.1      | R1:R1:R1:00:00:01 | —           |
| Router eth1 | 192.168.2.1      | R1:R1:R1:00:00:02 | —           |
| Servidor    | 192.168.2.50 /24 | SS:SS:SS:SS:SS:50 | 192.168.2.1 |

**PC-A le envía “Hola” al servidor.**

|  #  | Dónde         |     Capa      | Qué pasa                                                                                                                |
| :-: | ------------- | :-----------: | ----------------------------------------------------------------------------------------------------------------------- |
|  1  | PC-A          |       3       | Arma el paquete: IP origen `192.168.1.10` → IP destino `192.168.2.50`, TTL 64.                                          |
|  2  | PC-A          |       3       | ¿Local o remoto? `192.168.2.x` ≠ `192.168.1.x` → **remoto** → se lo envía al gateway `192.168.1.1`.                     |
|  3  | PC-A          |       2       | ARP: “¿Quién tiene **192.168.1.1**?” (broadcast) → el router responde con `R1:..:01`.                                   |
|  4  | PC-A          |     2 → 1     | Arma la trama: **MAC** `AA:..:01` → `R1:..:01` · **IP** `192.168.1.10` → `192.168.2.50`. La convierte en señales.       |
|  5  | Switch 1      |       2       | Reenvía por MAC de destino. **No cambia nada.**                                                                         |
|  6  | Router (eth0) |   1 → 2 → 3   | Recupera bits · FCS ok · MAC destino es la suya → **desarma la trama** · EtherType `0x0800` → sube el paquete a capa 3. |
|  7  | Router        |       3       | IP destino `192.168.2.50` no es suya → tabla de rutas: `192.168.2.0/24` → **eth1** · TTL 64 → 63.                       |
|  8  | Router (eth1) |       2       | ARP en la Red 2: “¿Quién tiene **192.168.2.50**?” → `SS:..:50`.                                                         |
|  9  | Router (eth1) |     2 → 1     | **Trama nueva:** **MAC** `R1:..:02` → `SS:..:50` · **IP** igual · FCS nuevo.                                            |
| 10  | Servidor      | 1 → 2 → 3 → 7 | MAC destino es la suya · IP destino es la suya → “Hola”.                                                                |

**Compara la trama antes y después del router:**

|             | Red 1 (paso 4)         | Red 2 (paso 9)         |
| ----------- | ---------------------- | ---------------------- |
| MAC origen  | AA:..:01 (PC-A)        | R1:..:02 (router eth1) |
| MAC destino | R1:..:01 (router eth0) | SS:..:50 (servidor)    |
| IP origen   | 192.168.1.10           | 192.168.1.10           |
| IP destino  | 192.168.2.50           | 192.168.2.50           |
| TTL         | 64                     | 63                     |

> [!IMPORTANT]
> En la Red 1, la trama lleva **MAC destino = router**, pero **IP destino = servidor**. No es un error: así funciona.
> **La respuesta** hace lo mismo al revés: el servidor ve que `192.168.1.10` es remota y se la envía a **su** gateway, `192.168.2.1`.

---

## 6. La MAC cambia en cada salto

Las MAC **de los aparatos** no cambian. Lo que cambia son **las MAC escritas en la trama**: cada router quita el “sobre” de capa 2 y pone uno nuevo para el siguiente tramo.

**Ejemplo: PC de la casa → google.com**

| Tramo                           | MAC origen → destino     | IP origen → destino                    |
| ------------------------------- | ------------------------ | -------------------------------------- |
| PC → router de casa             | PC → router (LAN)        | 192.168.1.10 → 142.250.78.14           |
| Router de casa → router del ISP | router (WAN) → ISP       | **181.50.20.30** → 142.250.78.14 ← NAT |
| … → servidor                    | último router → servidor | 181.50.20.30 → 142.250.78.14           |

- **MAC:** cambia en **cada** tramo.
- **IP destino:** igual de principio a fin.
- **IP origen:** cambia **una vez**, en el router de la casa, por **NAT** (privada → pública). Ver [`05-dns-dhcp-nat`](05-dns-dhcp-nat.md).

> [!TIP]
> **Analogía:** la dirección final del paquete (IP) dice “Calle X, Madrid” y nunca cambia. La etiqueta del transportista (MAC) se reemplaza en cada bodega: “casa → bodega Bogotá”, “bodega → aeropuerto”, “aeropuerto Madrid → bodega Madrid”…

> [!NOTE]
> La **MAC aleatoria** de celulares y laptops (una MAC distinta por cada red Wi-Fi, por privacidad) es otra cosa: es el dispositivo cambiando su identidad, no los routers cambiando el sobre.

---

## 7. Problemas típicos

| Problema                          | Qué pasa                                                                                                                   | Solución                                                                           |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| **Tormenta de broadcast**         | Dos switches unidos con dos cables forman un círculo: el broadcast da vueltas para siempre (las tramas **no tienen TTL**). | **STP** (_Spanning Tree_) bloquea los caminos redundantes.                         |
| **ARP spoofing**                  | ARP no tiene seguridad: un atacante responde “yo soy 192.168.1.1” y todo el tráfico pasa por él.                           | Seguridad en el switch (_Dynamic ARP Inspection_), redes confiables, **HTTPS**.    |
| **Demasiados equipos en una red** | Todos tienen que leer cada broadcast → ruido.                                                                              | Dividir en subredes / VLANs más pequeñas. Ver [`03-subnetting`](03-subnetting.md). |

---

## 8. Diagnóstico

| Síntoma                                         | Causa probable                                      |
| ----------------------------------------------- | --------------------------------------------------- |
| Llegas a tu red local, pero no a Internet       | **Gateway** mal configurado o caído                 |
| No llegas a algunos equipos de tu propia red    | **Máscara** incorrecta (local ↔ remoto confundidos) |
| Dos equipos se interrumpen la conexión entre sí | **IP duplicada** (ARP responde con dos MACs)        |

| Qué ver              | Windows       | Linux / macOS |
| -------------------- | ------------- | ------------- |
| IP, máscara, gateway | `ipconfig`    | `ip addr`     |
| Caché ARP (IP → MAC) | `arp -a`      | `ip neigh`    |
| Tabla de rutas       | `route print` | `ip route`    |
| Camino salto a salto | `tracert`     | `traceroute`  |

> [!TIP]
> Ejecuta `arp -a`: vas a ver la IP de tu router (ej. `192.168.1.1`) **junto a su MAC**. Es el resultado real del paso 3 del ejemplo.

---

## 9. En AWS

| Concepto                    | En AWS                                                                                                                                          |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| Subred                      | **Subnet** de la VPC (ej. `10.0.1.0/24`)                                                                                                        |
| Gateway de la red           | El **router de la VPC**, en la dirección **`.1`** de cada subnet (AWS la reserva)                                                               |
| Tabla de rutas              | **Route table**: `local` = toda la VPC (entrega directa) · `0.0.0.0/0 → igw-…` = Internet Gateway                                               |
| Ruta por defecto de la casa | La misma `0.0.0.0/0`                                                                                                                            |
| Switch / capa 2             | **No hay switches reales ni broadcast.** La capa 2 está virtualizada: cuando una instancia hace ARP, responde la propia infraestructura de AWS. |

> [!NOTE]
> Como no hay broadcast real en la VPC, desaparecen las tormentas de broadcast y el ARP spoofing entre instancias. Pero la lógica que ve la instancia es la misma: **¿local o remoto? → gateway → route table**.

---

## 10. BGP y ASN ⏳

> Pendiente. Cómo se conectan entre sí las redes que forman Internet (sistemas autónomos, anuncios de rutas). En AWS aparece en **Direct Connect** y **Site-to-Site VPN**. Ver también [`08-vpn`](08-vpn.md).

---

## 11. Autoevaluación

- [ ] ¿Por qué no se puede construir Internet solo con MACs y switches?
- [ ] ¿Qué hace un switch cuando no conoce la MAC de destino?
- [ ] ¿Qué diferencia hay entre un broadcast y un flooding?
- [ ] ¿Por qué el ARP Request es broadcast y el ARP Reply es unicast?
- [ ] Si PC-A quiere hablar con un servidor de otra red, ¿por la MAC de quién pregunta con ARP?
- [ ] En el ejemplo, ¿qué campos cambian cuando la trama pasa por el router y cuáles no?
- [ ] ¿Por qué un switch no cuenta como un salto y un router sí?
- [ ] ¿Qué ruta gana si coinciden `192.168.2.0/24` y `0.0.0.0/0`?
- [ ] ¿Por qué una trama puede dar vueltas para siempre y un paquete IP no?
- [ ] ¿Cuál es la IP del router de la VPC en la subnet `10.0.3.0/24`?
