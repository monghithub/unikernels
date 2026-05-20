# Día 01 — Qué es un kernel y para qué sirve

> **Fase:** 1 · **Semana:** 1 · **Tipo:** Lectura · **Duración:** 30 min

---

## Contexto y motivación

Cuando escribes un programa en Go, Python o Rust y llamas a `os.ReadFile()`, `open()` o `std::fs::read()`, el hardware no entiende esa instrucción. El disco no sabe qué es un "archivo"; la RAM no sabe qué es un "proceso". Entre tu código y el silicio hay una capa de software que traduce esas ideas de alto nivel a operaciones físicas concretas. Esa capa es el kernel.

Para entender los unikernels —el objetivo de este curso— necesitas comprender primero qué problema resuelve el kernel tradicional, porque los unikernels no eliminan esa capa: la rediseñan radicalmente. Un unikernel elimina la frontera entre tu aplicación y el kernel y fusiona ambos en un solo binario. Para apreciar lo que eso significa —y por qué es importante en producción— tienes que saber qué separa esa frontera y por qué existe.

Esta primera lección te da el mapa completo: qué es el kernel, qué problemas surgieron antes de que existiera, qué servicios ofrece hoy y por qué la distinción "kernel/usuario" es tan fundamental. Todo lo que aprendas aquí será el terreno sobre el que construiremos las semanas siguientes.

---

## Concepto principal

El kernel es el componente central de un sistema operativo. Es un programa —un binario— que se carga en memoria antes que cualquier otra cosa y que nunca deja de ejecutarse mientras el sistema está encendido. Su función es actuar de **árbitro y traductor** entre los programas que quieren hacer cosas (tu aplicación) y el hardware que puede hacerlas (CPU, RAM, disco, red).

```
  ┌─────────────────────────────────────────────────────┐
  │                  ESPACIO DE USUARIO                  │
  │   ┌──────────┐   ┌──────────┐   ┌──────────────┐   │
  │   │  tu app  │   │  nginx   │   │  base de     │   │
  │   │  (Go)    │   │          │   │  datos       │   │
  └───┴────┬─────┴───┴────┬─────┴───┴──────┬───────┘───┘
           │  syscall      │  syscall        │  syscall
  ─────────┼───────────────┼─────────────────┼────────────
  ┌────────▼───────────────▼─────────────────▼──────────┐
  │                       KERNEL                         │
  │   ┌──────────┐   ┌──────────┐   ┌──────────────┐   │
  │   │ gestión  │   │ gestión  │   │  drivers /   │   │
  │   │ memoria  │   │ procesos │   │  dispositivos│   │
  │   └──────────┘   └──────────┘   └──────────────┘   │
  └───────────────────────────┬─────────────────────────┘
                              │ acceso directo al HW
  ┌─────────────────────────────────────────────────────┐
  │                     HARDWARE                         │
  │    CPU       RAM       Disco NVMe       NIC          │
  └─────────────────────────────────────────────────────┘
```

La interfaz entre el espacio de usuario y el kernel se llama **system call** (syscall). Cuando tu programa llama a `read()`, lo que ocurre es: tu código emite una instrucción especial de CPU (`syscall` en x86-64) que transfiere el control al kernel, el kernel ejecuta la operación en nombre de tu programa, y luego devuelve el resultado. Tu programa nunca toca el hardware directamente.

---

## Desarrollo (los 30 minutos)

### Parte 1 — El problema que el kernel resuelve (~10 min)

**El mundo sin kernel: cada programa habla con el hardware directamente**

Imagina que no existiera el kernel. Tu programa Go que hace `os.ReadFile("datos.csv")` tendría que:

1. Saber el modelo exacto del controlador NVMe instalado en la máquina.
2. Enviar comandos de bajo nivel al bus PCIe para iniciar una transferencia DMA.
3. Gestionar la interrupción que llega cuando el disco termina la lectura.
4. Calcular en qué bloque físico del disco están los bytes del archivo.
5. Copiar los datos desde el buffer del controlador a su propia memoria.

