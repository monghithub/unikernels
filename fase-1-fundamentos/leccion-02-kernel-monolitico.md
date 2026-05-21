# Día 02 — Kernel monolítico: Linux por dentro

> **Fase:** 1 · **Semana:** 1 · **Tipo:** Lectura · **Duración:** 30 min

---

## Contexto y motivación

En la lección anterior establecimos que el kernel es el intermediario entre el
hardware y las aplicaciones: gestiona memoria, tiempo de CPU, dispositivos y
comunicación entre procesos. Ahora toca responder una pregunta más concreta:
¿cómo organiza Linux todo ese código internamente?

Linux es el kernel más desplegado del mundo. Corre en teléfonos Android, en el
99 % de los servidores de internet, en los supercomputadores del Top500 y —esto
nos importa especialmente— en casi todas las plataformas donde los unikernels
compiten o de las que aprenden. Entender su arquitectura no es un ejercicio
académico: es el mapa que nos permite después comprender *qué partes descarta un
unikernel* y *por qué eso es posible*.

---

## Concepto principal

**Un kernel monolítico ejecuta todos los servicios del sistema operativo en un
único espacio de direcciones con privilegio máximo (ring 0).** No hay fronteras
de proceso entre el gestor de memoria, el planificador, los drivers o la pila de
red: todo comparte el mismo espacio y se comunica mediante llamadas a función
ordinarias, sin ningún coste de cambio de contexto entre subsistemas.

Linux es monolítico, pero *modular*: los drivers y algunos subsistemas pueden
cargarse y descargarse en tiempo de ejecución como módulos del kernel (`.ko`),
aunque una vez cargados siguen corriendo en ring 0 con acceso total al espacio
del kernel.

---

## Desarrollo (los 30 minutos)

### Parte 1 — Qué significa "monolítico" (~10 min)

#### El espacio de memoria del kernel

Cuando arranca Linux, el procesador entra en el modo más privilegiado disponible
(ring 0 en x86, EL1/EL2 en ARM). En ese modo no hay restricciones de acceso a
instrucciones ni a direcciones de memoria. El kernel coloca en ese espacio:

- su propio código y datos estáticos,
- las estructuras de control de todos los procesos,
- las tablas de página,
- los buffers de red y disco (page cache),
- el código de todos los drivers activos.

Cuando una aplicación de usuario (ring 3) llama al kernel mediante una syscall,
el procesador eleva el nivel de privilegio, salta a la dirección del manejador
del kernel y ejecuta código dentro de ese espacio único. Al volver, baja de
nuevo a ring 3.

```
 Espacio de usuario (ring 3)
 ┌────────────────────────────────────────────┐
 │  app1 (PID 1001)  │  app2 (PID 1002)  │ … │
 └────────────────────────────────────────────┘
            │  syscall (int 0x80 / syscall)
            ▼
 ══════════════ barrera ring 3 / ring 0 ══════════
            │
 Espacio del kernel (ring 0)
 ┌────────────────────────────────────────────┐
 │         UN SOLO ESPACIO DE DIRECCIONES     │
 │                                            │
 │  scheduler │ MM │ VFS │ net │ IPC │ …     │
 │  driver A  │ driver B  │  driver C  │ …   │
 └────────────────────────────────────────────┘
            │
            ▼
       Hardware (CPU, RAM, disco, NIC…)
```

**Clave conceptual:** la frontera es entre user space y kernel space, no entre
subsistemas. Dentro del kernel, el scheduler puede llamar directamente a
funciones del gestor de memoria sin ningún overhead.

#### Por qué se llama "monolítico"

El término viene del griego *monos* (uno) + *lithos* (piedra): una sola roca.
En los años 80, cuando Linus Torvalds escribió la primera versión de Linux,
Tanenbaum le criticó precisamente esto en el famoso debate `comp.os.minix`
(1992): *"Linux is a monolithic kernel. This is a giant step back to the
1970s."* Torvalds defendió que la monolítica era la arquitectura pragmática para
obtener rendimiento real. Décadas después, Linux sigue siendo monolítico y sigue
ganando en la mayoría de benchmarks de producción.

