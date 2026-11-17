# 🌐 Modelo OSI

> **Fase:** 01 · Networking **Estado:** 🟡 En progreso (capas 1 y 2 ✅)
> **Siguiente tema:** [`01-tcp-ip.md`](01-tcp-ip.md)

---

## 📑 Contenido

1. [¿Qué es el modelo OSI?](#1-qué-es-el-modelo-osi)
2. [Las 7 capas de un vistazo](#2-las-7-capas-de-un-vistazo)
3. [Encapsulamiento](#3-encapsulamiento)
4. [OSI vs. TCP/IP](#4-osi-vs-tcpip)
5. [Capas en detalle](#5-capas-en-detalle)
6. [Diagnóstico capa por capa](#6-diagnóstico-capa-por-capa)
7. [Autoevaluación](#7-autoevaluación)
8. [Recursos](#8-recursos)

---

## 1. ¿Qué es el modelo OSI?

El **modelo OSI** (_Open Systems Interconnection_) es un **modelo de referencia** creado por la ISO en 1984. Divide la comunicación entre dos equipos en **7 capas**, y cada una resuelve **un solo problema**:

- Cada capa **usa los servicios** de la capa de abajo.
- Cada capa **ofrece un servicio** a la capa de arriba.
- Cada capa **solo “habla” con su capa par** en el otro equipo (la capa 3 de PC-A se entiende con la capa 3 del router).

No es un protocolo ni un software: es un **mapa mental**. En la práctica, Internet funciona con **TCP/IP**, pero OSI se sigue usando porque da un **lenguaje común**: decir _“es un problema de capa 3”_ o _“es un balanceador L7”_ se entiende en cualquier equipo de trabajo.

### ¿Para qué me sirve?

| Uso                 | Ejemplo                                                          |
| ------------------- | ---------------------------------------------------------------- |
| 🧠 **Entender**     | Saber qué hace cada dispositivo y cada protocolo.                |
| 🔧 **Diagnosticar** | Revisar un fallo de abajo hacia arriba sin adivinar.             |
| ☁️ **AWS**          | Elegir entre ALB (L7) y NLB (L4), o entre Security Group y NACL. |

> [!TIP]
> **Analogía:** enviar un paquete por correo. Escribes la carta (7), la traduces al idioma del destinatario (6), acuerdas una conversación por cartas (5), la numeras por páginas (4), le pones la dirección de la ciudad (3), el cartero la lleva a la casa correcta del barrio (2) y el camión la transporta físicamente (1).

---

## 2. Las 7 capas de un vistazo

```
        ┌──────────────────────┐
  7     │     Aplicación       │  HTTP · DNS · SSH         ┐
  6     │    Presentación      │  TLS · JSON · UTF-8       ├─ Datos
  5     │       Sesión         │  Sesiones · RPC           ┘
        ├──────────────────────┤
  4     │     Transporte       │  TCP · UDP                → Segmento   · Puerto
        ├──────────────────────┤
  3     │        Red           │  IP · ICMP                → Paquete    · IP
        ├──────────────────────┤
  2     │   Enlace de datos    │  Ethernet · Wi-Fi · ARP   → Trama      · MAC
        ├──────────────────────┤
  1     │       Física         │  Cable · Fibra · Radio    → Bits
        └──────────────────────┘
```

|  #  | Capa                | Pregunta que responde                             | PDU                  | Dirección     | Dispositivos                        |
| :-: | ------------------- | ------------------------------------------------- | -------------------- | ------------- | ----------------------------------- |
|  7  | **Aplicación**      | ¿Qué servicio quiere usar el usuario?             | Datos                | URL / dominio | —                                   |
|  6  | **Presentación**    | ¿En qué formato van los datos? ¿Van cifrados?     | Datos                | —             | —                                   |
|  5  | **Sesión**          | ¿Cómo se abre, mantiene y cierra la conversación? | Datos                | —             | —                                   |
|  4  | **Transporte**      | ¿A qué aplicación va y cómo llega completo?       | Segmento / Datagrama | **Puerto**    | Firewall, LB L4                     |
|  3  | **Red**             | ¿Cómo llego a otra red?                           | Paquete              | **IP**        | **Router**                          |
|  2  | **Enlace de datos** | ¿Para quién es en esta red local y llegó sano?    | Trama                | **MAC**       | **Switch**, NIC, Access Point       |
|  1  | **Física**          | ¿Cómo muevo bits por un medio?                    | Bits                 | —             | Cable, fibra, antena, hub, PHY, ONT |

> [!NOTE]
>
> - **TLS** no encaja perfecto en una sola capa: se dice que vive “entre la 4 y la 7”.
> - **ARP** vive entre la capa 2 y la 3: traduce una IP a una MAC.

> [!TIP]
> **Para memorizar (de 1 a 7):** “**F**ernando **E**s **R**ápido **T**rayendo **S**andías **P**ara **A**na”
> → **F**ísica · **E**nlace · **R**ed · **T**ransporte · **S**esión · **P**resentación · **A**plicación

---

## 3. Encapsulamiento

Al **enviar**, cada capa le **agrega su encabezado** a lo que recibe de arriba.
Al **recibir**, cada capa **quita su encabezado** y entrega el resto hacia arriba.

```
 ENVÍO ↓                                                               ↑ RECEPCIÓN
 Capa 7-5   [                      Datos "Hola"                    ]
 Capa 4     [ TCP (puertos) │ Datos                                ]   → Segmento
 Capa 3     [ IP (IPs) │ TCP │ Datos                               ]   → Paquete
 Capa 2     [ MACs │ Tipo │ IP │ TCP │ Datos │ FCS                 ]   → Trama
 Capa 1     10101010 … 10101011 0100 1011 …  (señales en el cable)     → Bits
```

> [!IMPORTANT]
> **Regla de oro:** la **MAC cambia en cada salto** (cada router arma una trama nueva), pero la **IP de origen y destino no cambia** de extremo a extremo (salvo que haya NAT).
>
> En cada salto el proceso es el mismo: **señales → bits** (capa 1) **→ trama** (capa 2) **→ decisión → bits → señales** por el siguiente medio.

---

## 4. OSI vs. TCP/IP

OSI sirve para **explicar y diagnosticar**; TCP/IP es lo que **realmente funciona** en Internet.

| Modelo TCP/IP | Capas OSI | Protocolos          |
| ------------- | :-------: | ------------------- |
| Aplicación    | 7 · 6 · 5 | HTTP, DNS, TLS, SSH |
| Transporte    |     4     | TCP, UDP            |
| Internet      |     3     | IP, ICMP            |
| Acceso a red  |   2 · 1   | Ethernet, Wi-Fi     |

---

## 5. Capas en detalle

**Leyenda:** ✅ entendida · 🟡 estudiando · ⏳ pendiente

### 🔌 Capa 1 · Física ✅

> **Qué hace:** convierte bits en señales (eléctricas, de luz o de radio) y al revés. **No sabe qué significan los bits ni para quién son.**

| PDU  | Dirección | Dispositivos                                   | En AWS                                                 |
| ---- | --------- | ---------------------------------------------- | ------------------------------------------------------ |
| Bits | —         | Cable, fibra, antena, hub, repetidor, PHY, ONT | La gestiona AWS (_Direct Connect_ es tu enlace físico) |

#### Tres formas de mover bits

| Medio                                      | Cómo viaja el bit                                                                                                                                                                        | Ventajas / límites                                                               |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| 🟠 **Cobre** (UTP Cat5e, Cat6)             | Variaciones de **voltaje**.                                                                                                                                                              | Barato · hasta **100 m** · sensible a interferencia (EMI).                       |
| 🔵 **Fibra óptica** (monomodo / multimodo) | Un **láser** se enciende, se apaga o cambia de intensidad dentro de un hilo de vidrio. La luz no se escapa porque rebota en las paredes (**reflexión total interna**).                   | Inmune a la interferencia eléctrica · llega a **kilómetros**.                    |
| 🟣 **Radio** (Wi-Fi)                       | **Ondas electromagnéticas** (como la luz, pero de menor frecuencia: 2,4 / 5 / 6 GHz). La corriente en la antena emisora crea la onda, y la onda mueve electrones en la antena receptora. | No necesitan aire (funcionan en el vacío) · paredes, metal y agua las debilitan. |

> [!NOTE]
> **2,4 GHz vs. 5 GHz:** 2,4 GHz (onda de ~12,5 cm) atraviesa mejor las paredes; 5 GHz (~6 cm) es más rápida pero tiene menos alcance.

#### Codificación y modulación

Cómo se representan los bits como señal:

| Medio     | Técnica                                                                                                                                                                     |
| --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Cable** | 100BASE-TX: 4B/5B + MLT-3 · 1000BASE-T: PAM-5 por los 4 pares.                                                                                                              |
| **Radio** | **QAM:** se cambian la amplitud y la fase de la onda (256-QAM = 8 bits por símbolo · 1024-QAM = 10 bits). **OFDM:** el canal se divide en muchas subportadoras en paralelo. |

#### Conceptos clave

| Concepto                          | Resumen                                                                                                                                                                                                   |
| --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sincronización**                | Preámbulo `10101010` × 7 ajusta el reloj; SFD `10101011` marca el inicio de la trama.                                                                                                                     |
| **Límites físicos**               | UTP hasta **100 m**; atenuación e interferencia (EMI) dañan bits.                                                                                                                                         |
| **Medio dedicado vs. compartido** | El cable entre un equipo y el switch es **dedicado**. El aire (Wi-Fi) y la fibra GPON son **compartidos** → hacen falta **turnos** para no hablar a la vez y **cifrado** para que otros no lean lo ajeno. |
| **Ancho de banda vs. latencia**   | Cuántos bits por segundo caben vs. cuánto tarda un bit en llegar.                                                                                                                                         |
| **Dúplex**                        | _Half_: por turnos · _Full_: enviar y recibir a la vez.                                                                                                                                                   |

#### Hub vs. switch

|            | Hub                                                                                                              | Switch                                                                                       |
| ---------- | ---------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| Capa       | 1                                                                                                                | 2                                                                                            |
| Qué hace   | Repetidor multipuerto: **regenera** la señal bit a bit, pero la copia a **todos** los puertos sin entender nada. | Recibe la trama completa, revisa el FCS y la reenvía **solo** por el puerto que corresponde. |
| Colisiones | Todos comparten **un** dominio de colisión (half duplex).                                                        | Cada puerto es su **propio** dominio de colisión (full duplex).                              |

---

### 🔗 Capa 2 · Enlace de datos ✅

> **Qué hace:** les da sentido a los bits **dentro de la misma red local**: quién envía, para quién es y si llegó sana.

| PDU   | Dirección | Dispositivos              | En AWS                                                                        |
| ----- | --------- | ------------------------- | ----------------------------------------------------------------------------- |
| Trama | MAC       | Switch, NIC, Access Point | Cada ENI tiene MAC, pero AWS no expone la capa 2 (no hay broadcast en la VPC) |

> [!IMPORTANT]
> **La MAC no valida señales.** La capa 1 recupera los bits · el **FCS** valida que llegaron sanos · la **MAC** dice para quién son.

#### Estructura de la trama Ethernet

```
| MAC destino | MAC origen | EtherType | Datos (46–1500 B) | FCS  |
|   6 bytes   |  6 bytes   |  2 bytes  |      payload      | 4 B  |
```

El preámbulo y el SFD van antes, en la frontera con la capa 1.

#### Conceptos clave

| Concepto                   | Resumen                                                                                                                                                                                                           |
| -------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **MAC**                    | 48 bits (6 bytes). Los primeros 3 bytes (**OUI**) identifican al fabricante.                                                                                                                                      |
| **Tipos de destino**       | _Unicast_ (uno) · _Broadcast_ `FF:FF:FF:FF:FF:FF` (todos) · _Multicast_ (un grupo).                                                                                                                               |
| **FCS / CRC**              | Detecta errores **accidentales**. Si falla, la trama se **descarta en silencio**; reenviar le toca a capas superiores como TCP.                                                                                   |
| **EtherType**              | `0x0800` IPv4 · `0x0806` ARP · `0x86DD` IPv6.                                                                                                                                                                     |
| **Recepción en la NIC**    | Compara la MAC de destino con la suya. Si no es suya (ni broadcast ni un multicast que le interese), la descarta **sin molestar al sistema operativo**.                                                           |
| **Switch**                 | **Aprende** la MAC de origen · **busca** la MAC de destino · si no la conoce → _flooding_. Un switch _store-and-forward_ revisa el FCS antes de reenviar, así que no propaga tramas dañadas.                      |
| **Broadcast vs. flooding** | _Broadcast_: el **emisor** le habla a todos a propósito (`FF:FF:…`) y todos lo leen. _Flooding_: el **switch** envía una trama unicast a todos porque aún no sabe dónde está el destino; solo el dueño la acepta. |
| **ARP**                    | Traduce IP → MAC **dentro de la red local**. “¿Quién tiene la IP X?” → pregunta en _broadcast_, respuesta en _unicast_; se guarda en la **caché ARP** (`arp -a`).                                                 |
| **MTU**                    | 1500 bytes de datos por trama (Ethernet estándar).                                                                                                                                                                |
| **Dominio de broadcast**   | Hasta dónde llega un broadcast. El switch **no** lo corta; el router **sí**.                                                                                                                                      |
| **VLAN (802.1Q)**          | Divide un switch en varias redes lógicas = varios dominios de broadcast.                                                                                                                                          |

#### Wi-Fi (802.11) vs. Ethernet (802.3)

|                     | Ethernet (802.3)            | Wi-Fi (802.11)                                                              |
| ------------------- | --------------------------- | --------------------------------------------------------------------------- |
| Dispositivo central | Switch                      | **Access Point (AP)**                                                       |
| MACs en la trama    | 2 (origen y destino)        | Hasta **4** (origen, destino, AP y una más en enlaces entre APs)            |
| Confirmación        | No (de eso se encarga TCP)  | Cada trama unicast se confirma con un **ACK**; si no llega, se retransmite. |
| Acceso al medio     | Full duplex, sin colisiones | **CSMA/CA**: escuchar antes de hablar para no transmitir a la vez           |

El AP **traduce** la trama 802.11 a una trama Ethernet 802.3 normal al pasarla al cable.

#### Seguridad en capa 2

| Medio     | Cómo se protege                                                                                                                                                              |
| --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Cable** | La seguridad es **física** (hay que conectarse al switch). En empresas se puede usar **802.1X** por puerto.                                                                  |
| **Wi-Fi** | **WPA2 / WPA3.** La clave nunca viaja por el aire: con el **4-way handshake** ambos lados demuestran que la conocen y derivan una **clave de sesión única por dispositivo**. |

- En la trama Wi-Fi el **encabezado** (con las MACs) va **en claro**; los **datos** van **cifrados**.
- El **MIC** detecta modificaciones **intencionales**; el FCS solo detecta daños **accidentales**.

> [!WARNING]
> El cifrado Wi-Fi solo protege **el tramo por el aire**. Para proteger de punta a punta se usa **TLS / HTTPS** (capas superiores).

---

### 🏠 Aplicado: cómo llega Internet a la casa

Lo que se paga es el **acceso a Internet**; el Wi-Fi es solo el último tramo, de unos metros, dentro de la casa.

```
Central del ISP ══ fibra ══ postes ══ splitter GPON ══ fibra ══ casa
                                                                │
                                         [ ONT + router + switch + AP ]
                                                                │
                                              cable / Wi-Fi → dispositivos
```

#### La “cajita” del operador: 4 aparatos en uno

| Aparato    | Función                                        | Capa  |
| ---------- | ---------------------------------------------- | :---: |
| **ONT**    | Luz ↔ bits                                     |   1   |
| **Switch** | Los puertos LAN de atrás                       |   2   |
| **AP**     | Bits ↔ ondas de radio                          | 1 y 2 |
| **Router** | Une la red de la casa con la del operador (IP) |   3   |

#### GPON

Una fibra se reparte entre **32–64 casas** con un splitter **pasivo** (sin electricidad).

- **Bajada:** todas las ONT reciben todo; cada una toma lo suyo, y el tráfico va **cifrado**.
- **Subida:** cada casa transmite en **turnos de tiempo** asignados por la central.

**Otras tecnologías de acceso:** coaxial (HFC / DOCSIS → cable módem) · par telefónico (ADSL / VDSL → módem DSL) · móvil 4G / 5G · satélite.

> [!NOTE]
> Cada tramo tiene su **propia capa 1** (radio → cobre → luz); la **capa 2 se arma y se desarma** en cada salto; en el router la decisión ya se toma **por IP** (capa 3).

#### ¿“Internet propio”?

Internet es una **red de redes**: siempre hay que conectarse a otra red. Estos son los niveles posibles:

| Nivel                           | Qué implica                                                                                                                              |
| ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| 1️⃣ **Red casera propia**        | Router con OpenWrt / OPNsense, VLANs, firewall, servidor local (Raspberry Pi, Pi-hole, Nextcloud).                                       |
| 2️⃣ **Exponer un servicio**      | IP pública del operador (ojo con **CGNAT**), o un túnel (Cloudflare Tunnel, Tailscale, VPN a una instancia en la nube).                  |
| 3️⃣ **Red con identidad propia** | **ASN** + bloque de IPs asignados por **LACNIC** (requiere una organización legalmente constituida) y **BGP** con uno o más proveedores. |
| 4️⃣ **Ser ISP**                  | En Colombia, inscripción en el **Registro Único de TIC** del MinTIC.                                                                     |

---

### 🗺️ Capa 3 · Red 🟡

> **Qué hace:** lleva un paquete **entre redes distintas**. La capa 2 solo llega hasta el router; desde ahí manda la capa 3.

#### Puente entre capa 2 y capa 3

**Capa 3** decide el **destino final** y el siguiente salto · **capa 2** lleva la trama **hasta ese siguiente salto**.

| Puente               | Qué resuelve                                                                                                               |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **EtherType**        | `0x0800` le dice a la capa 2 que el contenido es IPv4 y debe subirlo a la capa 3.                                          |
| **¿Local o remoto?** | El equipo compara la IP de destino con su red (usando la máscara). Local → entrega directa. Remoto → al **gateway**.       |
| **ARP**              | Consigue la MAC de la IP a la que se entregará la trama: la del destino (si es local) o la del **gateway** (si es remoto). |

> [!IMPORTANT]
> Si el destino está en otra red, la trama lleva **MAC destino = gateway**, pero el paquete lleva **IP destino = servidor final**. Nunca se hace ARP por la IP de un equipo de otra red.

📘 Ejemplo completo paso a paso (PC-A → router → servidor): [`04-routing-switching`](04-routing-switching.md)

**Temas que vienen:** por qué la MAC solo sirve dentro de la red local, qué hace el router, IP y subredes, tablas de rutas, gateway por defecto, NAT / CGNAT, y BGP + ASN (cómo se conectan las redes entre sí). Es la **base directa de las VPC de AWS**.

| PDU     | Dirección | Dispositivos | En AWS                                         |
| ------- | --------- | ------------ | ---------------------------------------------- |
| Paquete | IP        | Router       | VPC, subnets, _route tables_, IGW, NAT Gateway |

#### Conceptos clave

| Concepto                    | Resumen                                                                                            | Profundizar                                       |
| --------------------------- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------- |
| **Dirección IP**            | Parte de red + parte de host.                                                                      | [`02-ipv4-cidr`](02-ipv4-cidr.md)                 |
| **Máscara y CIDR**          | `/24` = `255.255.255.0` → 256 direcciones.                                                         | [`02-ipv4-cidr`](02-ipv4-cidr.md)                 |
| **Subnetting**              | Dividir una red en redes más pequeñas.                                                             | [`03-subnetting`](03-subnetting.md)               |
| **IPs privadas (RFC 1918)** | `10.0.0.0/8` · `172.16.0.0/12` · `192.168.0.0/16`. **Las usan todas las VPC.**                     | [`02-ipv4-cidr`](02-ipv4-cidr.md)                 |
| **Gateway por defecto**     | A dónde envío el paquete si el destino no está en mi red.                                          | [`04-routing-switching`](04-routing-switching.md) |
| **Tabla de rutas**          | El router decide el siguiente salto.                                                               | [`04-routing-switching`](04-routing-switching.md) |
| **TTL**                     | Cada router le resta 1; en 0 se descarta (evita bucles).                                           | —                                                 |
| **ICMP**                    | Mensajes de control: `ping`, `traceroute`.                                                         | —                                                 |
| **NAT / CGNAT**             | Traduce IP privada ↔ pública. Con CGNAT el operador comparte una IP pública entre muchos clientes. | [`05-dns-dhcp-nat`](05-dns-dhcp-nat.md)           |
| **BGP + ASN**               | Cómo se conectan entre sí las redes que forman Internet.                                           | [`04-routing-switching`](04-routing-switching.md) |
| **IPv6**                    | Direcciones de 128 bits.                                                                           | [`11-ipv6`](11-ipv6.md)                           |

---

### 🚚 Capa 4 · Transporte ⏳

> **Qué hace:** entrega los datos a la **aplicación correcta** (puerto) y, con TCP, garantiza que lleguen **completos y en orden**.

| PDU                              | Dirección | Dispositivos    | En AWS                         |
| -------------------------------- | --------- | --------------- | ------------------------------ |
| Segmento (TCP) / Datagrama (UDP) | Puerto    | Firewall, LB L4 | Security Groups, NACL, **NLB** |

#### TCP vs. UDP

|           | TCP                                             | UDP                      |
| --------- | ----------------------------------------------- | ------------------------ |
| Conexión  | Sí (_three-way handshake_: SYN → SYN-ACK → ACK) | No                       |
| Confiable | Sí: confirma, reordena y reenvía                | No                       |
| Velocidad | Más lento                                       | Más rápido               |
| Uso       | HTTP, SSH, bases de datos                       | DNS, video, juegos, VoIP |

#### Conceptos clave

| Concepto                   | Resumen                                                                                       | Profundizar                                 |
| -------------------------- | --------------------------------------------------------------------------------------------- | ------------------------------------------- |
| **Puertos a memorizar**    | `22` SSH · `53` DNS · `80` HTTP · `443` HTTPS · `3306` MySQL · `5432` PostgreSQL · `3389` RDP | —                                           |
| **Socket**                 | IP + puerto, ej. `192.168.1.20:443`.                                                          | —                                           |
| **Puertos efímeros**       | Puerto alto y aleatorio del cliente (`1024–65535`). **Clave para las NACL.**                  | [`06-firewalls`](06-firewalls.md)           |
| **Stateful vs. stateless** | Security Group recuerda la conexión; NACL no.                                                 | [`06-firewalls`](06-firewalls.md)           |
| **Load balancer L4**       | Reparte por IP y puerto (NLB).                                                                | [`09-load-balancing`](09-load-balancing.md) |

---

### 🤝 Capas 5 y 6 · Sesión y Presentación ⏳

> **Qué hacen:** la 5 **organiza la conversación**; la 6 **da formato** a los datos. En la práctica se estudian junto con la capa 7.

| Capa                 | Responsabilidad                                  | Ejemplos                             | Profundizar                       |
| -------------------- | ------------------------------------------------ | ------------------------------------ | --------------------------------- |
| **5 · Sesión**       | Abrir, mantener y cerrar la conversación.        | Sesiones, _keep-alive_, reanudar TLS | —                                 |
| **6 · Presentación** | Formato, codificación, compresión y **cifrado**. | TLS, JSON, UTF-8, gzip, JPEG         | [`07-tls-https`](07-tls-https.md) |

---

### 💻 Capa 7 · Aplicación ⏳

> **Qué hace:** define los **protocolos que usan las aplicaciones** (no es la aplicación en sí: Chrome no es capa 7, HTTP sí).

| PDU   | Dirección     | Dispositivos      | En AWS                                          |
| ----- | ------------- | ----------------- | ----------------------------------------------- |
| Datos | URL / dominio | Proxy, WAF, LB L7 | **ALB**, Route 53, API Gateway, CloudFront, WAF |

#### Conceptos clave

| Concepto             | Resumen                                                                      | Profundizar                                 |
| -------------------- | ---------------------------------------------------------------------------- | ------------------------------------------- |
| **HTTP**             | Métodos (GET, POST, PUT, DELETE), códigos (2xx, 3xx, 4xx, 5xx), encabezados. | [`10-http-and-apis`](10-http-and-apis.md)   |
| **DNS**              | Nombre → IP. Registros A, AAAA, CNAME, MX.                                   | [`05-dns-dhcp-nat`](05-dns-dhcp-nat.md)     |
| **DHCP**             | Asigna IP, máscara, gateway y DNS automáticamente.                           | [`05-dns-dhcp-nat`](05-dns-dhcp-nat.md)     |
| **Load balancer L7** | Enruta según URL, host o encabezados (ALB).                                  | [`09-load-balancing`](09-load-balancing.md) |
| **WAF**              | Firewall que entiende HTTP (bloquea SQLi, XSS…).                             | [`06-firewalls`](06-firewalls.md)           |

---

## 6. Diagnóstico capa por capa

Cuando algo “no funciona”, se revisa **de abajo hacia arriba**:

| Capa | Pregunta                               | Herramienta (Windows)                 |
| :--: | -------------------------------------- | ------------------------------------- |
|  1   | ¿Hay enlace? ¿Cable conectado?         | Luces del puerto                      |
|  2   | ¿Conozco la MAC del gateway?           | `arp -a`                              |
|  3   | ¿Llego a la IP? ¿Hay ruta?             | `ping` · `tracert` · `route print`    |
|  4   | ¿El puerto está abierto?               | `Test-NetConnection <host> -Port 443` |
|  7   | ¿Resuelve el nombre? ¿Responde la app? | `nslookup` · `curl`                   |

> [!TIP]
> **En AWS es igual:** _route table_ (3) → Security Group / NACL (4) → la aplicación (7).

---

## 7. Autoevaluación

- [ ] ¿Por qué la MAC cambia en cada salto y la IP no?
- [ ] ¿Qué diferencia hay entre un switch y un router?
- [ ] ¿Qué diferencia hay entre un broadcast y un flooding?
- [ ] Si PC-A quiere hablar con un servidor de otra red, ¿por la MAC de quién pregunta con ARP?
- [ ] Si el FCS no coincide, ¿quién se entera y quién vuelve a enviar los datos?
- [ ] ¿Por qué un switch no corta los broadcasts?
- [ ] ¿Por qué el Wi-Fi necesita ACK y CSMA/CA y el cable no?
- [ ] ¿Qué diferencia hay entre el FCS y el MIC?
- [ ] ¿Por qué WPA2 no basta para proteger una contraseña que envío a un sitio web?
- [ ] ¿Qué hace cada uno de los 4 aparatos de la “cajita” del operador?
- [ ] ¿En qué capa trabajan un ALB y un NLB?
- [ ] ¿Por qué una NACL necesita reglas para los puertos efímeros y un Security Group no?

---

## 8. Recursos

| Tipo             | Recurso |
| ---------------- | ------- |
| 🎥 Video         |         |
| 📄 Documentación |         |
| 🧪 Lab           |         |