Eso es lo que hacían los programas en los años 50: cada programa incluía sus propios "drivers". Los problemas que surgieron fueron inmediatos:

- **Colisión**: si dos programas intentaban escribir al disco al mismo tiempo, corrompían datos.
- **Monopolio**: si un programa se colgaba en un bucle, la CPU no ejecutaba nada más.
- **Inseguridad**: cualquier programa podía leer o machacar la memoria de cualquier otro.
- **Reescritura perpetua**: cada programador reescribía el mismo código de acceso a disco una y otra vez para cada modelo de hardware.

El kernel nació para resolver todos estos problemas con una idea central: **un único árbitro de confianza que intermedie todo acceso al hardware**.

**La abstracción como herramienta**

El kernel convierte hardware heterogéneo en abstracciones uniformes:

| Hardware real                        | Abstracción del kernel |
|--------------------------------------|------------------------|
| Sectores de disco NVMe / SATA / SAS  | Archivo (`/dev/sda`)   |
| Celdas de DRAM con direcciones físicas | Página de memoria virtual |
| Paquetes Ethernet con MACs y CRCs    | Socket TCP/IP          |
| Núcleos de CPU                       | Proceso / hilo         |
| Registros de dispositivos USB        | Archivo en `/dev/`     |

Gracias a estas abstracciones, tu programa Go llama a la misma función `os.ReadFile()` sin importar si el disco es un NVMe PCIe 5.0, un USB 2.0 o una unidad de red NFS montada en Europa. El kernel se encarga de la diferencia.

---

### Parte 2 — Los servicios que ofrece el kernel (~15 min)

Un kernel moderno (Linux, FreeBSD, Windows NT, XNU de macOS) ofrece cinco servicios fundamentales. Todos los demás se construyen sobre ellos.

#### 1. Gestión de memoria

Cada proceso cree que tiene para sí solo un espacio de memoria contiguo que empieza en la dirección 0. Eso es mentira: es **memoria virtual**. El kernel mantiene una tabla de traducción (la "page table") que mapea cada dirección virtual a una dirección física real en la RAM.

```
  Proceso A                    RAM física
  ─────────────                ─────────────────
  0x0000 - 0x1000  ──────────► frame 47 (0x2F000)
  0x1000 - 0x2000  ──────────► frame 12 (0x0C000)
  0x2000 - 0x3000  ──────────► swap en disco
                               frame 23 (libre)

  Proceso B
  ─────────────
  0x0000 - 0x1000  ──────────► frame 91 (0x5B000)  ← misma dir. virtual,
  0x1000 - 0x2000  ──────────► frame 23 (0x17000)    física diferente
```

Esto garantiza **aislamiento**: el proceso A no puede leer ni escribir en las páginas del proceso B, porque simplemente no tiene entradas en su page table que apunten a esos frames. Si lo intenta, la CPU lanza una excepción (`segfault`) y el kernel mata el proceso.

Adicionalmente, el kernel gestiona el **swap**: cuando la RAM se llena, mueve páginas poco usadas al disco y las trae de vuelta cuando se necesitan, de forma transparente para los procesos.

#### 2. Gestión de procesos

El kernel es el único que puede crear, pausar y matar procesos. Para la CPU solo existe un flujo de instrucciones en cada núcleo en cada instante; el kernel crea la ilusión de multitarea dividiendo el tiempo del procesador en pequeñas rebanadas y alternando qué proceso se ejecuta.

Este mecanismo se llama **scheduler** (planificador). En Linux, el scheduler por defecto es el CFS (Completely Fair Scheduler), que intenta dar a cada proceso una proporción justa del tiempo de CPU ponderada por su prioridad.

```
  Timeline de CPU (1 núcleo):

  ──[proceso A]──[proceso B]──[proceso A]──[kernel]──[proceso C]──[proceso A]──►
        4ms          4ms          4ms       (ISR)        4ms          4ms
                                          interrupción
                                           de reloj
```