#### Módulos: modularidad sin separación

Para no tener que recompilar el kernel entero cada vez que se añade soporte para
un nuevo dispositivo, Linux introdujo los **módulos del kernel** (`LKM –
Loadable Kernel Modules`). Un módulo es un objeto `.ko` que se carga con
`insmod` / `modprobe` y se descarga con `rmmod`. Una vez cargado, el código del
módulo reside en ring 0 con acceso total. La modularidad es de *desarrollo y
distribución*, no de aislamiento en ejecución.

---

### Parte 2 — Los subsistemas principales de Linux (~15 min)

Un kernel monolítico no significa código desorganizado. Linux está estructurado
en subsistemas bien delimitados que se comunican mediante APIs internas. Aquí
están los más relevantes:

```
 ┌──────────────────────────────────────────────────────────────┐
 │                    APLICACIONES (ring 3)                     │
 └──────────────────────────┬───────────────────────────────────┘
                            │  System Call Interface
 ┌──────────────────────────▼───────────────────────────────────┐
 │                  KERNEL LINUX (ring 0)                       │
 │                                                              │
 │  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌─────────────┐  │
 │  │ Process │  │  Memory  │  │  Virtual │  │   Network   │  │
 │  │  Sched  │  │  Manager │  │    FS    │  │    Stack    │  │
 │  │ (CFS…)  │  │  (MM)    │  │  (VFS)   │  │  (TCP/IP)   │  │
 │  └────┬────┘  └────┬─────┘  └────┬─────┘  └──────┬──────┘  │
 │       │            │              │                │          │
 │  ┌────▼────────────▼──────────────▼────────────────▼──────┐  │
 │  │                  IPC / Señales / Pipes                  │  │
 │  └──────────────────────────┬──────────────────────────────┘  │
 │                             │                                │
 │  ┌──────────────────────────▼──────────────────────────────┐  │
 │  │               DRIVERS DE DISPOSITIVO                    │  │
 │  │   blk (disco) │  char │  NIC │  USB │  GPU │  …        │  │
 │  └──────────────────────────┬──────────────────────────────┘  │
 └─────────────────────────────┼────────────────────────────────┘
                               │
 ┌─────────────────────────────▼────────────────────────────────┐
 │              HARDWARE (CPU · RAM · Disco · NIC · …)          │
 └──────────────────────────────────────────────────────────────┘
```

#### Process Scheduler

Decide qué proceso/hilo corre en cada CPU en cada instante. El planificador
principal desde Linux 2.6.23 es **CFS** (Completely Fair Scheduler). También
existe un planificador de tiempo real (`SCHED_FIFO`, `SCHED_RR`) para tareas de
baja latencia. El scheduler se invoca directamente desde interrupciones de timer
y desde rutas de código donde un proceso voluntariamente cede la CPU
(`schedule()`).

#### Memory Manager (MM)

Gestiona la memoria virtual y física:

- **Paginación y page tables:** cada proceso ve su propio espacio virtual de
  direcciones; MM mantiene las tablas de página que traducen esas direcciones a
  física (con soporte del MMU del procesador).
- **Page cache:** las páginas de fichero leídas del disco se guardan en RAM para
  reutilizarlas. Es el mayor consumidor de RAM en servidores.
- **Slab / SLUB allocator:** asigna objetos del kernel de tamaño fijo de forma
  eficiente.
- **OOM killer:** cuando la RAM se agota elige qué proceso matar.

#### Virtual File System (VFS)

Es una capa de abstracción que permite al kernel soportar decenas de sistemas de
ficheros (ext4, btrfs, xfs, tmpfs, procfs, sysfs, NFS…) con una única interfaz
de syscalls: `open`, `read`, `write`, `close`. La VFS define objetos como
`inode`, `dentry` y `file`; cada sistema de ficheros concreto implementa sus
operaciones rellenando punteros de función en esas estructuras.

```
 open("/etc/passwd", O_RDONLY)
        │
        ▼  VFS
   ┌─────────────────────┐
   │  dentry cache       │  ← ¿está en caché el path?
   │  inode cache        │  ← metadatos del fichero
   └──────┬──────────────┘
          │
          ▼  dispatch a filesystem concreto
   ┌──────────────────────┐
   │  ext4_file_read_iter │  (o btrfs, xfs, nfs…)
   └──────────────────────┘
```

