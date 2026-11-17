# 🧱 Modelo TCP/IP

> **Fase:** 01 · Networking **Estado:** 🟡 En progreso
> **Anterior:** [`00-osi-model.md`](00-osi-model.md) · **Siguiente:** [`02-ipv4-cidr.md`](02-ipv4-cidr.md)

---

## 📑 Contenido

1. [¿Qué es TCP/IP?](#1-qué-es-tcpip)
2. [Las 4 capas y dónde viven](#2-las-4-capas-y-dónde-viven)
3. [Encapsulamiento real: encabezados y tamaños](#3-encapsulamiento-real-encabezados-y-tamaños)
4. [Capa de transporte a fondo: puertos, TCP y UDP](#4-capa-de-transporte-a-fondo-puertos-tcp-y-udp)
5. [Viaje completo: abrir una página web](#5-viaje-completo-abrir-una-página-web)
6. [Internet: una red de redes](#6-internet-una-red-de-redes)
7. [Diagnóstico](#7-diagnóstico)
8. [En AWS](#8-en-aws)
9. [Autoevaluación](#9-autoevaluación)

---

## 1. ¿Qué es TCP/IP?

**TCP/IP** es el conjunto de protocolos con el que **realmente funciona Internet**. Nació en los años 70 en ARPANET (la red de investigación del Departamento de Defensa de EE. UU.) y se volvió su estándar oficial en 1983, **antes** de que existiera el modelo OSI (1984).

|        | OSI                                                              | TCP/IP                                         |
| ------ | ---------------------------------------------------------------- | ---------------------------------------------- |
| Qué es | Modelo **teórico** de referencia                                 | Conjunto de **protocolos reales**              |
| Capas  | 7                                                                | 4                                              |
| Uso    | Explicar y diagnosticar (“problema de capa 3”, “balanceador L7”) | Lo que corre en cada equipo, router y servidor |

> [!TIP]
> **OSI es el mapa; TCP/IP es el territorio.** Se estudian los dos porque en el trabajo se usa el vocabulario de OSI para hablar de equipos que funcionan con TCP/IP.

---

## 2. Las 4 capas y dónde viven

| TCP/IP           |    OSI    | Qué resuelve                                 | PDU                  | Protocolos                | **¿Dónde vive?**                               |
| ---------------- | :-------: | -------------------------------------------- | -------------------- | ------------------------- | ---------------------------------------------- |
| **Aplicación**   | 7 · 6 · 5 | Qué quiere hacer el usuario y en qué formato | Datos                | HTTP, DNS, TLS, SSH, DHCP | En la **aplicación** (navegador, servidor web) |
| **Transporte**   |     4     | A qué **aplicación** va y si llega completo  | Segmento / Datagrama | TCP, UDP                  | En el **sistema operativo** (kernel)           |
| **Internet**     |     3     | Cómo llegar a **otra red**                   | Paquete              | IP, ICMP                  | En el **sistema operativo** (kernel)           |
| **Acceso a red** |   2 · 1   | Cómo entregar en **esta red** y mover bits   | Trama / Bits         | Ethernet, Wi-Fi, ARP      | En la **tarjeta de red** y su driver           |

> [!NOTE]
> Por eso una aplicación nunca “arma tramas”: le entrega los datos al sistema operativo por un **socket** (“envía esto a 142.250.78.14, puerto 443”), y el sistema operativo y la tarjeta de red hacen todo lo demás.

---

## 3. Encapsulamiento real: encabezados y tamaños

### 3.1 Cómo se ve una trama real

Cada capa agrega su encabezado. Así se ve una trama con un pedazo de una respuesta HTTPS:

```
┌──────────────┬──────────────┬──────────────┬───────────────────────────┬──────┐
│ Ethernet     │ IPv4         │ TCP          │ Datos (TLS → HTTP)        │ FCS  │
│ 14 bytes     │ 20 bytes     │ 20 bytes     │ hasta 1460 bytes          │ 4 B  │
└──────────────┴──────────────┴──────────────┴───────────────────────────┴──────┘
 └─ capa 2 ───┘└─ capa 3 ────┘└─ capa 4 ────┘└─ capa 7 ─────────────────┘└ capa 2┘
               └────────────── MTU: 1500 bytes ───────────────────────────┘
               └──────── paquete IP ──────────────────────────────────────┘
                              └──────── segmento TCP ─────────────────────┘
```

| Encabezado          | Campos más importantes                                                                                                                                         |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Ethernet** (14 B) | MAC destino · MAC origen · EtherType (`0x0800` = IPv4)                                                                                                         |
| **IPv4** (20 B)     | IP origen · IP destino · **TTL** · **Protocolo** (`6` TCP · `17` UDP · `1` ICMP) · checksum. Detalle en [`02-ipv4-cidr`](02-ipv4-cidr.md#9-el-encabezado-ipv4) |
| **TCP** (20 B)      | Puerto origen · puerto destino · **número de secuencia** · **número de ACK** · flags · ventana                                                                 |
| **UDP** (8 B)       | Puerto origen · puerto destino · longitud · checksum. Nada más.                                                                                                |

### 3.2 Los dos límites: MTU y MSS

| Límite                                     | Qué mide                                                       | Valor típico                                 | Lo define       |
| ------------------------------------------ | -------------------------------------------------------------- | -------------------------------------------- | --------------- |
| **MTU** (_Maximum Transmission Unit_)      | Lo máximo que cabe **dentro** de una trama: el paquete IP completo | **1500 bytes** en Ethernet                   | La red (capa 2) |
| **MSS** (_Maximum Segment Size_)           | Lo máximo de **datos** (sin encabezados) por segmento TCP      | 1500 − 20 (IP) − 20 (TCP) = **1460 bytes**   | TCP (capa 4)    |

> [!TIP]
> **Analogía:** la MTU es el tamaño de la **caja** que acepta la empresa de envíos. El MSS es **cuánto producto** cabe dentro, después de poner el relleno y las etiquetas (los encabezados IP y TCP).

```
Tamaño de la trama = 14 (Ethernet) + 20 (IP) + 20 (TCP) + datos + 4 (FCS)
Con datos = 1460   → 14 + 20 + 20 + 1460 + 4 = 1518 bytes   (la trama Ethernet más grande)
```

### 3.3 Ejemplo: así se parte una página de 5.000 bytes

El servidor te va a enviar un HTML de **5.000 bytes**. Pero en una trama solo caben **1.460 bytes de datos**. ¿Qué pasa?

**① La aplicación no parte nada.** El servidor web le entrega los 5.000 bytes **de una vez** al sistema operativo por el socket.

**② TCP (en el sistema operativo) los parte en segmentos de máximo 1.460 bytes** (el MSS):

```
5.000 bytes de datos
├───── 1460 ─────┼───── 1460 ─────┼───── 1460 ─────┼─ 620 ─┤
   segmento 1       segmento 2       segmento 3      seg. 4
```

5.000 ÷ 1.460 = 3,4 → **4 segmentos** (tres llenos y uno con lo que sobra: 5.000 − 3 × 1.460 = **620**).

**③ Cada segmento se numera** con el **número de secuencia**: el número del **primer byte** que lleva.

|  Seg.  | Bytes que lleva | Nº de secuencia (relativo) | Tamaño de los datos | Tamaño de la trama completa |
| :----: | --------------- | :------------------------: | :-----------------: | :-------------------------: |
|   1    | 1 – 1.460       |           **1**            |        1.460        |          1.518 B            |
|   2    | 1.461 – 2.920   |         **1.461**          |        1.460        |          1.518 B            |
|   3    | 2.921 – 4.380   |         **2.921**          |        1.460        |          1.518 B            |
|   4    | 4.381 – 5.000   |         **4.381**          |         620         |      14+20+20+620+4 = **678 B** |

**④ Cada segmento viaja en su propio paquete IP**, dentro de su propia trama. Son 4 “sobres” independientes: pueden tomar caminos distintos, llegar desordenados o perderse uno solo.

**⑤ El receptor los reordena** usando el número de secuencia, junta los 5.000 bytes y se los entrega a la aplicación (el navegador) **como si hubieran llegado de una sola vez**.

> [!IMPORTANT]
> **¿Es como una paginación?** Se parece bastante, pero con 3 diferencias:
>
> | Paginación (ej. una API)                  | Segmentación TCP                                                                |
> | ----------------------------------------- | ------------------------------------------------------------------------------- |
> | Numera **páginas** (1, 2, 3…)             | Numera **bytes**: el segmento dice “empiezo en el byte 1.461”                   |
> | El cliente **pide** cada página           | El emisor **envía varias seguidas** sin esperar (hasta lo que permite la ventana) |
> | La aplicación **ve** las páginas          | La aplicación **no ve nada**: recibe un flujo continuo de bytes ya ordenado     |
>
> La idea que sí es igual: cada pedazo se puede **identificar, ordenar y volver a pedir por separado**. Si se pierde el segmento 2, **solo se reenvía el 2** (ver la sección 4.3).

### 3.4 ¿Cómo sabe cada lado cuál es el MSS?

Lo **anuncian en el handshake**: cada lado pone su MSS en el `SYN` (“yo puedo recibir segmentos de hasta 1.460 bytes”), y cada uno respeta el del otro. Ver la sección 4.3.

### 3.5 ¿Y si un paquete no cabe en algún tramo del camino?

No todos los tramos tienen MTU 1500 (una VPN o un enlace PPPoE agregan sus propios encabezados y dejan menos espacio). Hay dos opciones:

| Opción                                | Cómo funciona                                                                                                                                         | Problema                                                                  |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| **Fragmentación IP** (capa 3)         | El router parte el **paquete** en fragmentos; el destino los vuelve a unir.                                                                           | Si se pierde **un** fragmento, se pierde **todo** el paquete. Es lento.   |
| **Path MTU Discovery** (lo normal hoy) | El emisor marca el paquete con **DF** (_Don't Fragment_). Si no cabe, el router lo descarta y responde con **ICMP** “demasiado grande, el máximo es X”. El emisor achica sus segmentos. | Si un firewall bloquea **todo** el ICMP, el emisor nunca se entera → la conexión se “congela”. |

> [!WARNING]
> **Síntoma clásico de MTU:** el `ping` funciona y la página empieza a cargar, pero **se queda pegada** con archivos grandes. Los paquetes pequeños caben; los de 1.500 no, y el aviso ICMP está bloqueado. **Por eso no se debe bloquear todo el ICMP.**

> [!NOTE]
> **Segmentación ≠ fragmentación.** La **segmentación** la hace **TCP** en el origen, a propósito, y cada pedazo tiene su número de secuencia. La **fragmentación** la hace **IP** en el camino, solo si hace falta, y conviene evitarla. UDP no segmenta: si la aplicación envía un datagrama más grande que la MTU, IP lo fragmenta.

### 3.6 La sobrecarga: por qué importa el tamaño

Cada trama lleva **58 bytes de “sobre”** (14 + 20 + 20 + 4), lleve 1 byte o 1.460 bytes de datos.

| Caso                                     | Datos por trama | Trama   | % útil       |
| ---------------------------------------- | :-------------: | :-----: | :----------: |
| Mensaje pequeño (ej. un “Hola”)          | 4 B             | 64 B*   | **~6 %**     |
| Trama llena (MTU 1500)                   | 1.460 B         | 1.518 B | **~96 %**    |
| _Jumbo frame_ (MTU 9001, dentro de AWS)  | 8.949 B         | 9.019 B | **~99 %**    |

\* Ethernet exige un mínimo de 64 bytes por trama: si no alcanza, se rellena.

Un archivo de **1 MB** viaja en unas **700 tramas** (1.000.000 ÷ 1.460 ≈ 685), cada una con su propio sobre.

> [!TIP]
> **En AWS:**
>
> - Dentro de una VPC, muchas instancias usan **jumbo frames** de **9001** bytes: menos tramas para mover lo mismo.
> - Hacia **Internet** (Internet Gateway) y entre regiones se usa **1500**.
> - Por **Site-to-Site VPN** el máximo es todavía menor (**1446**), porque el cifrado IPsec agrega su propio encabezado.

---

## 4. Capa de transporte a fondo: puertos, TCP y UDP

### 4.1 Puertos: a qué aplicación va

La IP lleva el paquete **al equipo**; el **puerto** lo lleva **a la aplicación** dentro del equipo. Es un número de 16 bits (0–65535).

| Rango         | Nombre                            | Ejemplos                                                                |
| ------------- | --------------------------------- | ----------------------------------------------------------------------- |
| 0 – 1023      | **Bien conocidos** (_well-known_) | `22` SSH · `53` DNS · `80` HTTP · `443` HTTPS                           |
| 1024 – 49151  | **Registrados**                   | `3306` MySQL · `5432` PostgreSQL · `3389` RDP · `8080` HTTP alternativo |
| 49152 – 65535 | **Dinámicos / efímeros**          | El puerto que el sistema operativo le asigna al **cliente**             |

> [!NOTE]
> Cada sistema operativo usa su propio rango efímero (Linux suele usar `32768–60999`). Por eso, en AWS, las reglas de NACL para el tráfico de respuesta se escriben con `1024–65535`, para cubrirlos todos.

### 4.2 Socket y conexión

#### La analogía

| Concepto     | Analogía                                                   | Ejemplo                                       |
| ------------ | ---------------------------------------------------------- | --------------------------------------------- |
| **IP**       | La dirección de un **edificio**                            | `142.250.78.14`                               |
| **Puerto**   | El número de un **apartamento** dentro del edificio        | `443`                                         |
| **Socket**   | Edificio + apartamento: **un extremo** de la comunicación  | `142.250.78.14:443`                           |
| **Conexión** | **Una llamada** entre un apartamento y otro                | `192.168.1.10:51734` ⇄ `142.250.78.14:443`    |

Un socket es **una punta**. Una conexión necesita **dos puntas**: la tuya y la del servidor.

#### Paso a paso: qué pasa cuando abres una página

**① El servidor abre un socket y se queda escuchando** (estado `LISTEN`):

```
Servidor web:   0.0.0.0:443   LISTEN     ← "atiendo a cualquiera que llegue al puerto 443"
```

**② Tu navegador quiere conectarse.** Le pide al sistema operativo un socket, y el SO le asigna un **puerto efímero libre** al azar (por ejemplo `51734`). Tú nunca eliges ese número.

**③ Se forma la conexión**, que queda identificada por **5 valores** (la _5-tupla_):

| Protocolo | IP origen      | Puerto origen | IP destino      | Puerto destino |
| :-------: | -------------- | :-----------: | --------------- | :------------: |
| TCP       | 192.168.1.10   | **51734**     | 142.250.78.14   | 443            |

**④ La respuesta usa la misma 5-tupla, al revés:**

```
Ida:     192.168.1.10:51734  ──►  142.250.78.14:443
Vuelta:  142.250.78.14:443   ──►  192.168.1.10:51734   ← así tu SO sabe que es para esa pestaña
```

#### ¿Por qué hacen falta los 5 valores? Ejemplo con 3 pestañas y 2 equipos

Abres **3 pestañas** de Google en tu PC, y tu hermano abre **1** desde su celular en la misma casa (antes del NAT):

| Conexión              | Protocolo | IP origen        | Puerto origen | IP destino      | Puerto destino |
| --------------------- | :-------: | ---------------- | :-----------: | --------------- | :------------: |
| Tu pestaña 1          | TCP       | 192.168.1.10     | **51734**     | 142.250.78.14   | 443            |
| Tu pestaña 2          | TCP       | 192.168.1.10     | **51735**     | 142.250.78.14   | 443            |
| Tu pestaña 3          | TCP       | 192.168.1.10     | **51736**     | 142.250.78.14   | 443            |
| Celular de tu hermano | TCP       | **192.168.1.25** | 51734         | 142.250.78.14   | 443            |

- El servidor tiene **un solo** socket escuchando en el `443`, pero **4 conexiones** activas.
- Tus 3 pestañas solo se diferencian por el **puerto origen**.
- El celular puede usar **el mismo** puerto `51734` que tu pestaña 1: no hay conflicto porque **la IP origen es otra**. Basta con que **uno** de los 5 valores cambie para que sea otra conexión.

> [!IMPORTANT]
> Cuando llega un segmento, el sistema operativo **lee los 5 valores y busca a qué conexión pertenece**. Así sabe a qué pestaña entregar cada pedazo de cada página, aunque lleguen mezclados.

#### Míralo en tu PC

Abre una página y ejecuta `netstat -ano` (o `netstat -an | findstr 443`):

```
  Proto  Dirección local        Dirección remota        Estado
  TCP    192.168.1.10:51734     142.250.78.14:443       ESTABLISHED
  TCP    192.168.1.10:51735     142.250.78.14:443       ESTABLISHED
  TCP    192.168.1.10:51740     20.190.160.20:443       ESTABLISHED
```

Cada línea es **una conexión** = una 5-tupla. La “dirección local” es tu socket; la “remota”, el del servidor.

#### ¿Dónde vuelve a aparecer la 5-tupla?

| Dónde                          | Para qué la usa                                                                 |
| ------------------------------ | ------------------------------------------------------------------------------- |
| **NAT** del router de la casa  | Su tabla anota cada 5-tupla para saber a qué equipo devolverle cada respuesta.  |
| **Firewall stateful** / **Security Group** | Recuerda las conexiones abiertas y deja pasar sus respuestas automáticamente. |
| **VPC Flow Logs**              | Registran cada flujo con sus 5 valores y si se aceptó o rechazó.                |
| **Load balancer**              | Mantiene cada conexión con el mismo servidor.                                   |

### 4.3 TCP: confiable y ordenado

#### ¿Por qué hace falta un handshake?

Antes de enviar datos, los dos lados tienen que acordar 3 cosas:

1. **Que los dos están vivos** y que el camino funciona **en ambas direcciones**.
2. **Desde qué número empieza a contar cada uno** (el número de secuencia inicial, **ISN**). Cada lado elige uno **al azar**, por seguridad: así nadie puede adivinarlo e inyectar datos falsos.
3. **Las reglas** de la conexión: el **MSS** (sección 3.4), el tamaño de la **ventana** y opciones como **SACK**.

#### Three-way handshake, paso a paso

Tu navegador (`192.168.1.10:51734`) se conecta al servidor (`142.250.78.14:443`). El cliente elige ISN = **1000** y el servidor ISN = **5000**.

```
  Cliente                                                           Servidor
  SYN-SENT                                                          LISTEN
     │                                                                 │
  ①  │ ── SYN ────────────────────────────────────────────────────────►│
     │    seq=1000            MSS=1460, ventana=64240, SACK ok          │
     │                                                                 │  SYN-RECEIVED
  ②  │◄──────────────────────────────────────────────────── SYN-ACK ── │
     │    seq=5000  ack=1001  MSS=1460, ventana=65535, SACK ok          │
     │                                                                 │
  ③  │ ── ACK ────────────────────────────────────────────────────────►│
     │    seq=1001  ack=5001                                            │
  ESTABLISHED                                                     ESTABLISHED
```

| Paso | Flags      | Qué dice en palabras                                                                                                   |
| :--: | ---------- | ---------------------------------------------------------------------------------------------------------------------- |
|  ①   | `SYN`      | “Quiero conectarme. **Empiezo a contar en 1000.** Puedo recibir segmentos de hasta 1.460 bytes.”                       |
|  ②   | `SYN, ACK` | “Recibí tu 1000; **espero tu byte 1001** (`ack=1001`). Yo **empiezo a contar en 5000**. Mi MSS también es 1.460.”      |
|  ③   | `ACK`      | “Recibí tu 5000; **espero tu byte 5001**.” Desde aquí ya se pueden enviar datos.                                        |

> [!IMPORTANT]
> **¿Por qué `ack = seq + 1`?** El `ack` no dice “recibí el X”, sino **“ya tengo todo hasta antes del X; mándame el X”**. El `SYN` “gasta” un número de secuencia, así que después del `SYN` con `seq=1000` el siguiente esperado es el **1001**.

> [!NOTE]
> **Costo:** el handshake toma **1 viaje de ida y vuelta** (RTT) antes de enviar el primer byte de datos. Si el servidor está a 100 ms, son 100 ms solo para conectarse (y luego el handshake de TLS agrega más). Por eso los navegadores **reutilizan** las conexiones (_keep-alive_) y HTTP/3 usa QUIC, que junta ambos handshakes.

#### Después del handshake: la transferencia de datos

Continuamos el ejemplo. El navegador envía la petición (300 bytes) y el servidor responde con el HTML de 5.000 bytes de la sección 3.3:

```
  Cliente                                                           Servidor
     │ ── GET /  (seq=1001, 300 bytes) ──────────────────────────────►│
     │◄─────────────────────────────────────────────── ack=1301 ─────│  "tengo hasta el 1300"
     │                                                                 │
     │◄──────────────────────────── Seg 1: seq=5001, 1460 bytes ─────│
     │                     ✗ ───── Seg 2: seq=6461, 1460 bytes ──────│  ← se pierde en el camino
     │◄──────────────────────────── Seg 3: seq=7921, 1460 bytes ─────│
     │◄──────────────────────────── Seg 4: seq=9381,  620 bytes ─────│
     │                                                                 │
     │ ── ack=6461 ──────────────────────────────────────────────────►│  "tengo hasta el 6460"
     │ ── ack=6461  (SACK: tengo 7921–9380) ─────────────────────────►│  "sigo esperando el 6461…"
     │ ── ack=6461  (SACK: tengo 7921–10000) ────────────────────────►│  "…pero ya tengo 3 y 4"
     │                                                                 │
     │◄───────────────── Seg 2 retransmitido: seq=6461, 1460 bytes ──│  solo reenvía el que falta
     │ ── ack=10001 ─────────────────────────────────────────────────►│  "tengo los 5.000 bytes"
```

Qué pasó:

1. El servidor envió los **4 segmentos seguidos**, sin esperar un ACK por cada uno (lo permite la **ventana**).
2. El segmento 2 se perdió (por ejemplo, una trama con **FCS malo** que algún equipo descartó en la capa 2).
3. El cliente sigue respondiendo `ack=6461` (“me falta el 6461”): son **ACKs duplicados**. Con **SACK** además avisa qué bloques sí tiene.
4. El servidor ve los ACKs duplicados (o vence su **temporizador de retransmisión**) y **reenvía solo el segmento 2**.
5. El cliente ya tiene todo, lo **ordena** y le entrega los 5.000 bytes al navegador. El navegador **nunca supo** que hubo una pérdida.

#### Los mecanismos de TCP, en resumen

| Mecanismo                 | Qué hace                                                                                                          | En el ejemplo                                 |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------- | --------------------------------------------- |
| **Número de secuencia**   | Numera cada **byte** → el receptor **reordena** y detecta huecos.                                                 | `seq=5001`, `6461`, `7921`, `9381`            |
| **ACK acumulativo**       | “Tengo todo hasta antes del X.”                                                                                   | `ack=6461` = “me falta desde el 6461”         |
| **SACK**                  | “Además, tengo estos bloques sueltos.” Evita reenviar lo que ya llegó.                                            | “tengo 7921–10000”                            |
| **Retransmisión**         | Si un ACK no llega a tiempo, o llegan ACKs duplicados, se **reenvía**.                                            | Se reenvía solo el segmento 2                 |
| **Ventana** (_window_)    | El receptor dice cuántos bytes puede recibir **sin confirmar** → **control de flujo** (no ahogar al receptor).    | Los 4 segmentos salieron sin esperar          |
| **Control de congestión** | Si hay pérdidas, el emisor **baja la velocidad** para no saturar la red; luego vuelve a subir poco a poco.        | Tras la pérdida, el servidor envía más lento  |

#### Cerrar la conexión

Cada lado cierra **su** dirección por separado (por eso son 4 mensajes):

```
  Cliente                                         Servidor
     │ ── FIN ────────────────────────────────────►│   "terminé de enviar"
     │◄───────────────────────────────────── ACK ──│
     │◄───────────────────────────────────── FIN ──│   "yo también terminé"
     │ ── ACK ────────────────────────────────────►│
  TIME-WAIT (espera unos segundos por si llega algo tarde)
```

O de golpe, con **`RST`**: “este puerto no acepta conexiones” o “algo salió mal, corto ya”.

| Flag  | Significado                                    |
| ----- | ---------------------------------------------- |
| `SYN` | Iniciar conexión (sincronizar números)         |
| `ACK` | Confirmo lo recibido                           |
| `FIN` | Terminé de enviar                              |
| `RST` | Cortar la conexión ya                          |
| `PSH` | Entregar los datos a la aplicación sin esperar |

#### Estados de una conexión (los que verás en `netstat`)

| Estado        | Significado                                                             |
| ------------- | ----------------------------------------------------------------------- |
| `LISTEN`      | El servidor espera conexiones en ese puerto.                            |
| `SYN-SENT`    | El cliente envió el `SYN` y espera respuesta.                           |
| `SYN-RECEIVED`| El servidor respondió con `SYN-ACK` y espera el último `ACK`.           |
| `ESTABLISHED` | Conexión abierta: ya se envían datos.                                   |
| `FIN-WAIT` / `CLOSE-WAIT` | Uno de los dos lados ya cerró y el otro todavía no.         |
| `TIME-WAIT`   | Conexión cerrada; se espera un rato por si llegan segmentos atrasados.  |

#### Cuando el handshake falla

| Qué pasa con el `SYN`                                 | Qué ve el cliente                                    | Causa típica                                            |
| ----------------------------------------------------- | ---------------------------------------------------- | ------------------------------------------------------- |
| El servidor responde **`RST`**                        | _Connection refused_ (inmediato)                     | Nadie escucha en ese puerto (el servicio está caído).   |
| **No llega respuesta**: el cliente reenvía el `SYN` varias veces | _Connection timed out_ (tras varios segundos) | Un firewall / Security Group **descarta en silencio**, ruta incorrecta o equipo apagado. |

> [!WARNING]
> **SYN flood:** un atacante envía miles de `SYN` y nunca completa el handshake, para llenar la tabla de conexiones a medio abrir del servidor. Defensas: _SYN cookies_ en el servidor y, en AWS, **AWS Shield** (incluido gratis en su versión Standard).

### 4.4 UDP: rápido y sin garantías

- **No hay handshake**, ni ACK, ni retransmisión, ni orden. Encabezado de solo **8 bytes**.
- Si se pierde un datagrama, **se pierde**. La aplicación decide si le importa.

|                      | TCP                                       | UDP                                                                     |
| -------------------- | ----------------------------------------- | ----------------------------------------------------------------------- |
| Conexión             | Sí (handshake)                            | No                                                                      |
| Confiable / ordenado | Sí                                        | No                                                                      |
| Encabezado           | 20 bytes                                  | 8 bytes                                                                 |
| Latencia             | Mayor                                     | Menor                                                                   |
| Usos                 | HTTP/1.1 y 2, SSH, bases de datos, correo | DNS, VoIP, videollamadas, juegos, streaming en vivo, **HTTP/3 (QUIC)** |

> [!TIP]
> **¿Por qué la videollamada usa UDP?** Un fragmento de audio que llega tarde ya no sirve: es mejor perderlo que esperar una retransmisión.
> **¿Y DNS?** Una pregunta y una respuesta pequeñas: no vale la pena un handshake. (Si la respuesta es muy grande, DNS usa TCP.)

---

## 5. Viaje completo: abrir una página web

Escribes `https://ejemplo.com` en el navegador desde tu casa.

```
PC (192.168.1.10) ─ Wi-Fi ─ [router casa / NAT] ═ fibra ═ ISP ═ … Internet … ═ servidor (93.184.215.14)
```

|  #  | Qué pasa                                                                                                                                              | Capas que trabajan |
| :-: | ----------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------ |
|  0  | **Al conectarte al Wi-Fi**, DHCP te dio IP, máscara, gateway y DNS. Ver [`05-dns-dhcp-nat`](05-dns-dhcp-nat.md).                                      | 7 → 1              |
|  1  | **DNS:** el navegador pregunta “¿qué IP tiene `ejemplo.com`?” → consulta **UDP al puerto 53** del servidor DNS.                                       | 7 · 4 (UDP)        |
|  2  | El DNS no está en tu red → **¿local o remoto?** → remoto → **ARP** por la MAC del **gateway**.                                                        | 3 · 2              |
|  3  | Llega la respuesta: `93.184.215.14`.                                                                                                                  | 7                  |
|  4  | **TCP handshake** al puerto **443**: `SYN` → `SYN-ACK` → `ACK`. Puerto origen efímero, ej. `51734`.                                                   | 4                  |
|  5  | **TLS handshake:** se verifica el certificado y se acuerdan las claves de cifrado. Ver [`07-tls-https`](07-tls-https.md).                             | 6 · 5              |
|  6  | **HTTP:** `GET /` viaja **cifrado** dentro de TLS, dentro de TCP, dentro de IP, dentro de una trama Wi-Fi.                                            | 7 → 1              |
|  7  | En el **router de casa**: la trama Wi-Fi se desarma, el **NAT** cambia la IP origen `192.168.1.10` → IP pública, y sale una trama nueva por la fibra. | 3 · 2 · 1          |
|  8  | **Decenas de routers** en el camino: en cada uno, nueva trama (MAC nueva), TTL − 1, misma IP destino. Lo ves con `tracert`.                           | 3 · 2 · 1          |
|  9  | El servidor desarma todo: trama → paquete → segmento (puerto 443 → servidor web) → TLS → HTTP. Responde con `200 OK` y el HTML.                       | 1 → 7              |
| 10  | La respuesta vuelve por el camino inverso; el NAT de tu router la traduce de vuelta a `192.168.1.10:51734`.                                           | 3 · 4              |

> [!IMPORTANT]
> En este viaje se ven todas las reglas juntas:
>
> - La **MAC** cambia en cada salto; la **IP destino** nunca (paso 8).
> - La **IP origen** cambia una vez, por **NAT** (paso 7).
> - El **puerto** identifica la aplicación y la conexión (pasos 4 y 10).
> - **TLS** protege el contenido de punta a punta; el cifrado Wi-Fi solo protegía el tramo del aire.

---

## 6. Internet: una red de redes

Nadie es “dueño de Internet”: es la **interconexión** de decenas de miles de redes independientes.

| Pieza                          | Qué es                                                                                                                                                             |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Sistema autónomo (AS)**      | Una red con una sola política de enrutamiento: tu ISP, Google, AWS, una universidad. Se identifica con un **ASN**.                                                 |
| **BGP**                        | El protocolo con el que los AS se anuncian entre sí qué redes (bloques de IP) pueden alcanzar. Ver [`04-routing-switching`](04-routing-switching.md#10-bgp-y-asn). |
| **Tránsito**                   | Un AS **le paga** a otro más grande para llegar al resto de Internet.                                                                                              |
| **Peering**                    | Dos AS intercambian tráfico **directamente**, normalmente sin pagarse.                                                                                             |
| **IXP** (punto de intercambio) | Edificio donde muchos AS se conectan para hacer peering. En Colombia: **NAP Colombia**.                                                                            |
| **Cables submarinos**          | Conectan los continentes. A Colombia llegan por la costa Caribe y la Pacífica.                                                                                     |
| **RIR**                        | Reparten las IPs y los ASN por región. Latinoamérica: **LACNIC**.                                                                                                  |

**Del ISP a tu casa:** fibra (GPON), coaxial, DSL o móvil. Ver la sección “Cómo llega Internet a la casa” en [`00-osi-model`](00-osi-model.md).

### ¿“Internet propio”?

Siempre hay que conectarse a otra red. Niveles posibles:

| Nivel                      | Qué implica                                                                                           |
| -------------------------- | ----------------------------------------------------------------------------------------------------- |
| 1️⃣ **Red casera propia**   | Tu router (OpenWrt / OPNsense), VLANs, firewall, servidor local.                                      |
| 2️⃣ **Exponer un servicio** | IP pública del operador (ojo con **CGNAT**) o un túnel (Cloudflare Tunnel, Tailscale, VPN a la nube). |
| 3️⃣ **Ser un AS**           | **ASN** + bloque de IPs de **LACNIC** (requiere una organización legal) + **BGP** con proveedores.    |
| 4️⃣ **Ser ISP**             | En Colombia: inscripción en el **Registro Único de TIC** del MinTIC.                                  |

---

## 7. Diagnóstico

| Qué ver                                | Windows                                    | Linux / macOS                 |
| -------------------------------------- | ------------------------------------------ | ----------------------------- |
| Conexiones abiertas (5-tupla y estado) | `netstat -ano`                             | `ss -tunap`                   |
| ¿Está abierto un puerto remoto?        | `Test-NetConnection ejemplo.com -Port 443` | `nc -zv ejemplo.com 443`      |
| Resolver un nombre                     | `nslookup ejemplo.com`                     | `dig ejemplo.com`             |
| Ver los saltos                         | `tracert ejemplo.com`                      | `traceroute ejemplo.com`      |
| Petición HTTP completa                 | `curl -v https://ejemplo.com`              | `curl -v https://ejemplo.com` |

> [!TIP]
> Abre una página y ejecuta `netstat -ano`: vas a ver tus conexiones con estado `ESTABLISHED`, tu IP con un **puerto efímero** y la IP del servidor con el puerto **443**.

| Error típico             | Capa  | Qué significa                                                                           |
| ------------------------ | :---: | --------------------------------------------------------------------------------------- |
| _Connection refused_     |   4   | Llegaste al equipo, pero nadie escucha en ese puerto (respondió `RST`).                 |
| _Connection timed out_   | 3 / 4 | No hubo respuesta: ruta incorrecta, firewall que descarta en silencio o equipo apagado. |
| _Could not resolve host_ |   7   | Falló el DNS.                                                                           |

---

## 8. En AWS

| Concepto                   | En AWS                                                                                                                                                                                                         |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Puertos**                | Toda regla de **Security Group** y **NACL** es “protocolo + puerto + origen/destino”.                                                                                                                          |
| **Stateful vs. stateless** | El **Security Group** recuerda la conexión (la respuesta sale sola). La **NACL** no: hay que permitir también la respuesta hacia los **puertos efímeros** `1024–65535`. Ver [`06-firewalls`](06-firewalls.md). |
| **L4 vs. L7**              | **NLB** reparte por IP y puerto (TCP/UDP). **ALB** entiende HTTP (URL, host, encabezados). Ver [`09-load-balancing`](09-load-balancing.md).                                                                    |
| **5-tupla**                | Los **VPC Flow Logs** registran justamente: IP origen, IP destino, puerto origen, puerto destino, protocolo, y si se aceptó o rechazó.                                                                         |
| **MTU**                    | 9001 dentro de la VPC (jumbo frames); 1500 hacia Internet.                                                                                                                                                     |
| **Red de redes**           | AWS es un AS con su propia red global; **Direct Connect** y **Site-to-Site VPN** usan BGP para conectarse con tu red.                                                                                          |

---

## 9. Autoevaluación

- [ ] ¿Por qué se estudia OSI si lo que funciona es TCP/IP?
- [ ] ¿Qué capas viven en la tarjeta de red, cuáles en el sistema operativo y cuáles en la aplicación?
- [ ] ¿Qué diferencia hay entre MTU y MSS? ¿Por qué el MSS típico es 1460 bytes?
- [ ] ¿En cuántos segmentos se parte un archivo de 4.000 bytes? ¿Con qué números de secuencia (relativos)?
- [ ] ¿Quién parte los datos en segmentos: la aplicación, TCP o IP?
- [ ] ¿Qué diferencia hay entre segmentación (TCP) y fragmentación (IP)?
- [ ] ¿Por qué bloquear todo el ICMP puede hacer que una página “se quede pegada”?
- [ ] ¿Qué diferencia hay entre un socket y una conexión?
- [ ] ¿Qué 5 valores identifican una conexión?
- [ ] Si abro 3 pestañas al mismo sitio, ¿qué cambia entre las 3 conexiones? ¿Y si otro equipo usa el mismo puerto origen?
- [ ] ¿Qué pasa en cada paso del _three-way handshake_? ¿Por qué `ack = seq + 1`?
- [ ] ¿Qué se negocia en el handshake además de “estoy vivo”?
- [ ] Si se pierde el segmento 2 de 4, ¿qué responde el receptor y qué se reenvía?
- [ ] Si una trama se descarta por FCS, ¿quién se da cuenta y quién la reenvía?
- [ ] ¿Por qué DNS y las videollamadas usan UDP?
- [ ] ¿Qué diferencia hay entre _connection refused_ y _connection timed out_?
- [ ] ¿Por qué una NACL necesita reglas para los puertos efímeros y un Security Group no?
- [ ] ¿Qué diferencia hay entre tránsito y peering?
