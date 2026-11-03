# ACT4 – Seguridad en el protocolo DHCP

---

## 1. El proceso DORA y el broadcast

Para obtener una IP, el cliente y el servidor intercambian cuatro mensajes (UDP, cliente en el puerto 68 y servidor en el 67). Como el cliente todavía no tiene dirección IP, casi todo el intercambio viaja por difusión (broadcast):

| Paso | Mensaje | De → a (IP) | Qué hace |
| --- | --- | --- | --- |
| 1 | **D**HCPDISCOVER | 0.0.0.0 → 255.255.255.255 | El cliente busca cualquier servidor DHCP del segmento |
| 2 | **O**HCPOFFER | servidor → broadcast o unicast al cliente | Cada servidor que lo oye ofrece IP, máscara, gateway, DNS y lease |
| 3 | **R**HCPREQUEST | 0.0.0.0 → 255.255.255.255 | El cliente acepta una oferta (opción 54) y lo anuncia para que el resto retire la suya |
| 4 | **A**HCPACK | servidor → cliente | El servidor confirma la concesión y el cliente configura su IP |

Regla mnemotécnica: **D-O-R-A** (Discover, Offer, Request, Ack).

**Implicaciones de seguridad del broadcast.** Si un atacante está en el mismo segmento (la misma VLAN o la misma *Red Interna* de VirtualBox), el uso masivo de difusión le da ventaja:

- **Visibilidad total y gratuita.** Recibe todos los DISCOVER y REQUEST sin hacer nada especial: ve MACs, nombres de equipo y qué IP pide cada uno. Es reconocimiento sin coste.
- **Puede suplantar al servidor (Rogue DHCP).** Cualquier equipo puede responder al DISCOVER. Como el cliente suele quedarse con la **primera oferta que llega**, si el atacante responde rápido le impone su gateway y su DNS, lo que permite un *Man-in-the-Middle*.
- **El switch no distingue.** Reparte el broadcast por todos los puertos de la VLAN y no diferencia un servidor legítimo de uno falso.
- **Amplifica un DoS.** Una ráfaga de DISCOVER falsos llega a todo el segmento y el servidor debe procesarlos uno a uno.

El broadcast tiene un límite: **no cruza routers**. Con *DHCP relay* (`ip helper-address`) el servidor puede estar en otra red, pero el atacante sigue necesitando estar en la VLAN del cliente.

---

## 2. Anatomía del paquete DHCP (capa 7 vs capa 2)

DHCP hereda el formato de BOOTP: una cabecera fija de 236 bytes, la *magic cookie* `99.130.83.99` y después las opciones. Los campos relevantes para la seguridad:

| Campo | Tamaño | Contenido |
| --- | --- | --- |
| op | 1 B | 1 = petición del cliente, 2 = respuesta del servidor |
| htype / hlen | 2 B | Tipo y longitud de dirección hardware (1 y 6 para Ethernet) |
| xid | 4 B | ID aleatorio que une los cuatro mensajes de un mismo cliente |
| flags | 2 B | Incluye el bit *Broadcast* |
| yiaddr | 4 B | "Your IP": la dirección que ofrece el servidor |
| giaddr | 4 B | IP del relay, si lo hay |
| **chaddr** | 16 B | Dirección hardware del cliente (la MAC va en los 6 primeros bytes) |
| options | variable | Opción 53 (tipo de mensaje), 50 (IP pedida), 54 (servidor), 51 (lease), 61 (client-id)... |

**La diferencia clave: `chaddr` frente a la MAC de origen Ethernet.** Son dos cosas distintas:

- La **MAC de origen de la trama** (capa 2) la pone la tarjeta o el driver y solo vive en el enlace local; cada router la reescribe.
- El **`chaddr`** es un dato dentro del payload (capa 7) que escribe el software cliente.

Normalmente coinciden, pero **nada en el protocolo obliga a que sean iguales**.