#### Network Stack

Implementa la pila TCP/IP completa dentro del kernel: IP, TCP, UDP, ICMP,
sockets, routing, netfilter/iptables. La comunicación entre la aplicación y el
stack sigue el camino: `syscall send()` → socket layer → TCP → IP → driver NIC
→ hardware. Todos esos pasos ocurren dentro de ring 0 sin cambios de espacio.

#### IPC (Inter-Process Communication)

Aunque el kernel en sí no necesita IPC entre subsistemas (usa llamadas a
función), expone mecanismos IPC a las aplicaciones:

- **Pipes y FIFOs**
- **Señales** (SIGTERM, SIGKILL, …)
- **Semáforos y mutexes** (POSIX, System V)
- **Memoria compartida**
- **Sockets UNIX**

#### Drivers de dispositivo

El mayor volumen de código de Linux está en drivers. Un driver expone al kernel
una interfaz estándar (operaciones de bloque, operaciones de carácter, etc.) y
habla con el hardware en su idioma específico (registros MMIO, DMA, interrupciones
PCI, protocolo USB…).

#### El tamaño real del kernel Linux

Para poner cifras concretas al concepto "monolítico":

| Versión   | Líneas de código (aprox.) | Ficheros `.c` + `.h` |
|-----------|--------------------------|----------------------|
| 1.0 (1994)| ~176 000                  | ~400                 |
| 2.6 (2003)| ~5 900 000                | ~17 000              |
| 5.0 (2019)| ~26 000 000               | ~68 000              |
| 6.x (2024)| **~36 000 000**           | ~90 000              |

Más del 60 % del código son drivers. El subsistema de red representa
aproximadamente el 5 %, el MM otro 5 %, y el scheduler menos del 1 %.

Fuente de los datos: `cloc` aplicado al árbol fuente oficial de kernel.org.

---

### Parte 3 — Velocidad y riesgos del modelo monolítico (~5 min)

#### Por qué es tan rápido

La ventaja principal del diseño monolítico es la **ausencia de IPC entre
subsistemas**. Cuando el scheduler necesita información del MM, hace una llamada
a función ordinaria: instrucción `call` + `ret`. No hay:

- cambio de espacio de memoria,
- serialización de mensajes,
- cambio de nivel de privilegio,
- copia de datos entre espacios.

En comparación, un microkernel (siguiente lección) tiene que hacer un round-trip
IPC completo para que dos servicios del sistema se comuniquen. En cargas de
trabajo con alta frecuencia de syscalls o con muchas interacciones
scheduler-MM-VFS, esa diferencia acumulada se nota en latencia y throughput.

#### Por qué es arriesgado

La contrapartida es que **todo el código del kernel comparte el mismo espacio de
memoria y el mismo nivel de privilegio**. Un bug en el driver de una tarjeta de
red wifi —aunque no tenga ninguna relación con el sistema de ficheros— puede:

- corromper estructuras de datos del MM,
- causar un `NULL pointer dereference` que provoque kernel panic,
- ser explotado para escalar privilegios (vulnerabilidades tipo
  `CVE-2021-4154`, `CVE-2022-0847` Dirty Pipe…).

El sistema entero cae o queda comprometido. No hay aislamiento de fallos entre
subsistemas.

Mecanismos que Linux usa para mitigar esto (pero no eliminar el riesgo):

- **KASLR** (Kernel Address Space Layout Randomization): aleatoriza la base del
  kernel para dificultar exploits.
- **SMEP/SMAP**: el kernel no puede ejecutar ni leer/escribir arbitrariamente
  páginas de usuario.
- **Kernel lockdown**: restringe qué puede hacer un proceso con `CAP_SYS_ADMIN`.
- **eBPF con verificador**: permite extender el kernel con programas seguros
  verificados estáticamente.

Aun así, ~2-3 vulnerabilidades críticas del kernel Linux se publican cada mes.

---

## Puntos clave