Cada cambio de contexto implica guardar el estado completo del proceso que sale (registros de CPU, puntero a su page table, etc.) y restaurar el estado del proceso que entra. Esto tiene un coste: entre 1 y 10 microsegundos típicamente en hardware moderno.

#### 3. Drivers y gestión de dispositivos

Ningún dispositivo de hardware habla el mismo idioma. Un driver es un módulo de software que conoce el protocolo específico de un tipo de dispositivo y expone una interfaz uniforme al resto del kernel.

En Linux, la mayoría de los dispositivos se presentan como archivos en `/dev/` o en `/sys/`. Esto es la filosofía Unix: "todo es un archivo". Escribir en `/dev/sda` significa escribir bytes directamente al disco. Escribir en `/dev/null` descarta los bytes. Leer de `/dev/urandom` devuelve bytes aleatorios generados por el hardware.

#### 4. Sistema de ficheros (VFS)

El kernel implementa una capa llamada **VFS** (Virtual File System) que proporciona una interfaz unificada de archivos y directorios independiente del sistema de ficheros subyacente. Tu programa llama a `open("/home/user/datos.csv")` sin saber si ese path está en ext4, btrfs, xfs, NFS, tmpfs o procfs. El VFS delega al driver del FS correcto.

```
  open("/proc/cpuinfo")          open("/home/user/datos.csv")
         │                                    │
         ▼                                    ▼
  ┌─────────────────────────────────────────────┐
  │              VFS (capa virtual)              │
  └────────┬────────────────────────┬────────────┘
           │                        │
           ▼                        ▼
      driver procfs             driver ext4
    (genera datos al vuelo)   (lee del disco NVMe)
```

#### 5. Pila de red

El kernel implementa la pila TCP/IP completa. Cuando tu programa Go llama a `net.Dial("tcp", "example.com:443")`, el kernel:

1. Resuelve la dirección IP (puede delegar a una librería de usuario, pero el socket DNS es una syscall).
2. Establece la conexión TCP (three-way handshake) gestionando los buffers de envío y recepción.
3. Encapsula los datos en segmentos TCP, luego en datagramas IP, luego en frames Ethernet.
4. Envía los frames al driver de la tarjeta de red (NIC).
5. Gestiona las retransmisiones si se pierden paquetes.

Tu programa solo ve un stream de bytes. La complejidad de la red es invisible.

---

### Parte 3 — La distinción kernel/usuario y su relevancia para unikernels (~5 min)

**Por qué existe la frontera**

La separación entre espacio de kernel y espacio de usuario no es solo conceptual: está implementada en el hardware de la CPU mediante **anillos de privilegio** (privilege rings). En x86-64:

```
  Ring 0  ──  Kernel: acceso total al hardware, a todos los registros,
              a toda la memoria física. Una instrucción errónea aquí
              puede colgar la máquina entera.

  Ring 1  ──  (sin uso en sistemas modernos)
  Ring 2  ──  (sin uso en sistemas modernos)

  Ring 3  ──  Espacio de usuario: acceso restringido. No puede ejecutar
              instrucciones privilegiadas. No puede acceder a memoria
              fuera de su espacio virtual. Un error aquí solo mata
              ese proceso.
```

Para cruzar de Ring 3 a Ring 0 el proceso debe emitir una instrucción `syscall`. La CPU comprueba que la llamada sea legítima (está en la tabla de syscalls del kernel), ejecuta el código del kernel en Ring 0, y devuelve el control al proceso en Ring 3 cuando termina. Este cambio de anillo tiene un coste medible.

**El coste de la frontera**

Cada syscall implica:
- Un cambio de contexto parcial (flush de algunos cachés de CPU en sistemas modernos).
- En sistemas con mitigaciones de Spectre/Meltdown activas (prácticamente todos desde 2018): un flush adicional del TLB (Translation Lookaside Buffer), que puede costar entre 100 y 1000 ns por syscall.

Una aplicación de red de alto rendimiento en Linux puede emitir millones de syscalls por segundo. El coste de las mitigaciones de seguridad se mide en 5-30% de degradación de rendimiento en cargas de trabajo intensivas.

