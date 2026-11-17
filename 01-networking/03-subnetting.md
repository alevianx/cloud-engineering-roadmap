# ✂️ Subnetting

> **Fase:** 01 · Networking **Estado:** 🟡 En progreso
> **Anterior:** [`02-ipv4-cidr.md`](02-ipv4-cidr.md) · **Siguiente:** [`04-routing-switching.md`](04-routing-switching.md)
> **Requisito:** saber calcular red, broadcast y hosts de un prefijo (ver [`02-ipv4-cidr`](02-ipv4-cidr.md#4-calcular-una-red-paso-a-paso)).

---

## 📑 Contenido

1. [¿Qué es y para qué sirve?](#1-qué-es-y-para-qué-sirve)
2. [La idea clave: tomar bits prestados](#2-la-idea-clave-tomar-bits-prestados)
3. [Método 1: subredes iguales](#3-método-1-subredes-iguales)
4. [Método 2: partir de los hosts que necesito](#4-método-2-partir-de-los-hosts-que-necesito)
5. [VLSM: subredes de distintos tamaños](#5-vlsm-subredes-de-distintos-tamaños)
6. [La regla de alineación](#6-la-regla-de-alineación)
7. [Superredes: el proceso inverso](#7-superredes-el-proceso-inverso)
8. [Diseño real: una VPC en AWS](#8-diseño-real-una-vpc-en-aws)
9. [Chuleta](#9-chuleta)
10. [Errores comunes](#10-errores-comunes)
11. [Ejercicios](#11-ejercicios)
12. [Autoevaluación](#12-autoevaluación)

---

## 1. ¿Qué es y para qué sirve?

**Subnetting** es partir una red grande en **redes más pequeñas** (subredes). Cada subred es una red independiente: su propio rango, su propio broadcast y, para hablar con las demás, un **router**.

| Para qué                   | Ejemplo                                                                                             |
| -------------------------- | --------------------------------------------------------------------------------------------------- |
| 📢 **Menos ruido**         | Cada subred es un **dominio de broadcast** distinto: los ARP y DHCP de Ventas no molestan a TI.     |
| 🔒 **Seguridad**           | Entre subredes pasa por un router o firewall → se puede controlar quién habla con quién.            |
| 🗂️ **Orden**               | Un rango por área, piso o función: se sabe qué es cada IP con solo verla.                           |
| ♻️ **No desperdiciar IPs** | A cada red se le da el tamaño que necesita.                                                         |
| ☁️ **AWS**                 | Una VPC **siempre** se divide en subnets: públicas y privadas, una por zona de disponibilidad (AZ). |

> [!TIP]
> **Analogía:** un edificio (la red) se divide en apartamentos (subredes). Cada apartamento tiene su propia numeración interna, y para ir de uno a otro hay que pasar por el pasillo (el router).

---

## 2. La idea clave: tomar bits prestados

Para crear subredes se **toman bits prestados de la parte de host** y se usan como parte de red. El prefijo crece.

```
192.168.10.0/24   →   último octeto:   H H H H H H H H     (8 bits de host)

Tomo 2 bits:
192.168.10.0/26   →   último octeto:   S S H H H H H H     (2 de subred + 6 de host)
```

Con 2 bits de subred hay **4 combinaciones**, es decir, 4 subredes:

| Bits `SS` | Último octeto (binario) | Subred                | Rango       |
| :-------: | ----------------------- | --------------------- | ----------- |
|   `00`    | `00 000000`             | 192.168.10.**0**/26   | .0 – .63    |
|   `01`    | `01 000000`             | 192.168.10.**64**/26  | .64 – .127  |
|   `10`    | `10 000000`             | 192.168.10.**128**/26 | .128 – .191 |
|   `11`    | `11 000000`             | 192.168.10.**192**/26 | .192 – .255 |

> [!IMPORTANT]
> **La regla de todo el tema:** cada bit que tomas **duplica** la cantidad de subredes y **parte a la mitad** su tamaño.
>
> ```
> Subredes creadas = 2^(bits prestados)
> Hosts por subred = 2^(bits de host que quedan) − 2
> ```

| Bits prestados (desde `/24`) | Prefijo | Subredes | Direcciones c/u | Hosts c/u |
| :--------------------------: | :-----: | :------: | :-------------: | :-------: |
|              0               |  `/24`  |    1     |       256       |    254    |
|              1               |  `/25`  |    2     |       128       |    126    |
|              2               |  `/26`  |    4     |       64        |    62     |
|              3               |  `/27`  |    8     |       32        |    30     |
|              4               |  `/28`  |    16    |       16        |    14     |
|              5               |  `/29`  |    32    |        8        |     6     |
|              6               |  `/30`  |    64    |        4        |     2     |

---

## 3. Método 1: subredes iguales

**Problema:** tengo `192.168.10.0/24` y necesito **6 subredes** iguales. ¿Cuál es el prefijo y cuáles son las subredes?

**① ¿Cuántos bits presto?** El menor `n` con 2ⁿ ≥ 6 → 2³ = 8 → **3 bits**.

**② Nuevo prefijo:** /24 + 3 = **/27** (máscara `255.255.255.224`).

**③ Tamaño del bloque:** 256 − 224 = **32**.

**④ Listar las subredes** saltando de 32 en 32:

|  #  | Subred            | Primer host | Último host | Broadcast |
| :-: | ----------------- | ----------- | ----------- | --------- |
|  1  | 192.168.10.0/27   | .1          | .30         | .31       |
|  2  | 192.168.10.32/27  | .33         | .62         | .63       |
|  3  | 192.168.10.64/27  | .65         | .94         | .95       |
|  4  | 192.168.10.96/27  | .97         | .126        | .127      |
|  5  | 192.168.10.128/27 | .129        | .158        | .159      |
|  6  | 192.168.10.160/27 | .161        | .190        | .191      |
|  7  | 192.168.10.192/27 | _libre_     |             |           |
|  8  | 192.168.10.224/27 | _libre_     |             |           |

Resultado: 8 subredes de **30 hosts**; uso 6 y me quedan 2 para crecer.

> [!NOTE]
> Fíjate en el patrón: el **broadcast** de una subred es siempre la **siguiente subred − 1**.

---

## 4. Método 2: partir de los hosts que necesito

En la práctica casi siempre se sabe **cuántos equipos** habrá, no cuántas subredes.

**Problema:** cada área necesita hasta **50 equipos**. ¿Qué prefijo uso?

**① ¿Cuántos bits de host necesito?** El menor `h` con 2ʰ − 2 ≥ 50:

- 2⁵ − 2 = 30 → no alcanza.
- 2⁶ − 2 = **62** → alcanza.

**② Prefijo:** 32 − 6 = **/26**.

| Necesito hasta… | Bits de host | Prefijo | Hosts |
| --------------: | :----------: | :-----: | ----: |
|               2 |      2       |  `/30`  |     2 |
|               6 |      3       |  `/29`  |     6 |
|              14 |      4       |  `/28`  |    14 |
|              30 |      5       |  `/27`  |    30 |
|              62 |      6       |  `/26`  |    62 |
|             126 |      7       |  `/25`  |   126 |
|             254 |      8       |  `/24`  |   254 |
|             510 |      9       |  `/23`  |   510 |
|           1.022 |      10      |  `/22`  | 1.022 |

> [!TIP]
> **Siempre deja margen.** Si hoy son 50 equipos, mañana serán 70. Cambiar el tamaño de una subred después suele obligar a cambiar las IPs de todos los equipos.

---

## 5. VLSM: subredes de distintos tamaños

Con subredes iguales se desperdicia: si Ventas necesita 100 hosts y el enlace entre dos routers necesita 2, darles el mismo tamaño es absurdo. **VLSM** (_Variable Length Subnet Mask_) le da a cada subred **su propio prefijo**.

**Problema:** oficina con la red `192.168.10.0/24`:

| Área                     | Equipos |
| ------------------------ | ------: |
| Ventas                   |     100 |
| TI                       |      50 |
| Gerencia                 |      20 |
| Enlace router ↔ firewall |       2 |

**① Ordena de mayor a menor** y elige el prefijo de cada una (tabla de la sección 4):

| Área     | Equipos |   Prefijo   | Tamaño del bloque |
| -------- | ------: | :---------: | :---------------: |
| Ventas   |     100 | `/25` (126) |        128        |
| TI       |      50 | `/26` (62)  |        64         |
| Gerencia |      20 | `/27` (30)  |        32         |
| Enlace   |       2 |  `/30` (2)  |         4         |

**② Asigna en orden**, cada una empezando donde terminó la anterior:

| Área     | Subred                | Rango de hosts | Broadcast |
| -------- | --------------------- | -------------- | --------- |
| Ventas   | **192.168.10.0/25**   | .1 – .126      | .127      |
| TI       | **192.168.10.128/26** | .129 – .190    | .191      |
| Gerencia | **192.168.10.192/27** | .193 – .222    | .223      |
| Enlace   | **192.168.10.224/30** | .225 – .226    | .227      |
| _Libre_  | 192.168.10.228 – .255 |                |           |

```
.0                       .128          .192     .224 .228            .255
├──────── Ventas /25 ─────┼── TI /26 ───┼─ Ger ──┼Enl┼──── libre ──────┤
```

> [!IMPORTANT]
> **¿Por qué de mayor a menor?** Porque cada subred tiene que empezar en un múltiplo de su tamaño (sección 6). Si pones primero las pequeñas, dejas “huecos” donde ya no cabe una grande.

---

## 6. La regla de alineación

Una subred **solo puede empezar en un múltiplo de su tamaño de bloque**.

| Prefijo (en el último octeto) | Bloque | Inicios válidos                 |
| :---------------------------: | :----: | ------------------------------- |
|             `/25`             |  128   | .0 · .128                       |
|             `/26`             |   64   | .0 · .64 · .128 · .192          |
|             `/27`             |   32   | .0 · .32 · .64 · .96 · .128 · … |
|             `/28`             |   16   | .0 · .16 · .32 · .48 · …        |

> [!WARNING]
> `192.168.10.32/26` **no es una subred válida**: 32 no es múltiplo de 64. Si haces la cuenta (AND), `.32` con máscara `/26` da red `.0` → es solo un host dentro de `192.168.10.0/26`. Muchas herramientas (y la consola de AWS) te lo rechazan.

Lo mismo vale en otros octetos: un `/20` (bloque de 16 **en el tercer octeto**) puede empezar en `10.0.0.0`, `10.0.16.0`, `10.0.32.0`…, pero no en `10.0.8.0`.

---

## 7. Superredes: el proceso inverso

El subnetting **parte** una red; la **sumarización** (o superred) **junta** varias redes contiguas en una sola, **quitando** bits de red.

```
192.168.0.0/24
192.168.1.0/24      →    192.168.0.0/22
192.168.2.0/24
192.168.3.0/24
```

Con eso, un router anuncia **una** ruta en vez de cuatro. Así es como las tablas de rutas de Internet no explotan: un ISP anuncia su bloque grande, no cada subred de cada cliente.

**Condiciones:** las redes deben ser **contiguas** y el bloque resultante debe estar **alineado** (un `/22` tiene bloque 4 en el tercer octeto → debe empezar en 0, 4, 8…). `192.168.1.0` a `192.168.4.0` **no** se pueden juntar en un solo `/22`.

> [!TIP]
> **En AWS:** se usa en las route tables cuando conectas muchas VPC o tu oficina (Transit Gateway, VPN): una ruta `10.0.0.0/8` puede cubrir todas tus VPC si planeaste bien los rangos.

---

## 8. Diseño real: una VPC en AWS

### Reglas de AWS que condicionan el diseño

| Regla                                                             | Consecuencia                                                                                                                          |
| ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| La VPC va de **`/16`** a **`/28`**                                | Normalmente se usa `/16` para tener espacio.                                                                                          |
| Una subnet vive en **una sola AZ**                                | Para alta disponibilidad se crea **una subnet por AZ** de cada tipo.                                                                  |
| **5 IPs reservadas** por subnet                                   | Un `/24` deja 251, un `/28` deja 11.                                                                                                  |
| El CIDR de una subnet **no se puede cambiar**                     | Hay que acertar el tamaño desde el principio (o crear otra subnet y migrar).                                                          |
| Los rangos **no deben solaparse** con otras VPC ni con tu oficina | Si no, no podrás conectarlas (peering, VPN, Transit Gateway).                                                                         |
| **Pública** o **privada** no depende del rango                    | Depende de la **route table**: si tiene `0.0.0.0/0 → IGW` es pública. Ver [`04-routing-switching`](04-routing-switching.md#9-en-aws). |

### Ejemplo: VPC `10.0.0.0/16` en 3 zonas, con 3 capas

| Capa                                            | AZ-a                     | AZ-b         | AZ-c         | Tamaño | IPs usables c/u |
| ----------------------------------------------- | ------------------------ | ------------ | ------------ | :----: | :-------------: |
| 🌐 **Pública** (load balancers, NAT Gateway)    | 10.0.0.0/24              | 10.0.1.0/24  | 10.0.2.0/24  | `/24`  |       251       |
| ⚙️ **Privada – aplicación** (EC2, contenedores) | 10.0.16.0/20             | 10.0.32.0/20 | 10.0.48.0/20 | `/20`  |      4.091      |
| 🗄️ **Privada – datos** (RDS, caché)             | 10.0.64.0/24             | 10.0.65.0/24 | 10.0.66.0/24 | `/24`  |       251       |
| 📦 **Reserva**                                  | 10.0.80.0 – 10.0.255.255 |              |              |        |   para crecer   |

**Por qué así:**

- **Públicas pequeñas:** ahí solo viven los load balancers y los NAT Gateway.
- **Privadas de aplicación grandes:** ahí corre casi todo, y servicios como **EKS** pueden darle una IP de la VPC a **cada contenedor**: las IPs se acaban rápido.
- **Datos aparte:** se protegen con sus propios controles y casi no crecen.
- **Rangos alineados:** los `/20` empiezan en 16, 32 y 48 (múltiplos de 16). Los `/24` de datos empiezan en 64, después de los `/20`, sin solaparse.
- **Más de la mitad libre:** para agregar una AZ más o una capa nueva sin rediseñar.

### Plan entre ambientes

Elige un rango distinto para cada VPC desde el día uno, para poder conectarlas después:

| Ambiente              | CIDR             |
| --------------------- | ---------------- |
| Desarrollo            | `10.10.0.0/16`   |
| Pruebas               | `10.20.0.0/16`   |
| Producción            | `10.30.0.0/16`   |
| Oficina (on-premises) | `192.168.0.0/16` |

Todas las VPC caben dentro de `10.0.0.0/8`, así que una sola ruta puede cubrirlas (sección 7).

> [!WARNING]
> Evita `192.168.0.0/24` y `192.168.1.0/24` en AWS: son los rangos de casi todos los routers caseros. Si alguien se conecta por VPN desde su casa, habrá conflicto.

---

## 9. Chuleta

**Potencias de 2:**

| 2¹  | 2²  | 2³  | 2⁴  | 2⁵  | 2⁶  | 2⁷  | 2⁸  | 2⁹  |  2¹⁰  |  2¹¹  |  2¹²  |
| :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :---: | :---: | :---: |
|  2  |  4  |  8  | 16  | 32  | 64  | 128 | 256 | 512 | 1.024 | 2.048 | 4.096 |

**Prefijo → máscara → bloque** (el bloque se aplica en el octeto donde cae el corte):

| Prefijo            | `/17` `/25` | `/18` `/26` | `/19` `/27` | `/20` `/28` | `/21` `/29` | `/22` `/30` | `/23` `/31` |             `/16` `/24`             |
| ------------------ | :---------: | :---------: | :---------: | :---------: | :---------: | :---------: | :---------: | :---------------------------------: |
| Máscara del octeto |     128     |     192     |     224     |     240     |     248     |     252     |     254     |                 255                 |
| Bloque             |     128     |     64      |     32      |     16      |      8      |      4      |      2      | 1 (el corte cae justo en el octeto) |

> [!TIP]
> **El método en 4 pasos (sirve para todo):**
>
> 1. ¿Cuántos hosts o subredes necesito? → potencia de 2 → **prefijo**.
> 2. **Bloque** = 256 − máscara del octeto donde cae el corte.
> 3. Las subredes empiezan en **múltiplos del bloque**.
> 4. **Broadcast** = siguiente subred − 1.

---

## 10. Errores comunes

| Error                             | Consecuencia                                       | Cómo evitarlo                                         |
| --------------------------------- | -------------------------------------------------- | ----------------------------------------------------- |
| **Subredes que se solapan**       | Rutas ambiguas: el tráfico va al lugar equivocado. | Anota cada subred en una tabla con su rango completo. |
| **Subred desalineada** (`.32/26`) | No es una subred válida.                           | Empieza siempre en un múltiplo del bloque.            |
| **Olvidar red y broadcast**       | Te faltan 2 IPs.                                   | Hosts = 2ʰ − 2.                                       |
| **Olvidar las 5 de AWS**          | Te faltan 3 IPs más de las que creías.             | En AWS: 2ʰ − 5.                                       |
| **Subred justa, sin margen**      | Al crecer hay que rediseñar y cambiar IPs.         | Dimensiona con margen y deja espacio libre.           |
| **Asignar VLSM de menor a mayor** | Huecos donde ya no cabe la subred grande.          | Siempre de mayor a menor.                             |
| **Mismo rango en VPC y oficina**  | VPN y peering imposibles.                          | Plan de rangos por ambiente (sección 8).              |

---

## 11. Ejercicios

1. Divide `10.0.0.0/24` en **8 subredes iguales**. ¿Qué prefijo usas? Lista las 3 primeras y la última.
2. Necesito una subred para **500 equipos**. ¿Qué prefijo uso?
3. **VLSM** con `172.16.0.0/24`: A = 60 equipos, B = 28, C = 12, D = 2 (enlace).
4. ¿`10.0.1.0/24` se solapa con `10.0.0.0/23`?
5. Sumariza `10.1.4.0/24`, `10.1.5.0/24`, `10.1.6.0/24` y `10.1.7.0/24` en una sola ruta.
6. En AWS necesito **100 instancias** en una subnet. ¿Sirve un `/26`? ¿Y un `/25`?
7. ¿`10.0.8.0/20` es una subred válida?
<details>
<summary>Ver soluciones</summary>

1. 8 = 2³ → 3 bits → **`/27`**, bloque 32. `10.0.0.0/27`, `10.0.0.32/27`, `10.0.0.64/27` … `10.0.0.224/27`.
2. 2⁹ − 2 = 510 ≥ 500 → 9 bits de host → **`/23`**.
3. De mayor a menor:
   - A (60) → `/26` → **172.16.0.0/26** (.1 – .62, broadcast .63)
   - B (28) → `/27` → **172.16.0.64/27** (.65 – .94, broadcast .95)
   - C (12) → `/28` → **172.16.0.96/28** (.97 – .110, broadcast .111)
   - D (2) → `/30` → **172.16.0.112/30** (.113 – .114, broadcast .115)
4. **Sí.** `10.0.0.0/23` cubre de `10.0.0.0` a `10.0.1.255`, y eso incluye todo `10.0.1.0/24`.
5. **`10.1.4.0/22`** (4 redes = 2² → se quitan 2 bits; 4 es múltiplo de 4, así que está alineado).
6. `/26` → 64 − 5 = 59: **no** sirve. `/25` → 128 − 5 = 123: **sí** sirve.
7. **No.** Un `/20` tiene bloque 16 en el tercer octeto; 8 no es múltiplo de 16. Las válidas cercanas son `10.0.0.0/20` y `10.0.16.0/20`.

</details>

---

## 12. Autoevaluación

- [ ] ¿Qué pasa con la cantidad y el tamaño de las subredes cada vez que presto un bit?
- [ ] ¿Cómo elijo el prefijo si sé cuántos equipos habrá?
- [ ] ¿Por qué en VLSM se asigna de mayor a menor?
- [ ] ¿Por qué `192.168.10.32/26` no es una subred válida?
- [ ] ¿Qué condiciones deben cumplir varias redes para sumarizarlas?
- [ ] ¿Por qué en AWS se crea una subnet por AZ?
- [ ] ¿Qué hace que una subnet de AWS sea pública: su rango o su route table?
- [ ] ¿Por qué las subnets privadas de aplicación suelen ser más grandes que las públicas?
- [ ] ¿Por qué hay que planear rangos distintos para cada VPC desde el principio?