- **Monolítico = un único espacio de memoria en ring 0** para todo el código del
  OS: scheduler, MM, VFS, drivers, red, IPC.
- **Comunicación entre subsistemas = llamada a función ordinaria**, sin overhead
  de IPC. Eso lo hace muy rápido.
- **Un fallo en cualquier subsistema puede afectar a todo el kernel**, porque no
  hay aislamiento. Eso lo hace vulnerable.
- **Los módulos del kernel (`.ko`) son cargables dinámicamente** pero siguen
  ejecutando en ring 0: modularidad de desarrollo, no de seguridad.
- **Linux tiene ~36 millones de líneas de código** (2024); más del 60 % son
  drivers que la mayoría de sistemas nunca usan.
- **Los unikernels extraen solo los subsistemas que necesita una aplicación**
  (próximas fases): en vez de cargar 36 M de líneas, cargan decenas de miles.
  Esto elimina la superficie de ataque y reduce el footprint de memoria.

---

## Preguntas de autoevaluación

1. ¿En qué ring de la CPU corre el código del kernel Linux? ¿Y el código de una
   aplicación de usuario?

2. ¿Cuál es la diferencia entre un módulo del kernel y un driver compilado
   estáticamente en el kernel? ¿Afecta esa diferencia al aislamiento de fallos?

3. ¿Por qué la comunicación entre el scheduler y el MM es más rápida en Linux
   que en un microkernel como MINIX o seL4?

4. Un driver de audio tiene un buffer overflow explotable. ¿Qué puede conseguir
   un atacante en un kernel monolítico que no podría conseguir si el driver
   corriera en un proceso de usuario separado (modelo microkernel)?

5. Si un unikernel va a ejecutar una sola aplicación web que solo necesita TCP/IP
   y acceso a disco, ¿qué subsistemas de Linux podría descartar completamente?

6. ¿Qué es KASLR y qué tipo de ataque dificulta? ¿Lo elimina por completo?

---

## Respuestas de autoevaluación

**Pregunta 1.** El kernel Linux corre en **ring 0** (el nivel de privilegio máximo de la CPU, con acceso irrestricto a instrucciones y memoria). El código de una aplicación de usuario corre en **ring 3** (el nivel menos privilegiado, con acceso a instrucciones y memoria restringidos por hardware).

---

**Pregunta 2.** Un módulo `.ko` se carga dinámicamente con `insmod`/`modprobe` y puede descargarse con `rmmod` sin reiniciar el sistema. Un driver compilado estáticamente forma parte del bzImage del kernel y siempre está presente en memoria.

La diferencia **no afecta al aislamiento de fallos en absoluto**. Ambos corren en ring 0 una vez activos. Un bug en un módulo `.ko` tiene exactamente el mismo potencial de corromper el espacio del kernel y causar un kernel panic que un bug en código compilado estáticamente. La modularidad de Linux es de desarrollo y distribución (añadir soporte para hardware sin recompilar el kernel entero), no de seguridad.

---

**Pregunta 3.** En Linux, el scheduler llama al MM mediante una **llamada a función C ordinaria** (`call`/`ret`) dentro del mismo espacio de kernel en ring 0. Sin cambio de contexto, sin cambio de ring, sin copia de datos, sin serialización: una instrucción de máquina.

En un microkernel, scheduler y MM son componentes separados que se comunican mediante IPC: hay que serializar un mensaje, ejecutar el round-trip (cambio de contexto entre procesos, posible cambio de espacio de memoria, desencolar el mensaje en el otro extremo), procesar y devolver la respuesta. Ese proceso puede costar varios microsegundos — uno o dos órdenes de magnitud más que una llamada a función.

---

**Pregunta 4.** En un kernel monolítico (ring 0), un buffer overflow explotable en el driver de audio otorga al atacante **control total del procesador**. Puede leer y escribir cualquier dirección física de memoria, modificar las tablas de páginas de cualquier proceso, instalar rootkits, deshabilitar mecanismos de seguridad del kernel y tomar control completo de la máquina.

