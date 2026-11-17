# 🔢 IPv4 y CIDR

> **Fase:** 01 · Networking **Estado:** 🟡 En progreso
> **Anterior:** [`01-tcp-ip.md`](01-tcp-ip.md) · **Siguiente:** [`03-subnetting.md`](03-subnetting.md)

---

## 📑 Contenido

1. [¿Qué es una dirección IPv4?](#1-qué-es-una-dirección-ipv4)
2. [Binario en 5 minutos](#2-binario-en-5-minutos)
3. [Red y host: la máscara](#3-red-y-host-la-máscara)
4. [Calcular una red paso a paso](#4-calcular-una-red-paso-a-paso)
5. [Tabla de prefijos](#5-tabla-de-prefijos)
6. [¿Local o remoto? La cuenta que hace cada equipo](#6-local-o-remoto-la-cuenta-que-hace-cada-equipo)
7. [Clases (historia) y por qué llegó CIDR](#7-clases-historia-y-por-qué-llegó-cidr)
8. [Rangos especiales: privadas, públicas y reservadas](#8-rangos-especiales-privadas-públicas-y-reservadas)
9. [El encabezado IPv4](#9-el-encabezado-ipv4)
10. [En AWS](#10-en-aws)
11. [Ejercicios](#11-ejercicios)
12. [Autoevaluación](#12-autoevaluación)

---

## 1. ¿Qué es una dirección IPv4?

Una dirección IPv4 es un número de **32 bits**. Para leerlo fácil se escribe en **4 octetos** (grupos de 8 bits) en decimal, separados por puntos:

```
  192    .   168    .    1     .    10
11000000 . 10101000 . 00000001 . 00001010
```

- Cada octeto va de **0 a 255** (8 bits).
- Total de direcciones posibles: 2³² ≈ **4.300 millones**. Hoy no alcanzan para todos los dispositivos del mundo (ver [sección 8](#agotamiento-de-ipv4)).

A diferencia de la MAC (plana, como una cédula), la IP es **jerárquica**, como una dirección postal: una parte dice **en qué red** está el equipo y otra **qué equipo es**. Por eso los routers pueden trabajar con redes en vez de con equipos individuales (ver [`04-routing-switching`](04-routing-switching.md#1-por-qué-hacen-falta-dos-capas)).

---

## 2. Binario en 5 minutos

Cada bit de un octeto tiene un valor fijo:

| Bit       |  8  |  7  |  6  |  5  |  4  |  3  |  2  |  1  |
| --------- | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| **Valor** | 128 | 64  | 32  | 16  |  8  |  4  |  2  |  1  |

**Decimal → binario:** se van restando los valores de izquierda a derecha.

```
192 = 128 + 64                → 1 1 0 0 0 0 0 0
168 = 128 + 32 + 8            → 1 0 1 0 1 0 0 0
 10 = 8 + 2                   → 0 0 0 0 1 0 1 0
 70 = 64 + 4 + 2              → 0 1 0 0 0 1 1 0
```

**Valores que aparecen siempre en las máscaras** (unos seguidos desde la izquierda):

| Binario    | Decimal |
| ---------- | :-----: |
| `10000000` |   128   |
| `11000000` |   192   |
| `11100000` |   224   |
| `11110000` |   240   |
| `11111000` |   248   |
| `11111100` |   252   |
| `11111110` |   254   |
| `11111111` |   255   |

> [!TIP]
> Memoriza esta tabla: con ella calculas cualquier red sin calculadora.

---

## 3. Red y host: la máscara

La **máscara** dice cuántos bits de la IP son la **red**. Los bits en `1` son red; los bits en `0` son host.

```
IP:       11000000.10101000.00000001.00001010   (192.168.1.10)
Máscara:  11111111.11111111.11111111.00000000   (255.255.255.0)
          └──────────── red (24 bits) ───────┘└ host (8) ┘
```

**CIDR** (_Classless Inter-Domain Routing_) escribe lo mismo de forma corta: **cuántos unos** tiene la máscara.

```
192.168.1.10 / 24      =   192.168.1.10   con máscara 255.255.255.0
```

| Concepto             | Qué es                                                                            |
| -------------------- | --------------------------------------------------------------------------------- |
| **Prefijo** (`/24`)  | Bits de red.                                                                      |
| **Bits de host**     | 32 − prefijo. En un `/24`: 8.                                                     |
| **Dirección de red** | Todos los bits de host en `0`. Identifica a la red; no se asigna a ningún equipo. |
| **Broadcast**        | Todos los bits de host en `1`. “Todos los equipos de esta red”; no se asigna.     |
| **Hosts usables**    | Todo lo que queda en medio.                                                       |

**Fórmulas:**

```
Direcciones totales = 2^(32 − prefijo)
Hosts usables       = 2^(32 − prefijo) − 2      (se restan red y broadcast)
```

---

## 4. Calcular una red paso a paso

### Ejemplo 1 · Fácil: `192.168.1.10/24`

El corte cae justo entre octetos, así que es directo:

|               | Resultado        |
| ------------- | ---------------- |
| Red           | `192.168.1.0`    |
| Primer host   | `192.168.1.1`    |
| Último host   | `192.168.1.254`  |
| Broadcast     | `192.168.1.255`  |
| Hosts usables | 2⁸ − 2 = **254** |

### Ejemplo 2 · Corte dentro de un octeto: `192.168.1.70/26`

**① La máscara.** `/26` = 24 bits (3 octetos de 255) + 2 bits más → `11000000` = **192** → `255.255.255.192`.

**② La operación AND.** El equipo calcula su red con un **AND bit a bit** entre la IP y la máscara (1 AND 1 = 1; cualquier otra combinación = 0). Solo hace falta mirar el octeto donde cae el corte:

```
IP (último octeto):       70  = 0 1 | 0 0 0 1 1 0
Máscara:                 192  = 1 1 | 0 0 0 0 0 0
                                ─────────────────
AND (red):                64  = 0 1 | 0 0 0 0 0 0
```

**③ El atajo: tamaño del bloque.** 256 − 192 = **64** → las redes `/26` van de 64 en 64: `.0`, `.64`, `.128`, `.192`. El 70 cae en el bloque que empieza en **64**.

|               | Resultado                              |
| ------------- | -------------------------------------- |
| Red           | `192.168.1.64`                         |
| Primer host   | `192.168.1.65`                         |
| Último host   | `192.168.1.126`                        |
| Broadcast     | `192.168.1.127` (la siguiente red − 1) |
| Hosts usables | 2⁶ − 2 = **62**                        |

> [!IMPORTANT]
> **Método rápido para cualquier red:**
>
> 1. Busca el octeto donde cae el corte.
> 2. Tamaño del bloque = 256 − valor de la máscara en ese octeto.
> 3. Red = el múltiplo del bloque igual o menor que la IP.
> 4. Broadcast = red + bloque − 1.

---

## 5. Tabla de prefijos

| Prefijo | Máscara         | Direcciones |      Hosts usables | Usables en AWS (−5) | Uso típico                                                 |
| :-----: | --------------- | ----------: | -----------------: | ------------------: | ---------------------------------------------------------- |
|  `/8`   | 255.0.0.0       |  16.777.216 |         16.777.214 |                   — | Rango privado `10.0.0.0/8`                                 |
|  `/16`  | 255.255.0.0     |      65.536 |             65.534 |              65.531 | **VPC** (tamaño máximo)                                    |
|  `/20`  | 255.255.240.0   |       4.096 |              4.094 |               4.091 | Subnet grande                                              |
|  `/24`  | 255.255.255.0   |         256 |                254 |             **251** | Red de casa · subnet típica                                |
|  `/25`  | 255.255.255.128 |         128 |                126 |                 123 |                                                            |
|  `/26`  | 255.255.255.192 |          64 |                 62 |                  59 |                                                            |
|  `/27`  | 255.255.255.224 |          32 |                 30 |                  27 |                                                            |
|  `/28`  | 255.255.255.240 |          16 |                 14 |                  11 | **Subnet mínima en AWS**                                   |
|  `/29`  | 255.255.255.248 |           8 |                  6 |                   — |                                                            |
|  `/30`  | 255.255.255.252 |           4 |                  2 |                   — | Enlace entre dos routers                                   |
|  `/32`  | 255.255.255.255 |           1 | 1 (un solo equipo) |                   — | “Solo esta IP”, ej. `181.50.20.30/32` en un Security Group |

> [!TIP]
> **Cada bit menos de prefijo duplica el tamaño:** `/24` = 256 → `/23` = 512 → `/22` = 1024. Y al revés: cada bit más lo parte a la mitad. Esa es la base del [`03-subnetting`](03-subnetting.md).

---

## 6. ¿Local o remoto? La cuenta que hace cada equipo

Antes de enviar cualquier paquete, el equipo hace el **AND** con su propia máscara para su IP y para la IP de destino. Si da la misma red → **local** (ARP directo al destino). Si no → **remoto** (al gateway). Ver el ejemplo completo en [`04-routing-switching`](04-routing-switching.md#5-ejemplo-completo-pc-a--router--servidor).

| Mi IP / máscara     | Destino       | Mi red      | Red del destino   | Resultado            |
| ------------------- | ------------- | ----------- | ----------------- | -------------------- |
| 192.168.1.10/24     | 192.168.1.200 | 192.168.1.0 | 192.168.1.0       | **Local**            |
| 192.168.1.10/24     | 192.168.2.50  | 192.168.1.0 | 192.168.2.0       | **Remoto** → gateway |
| 192.168.1.10/**25** | 192.168.1.200 | 192.168.1.0 | 192.168.1.**128** | **Remoto** → gateway |

> [!WARNING]
> La tercera fila muestra por qué una **máscara mal configurada** rompe la red: con `/25`, el equipo cree que `.200` está en otra red y se lo envía al router, aunque esté conectado al mismo switch.

---

## 7. Clases (historia) y por qué llegó CIDR

Al principio, el tamaño de la red dependía del **primer octeto** (direccionamiento _classful_):

| Clase | Primer octeto | Máscara fija | Hosts por red |
| :---: | :-----------: | :----------: | ------------: |
|   A   |    1 – 126    |     `/8`     |    16.777.214 |
|   B   |   128 – 191   |    `/16`     |        65.534 |
|   C   |   192 – 223   |    `/24`     |           254 |
|   D   |   224 – 239   |      —       |     Multicast |
|   E   |   240 – 255   |      —       |  Experimental |

**El problema:** una empresa con 500 equipos no cabía en una clase C (254) y recibía una clase B (65.534): **se desperdiciaban más de 65.000 direcciones**.

**La solución (1993): CIDR.** La máscara puede tener **cualquier** largo (`/22`, `/27`…), así que cada quien recibe un tamaño que se ajusta a lo que necesita. Hoy las clases son solo historia, pero todavía se oye “una clase C” para decir “un `/24`”.

---

## 8. Rangos especiales: privadas, públicas y reservadas

### Privadas (RFC 1918)

Se pueden usar **libremente** dentro de cualquier red, pero **no se enrutan en Internet**: para salir necesitan **NAT**. Las usan las casas, las empresas y **todas las VPC**.

| Rango                         | CIDR             | Tamaño | Dónde lo ves                                           |
| ----------------------------- | ---------------- | ------ | ------------------------------------------------------ |
| 10.0.0.0 – 10.255.255.255     | `10.0.0.0/8`     | 16,7 M | Empresas · **VPC de AWS** (ej. `10.0.0.0/16`)          |
| 172.16.0.0 – 172.31.255.255   | `172.16.0.0/12`  | 1 M    | Empresas · la VPC por defecto de AWS (`172.31.0.0/16`) |
| 192.168.0.0 – 192.168.255.255 | `192.168.0.0/16` | 65 mil | Redes de casa                                          |

> [!WARNING]
> `172.32.1.1` **no** es privada: el rango privado de 172 va solo de `.16` a `.31`.

### Otros rangos que debes reconocer

| Rango             | Nombre                 | Para qué                                                                                                                     |
| ----------------- | ---------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `100.64.0.0/10`   | **CGNAT** (RFC 6598)   | IPs que usa el operador entre su red y tu router cuando aplica CGNAT.                                                        |
| `127.0.0.0/8`     | **Loopback**           | “Yo mismo”. `127.0.0.1` = `localhost`; nunca sale del equipo.                                                                |
| `169.254.0.0/16`  | **Link-local / APIPA** | IP que se pone un equipo cuando no consigue DHCP. En AWS, `169.254.169.254` es el **servicio de metadatos** de la instancia. |
| `0.0.0.0/0`       | **Todo**               | En una tabla de rutas o un Security Group: “cualquier IP”.                                                                   |
| `255.255.255.255` | Broadcast limitado     | “Todos en mi red local” (lo usa DHCP).                                                                                       |
| `224.0.0.0/4`     | Multicast              | Envío a un grupo.                                                                                                            |

### Pública vs. privada

|                         | Privada                                | Pública                 |
| ----------------------- | -------------------------------------- | ----------------------- |
| ¿Única en el mundo?     | No: miles de casas usan `192.168.1.10` | Sí                      |
| ¿Se enruta en Internet? | No                                     | Sí                      |
| ¿Quién la asigna?       | Tú (o tu router por DHCP)              | LACNIC → ISP / AWS → tú |

### Agotamiento de IPv4

- **IANA** entregó sus últimos bloques a los registros regionales en **2011**.
- **LACNIC** agotó su reserva para nuevas asignaciones en **2020**.
- Hoy las IPv4 públicas se **reutilizan, se compran o se alquilan** a precios altos.

Por eso existen tres cosas:

1. **NAT:** muchas IPs privadas salen por **una** pública (tu casa). Ver [`05-dns-dhcp-nat`](05-dns-dhcp-nat.md).
2. **CGNAT:** el operador comparte **una** IP pública entre **muchos clientes**. Señal de que estás detrás de CGNAT: la IP WAN de tu router está en `100.64.0.0/10` o no coincide con la que ves en un sitio tipo “cuál es mi IP”.
3. **IPv6:** 128 bits, direcciones prácticamente ilimitadas. Ver [`11-ipv6`](11-ipv6.md).

---

## 9. El encabezado IPv4

20 bytes (sin opciones) que van antes del segmento TCP/UDP:

| Campo                       | Para qué sirve                                                                                                                     |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| **Versión**                 | `4`                                                                                                                                |
| **Longitud del encabezado** | Normalmente 20 bytes.                                                                                                              |
| **DSCP**                    | Prioridad (QoS): por ejemplo, darle preferencia a la voz.                                                                          |
| **Longitud total**          | Tamaño del paquete completo (máx. 65.535; en Ethernet, normalmente ≤ 1500).                                                        |
| **ID · Flags · Offset**     | Para **fragmentar**: si un paquete no cabe en la MTU del siguiente enlace, se parte. El flag **DF** (_Don’t Fragment_) lo prohíbe. |
| **TTL**                     | Cada router le resta 1; en 0 se descarta. `tracert` lo aprovecha.                                                                  |
| **Protocolo**               | Qué viene adentro: `1` ICMP · `6` TCP · `17` UDP.                                                                                  |
| **Checksum**                | Verifica **solo el encabezado**. Cada router lo recalcula porque cambia el TTL.                                                    |
| **IP origen**               | Cambia solo con NAT.                                                                                                               |
| **IP destino**              | No cambia en todo el camino.                                                                                                       |

> [!NOTE]
> El campo **Protocolo** cumple en la capa 3 el mismo papel que el **EtherType** en la capa 2: le dice a la capa de arriba qué hay adentro.

---

## 10. En AWS

| Regla                           | Detalle                                                                                                                                                                           |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Tamaño de la VPC**            | Bloque CIDR entre **`/16`** (65.536 IPs) y **`/28`** (16 IPs). Se pueden agregar CIDR secundarios después.                                                                        |
| **Usar rangos RFC 1918**        | Lo normal es `10.x.x.x/16`.                                                                                                                                                       |
| **5 IPs reservadas por subnet** | En `10.0.1.0/24`: `.0` red · **`.1` router de la VPC** · `.2` DNS de AWS · `.3` reservada · `.255` broadcast (aunque no hay broadcast real). → Un `/24` deja **251** IPs usables. |
| **No solapar rangos**           | Si tu VPC usa `192.168.1.0/24` y tu oficina también, la **VPN** o el **peering** no podrán enrutar entre ellas. Elige rangos poco comunes desde el principio.                     |
| **IPs públicas**                | Una instancia en una subnet pública puede tener una IP pública o una **Elastic IP** (fija). AWS **cobra** por cada IPv4 pública desde 2024: otra consecuencia del agotamiento.    |
| **`/32` en Security Groups**    | “Solo mi IP”: ej. permitir SSH solo desde `181.50.20.30/32`.                                                                                                                      |
| **`0.0.0.0/0`**                 | En una route table: ruta por defecto. En un Security Group: **cualquier IP** (cuidado con abrir puertos así).                                                                     |

---

## 11. Ejercicios

Calcula red, broadcast, rango de hosts y cantidad de hosts:

1. `10.0.1.130/25`
2. `172.16.5.200/27`
3. ¿`172.32.1.1` es privada?
4. ¿`192.168.1.10/25` y `192.168.1.100/25` están en la misma red?
5. En AWS, ¿cuántas IPs usables tiene una subnet `/26`? ¿Cuál es la IP del router en `10.0.3.0/24`?

<details>
<summary>Ver soluciones</summary>

1. Bloque 256 − 128 = 128 → red `10.0.1.128` · hosts `.129`–`.254` · broadcast `10.0.1.255` · **126** hosts.
2. Bloque 256 − 224 = 32 → 200 cae en el bloque que empieza en 192 → red `172.16.5.192` · hosts `.193`–`.222` · broadcast `172.16.5.223` · **30** hosts.
3. **No.** El rango privado es `172.16.0.0` – `172.31.255.255`.
4. **Sí.** Con `/25` los bloques son `.0–.127` y `.128–.255`; ambas están en `.0–.127`.
5. 64 − 5 = **59**. El router de la VPC es `10.0.3.1`.

</details>

---

## 12. Autoevaluación

- [ ] ¿Por qué una IP es jerárquica y una MAC no?
- [ ] ¿Cuánto vale `11100000` en decimal?
- [ ] ¿Qué significa `/24` y cuántos hosts usables tiene?
- [ ] ¿Por qué se restan 2 direcciones a los hosts usables? ¿Y por qué 5 en AWS?
- [ ] ¿Cómo calculo la red y el broadcast de `192.168.1.70/26` con el método del bloque?
- [ ] ¿Qué pasa si configuro mal la máscara de un equipo?
- [ ] ¿Por qué CIDR reemplazó a las clases?
- [ ] ¿Cuáles son los tres rangos privados de RFC 1918?
- [ ] ¿Cómo sé si mi ISP me tiene detrás de CGNAT?
- [ ] ¿Para qué sirve `169.254.169.254` en AWS?
- [ ] ¿Por qué no debo usar el mismo rango en mi VPC y en mi oficina?