**Por qué el servidor usa `chaddr` y no la MAC de la trama.** Cuando hay un relay de por medio, la trama que le llega al servidor lleva la MAC del router, no la del cliente. Si el servidor se fiara de la MAC de capa 2, todos los clientes de otra subred parecerían el mismo equipo. Usando `chaddr` (o el client-id, opción 61) funciona igual en local y a través de relays. Además lo necesita para responder en unicast a alguien que todavía no tiene ni IP ni entrada ARP.

**Vulnerabilidad que introduce.** El servidor confía en un campo que **el cliente rellena libremente**. Con eso un atacante puede:

- Enviar miles de DISCOVER con un `chaddr` aleatorio cada vez, de modo que el servidor cree que son miles de clientes distintos (base del Starvation).
- Poner el `chaddr` de una víctima para renovar o **liberar** (DHCPRELEASE) su concesión, o quitarle una reserva estática.
- Saltarse filtros basados en MAC, porque solo comprueban el `chaddr`, que es justo lo que el atacante falsifica.

---

## 3. La carencia de autenticación

**DHCP no incluye ningún mecanismo obligatorio de autenticación ni de cifrado.** El propio RFC 2131 lo reconoce en su sección de *Security Considerations*: se apoya en UDP/IP (inseguros de serie), todo viaja en claro, y advierte de que un servidor no autorizado puede engañar a los clientes y un cliente malicioso puede agotar los recursos del servidor.

Qué lo permite:

- **Responder a un DISCOVER:** el cliente acepta cualquier OFFER con el `xid` correcto, y ese `xid` viaja en claro en el broadcast, así que lo conoce todo el segmento. No hay firma ni certificado que identifique al servidor legítimo.
- **Pedir miles de IPs:** el servidor solo diferencia clientes por `chaddr`/client-id, que elige el propio cliente. Para el protocolo, 1.000 `chaddr` distintos equivalen a 1.000 clientes legítimos. No hay límite por puerto ni por MAC real.

**El RFC 3118 (*Authentication for DHCP Messages*, 2001)** intentó resolverlo. Añade la opción 90 con un HMAC-MD5 y detección de repeticiones, para probar origen e integridad. Define dos modos: un token con contraseña en claro (muy básico) y *delayed authentication* con clave compartida repartida fuera de banda.

**Por qué su adopción ha sido prácticamente nula:**

- **Las claves hay que repartirlas a mano** antes de la primera conexión, justo lo contrario a la configuración cero que es la gracia de DHCP. En redes con invitados, BYOD o IoT resulta inviable.
- **Casi nadie lo implementó:** ni Windows, ni macOS, ni Android, ni iOS, ni en la práctica isc-dhcp-server.
- **Protege a medias:** no cifra, usa MD5 y tiene problemas con los relays.
- **Había alternativas más baratas:** la industria resolvió el problema en la infraestructura (DHCP Snooping, Port Security, 802.1X) sin necesidad de tocar cada cliente.

---

## 4. La mecánica del DHCP Starvation

El objetivo es dejar al servidor legítimo **sin direcciones libres** para que no pueda atender a nadie. Es una denegación de servicio (DoS) y a menudo el paso previo para montar un servidor falso.

**Paso a paso:**

1. El atacante se conecta al mismo segmento que el servidor (o que un relay).
2. Fabrica un DISCOVER con `chaddr` aleatorio y `xid` nuevo, y lo envía en broadcast.
3. El servidor aparta una IP libre para ese `chaddr` y responde con un OFFER.
4. El atacante contesta con REQUEST, el servidor envía ACK y esa IP queda ocupada **durante todo el lease**. Solo con el DISCOVER ya la bloquea unos segundos, pero completando DORA la retiene toda la concesión.
5. Repite con otro `chaddr` miles de veces por segundo hasta vaciar el pool. En un `/24` con rango `.100–.200` bastan **101 concesiones falsas**.
6. Opcional (y lo más peligroso): levanta su propio servidor DHCP. Los equipos nuevos solo reciben su oferta, con un gateway y un DNS controlados por él.