Si el driver corriera como proceso de usuario separado (modelo microkernel), el exploit solo comprometería ese proceso con privilegios mínimos. No podría acceder a la memoria del kernel ni de otros procesos, ni ejecutar instrucciones privilegiadas. La CPU le impediría físicamente cruzar esa frontera. El sistema continuaría funcionando con el daño contenido.

---

**Pregunta 5.** Una aplicación web que solo necesita TCP/IP y acceso a disco podría descartar de Linux:

- Todos los drivers de dispositivos no usados: USB, audio, Bluetooth, WiFi, GPU, puertos serie, SATA/SAS si usa solo NVMe (o viceversa), etc. (>60 % del código del kernel).
- Sistemas de ficheros no necesarios: btrfs, xfs, NFS, procfs, sysfs, tmpfs (si no los necesita).
- IPC de System V (colas de mensajes, semáforos System V, shared memory que no usa).
- Gestión multi-usuario, namespaces, cgroups si ejecuta una sola carga de trabajo.
- Módulos de seguridad mandatoria (SELinux, AppArmor): el aislamiento lo proporciona el hipervisor.
- Gran parte del MM (sin swap, gestión simplificada si es single-process).

En la práctica, un unikernel para ese caso usa una imagen de pocos MB frente a los ~30 MB de un kernel Linux estándar.

---

**Pregunta 6.** KASLR (Kernel Address Space Layout Randomization) aleatoriza la dirección base donde el kernel se carga en memoria en cada arranque. Dificulta los exploits que necesitan conocer la dirección exacta de funciones del kernel (como `commit_creds` o `prepare_kernel_cred`) para saltar a ellas o sobreescribirlas.

**No lo elimina por completo**: los exploits que utilizan vulnerabilidades de filtración de información de memoria (info leaks) pueden deducir la base del kernel en tiempo de ejecución y saltarse KASLR. Los exploits que no necesitan direcciones absolutas (como los que explotan primitivas de escritura arbitraria usando desplazamientos relativos) tampoco se ven afectados.

---

## Referencias

- **Código fuente de Linux:** <https://github.com/torvalds/linux> — El árbol
  completo. El directorio `kernel/` contiene el scheduler; `mm/` la gestión de
  memoria; `fs/` el VFS; `net/` la pila de red; `drivers/` los drivers.

- **The Linux Kernel documentation:** <https://www.kernel.org/doc/html/latest/>
  — Documentación oficial, especialmente las secciones "Core-API" y "Driver
  implementer's API guide".

- **"Linux Device Drivers" (Corbet, Rubini, Kroah-Hartman):**
  <https://lwn.net/Kernel/LDD3/> — Disponible libre online. Capítulos 1 y 2
  para la arquitectura general.

- **Debate Torvalds vs Tanenbaum (1992):**
  <https://www.oreilly.com/openbook/opensources/book/appa.html> — Contexto
  histórico del debate monolítico vs microkernel. Vale 20 minutos de lectura.

- **"Understanding the Linux Kernel" (Bovet & Cesati, O'Reilly):** Referencia
  clásica para profundizar en MM, scheduler y VFS. Algo desactualizado (cubre
  2.6) pero los fundamentos siguen siendo válidos.

- **LWN.net:** <https://lwn.net> — Noticias y artículos técnicos semanales
  sobre el desarrollo del kernel. Imprescindible para seguir la evolución real.

- **CVE-2022-0847 "Dirty Pipe":** <https://dirtypipe.cm4all.com/> — Artículo
  del descubridor. Ejemplo concreto de cómo un bug en una ruta de escritura de
  pipes permite escalar a root en un kernel monolítico.

---

## Notas personales

> Añade aquí tus notas, dudas y conexiones con lo que ya sabes al estudiar esta
> lección. Algunos puntos de partida:
>
> - ¿Cuántas líneas de código tiene el kernel que corre en tu máquina ahora
>   mismo? (`uname -r` y luego búscalo en kernel.org)
> - ¿Qué módulos tiene cargados tu sistema? (`lsmod | wc -l`)
> - ¿Cuánto ocupa el bzImage del kernel en tu /boot?

---

*← [Día 01 — Qué es un kernel](leccion-01-que-es-un-kernel.md) · [Día 03 — Microkernel](leccion-03-microkernel.md) →*