**El puente hacia los unikernels**

Ahora tienes el contexto necesario para entender el argumento central de los unikernels:

> Si tu aplicación va a ejecutarse en una máquina virtual de un proveedor cloud, aislada del mundo exterior por el hypervisor, ¿qué sentido tiene mantener la frontera kernel/usuario dentro de esa VM? ¿Qué ganas con el aislamiento entre procesos si solo va a ejecutarse un proceso? ¿Qué ganas con los drivers genéricos de Linux para 5.000 modelos de hardware si el "hardware" que ve tu VM son solo tres o cuatro dispositivos virtuales estándar?

Los unikernels responden estas preguntas eliminando las capas que no aportan valor en ese contexto específico y compilando la aplicación junto con solo los servicios del kernel que realmente necesita, en un único binario que arranca en milisegundos y expone una superficie de ataque mínima.

En las próximas lecciones veremos cómo Linux organiza esas capas internamente, qué diferencias hay con los microkernels, y cómo surgió históricamente la idea de un "library OS". Todo ese camino nos llevará a entender qué son Nanos, Unikraft y MirageOS, y por qué importan.

---

## Puntos clave

- El kernel es el árbitro que intermedia todo acceso al hardware; los programas nunca hablan directamente con el silicio.
- Su razón de ser es el aislamiento, la abstracción y la gestión de recursos compartidos (CPU, RAM, dispositivos).
- Los cinco servicios fundamentales son: gestión de memoria (virtual), gestión de procesos (scheduler), drivers, VFS y pila de red.
- La frontera kernel/usuario está implementada en hardware mediante anillos de privilegio de la CPU; cruzarla tiene un coste medible, especialmente con las mitigaciones de Spectre/Meltdown.
- Los unikernels cuestionan si esa frontera tiene sentido cuando la aplicación está aislada por un hypervisor y solo ejecuta una carga de trabajo.

---

## Preguntas de autoevaluación

1. Cuando tu programa Python hace `open("datos.csv", "r")`, ¿cuántas veces cruza la frontera kernel/usuario antes de devolverte los primeros bytes del archivo? Enumera los pasos a alto nivel.

2. El kernel de Linux pesa más de 30 MB compilado y contiene drivers para miles de dispositivos que tu servidor en AWS nunca va a ver. ¿Qué coste tiene eso en términos de memoria, superficie de ataque y tiempo de arranque? ¿Puede ser un problema?

3. Dos procesos en la misma máquina tienen cada uno un puntero a la dirección virtual `0x00007fff_deadbeef`. ¿Apuntan al mismo byte físico de RAM? ¿Por qué sí o por qué no?

---

## Referencias

| Recurso | Por qué leerlo |
|---------|----------------|
| *Operating Systems: Three Easy Pieces* — Arpaci-Dusseau (cap. 1-4, libre en ostep.org) | La mejor introducción moderna a kernels; cubre virtualización de CPU y memoria con ejemplos en C que puedes compilar. |
| `man 2 syscall` y `man 2 intro` en tu terminal Linux | Lista completa de syscalls del kernel Linux con descripción de cada una; es el contrato entre tu programa y el kernel. |
| *The Linux Programming Interface* — Kerrisk (cap. 2-3) | Profundiza en la interfaz de syscalls de Linux con ejemplos prácticos; capítulo 2 cubre el concepto de kernel vs. usuario en detalle. |
| Artículo "What is the Linux Kernel?" — kernel.org/doc/html/latest/admin-guide/README.html | Perspectiva oficial; breve pero densa en terminología que usarás en el resto del curso. |
| Charla "How the Kernel Manages Your Memory" — Gustavo Duarte (duartes.org/gustavo/blog/) | Visualizaciones excelentes del espacio de memoria de un proceso y la page table; complementa perfectamente la Parte 2. |

---

## Notas personales

> _Escribe aquí tus notas._

---

*← (inicio del curso) · [Día 02 — Kernel monolítico: Linux por dentro](leccion-02-kernel-monolitico.md) →*