**Starvation simple frente a complejo:**

| Aspecto | Simple (solo `chaddr`) | Complejo (MAC Ethernet + `chaddr`) |
| --- | --- | --- |
| MAC de origen de la trama | Siempre la MAC real del atacante | Una MAC aleatoria distinta en cada trama |
| `chaddr` | Aleatorio en cada mensaje | Igual a la MAC falsa de esa trama |
| Coherencia capa 2 / capa 7 | Incoherente (MAC ≠ chaddr) | Coherente, parece un cliente real |
| Lo que ve el switch | Una sola MAC en el puerto | Miles de MAC nuevas en el puerto |

**Por qué el escenario complejo es mucho más difícil de mitigar.** En el ataque simple, el switch solo ve una MAC en el puerto, así que el DHCP Snooping detecta enseguida que la MAC de la trama no coincide con el `chaddr` (opción *verify mac-address*) y descarta esos paquetes.

En el complejo, cada trama lleva una MAC nueva que **sí** coincide con su `chaddr`, por lo que esa comprobación no detecta nada. Además el switch ve miles de MAC aparentemente legítimas: se le llena la tabla CAM (puede derivar en *MAC flooding*) y la única defensa que queda es limitar MAC por puerto (Port Security), que a su vez puede afectar a equipos reales conectados tras un hub o un teléfono IP. Por eso herramientas como Yersinia usan este modo por defecto.

---

## 5. Herramientas y ciclo de vida del lease

**Yersinia.** Herramienta de pentesting de protocolos de capa 2 (STP, CDP, DHCP...), con interfaz gráfica, modo texto y línea de comandos. Para DHCP implementa el *sending DISCOVER packet*: inunda el segmento con DISCOVER usando MAC y `chaddr` aleatorios (el escenario complejo de doble capa). Es rápida y no requiere programar, pero es poco flexible: hace lo que trae predefinido.

**Scapy (Python).** Permite construir, enviar y recibir paquetes byte a byte: `Ether()/IP()/UDP()/BOOTP()/DHCP()`, controlando cada campo (MAC de origen, `chaddr`, `xid`, opciones). Con un bucle se genera tráfico masivo y, en modo escucha, se capturan los OFFER para completar el REQUEST.

**Ventajas de automatizar con un script frente a hacerlo a mano:**

- **Velocidad:** manualmente son minutos por concesión; un script agota el pool en segundos.
- **Aleatoriedad controlada:** genera MAC y `chaddr` únicos sin repetir, algo inviable a mano.
- **Reactividad:** escucha los OFFER y responde el REQUEST correcto automáticamente, cerrando el ciclo DORA.
- **Reproducibilidad:** se lanza igual antes y después de aplicar una defensa para comparar, y se ajusta (ritmo, rango, un solo `chaddr` para probar el Snooping). Es ideal para medir la eficacia de las mitigaciones.

**Ciclo de vida del lease (la concesión).** Al conceder una IP, el servidor fija un *lease time* T y el cliente arranca dos temporizadores: **T1** (renovar, 50 % de T) y **T2** (reenlazar, 87,5 % de T).

- En T1 intenta renovar en unicast con el mismo servidor.
- Si no responde, en T2 lo pide en broadcast a cualquier servidor.
- Si expira T sin renovar, libera la IP y vuelve al DISCOVER.

Lo importante: mientras la concesión sigue viva, esa IP **permanece ocupada aunque el equipo ya no esté conectado**. De ahí la pregunta sobre el lease time:

- **Lease de 7 días:** si el atacante agota el pool y se desconecta inmediatamente, esas IPs falsas siguen reservadas **7 días**. El servidor no sabe que esos "clientes" se fueron, así que la red queda sin direcciones una semana, salvo que el administrador borre a mano `dhcpd.leases` y reinicie el servicio.
- **Lease de 5 minutos:** las concesiones falsas caducan a los 5 minutos de irse el atacante y el pool se recupera solo. El inconveniente es que un lease tan corto dispara el tráfico de renovación y, mientras el ataque sigue activo, es más fácil mantener el pool agotado (hay que renovar constantemente). Ese equilibrio se analiza en el punto 9.

