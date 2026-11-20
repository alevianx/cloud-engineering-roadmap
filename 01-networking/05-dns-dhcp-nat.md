# 🧭 DNS, DHCP y NAT

> **Fase:** 01 · Networking **Estado:** 🟡 En progreso
> **Anterior:** [`04-routing-switching.md`](04-routing-switching.md) · **Siguiente:** [`06-firewalls.md`](06-firewalls.md)
> **Base:** IPs privadas y CGNAT en [`02-ipv4-cidr`](02-ipv4-cidr.md#8-rangos-especiales-privadas-públicas-y-reservadas) · 5-tupla en [`01-tcp-ip`](01-tcp-ip.md#42-socket-y-conexión)

---

## 📑 Contenido

1. [Los tres servicios invisibles](#1-los-tres-servicios-invisibles)
2. [DHCP: ¿quién soy en esta red?](#2-dhcp-quién-soy-en-esta-red)
3. [DNS: ¿dónde está ese nombre?](#3-dns-dónde-está-ese-nombre)
4. [NAT: ¿cómo salgo a Internet?](#4-nat-cómo-salgo-a-internet)
5. [Los tres juntos: conectarte y abrir una página](#5-los-tres-juntos-conectarte-y-abrir-una-página)
6. [Diagnóstico](#6-diagnóstico)
7. [En AWS](#7-en-aws)
8. [Ejercicios](#8-ejercicios)
9. [Autoevaluación](#9-autoevaluación)

---

## 1. Los tres servicios invisibles

Hasta aquí vimos **cómo** viajan los paquetes. Pero antes de enviar el primero, tu equipo necesita resolver tres preguntas, y lo hace sin que te des cuenta:

| Servicio | Pregunta que responde                              | Cuándo actúa                           | Capa · Puerto           |
| -------- | -------------------------------------------------- | -------------------------------------- | ----------------------- |
| **DHCP** | ¿Qué IP tengo yo, cuál es mi gateway y mi DNS?     | Al **conectarte** a una red            | 7 · UDP `67` / `68`     |
| **DNS**  | ¿Qué IP tiene `google.com`?                        | Antes de **cada conexión** a un nombre | 7 · UDP / TCP `53`      |
| **NAT**  | ¿Cómo salgo a Internet si mi IP es privada?        | En **cada paquete** que sale del router | 3 / 4 (cambia IP y puerto) |

> [!TIP]
> **Analogía:** llegas a vivir a un edificio.
>
> - **DHCP** es la **portería**: te asigna el número de apartamento y te dice dónde queda la salida y la guía telefónica.
> - **DNS** es la **guía telefónica**: buscas “Pizzería X” y te da el número.
> - **NAT** es el **conmutador del edificio**: todas tus llamadas salen con el número principal del edificio, y la portería anota quién llamó para pasarle la respuesta al apartamento correcto.

---

## 2. DHCP: ¿quién soy en esta red?

### 2.1 Qué entrega

**DHCP** (_Dynamic Host Configuration Protocol_) le da a cada equipo, automáticamente, todo lo que necesita para funcionar en la red:

| Dato                    | Ejemplo            | Para qué sirve                                                                  |
| ----------------------- | ------------------ | ------------------------------------------------------------------------------- |
| **IP**                  | `192.168.1.10`     | Tu dirección en esta red.                                                       |
| **Máscara**             | `255.255.255.0`    | Para saber qué es **local** y qué es **remoto** ([`02`](02-ipv4-cidr.md#6-local-o-remoto-la-cuenta-que-hace-cada-equipo)). |
| **Gateway por defecto** | `192.168.1.1`      | A quién enviarle lo que no es local.                                            |
| **Servidores DNS**      | `192.168.1.1`, `8.8.8.8` | A quién preguntarle por los nombres.                                      |
| **Tiempo de préstamo** (_lease_) | `24 horas` | Cuánto tiempo puedes usar esa IP antes de renovarla.                      |
| Dominio de búsqueda     | `empresa.local`    | Para escribir `impresora` en vez de `impresora.empresa.local`.                  |

Sin DHCP tendrías que configurar los 4 primeros datos **a mano** en cada equipo, y si dos personas escogen la misma IP, habrá conflicto.

### 2.2 El proceso DORA, paso a paso

Tu laptop se conecta al Wi-Fi de la casa. **No tiene IP todavía**, y tampoco sabe quién es el servidor DHCP (normalmente es el propio router).

```
  Laptop (sin IP)                                    Servidor DHCP (router 192.168.1.1)
     │                                                         │
  D  │ ── DISCOVER  (broadcast) ──────────────────────────────►│  "¿Hay algún servidor DHCP?"
     │                                                         │
  O  │◄──────────────────────────────────────────── OFFER ──── │  "Te ofrezco la 192.168.1.10"
     │                                                         │
  R  │ ── REQUEST  (broadcast) ───────────────────────────────►│  "Acepto la 192.168.1.10 de 192.168.1.1"
     │                                                         │
  A  │◄────────────────────────────────────────────── ACK ──── │  "Confirmado: es tuya por 24 h"
     │                                                         │
  Ya tiene IP, máscara, gateway y DNS
```

| Paso | Mensaje      | IP origen → destino               | MAC destino           | Contenido                                                               |
| :--: | ------------ | --------------------------------- | --------------------- | ----------------------------------------------------------------------- |
|  D   | **Discover** | `0.0.0.0` → `255.255.255.255`     | `FF:FF:FF:FF:FF:FF`   | “No tengo IP. Mi MAC es AA:..:01. ¿Quién me da una?”                    |
|  O   | **Offer**    | `192.168.1.1` → (a la laptop)     | Broadcast o la MAC de la laptop | “Te ofrezco `192.168.1.10/24`, gateway `.1`, DNS `.1`, por 24 h.” |
|  R   | **Request**  | `0.0.0.0` → `255.255.255.255`     | `FF:FF:FF:FF:FF:FF`   | “Acepto la oferta **del servidor 192.168.1.1**.”                        |
|  A   | **Ack**      | `192.168.1.1` → (a la laptop)     | La MAC de la laptop   | “Confirmado.”                                                           |

Puertos: el **servidor** escucha en **UDP 67** y el **cliente** en **UDP 68**.

> [!IMPORTANT]
> **¿Por qué el Request también es broadcast** si la laptop ya sabe quién le hizo la oferta? Porque en la red puede haber **varios** servidores DHCP que le ofrecieron IP. Al gritarlo a todos, los demás se enteran de que su oferta **no** fue aceptada y la liberan.

> [!NOTE]
> **¿Cómo sabe el servidor a quién le responde, si la laptop no tiene IP?** Por la **MAC** que viene dentro del Discover. Por eso DHCP necesita la capa 2: es de los pocos protocolos que funcionan **antes** de tener IP.

### 2.3 El préstamo (_lease_) y su renovación

La IP es **prestada**, no regalada. Con un lease de 24 horas:

```
 0 h ─────────── 12 h (50 %) ─────────────── 21 h (87,5 %) ──────── 24 h
 │               │                           │                       │
 Recibe la IP    T1: le pide al MISMO        T2: le pide a           Vence: deja de
                 servidor que se la renueve  CUALQUIER servidor      usar la IP y
                 (unicast)                   (broadcast)             empieza de cero
```

Si el equipo está encendido, renueva solo a la mitad del tiempo y casi siempre conserva **la misma IP**. Si te desconectas varios días, al volver puede tocarte otra.

### 2.4 Formas de asignar una IP

| Tipo                   | Cómo funciona                                                               | Cuándo se usa                                         |
| ---------------------- | --------------------------------------------------------------------------- | ----------------------------------------------------- |
| **Estática (manual)**  | La escribes tú en el equipo.                                                | Routers, servidores, equipos de red.                  |
| **Dinámica (DHCP)**    | El servidor la toma de un **pool** (ej. `.100` a `.200`).                   | Laptops, celulares, casi todo.                        |
| **Reserva DHCP**       | El servidor le da **siempre la misma IP** a una MAC concreta.               | Impresoras, cámaras, un servidor casero: IP fija sin configurarla en el equipo. |

> [!TIP]
> Si configuras IPs estáticas, ponlas **fuera del pool** del DHCP (ej. pool `.100–.200`, estáticas `.2–.99`). Si no, el DHCP puede entregarle esa misma IP a otro equipo → **IP duplicada**.

### 2.5 DHCP en redes con varias subredes: el relay

El Discover es **broadcast**, y los broadcast **no cruzan el router** ([`04`](04-routing-switching.md#2-switching-capa-2)). En una empresa con 10 VLANs, ¿hace falta un servidor DHCP en cada una?

No: el router hace de **DHCP relay** (en Cisco: `ip helper-address`). Recibe el broadcast en una VLAN y lo reenvía en **unicast** al servidor DHCP central, indicándole de qué red viene, para que el servidor elija una IP del pool correcto.

```
PC (VLAN 10) ── broadcast ──► Router (relay) ── unicast ──► Servidor DHCP central
                               "viene de 10.0.10.0/24"        (un pool por VLAN)
```

### 2.6 Problemas típicos de DHCP

| Problema                         | Síntoma                                                                 | Causa / solución                                                                             |
| -------------------------------- | ----------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| **No hay respuesta de DHCP**     | El equipo se pone una IP **`169.254.x.x`** (APIPA) → no navega.          | Servidor caído, cable o Wi-Fi mal, VLAN equivocada. ([`02`](02-ipv4-cidr.md#otros-rangos-que-debes-reconocer)) |
| **Pool agotado**                 | Los equipos nuevos no consiguen IP; los que ya estaban funcionan.       | Pool muy pequeño o lease muy largo (ej. en una cafetería con muchos visitantes).             |
| **DHCP falso** (_rogue DHCP_)    | Algunos equipos reciben otro gateway u otro DNS.                         | Alguien conectó un router casero, o un atacante. Defensa: **DHCP snooping** en el switch.    |
| **IP duplicada**                 | Dos equipos se cortan la conexión entre sí.                              | Una IP estática dentro del pool.                                                             |

### 2.7 Comandos

| Qué hacer                               | Windows                            | Linux                 |
| --------------------------------------- | ---------------------------------- | --------------------- |
| Ver IP, gateway, DNS, servidor DHCP y lease | `ipconfig /all`                | `ip addr` · `nmcli dev show` |
| Soltar la IP                            | `ipconfig /release`                | `sudo dhclient -r`    |
| Pedir una IP nueva (hacer DORA otra vez) | `ipconfig /renew`                 | `sudo dhclient`       |

> [!TIP]
> Ejecuta `ipconfig /all` y busca: **Servidor DHCP**, **Concesión obtenida** y **La concesión expira**. Ahí ves el lease real.

---

## 3. DNS: ¿dónde está ese nombre?

### 3.1 Por qué existe

Las personas recuerdan **nombres** (`google.com`); las redes solo entienden **IPs** (`142.250.78.14`). El **DNS** (_Domain Name System_) traduce uno en otro.

Además, separa el **nombre** de la **ubicación**: si Google cambia de servidor, cambia el registro DNS y nadie tiene que aprenderse una IP nueva. En la nube esto es clave: las IPs cambian todo el tiempo, los nombres no.

### 3.2 La jerarquía de nombres

DNS es una **base de datos distribuida** en forma de árbol. Ningún servidor lo sabe todo: cada nivel solo sabe **a quién preguntarle** por el nivel de abajo.

```
                               .   (raíz)
              ┌────────────────┼────────────────┐
             com               co               org        ← TLD (dominio de nivel superior)
         ┌────┴────┐       ┌───┴───┐
      google    ejemplo   gov    edu                       ← dominio (lo que compras)
                  │
          ┌───────┼───────┐
         www     api     mail                              ← subdominios (los creas tú, gratis)
```

Un nombre se lee **de derecha a izquierda**: `www.ejemplo.com.` → raíz (`.`) → `com` → `ejemplo` → `www`. El punto final es la raíz; casi nunca se escribe, pero siempre está.

| Nivel                   | Ejemplo                       | Quién lo administra                                                     |
| ----------------------- | ----------------------------- | ----------------------------------------------------------------------- |
| **Raíz**                | `.`                           | 13 nombres de servidores (`a` a `m.root-servers.net`), con **cientos de copias** repartidas por el mundo (_anycast_). |
| **TLD**                 | `.com`, `.org`, `.co`, `.io`  | Un **registro**: Verisign (`.com`); en Colombia, `.co` lo administra .CO Internet por concesión del MinTIC. |
| **Dominio**             | `ejemplo.com`                 | Tú, después de comprarlo en un **registrador** (GoDaddy, Namecheap, Route 53…). |
| **Subdominio**          | `api.ejemplo.com`             | Tú: creas los que quieras en tu zona.                                   |

### 3.3 Los actores de una consulta

| Actor                                  | Qué hace                                                                                         | Ejemplos                                          |
| -------------------------------------- | ------------------------------------------------------------------------------------------------ | ------------------------------------------------- |
| **Stub resolver** (tu equipo)          | Le hace **una** pregunta al resolver y espera la respuesta final. Revisa antes su caché y el archivo `hosts`. | El sistema operativo, el navegador.              |
| **Resolver recursivo**                 | Hace **todo el trabajo**: pregunta a la raíz, al TLD y al autoritativo, y **guarda en caché**.   | El de tu ISP, `8.8.8.8` (Google), `1.1.1.1` (Cloudflare), el de AWS en la VPC. |
| **Servidores raíz**                    | No conocen la respuesta, pero saben **quién maneja cada TLD**.                                   | `a.root-servers.net`                              |
| **Servidores del TLD**                 | Saben **qué servidores son autoritativos** para cada dominio.                                    | Los de `.com`                                     |
| **Servidor autoritativo**              | Tiene la **respuesta oficial** del dominio: sus registros.                                       | `ns-123.awsdns-45.com` (Route 53), Cloudflare DNS |

### 3.4 Resolución paso a paso

Escribes `www.ejemplo.com` en el navegador, y nadie lo ha consultado antes:

```
                                                        ┌──────────────────┐
                                                   ③ ──►│  Servidor raíz   │
                                                   ◄── ④│      ( . )       │
┌──────────┐  ②  "¿www.ejemplo.com?"  ┌───────────┐     └──────────────────┘
│ Tu PC    │ ───────────────────────► │ Resolver  │     ┌──────────────────┐
│ (stub)   │                          │ recursivo │ ⑤ ─►│  Servidor TLD    │
│          │ ◄─────────────────────── │ (1.1.1.1) │ ◄─ ⑥│     ( .com )     │
└──────────┘  ⑨  "93.184.215.14"      └───────────┘     └──────────────────┘
  ① caché                               caché ⑧         ┌──────────────────┐
                                                   ⑦ ──►│  Autoritativo    │
                                                   ◄────│  de ejemplo.com  │
                                                        └──────────────────┘
```

|  #  | Quién                    | Qué pasa                                                                                                  |
| :-: | ------------------------ | --------------------------------------------------------------------------------------------------------- |
|  ①  | Tu PC                    | Busca en la **caché del navegador**, la **caché del sistema** y el archivo **`hosts`**. No está.           |
|  ②  | Tu PC → resolver         | Pregunta al resolver que le dio el **DHCP**: “¿IP de `www.ejemplo.com`?” (**UDP 53**). Pregunta **recursiva**: “dame la respuesta final”. |
|  ③④ | Resolver → raíz          | “¿`www.ejemplo.com`?” → “No sé, pero `.com` lo manejan **estos servidores**.” (una **referencia**)        |
|  ⑤⑥ | Resolver → TLD `.com`    | “¿`www.ejemplo.com`?” → “No sé, pero `ejemplo.com` lo manejan **`ns1.ejemplo-dns.com`**.”                 |
|  ⑦  | Resolver → autoritativo  | “¿`www.ejemplo.com`?” → “**`A 93.184.215.14`**, TTL 3600.” Esta sí es la respuesta.                       |
|  ⑧  | Resolver                 | **Guarda en caché** la respuesta (y también quiénes manejan `.com` y `ejemplo.com`).                      |
|  ⑨  | Resolver → tu PC         | Te entrega la IP. Tu PC también la guarda en caché.                                                       |

> [!IMPORTANT]
> **Recursiva vs. iterativa:** tu PC hace **una** pregunta recursiva (“resuélvemelo tú”). El resolver hace preguntas **iterativas** (“dime lo que sepas”) y va siguiendo las referencias hasta llegar a la respuesta.

> [!NOTE]
> **Gracias a la caché, esto casi nunca pasa completo.** Si otro usuario del mismo resolver ya preguntó por `ejemplo.com` hace 5 minutos, la respuesta sale de la caché en milisegundos (pasos ③ a ⑦ se saltan). Y el resolver casi siempre tiene ya en caché los servidores de `.com`.

### 3.5 Tipos de registro

Cada dominio tiene una **zona**: un conjunto de **registros**. Estos son los que debes conocer:

| Registro  | Qué guarda                                     | Ejemplo                                                         |
| --------- | ---------------------------------------------- | --------------------------------------------------------------- |
| **A**     | Nombre → **IPv4**                              | `www.ejemplo.com  A  93.184.215.14`                             |
| **AAAA**  | Nombre → **IPv6**                              | `www.ejemplo.com  AAAA  2606:2800:21f:cb07::1`                  |
| **CNAME** | Nombre → **otro nombre** (un alias)            | `blog.ejemplo.com  CNAME  ejemplo.wordpress.com`                |
| **MX**    | A qué servidor llega el **correo** (con prioridad) | `ejemplo.com  MX  10 mail.ejemplo.com`                      |
| **TXT**   | Texto libre: verificaciones y políticas de correo (SPF, DKIM, DMARC) | `ejemplo.com  TXT  "v=spf1 include:_spf.google.com ~all"` |
| **NS**    | Qué servidores son **autoritativos** para la zona | `ejemplo.com  NS  ns-123.awsdns-45.com`                      |
| **SOA**   | Datos de la zona: servidor principal, contacto, número de versión | (uno por zona, se crea solo)                    |
| **PTR**   | **IP → nombre** (búsqueda inversa)             | `14.215.184.93.in-addr.arpa  PTR  www.ejemplo.com`              |
| **SRV**   | Servicio + puerto + servidor                   | `_sip._tcp.ejemplo.com  SRV  10 5 5060 sip.ejemplo.com`         |
| **CAA**   | Qué autoridades pueden emitir certificados TLS para el dominio | `ejemplo.com  CAA  0 issue "amazon.com"`        |

> [!WARNING]
> **Un CNAME no puede ir en la raíz del dominio** (`ejemplo.com`, el llamado _apex_), porque ahí ya existen los registros NS y SOA, y un CNAME no puede convivir con otros registros del mismo nombre. Por eso `www.ejemplo.com` sí puede ser CNAME, pero `ejemplo.com` no.
> **En AWS** esto se resuelve con un registro **Alias** de Route 53 (sección 7).

### 3.6 TTL y caché

Cada registro tiene un **TTL** (_Time To Live_, en segundos): cuánto tiempo pueden guardarlo en caché los resolvers.

| TTL                       | Ventaja                                 | Desventaja                                           |
| ------------------------- | --------------------------------------- | ---------------------------------------------------- |
| **Alto** (ej. 86400 = 1 día) | Menos consultas, respuestas más rápidas | Un cambio tarda **hasta 1 día** en verse en todo el mundo. |
| **Bajo** (ej. 60 = 1 min)    | Los cambios se ven casi de inmediato    | Más consultas (en Route 53 cuestan).                 |

> [!TIP]
> **Antes de migrar un servidor:** baja el TTL a 60 segundos **un día antes** (para que expire el TTL viejo), cambia la IP, y cuando todo funcione vuelve a subirlo. Esto es lo que la gente llama “esperar a que **propague** el DNS”: en realidad no se propaga nada, solo hay que esperar a que **venzan las cachés**.

> [!NOTE]
> **El TTL de DNS no es el TTL de IP.** El de IP cuenta **saltos** (routers); el de DNS cuenta **segundos** de caché. Mismo nombre, conceptos distintos.

### 3.7 Transporte y seguridad

| Tema                    | Detalle                                                                                                                         |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| **UDP 53**              | Lo normal: una pregunta y una respuesta pequeñas, sin handshake ([`01`](01-tcp-ip.md#44-udp-rápido-y-sin-garantías)).           |
| **TCP 53**              | Cuando la respuesta **no cabe** en UDP, y para copiar zonas completas entre servidores (_zone transfer_).                       |
| **DNS en claro**        | Por defecto viaja **sin cifrar**: tu ISP o cualquiera en el Wi-Fi puede ver qué sitios consultas.                               |
| **DoH / DoT**           | DNS cifrado: **DNS over HTTPS** (puerto 443) y **DNS over TLS** (puerto 853). Los navegadores modernos lo traen.                |
| **DNS spoofing**        | Un atacante responde con una IP falsa (por ejemplo, con un **DHCP falso** que te asigna su propio DNS). **HTTPS** te protege: el certificado no coincidiría. |
| **DNSSEC**              | Firma digital de los registros: el resolver puede comprobar que la respuesta es auténtica.                                      |

### 3.8 Comprar un dominio: registrador vs. DNS

Son **dos cosas distintas** que a veces da la misma empresa:

| Pieza                     | Qué hace                                                                 | Ejemplo                                   |
| ------------------------- | ------------------------------------------------------------------------ | ----------------------------------------- |
| **Registrador**           | Te **vende** el nombre y le dice al TLD qué servidores NS son los tuyos. | GoDaddy, Namecheap, Route 53 Domains      |
| **Hosting de DNS**        | Tiene tu **zona** con los registros (A, MX…) y responde las consultas.   | Route 53, Cloudflare                      |

Puedes comprar el dominio en GoDaddy y tener el DNS en Route 53: solo cambias los registros **NS** en el registrador para que apunten a Route 53.

### 3.9 Comandos

| Qué hacer                        | Windows (PowerShell / CMD)                    | Linux / macOS                    |
| -------------------------------- | --------------------------------------------- | -------------------------------- |
| Resolver un nombre               | `nslookup ejemplo.com` · `Resolve-DnsName ejemplo.com` | `dig ejemplo.com`       |
| Un tipo de registro              | `nslookup -type=mx ejemplo.com`               | `dig ejemplo.com MX`             |
| Preguntar a un resolver concreto | `nslookup ejemplo.com 1.1.1.1`                | `dig @1.1.1.1 ejemplo.com`       |
| Ver todo el camino (raíz → TLD → autoritativo) | —                               | `dig +trace ejemplo.com`         |
| Ver la caché DNS del equipo      | `ipconfig /displaydns`                        | `resolvectl statistics`          |
| Borrar la caché DNS              | `ipconfig /flushdns`                          | `sudo resolvectl flush-caches`   |
| Archivo `hosts`                  | `C:\Windows\System32\drivers\etc\hosts`       | `/etc/hosts`                     |

> [!TIP]
> El archivo **`hosts`** gana sobre el DNS. Si escribes `127.0.0.1  facebook.com` ahí, tu equipo nunca le preguntará al DNS por Facebook. Es útil para probar un sitio nuevo **antes** de cambiar el DNS real.

---

## 4. NAT: ¿cómo salgo a Internet?

### 4.1 Por qué existe

Las IPs privadas (`192.168.x.x`, `10.x.x.x`) **no se enrutan en Internet** ([`02`](02-ipv4-cidr.md#privadas-rfc-1918)): si un paquete sale con IP origen `192.168.1.10`, el servidor no tendría a dónde responder, porque millones de casas usan esa misma IP.

**NAT** (_Network Address Translation_) **reescribe** las direcciones del paquete al pasar por el router: cambia la IP privada por una pública a la salida, y la deshace a la vuelta. Es la razón por la que IPv4 sigue funcionando a pesar de que se agotó.

### 4.2 Tipos de NAT

| Tipo                                 | Traducción                                                        | Ejemplo / uso                                                    |
| ------------------------------------ | ----------------------------------------------------------------- | ---------------------------------------------------------------- |
| **NAT estático (1:1)**               | Una IP privada ↔ una IP pública fija.                             | Un servidor que debe ser alcanzable. **Así funciona el Internet Gateway de AWS.** |
| **NAT dinámico**                     | Una IP privada ↔ una IP de un **pool** de públicas, mientras dure. | Casi no se usa: hacen falta muchas IPs públicas.                 |
| **PAT / NAPT** (_NAT overload_)      | **Muchas** IPs privadas ↔ **una** IP pública, distinguiéndolas por el **puerto**. | **El de tu casa.** También el NAT Gateway de AWS. |

Cuando se dice “NAT” sin más, casi siempre se habla de **PAT**.

### 4.3 PAT paso a paso: la tabla NAT

En tu casa, tu PC y el celular de tu hermano abren Google al mismo tiempo, **con el mismo puerto origen** (`51734`). El router tiene la IP pública `181.50.20.30`.

```
 PC        192.168.1.10:51734 ──┐                                 ┌──► Google 142.250.78.14:443
                                ├──► [ Router NAT ] ═══ Internet ═══┤
 Celular   192.168.1.25:51734 ──┘    181.50.20.30                   │
```

**① A la salida**, el router reescribe la IP origen **y**, si hace falta, el puerto origen, y lo anota en su **tabla NAT**:

| Lado interno (privado)   | Lado externo (público)    | Destino              |
| ------------------------ | ------------------------- | -------------------- |
| `192.168.1.10:51734`     | `181.50.20.30:40001`      | `142.250.78.14:443`  |
| `192.168.1.25:51734`     | `181.50.20.30:40002`      | `142.250.78.14:443`  |

```
Paquete del PC antes del router:     192.168.1.10:51734  →  142.250.78.14:443
Paquete del PC después del router:   181.50.20.30:40001  →  142.250.78.14:443
                                     └── cambió ─────┘     └── igual ──────┘
```

**② Google** ve **dos conexiones desde la misma IP** (`181.50.20.30`), con puertos `40001` y `40002`. No sabe que detrás hay dos equipos.

**③ A la vuelta**, la respuesta llega a `181.50.20.30:40002`. El router busca en la tabla → es del **celular** → reescribe el destino a `192.168.1.25:51734` y la entrega.

> [!IMPORTANT]
> El router identifica cada conexión con la **5-tupla** ([`01`](01-tcp-ip.md#42-socket-y-conexión)). Como el **puerto** tiene 16 bits, una sola IP pública puede sostener **decenas de miles** de conexiones simultáneas.

### 4.4 La consecuencia: nadie puede entrar sin permiso

La tabla NAT solo tiene entradas para conexiones **que salieron desde adentro**. Si alguien en Internet envía un paquete a `181.50.20.30:8080` y no hay ninguna entrada para ese puerto, el router **no sabe a qué equipo dárselo** y lo descarta.

| Dirección                 | ¿Funciona por defecto?         | Cómo habilitarlo                                                                                    |
| ------------------------- | ------------------------------ | --------------------------------------------------------------------------------------------------- |
| **Adentro → afuera**      | ✅ Sí                          | —                                                                                                   |
| **Afuera → adentro**      | ❌ No                          | **Port forwarding** (DNAT): “todo lo que llegue al puerto `8080` público, envíalo a `192.168.1.50:80`”. |

> [!WARNING]
> **NAT no es un firewall.** Bloquea las conexiones entrantes **como efecto secundario**, no como política de seguridad. Un port forwarding mal pensado expone el equipo a todo Internet. La seguridad real la da un **firewall** ([`06-firewalls`](06-firewalls.md)).

### 4.5 SNAT vs. DNAT

| Tipo                   | Qué cambia      | Para qué                                     | Ejemplo                                  |
| ---------------------- | --------------- | -------------------------------------------- | ---------------------------------------- |
| **SNAT** (_source_)    | La IP **origen** | Que equipos privados **salgan** a Internet. | Tu casa · NAT Gateway de AWS             |
| **DNAT** (_destination_) | La IP **destino** | Que desde afuera se pueda **entrar** a un equipo privado. | Port forwarding · un load balancer |

### 4.6 CGNAT: NAT sobre NAT

Como las IPv4 se agotaron, muchos operadores ya **no le dan una IP pública a cada casa**. Ponen un NAT gigante en su red y comparten **una** IP pública entre cientos de clientes ([`02`](02-ipv4-cidr.md#agotamiento-de-ipv4)):

```
PC 192.168.1.10 ─► [NAT de tu router] ─► 100.64.23.7 ─► [CGNAT del operador] ─► 190.85.4.20 ─► Internet
                     1.ª traducción        (RFC 6598)      2.ª traducción          (compartida)
```

| Consecuencia                                                   | Solución                                                             |
| -------------------------------------------------------------- | -------------------------------------------------------------------- |
| **Port forwarding imposible**: tú no controlas el segundo NAT. | Un túnel (Cloudflare Tunnel, Tailscale) o una VPN a una instancia en la nube. |
| Algunos juegos y videollamadas P2P funcionan peor.             | Pedir una IP pública al operador (a veces con costo).                |
| Si un vecino hace algo indebido, sitios web pueden **bloquear la IP compartida**. | IPv6, que no necesita NAT ([`11-ipv6`](11-ipv6.md)).    |

### 4.7 Lo que NAT rompe

NAT rompe la idea original de Internet, en la que cada equipo tenía una dirección única y alcanzable:

- **Conexiones entrantes** y aplicaciones **P2P**: necesitan técnicas como **STUN** o **TURN** para atravesar el NAT (videollamadas, juegos).
- Protocolos que **escriben la IP dentro de los datos** (FTP activo, SIP): el NAT cambia el encabezado, pero no el contenido.
- **Rastreo**: detrás de una IP pública puede haber muchas personas.

> [!NOTE]
> Con **IPv6** hay direcciones de sobra y cada equipo puede tener una **pública**. NAT deja de ser necesario, y la protección contra conexiones entrantes la da un firewall, que es donde debió estar siempre.

---

## 5. Los tres juntos: conectarte y abrir una página

Llegas a la casa, la laptop se conecta al Wi-Fi y abres `https://ejemplo.com`:

|  #  | Servicio | Qué pasa                                                                                                   |
| :-: | :------: | ---------------------------------------------------------------------------------------------------------- |
|  1  | **DHCP** | DORA → recibes `192.168.1.10/24`, gateway `192.168.1.1`, DNS `192.168.1.1`.                                |
|  2  |   ARP    | Para hablar con el DNS (el router), ARP por la MAC de `192.168.1.1`.                                       |
|  3  | **DNS**  | Tu PC le pregunta al router; el router (que es un **reenviador** de DNS) le pregunta al resolver del ISP → `93.184.215.14`. |
|  4  | **NAT**  | La consulta DNS del router al ISP ya sale con la IP pública.                                               |
|  5  |   TCP    | Handshake a `93.184.215.14:443` desde `192.168.1.10:51734`.                                                |
|  6  | **NAT**  | El router traduce a `181.50.20.30:40001` y lo anota en su tabla.                                           |
|  7  | TLS/HTTP | La página carga. Cada respuesta pasa por la tabla NAT de vuelta a tu laptop.                               |

El viaje completo capa por capa está en [`01-tcp-ip`](01-tcp-ip.md#5-viaje-completo-abrir-una-página-web).

---

## 6. Diagnóstico

Cuando “no hay Internet”, estos tres servicios explican la mayoría de los casos:

| Síntoma                                                          | Qué falla        | Cómo comprobarlo                                                            |
| ---------------------------------------------------------------- | ---------------- | --------------------------------------------------------------------------- |
| Tu IP es `169.254.x.x`                                           | **DHCP**         | `ipconfig /all` → no hay servidor DHCP ni gateway.                          |
| `ping 8.8.8.8` funciona, pero `ping google.com` no               | **DNS**          | `nslookup google.com` falla → cambia el DNS o revisa el que te dio el DHCP. |
| Llegas al router (`ping 192.168.1.1`) pero no a `8.8.8.8`        | **Gateway / NAT / ISP** | `tracert 8.8.8.8` se corta en el primer o segundo salto.             |
| Cambiaste un registro DNS y algunos todavía ven el sitio viejo   | **Caché DNS**    | Espera el TTL · `ipconfig /flushdns` · `nslookup sitio 1.1.1.1`.            |
| Configuraste port forwarding y desde afuera no entra             | **NAT / CGNAT**  | Compara la IP WAN del router con la de un sitio “cuál es mi IP”. Si no coinciden (o es `100.64.x.x`), hay CGNAT. |
| Algunos equipos reciben un gateway o DNS raro                    | **DHCP falso**   | `ipconfig /all` → revisa qué IP aparece como **Servidor DHCP**.             |

> [!TIP]
> **Orden de diagnóstico:** ¿tengo IP? (DHCP) → ¿llego al gateway? → ¿llego a una IP de Internet? (NAT / ruta) → ¿resuelvo nombres? (DNS). Es el mismo “de abajo hacia arriba” del [`00-osi-model`](00-osi-model.md#6-diagnóstico-capa-por-capa).

---

## 7. En AWS

### 7.1 DHCP en la VPC

| Concepto                 | En AWS                                                                                                                       |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| **Servidor DHCP**        | Lo pone AWS. No lo administras, pero **sí** eliges sus opciones con un **DHCP option set** (dominio, servidores DNS, NTP).    |
| **IP privada**           | Se asigna al crear la interfaz de red (**ENI**) y **no cambia** mientras exista, aunque apagues la instancia.                 |
| **IP pública automática** | **Cambia** si detienes e inicias la instancia. Para que sea fija, usa una **Elastic IP**.                                    |
| **DNS que entrega**      | `AmazonProvidedDNS`: el resolver de la VPC, en la **IP base + 2** (en `10.0.0.0/16` → `10.0.0.2`) y también en `169.254.169.253`. |

### 7.2 DNS: Route 53

**Route 53** es el DNS de AWS (el nombre viene del **puerto 53**). Hace tres trabajos:

| Trabajo                    | Qué es                                                                                                          |
| -------------------------- | --------------------------------------------------------------------------------------------------------------- |
| **Registrador**            | Comprar dominios.                                                                                               |
| **DNS autoritativo**       | Tus zonas: **hosted zone pública** (responde a todo Internet) o **privada** (solo dentro de las VPC que asocies). |
| **Resolver**               | El resolver recursivo de cada VPC (la `.2`). Con **endpoints inbound / outbound** conecta el DNS de la VPC con el de tu oficina. |

**Registro Alias** (exclusivo de Route 53):

| | CNAME | Alias |
|---|-------|-------|
| ¿En el _apex_ (`ejemplo.com`)? | ❌ No | ✅ Sí |
| Apunta a | Cualquier nombre | Recursos de AWS: ALB, CloudFront, S3 (sitio web), API Gateway, otro registro de la zona |
| Costo de las consultas | Se cobran | **Gratis** hacia recursos de AWS |

> [!TIP]
> **Pregunta típica del examen:** “¿Cómo apunto `ejemplo.com` (sin `www`) a un load balancer?” → **Registro Alias**, no CNAME.

**Políticas de enrutamiento** (qué IP responder):

| Política                | Responde…                                                     | Uso                                           |
| ----------------------- | ------------------------------------------------------------- | --------------------------------------------- |
| **Simple**              | Siempre lo mismo.                                             | Un solo recurso.                              |
| **Weighted** (ponderada) | Según porcentajes (ej. 90 % / 10 %).                         | Probar una versión nueva con poco tráfico.    |
| **Latency**             | La región con **menor latencia** para el usuario.             | Apps en varias regiones.                      |
| **Failover**            | El principal; si su **health check** falla, el secundario.    | **Disaster recovery** activo-pasivo.          |
| **Geolocation**         | Según el **país o continente** del usuario.                   | Contenido o leyes por país.                   |
| **Geoproximity**        | Según la **distancia**, con un sesgo ajustable.               | Mover tráfico entre regiones gradualmente.    |
| **Multivalue answer**   | Hasta 8 IPs **sanas** al azar.                                | Reparto simple con health checks.             |
| **IP-based**            | Según la **IP de origen** del usuario.                        | Enviar los clientes de un ISP a un sitio concreto. |

> [!NOTE]
> Los **health checks** de Route 53 vigilan tus recursos, y si uno falla, Route 53 deja de responder con su IP. Es la base del failover entre regiones. Todo esto se profundiza en [`03-aws/08-route53`](../03-aws/08-route53.md).

### 7.3 NAT: Internet Gateway vs. NAT Gateway

```
                        ┌────────────────────── VPC 10.0.0.0/16 ──────────────────────┐
                        │                                                              │
 Internet ◄══► [ IGW ] ◄┼──► Subnet pública 10.0.0.0/24                                │
                        │      ├─ EC2 web (10.0.0.10 ↔ IP pública 3.85.x.x, NAT 1:1)    │
                        │      └─ NAT Gateway (Elastic IP 54.x.x.x)                     │
                        │               ▲                                              │
                        │               │ 0.0.0.0/0 → nat-…                            │
                        │      Subnet privada 10.0.16.0/20                             │
                        │      └─ EC2 app (10.0.16.25, sin IP pública)                 │
                        └──────────────────────────────────────────────────────────────┘
```

|                          | **Internet Gateway (IGW)**                                          | **NAT Gateway**                                                         |
| ------------------------ | ------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| Tipo de NAT              | **1:1** (IP privada ↔ IP pública de la instancia)                   | **PAT**: muchas privadas → su Elastic IP                                |
| Dirección                | **Entrada y salida**                                                | **Solo salida** (nadie de afuera puede iniciar conexiones)              |
| Para quién               | Subnets **públicas**                                                | Subnets **privadas** que necesitan salir (actualizaciones, APIs externas) |
| Dónde vive               | Se conecta a la VPC (uno por VPC)                                   | **Dentro de una subnet pública**, en **una AZ**                         |
| Route table              | Subnet pública: `0.0.0.0/0 → igw-…`                                  | Subnet privada: `0.0.0.0/0 → nat-…`                                     |
| Alta disponibilidad      | Ya es redundante                                                    | Es **zonal**: crea **uno por AZ** para no depender de una sola zona     |
| Costo                    | Gratis                                                              | **Se cobra por hora y por GB** procesado (puede ser caro)               |

> [!IMPORTANT]
> **La instancia nunca ve su IP pública.** Si entras por SSH a una EC2 y ejecutas `ip addr`, solo verás `10.0.0.10`: el Internet Gateway hace la traducción 1:1 afuera de la instancia. Para conocer su IP pública, la instancia le pregunta al **servicio de metadatos** (`169.254.169.254`).

> [!TIP]
> **Preguntas típicas del examen:**
>
> - “Las instancias de una subnet privada necesitan descargar actualizaciones sin ser accesibles desde Internet” → **NAT Gateway** en una subnet pública + ruta `0.0.0.0/0` hacia él.
> - “Que no se caiga si falla una AZ” → **un NAT Gateway por AZ**, y cada subnet privada apunta al de su zona.
> - “Bajar el costo del NAT al acceder a S3 o DynamoDB” → **Gateway VPC Endpoint** (gratis): el tráfico a S3 no pasa por el NAT.
> - “Lo mismo para IPv6” → **Egress-only Internet Gateway** (IPv6 no necesita NAT, pero sí bloquear lo entrante).

---

## 8. Ejercicios

1. Tu laptop tiene la IP `169.254.33.12`. ¿Qué falló y qué comando ejecutas primero?
2. ¿Por qué el DHCP Discover sale con IP origen `0.0.0.0`?
3. Tienes un lease de 8 horas. ¿A qué hora intenta renovarlo el equipo por primera vez?
4. `ping 8.8.8.8` responde, pero el navegador dice “no se puede encontrar el servidor”. ¿Qué servicio falla?
5. Quieres que `ejemplo.com` (sin `www`) apunte a un ALB en AWS. ¿Qué tipo de registro usas y por qué no un CNAME?
6. Vas a cambiar la IP de tu servidor web y el registro A tiene TTL 86400. ¿Qué haces antes?
7. En tu casa, dos equipos se conectan al mismo servidor con el mismo puerto origen. ¿Cómo los distingue el router al recibir las respuestas?
8. La IP WAN de tu router es `100.72.10.5`. ¿Puedes publicar un servidor casero con port forwarding?
9. Una EC2 en una subnet privada no puede descargar paquetes de Internet. La subnet tiene la ruta `0.0.0.0/0 → igw-…`. ¿Qué está mal?
10. Tienes un NAT Gateway en `us-east-1a` y subnets privadas en 3 AZ. ¿Qué riesgo hay?

<details>
<summary>Ver soluciones</summary>

1. **DHCP**: no recibió respuesta y se puso una IP APIPA. Primero `ipconfig /all` (¿hay gateway, servidor DHCP?) y luego `ipconfig /renew`.
2. Porque **todavía no tiene IP**. Por eso también va a `255.255.255.255` y el servidor lo identifica por la **MAC**.
3. A las **4 horas** (T1 = 50 %), pidiéndoselo en unicast al mismo servidor.
4. **DNS**: hay conectividad IP pero no se resuelven nombres. Comprueba con `nslookup`.
5. Un registro **Alias** de Route 53. Un CNAME **no puede ir en el apex**, porque ahí ya existen los registros NS y SOA.
6. Bajar el TTL (ej. a 60) **al menos 1 día antes**, para que venzan las cachés con el TTL viejo; luego cambias la IP y vuelves a subir el TTL.
7. Por la **tabla NAT**: a la salida le dio a cada uno un **puerto público distinto** (`40001`, `40002`), y a la vuelta busca ese puerto.
8. **No**: `100.72.10.5` está en `100.64.0.0/10`, el rango de **CGNAT**. El segundo NAT es del operador. Usa un túnel o pide una IP pública.
9. Una subnet privada **no** debe salir por el IGW (sus instancias no tienen IP pública, así que el IGW no tiene qué traducir). Necesita `0.0.0.0/0 → nat-…`, con el NAT Gateway en una subnet pública.
10. Si se cae `us-east-1a`, las subnets privadas de las **otras dos AZ también pierden salida** a Internet. Solución: un NAT Gateway por AZ.

</details>

---

## 9. Autoevaluación

**DHCP**

- [ ] ¿Qué 4 datos mínimos entrega DHCP?
- [ ] ¿Qué significa cada letra de DORA? ¿Por qué el Request es broadcast?
- [ ] ¿Cómo funciona DHCP si el servidor está en otra subred?
- [ ] ¿Qué significa tener una IP `169.254.x.x`?
- [ ] ¿Qué diferencia hay entre una IP estática y una reserva DHCP?

**DNS**

- [ ] ¿Qué diferencia hay entre un resolver recursivo y un servidor autoritativo?
- [ ] ¿Qué pasos sigue el resolver para encontrar `www.ejemplo.com` sin caché?
- [ ] ¿Para qué sirven los registros A, AAAA, CNAME, MX, NS y TXT?
- [ ] ¿Por qué no se puede poner un CNAME en `ejemplo.com`?
- [ ] ¿Qué es el TTL de DNS y qué tiene que ver con “la propagación”?
- [ ] ¿Cuándo usa DNS TCP en vez de UDP?

**NAT**

- [ ] ¿Por qué hace falta NAT con IPv4?
- [ ] ¿Qué diferencia hay entre NAT 1:1 y PAT?
- [ ] ¿Cómo sabe el router a qué equipo devolver cada respuesta?
- [ ] ¿Por qué NAT no es un firewall?
- [ ] ¿Qué es CGNAT y qué consecuencias tiene?

**AWS**

- [ ] ¿En qué IP está el resolver DNS de una VPC `10.0.0.0/16`?
- [ ] ¿Qué diferencia hay entre un registro Alias y un CNAME?
- [ ] ¿Qué política de Route 53 usarías para disaster recovery activo-pasivo?
- [ ] ¿Qué diferencia hay entre un Internet Gateway y un NAT Gateway?
- [ ] ¿Por qué se crea un NAT Gateway por AZ?
- [ ] ¿Por qué una instancia no ve su IP pública con `ip addr`?