---

## 6. Consecuencias colaterales y APIPA

**Qué les pasa a los equipos nuevos cuando el pool está agotado.** Envían su DISCOVER y no reciben OFFER (o reciben un NAK). Tras varios intentos sin éxito:

- Se quedan sin IP, gateway ni DNS, es decir, sin conectividad útil.
- Si hay un Rogue DHCP, puede ser el único que responda y entregar una configuración maliciosa.
- En muchos sistemas, el equipo se autoconfigura una dirección **APIPA**.

**APIPA (169.254.0.0/16).** Es la autoconfiguración *link-local* definida en el RFC 3927. Cuando no recibe respuesta, el equipo se asigna él mismo una IP de ese rango:

- Elige una IP pseudoaleatoria y comprueba con ARP que nadie más la usa; si hay conflicto, prueba otra.
- Windows la activa por defecto; en Linux depende (avahi-autoipd o la opción del cliente DHCP).
- Sigue enviando DISCOVER en segundo plano, de modo que si el servidor vuelve a tener direcciones, cambia a una IP normal.

Por eso, **un equipo con una `169.254.x.x` indica que no hay servidor DHCP disponible**, que es el síntoma típico de un Starvation.

**Limitaciones de las direcciones link-local:**

- **No son enrutables:** ningún router las reenvía, así que solo valen dentro del mismo enlace/VLAN. No hay acceso a Internet ni a otras subredes.
- **Sin gateway ni DNS:** no se resuelven nombres ni se sale de la red local.
- **Solo entre equipos en APIPA:** dos equipos `169.254.x.x` se ven entre sí, pero no alcanzan a los que tengan IP del rango corporativo. El resultado es una red partida y sin servicios.

---

## 7. Mitigación en la infraestructura (capa 2)

**DHCP Snooping.** Es una función del switch de acceso que inspecciona el tráfico DHCP de cada puerto y actúa como un cortafuegos DHCP de capa 2. Clasifica los puertos en dos tipos:

- **Puerto de confianza (*trusted*):** por él se permiten mensajes de servidor (OFFER, ACK, NAK). Se configuran como trusted los puertos que van al servidor DHCP legítimo o a otro switch (uplinks).
- **Puerto de no confianza (*untrusted*):** el de los equipos de usuario. Por defecto todos lo son, y en ellos el switch descarta cualquier respuesta de servidor.

**Cómo evita un Rogue DHCP:** si el atacante conecta un servidor falso a un puerto untrusted y envía un OFFER o un ACK, el switch los bloquea de inmediato, porque un puerto de usuario nunca debería originar respuestas de servidor. Así ningún cliente recibe la configuración maliciosa. Además, el Snooping puede limitar la tasa de mensajes DHCP por puerto y comprobar que la MAC coincide con el `chaddr`, lo que corta el Starvation simple.

**Base de datos de vinculación (*Snooping Binding Database*).** Es la tabla donde el switch anota cada concesión legítima que pasa por un puerto trusted. De cada transacción registra:

- La **MAC** del cliente
- La **IP** concedida
- El **lease** restante
- El **puerto** del switch y la **VLAN**

Esta tabla es la fuente de verdad para otras dos defensas: **Dynamic ARP Inspection** (valida que las respuestas ARP coincidan con el par IP-MAC registrado) e **IP Source Guard** (solo deja pasar tramas cuya IP de origen esté en la tabla). Con ello se impide también el ARP spoofing.

**Port Security.** Limita cuántas direcciones MAC de origen distintas admite un puerto físico. Contra el agotamiento de IPs:

- Si se fija un máximo de 1 o 2 MAC por puerto, el Starvation complejo (una MAC nueva por trama) dispara el límite en cuanto aparece la segunda.
- Ante la violación, el switch aplica la acción configurada: *shutdown* (apaga el puerto), *restrict* o *protect* (descarta las tramas sobrantes y avisa).
- Así se impide que un solo puerto genere miles de MAC falsas, y de paso se evita el MAC flooding.

Las tres funciones combinadas (Snooping + DAI/IP Source Guard + Port Security) son la defensa estándar en switches gestionables.

---

## 8. Mitigación en el propio host (Linux)

En VirtualBox no disponemos de un switch gestionable, así que el servidor Linux (Debian/Ubuntu con isc-dhcp-server) **debe protegerse a sí mismo** con el firewall del kernel.

**El módulo `hashlimit` de iptables.** iptables permite limitar la tasa de paquetes. El módulo `limit` normal lleva un único contador global; **`hashlimit` es más potente** porque mantiene una tabla hash con un contador independiente por clave (IP de origen, MAC, puerto...). Es decir, permite limitar **por cliente**, no para todo el tráfico junto.

Funciona con el modelo *token bucket*: cada clave acumula "fichas" a un ritmo (`--hashlimit-upto`, por ejemplo 5/seg) con un máximo de ráfaga (`--hashlimit-burst`). Mientras hay fichas, el paquete se acepta; si se agotan, el sobrante cae a la siguiente regla (normalmente DROP).

```bash
# DHCP va por UDP 67/68
iptables -A INPUT -p udp --dport 67 \
  -m hashlimit --hashlimit-name dhcp \
  --hashlimit-mode srcmac --hashlimit-upto 5/sec --hashlimit-burst 10 \
  -j ACCEPT
iptables -A INPUT -p udp --dport 67 -j DROP
```

**Por qué es útil contra el Starvation:** el ataque se basa en enviar DISCOVER a un ritmo muy alto. Limitando los que acepta el servidor (por MAC o, si todo viene del mismo host, en global), se descartan las ráfagas y el servidor conserva capacidad para los clientes reales. Es un paliativo: ralentiza mucho el agotamiento, aunque no distingue por sí solo un cliente legítimo de uno falso si el atacante baja el ritmo.

**ebtables.** Es el equivalente a iptables pero para el **puente (bridge) de capa 2**: filtra tramas Ethernet (por MAC, EtherType...), mientras que iptables trabaja en capa 3 (IP). Se usa cuando el Linux actúa como puente entre interfaces (por ejemplo, el host del laboratorio puenteando atacante y servidor).

**Comprobar que la MAC de capa 2 coincide con el `chaddr` de capa 7.** La idea es replicar en software lo que hace el *verify mac-address* del Snooping:

- Interceptar en el bridge las tramas DHCP (IPv4, UDP 67/68).
- Comparar la MAC de origen de la cabecera Ethernet con los 6 bytes del `chaddr` del payload.
- Si no coinciden, descartar la trama (es el síntoma del Starvation simple: varía el `chaddr` pero mantiene su MAC real).

ebtables compara bien campos de capa 2; para leer con precisión el offset del `chaddr` dentro del payload suele combinarse con el match `u32` o reglas a medida. Su limitación es que **no detiene el Starvation complejo** (ahí MAC y `chaddr` sí coinciden), para el que haría falta limitar MAC por puerto o usar hashlimit.

---

## 9. Las lease policies como defensa

**Equilibrar el lease time** es un compromiso:

- **Lease largo (días):** poco tráfico de renovación y estabilidad, pero ante un Starvation las IPs falsas quedan bloqueadas mucho tiempo aunque el atacante se marche.
- **Lease corto (minutos):** el pool se recupera rápido tras el ataque, pero multiplica las renovaciones (REQUEST/ACK constantes), carga el servidor y, con el ataque activo, facilita mantener el pool agotado.

Lo que suele hacer un administrador:

- Redes estables (oficina, equipos fijos): leases de horas (8-24 h) y reservas estáticas por MAC para servidores e impresoras.
- Redes muy dinámicas (wifi de invitados, aulas): leases cortos para reciclar IPs, asumiendo más renovaciones.
- **Dimensionar el pool con holgura** (bastantes más IPs que clientes) para que un pico no lo agote.
- Combinarlo siempre con las defensas de capa 2 y del host: el lease time **no evita** el ataque, solo decide cuánto dura su efecto.

**Detección de conflictos (`ping-check` / conflict-detection).** Antes de entregar una IP, el servidor comprueba que no esté ya en uso:

- En isc-dhcp-server, `ping-check true;` hace que el servidor envíe un ping (ICMP echo) a la IP candidata antes de ofrecerla. Si alguien responde, la marca como *abandoned* y ofrece otra. Espera `ping-timeout` segundos (1 por defecto), lo que añade algo de latencia a cada concesión.
- Por su parte, el cliente envía un ARP gratuito antes de usar la IP del ACK; si hay conflicto, manda un DHCPDECLINE y el servidor la marca como no disponible.

**De qué sirve aunque el pool esté corrupto.** Evita el daño de las **IPs duplicadas**. Si por el ataque o por leases inconsistentes el servidor fuera a entregar una IP que otro equipo ya usa, la detección lo descubre y no la asigna, evitando cortes por duplicidad. No recupera el pool agotado, pero impide que un pool en mal estado provoque colisiones de direcciones que tiren la conectividad de clientes que sí funcionaban.

---

## 10. El modo promiscuo en VirtualBox

**Qué es.** Normalmente una tarjeta de red solo procesa las tramas dirigidas a su propia MAC (más broadcast y multicast suscritos) y descarta el resto. En modo promiscuo la interfaz acepta **todas** las tramas que ve, vayan dirigidas a ella o no. Es imprescindible para capturar tráfico (Wireshark) y para recibir respuestas destinadas a MAC que no son la propia, como las MAC falsas del ataque.

**Por qué los hipervisores lo restringen por defecto.** En VirtualBox el adaptador viene en "Denegar". Razones:

- **Aislamiento y seguridad:** evita que una VM espíe el tráfico de otras VMs o del host que comparten el mismo segmento virtual.
- **Rendimiento:** entregar todas las tramas a una VM que no las necesita consume CPU y memoria del host.
- Por eso ofrece tres niveles: *Denegar*, *Permitir VMs* y *Permitir todo*. En el laboratorio se usa **"Permitir todo"** en la Red Interna para que Kali capture y procese todo el tráfico.

**Si Kali NO estuviera en modo promiscuo**, hay que separar **enviar** de **recibir**:

- **Enviar paquetes con MAC de origen inventadas: sí funcionaría.** El modo promiscuo solo afecta a la recepción. Scapy construye la trama en capa 2 y la inyecta con la MAC de origen que quiera. El envío de DISCOVER con MAC/`chaddr` falsos no depende del modo promiscuo; de hecho, un Starvation "a ciegas" (solo inundar sin leer respuestas) funcionaría igual.
- **Recibir los OFFER dirigidos a esas MAC falsas: no, o solo parcialmente.** El servidor puede responder el OFFER en unicast a la MAC del `chaddr`. Esa trama lleva como destino una MAC inventada que **no** es la de la interfaz de Kali, así que sin modo promiscuo la tarjeta la descarta y Scapy nunca la ve, por lo que no puede completar el REQUEST de esas IPs.
  - Matiz: muchos clientes piden la respuesta en broadcast (bit *Broadcast* del campo `flags`). Si el servidor lo respeta, Kali sí recibiría los OFFER aun sin modo promiscuo, porque el broadcast lo procesan todas las interfaces. Pero no está garantizado, así que para un ataque fiable que complete DORA con MAC falsas conviene activar el modo promiscuo.

**Conclusión:** el modo promiscuo no es necesario para *lanzar* las tramas falsas, pero sí para *recibir* de forma fiable las respuestas unicast destinadas a MAC que no son la real de Kali, es decir, para ejecutar el ataque completo y para capturar y analizar el intercambio.

---

## 11. Aspectos éticos y legales

**Cuándo está permitido realizar estas pruebas.** El Starvation es una variante de denegación de servicio (DoS), así que solo es lícito en condiciones de laboratorio o pentesting autorizado:

- **Entorno propio y aislado:** una red de laboratorio propia como la Red Interna de VirtualBox, sin conexión a producción ni a Internet. Es el caso de esta práctica.
- **Autorización expresa y por escrito:** sobre una red ajena, solo con permiso previo y documentado del titular del sistema (un contrato o *scope* de auditoría que delimite qué, cuándo y cómo se puede atacar). Sin ese consentimiento, la prueba es un delito aunque la intención sea "demostrar" un fallo.
- **Alcance y proporcionalidad:** ceñirse a lo autorizado, no tocar datos reales ni interrumpir servicios fuera del acuerdo, y documentarlo todo.

La regla práctica: lo que convierte el ataque en delito **no es la técnica, sino la falta de autorización** sobre un sistema ajeno.

**Consecuencias penales en España (Código Penal, delitos de daños informáticos).** Un DoS sin autorización encaja en los delitos de daños informáticos:

- **Art. 264 bis** – obstaculizar o interrumpir gravemente y sin autorización el funcionamiento de un sistema informático ajeno. Es el que mejor describe un DoS/Starvation. Pena base: **prisión de 6 meses a 3 años**. Agravado (apdo. 2): **prisión de 3 a 8 años y multa del triplo al décuplo** del perjuicio, si concurren daños de especial gravedad, un número elevado de sistemas afectados, perjuicio a servicios públicos esenciales o a infraestructuras críticas.
- **Art. 264** – dañar, borrar, alterar o hacer inaccesibles datos, programas o documentos electrónicos ajenos. Pena base: 6 meses a 3 años; agravado: 2 a 5 años y multa.
- **Art. 264 ter** – facilitar los delitos anteriores produciendo o distribuyendo herramientas o contraseñas concebidas para ello: 6 meses a 2 años o multa. Es relevante respecto a distribuir scripts o herramientas de ataque con esa finalidad.

Además puede haber responsabilidad civil por los daños, y si se accede a datos o sistemas protegidos se suma el art. 197 bis (acceso ilegal a sistemas). En el entorno controlado de la práctica, sobre máquinas propias y aisladas, no se da ninguno de estos supuestos, por lo que es un sitio seguro para aprender.

> *Esta es información general con fines académicos, no asesoramiento jurídico. Conviene contrastar siempre el texto vigente en el BOE.*

---

## Fuentes y referencias

- **RFC 2131** – *Dynamic Host Configuration Protocol* (formato BOOTP y *Security Considerations*)
- **RFC 3118** – *Authentication for DHCP Messages*
- **RFC 3927** – *Dynamic Configuration of IPv4 Link-Local Addresses* (APIPA)
- **RFC 951** – *Bootstrap Protocol (BOOTP)*
- Documentación de **isc-dhcp-server**: `dhcpd.conf` (`ping-check`, `ping-timeout`, lease times) y `dhcpd.leases`
- Manuales de **iptables** (`man iptables-extensions`, módulo `hashlimit`) y de **ebtables**
- Documentación de **Cisco** sobre DHCP Snooping, Dynamic ARP Inspection, IP Source Guard y Port Security
- **VirtualBox User Manual** – configuración de red y modo promiscuo (Denegar / Permitir VMs / Permitir todo)
- **Código Penal español**, arts. 264, 264 bis y 264 ter – [texto del capítulo IX, De los daños](https://papelea.com/es/leyes/codigo-penal/libro-ii_titulo-xiii_capitulo-ix) · análisis del [art. 264 bis](https://elderecho.com/en-jurisprudenciatuitatuit-analizamos-el-delito-de-danos-previsto-en-el-articulo-264-bis-codigo-penal) · texto vigente en el [BOE](https://www.boe.es)
