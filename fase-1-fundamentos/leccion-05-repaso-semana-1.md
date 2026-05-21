# Día 05 — Repaso semana 1: cuestionario de 40 preguntas

> **Fase:** 1 · **Semana:** 1 · **Tipo:** Repaso · **Duración:** 30 min

---

## Cómo usar este cuestionario

Divide los 30 minutos en dos partes: **20 min** para responder todas las preguntas por escrito (sin mirar las lecciones), **10 min** para revisar las soluciones y anotar qué conceptos necesitas reforzar.

Cada pregunta vale 2 puntos (1 punto las marcadas con ★). Puntuación máxima: 78 puntos.

- **70–78**: listo para avanzar.
- **50–69**: repasa las lecciones indicadas en los encabezados de bloque.
- **< 50**: dedica media sesión extra a releer las lecciones flojas antes del día 06.

---

## Mapa mental de la semana

```
               ┌─────────────────────────────────┐
               │             KERNEL               │
               │  árbitro entre app y hardware    │
               └──────────────┬──────────────────┘
          ┌───────────────────┼────────────────────┐
          ▼                   ▼                    ▼
   Monolítico           Microkernel          (Unikernel)
   (Linux)              (Mach/QNX/seL4)      — futuro →
   todo en ring 0       mínimo en ring 0
   rendimiento ↑        aislamiento ↑
   resiliencia ↓        latencia IPC ↑
          │                   │
          └─────────┬─────────┘
                    ▼
          Frontera user / kernel
          ┌─────────────────────────┐
          │  User space  (ring 3)   │
          │  Kernel space (ring 0)  │
          │  Cruce = syscall        │
          └─────────────────────────┘
```

---

## Bloque I — Qué es un kernel (lección 01)

**1.** ¿Cuál es la función principal del kernel en un sistema operativo moderno?

**2.** Enumera los cinco servicios fundamentales que ofrece un kernel moderno.

**3.** ¿Qué problema concreto se producía cuando no existía un kernel y cada programa hablaba directamente con el hardware? Menciona al menos tres consecuencias.

**4.** ¿Qué es la memoria virtual y qué garantía de seguridad proporciona entre procesos?

**5.** ¿Qué es el VFS (Virtual File System)? ¿Por qué una aplicación puede llamar a `open("/home/user/datos.csv")` sin saber si el disco es NVMe o NFS?

**6.** ¿Qué es una syscall? Explica la mecánica básica en tres pasos, sin entrar en detalles de instrucciones específicas de CPU.

**7 ★.** ¿En qué ring de la CPU corre el kernel? ¿Y las aplicaciones de usuario?

**8.** ¿Qué son las page tables y para qué sirven? ¿Qué estructura de datos del kernel Linux las representa por proceso?

**9.** ¿Por qué las mitigaciones de Spectre/Meltdown incrementan el coste de las syscalls? ¿Cuánto puede degradar el rendimiento en cargas de trabajo intensivas?

**10.** Formula en una frase el argumento central por el que los unikernels cuestionan la frontera kernel/usuario en entornos cloud.

---

## Bloque II — Kernel monolítico (lección 02)

**11.** Define "kernel monolítico" en términos de espacio de memoria y nivel de privilegio.

**12.** ¿Por qué Tanenbaum llamó a Linux "un paso atrás a los años 70" en su mensaje de 1992? ¿Cuál era su argumento?

**13.** ¿Qué es un módulo del kernel (`.ko`)? ¿Cómo se diferencia de un driver compilado estáticamente? ¿Afecta esa diferencia al aislamiento de fallos?

**14.** ¿Qué es el CFS (Completely Fair Scheduler)? ¿Desde qué versión de Linux es el planificador por defecto?

**15.** ¿Qué hace el OOM killer y cuándo se activa?

**16.** Un driver de audio en Linux tiene un buffer overflow explotable. ¿Qué puede conseguir un atacante? Contrástalo con lo que conseguiría si el driver corriera como proceso de usuario separado.

**17.** ¿Qué es KASLR? ¿Qué tipo de ataque mitiga y por qué no lo elimina por completo?

**18 ★.** ¿Qué porcentaje aproximado del código del kernel Linux (versión 6.x) son drivers? ¿Qué implica eso para un unikernel que solo necesita TCP/IP y acceso a disco?

**19.** ¿Cuál es la ventaja de rendimiento clave del kernel monolítico respecto al microkernel en la comunicación entre subsistemas? Sé preciso: ¿qué instrucción de CPU se usa y cuánto cuesta?

**20.** ¿Qué es el page cache en Linux, quién lo gestiona y por qué es el mayor consumidor de RAM en servidores?

---

## Bloque III — Microkernels (lección 03)

**21.** ¿Cuáles son los tres componentes que un microkernel mantiene en ring 0? Explica brevemente por qué cada uno debe estar en ring 0.

**22.** ¿En qué universidad se diseñó el kernel Mach y en qué kernel de producción actual se usa como base?

**23.** ¿Qué es XNU y en qué plataformas corre? ¿Es un microkernel puro? Justifica.

**24.** ¿Qué es QNX Neutrino y en qué tipo de sistemas se despliega principalmente? Da dos ejemplos de industria.

**25.** ¿Qué significa que seL4 esté "formalmente verificado"? ¿Qué herramienta matemática se usó?

**26.** ¿Qué garantiza la verificación formal de seL4 y qué explícitamente NO garantiza?

**27.** Describe el coste de rendimiento que paga un microkernel respecto a un kernel monolítico para leer un fichero. ¿Cuántos saltos de IPC hay y qué implica cada uno?

**28.** ¿Qué ocurre si el servidor de sistema de ficheros de un sistema basado en microkernel tiene un bug de segmentación y muere? Contrástalo con lo que ocurriría en Linux.

**29.** ¿Qué es un Library OS? ¿En qué se diferencia de un microkernel en cuanto a qué comparten las aplicaciones?

**30.** ¿Qué parte de la predicción de Tanenbaum en 1992 sobre los microkernels resultó correcta y qué parte fue errónea?

---

## Bloque IV — User space vs kernel space (lección 04)

**31 ★.** ¿Qué rango de direcciones virtuales ocupa el user space en x86-64 con 48 bits de direccionamiento?

**32.** ¿Por qué existe el "agujero canónico" en el espacio de direcciones virtuales de x86-64? ¿Qué ocurre si se intenta desreferenciar una dirección no canónica?

**33.** ¿Qué instrucción ejecuta una aplicación Linux en x86-64 para hacer una syscall? ¿Qué registro contiene el número de syscall y cuáles los argumentos?

**34.** ¿Qué registro de la CPU se carga en cada cambio de proceso (context switch)? ¿Por qué modificar ese registro es una operación privilegiada?

**35.** Un proceso llama a `malloc(128)` doscientas veces seguidas. ¿Cuántas syscalls al kernel se generarán aproximadamente? Justifica la respuesta explicando el rol de la libc.

**36.** Nombra tres instrucciones privilegiadas de x86-64 que solo pueden ejecutarse en ring 0 y explica por qué serían peligrosas si estuvieran disponibles en ring 3.

**37.** ¿Cuál es la diferencia de coste entre una syscall en Linux sin KPTI y con KPTI activo? ¿Qué operación extra hace el procesador con KPTI?

**38.** En un unikernel, ¿cómo se llama internamente al equivalente de una syscall? ¿Cuántos ciclos de overhead tiene esa llamada comparada con una syscall en Linux?

**39.** ¿Qué es `mm_struct` en Linux? ¿Qué información contiene y cuándo se usa?

**40.** Si un unikernel no tiene separación user/kernel space, ¿qué proporciona el aislamiento entre dos instancias del mismo unikernel corriendo en el mismo servidor físico? ¿Qué tecnología de hardware lo hace posible?

---

## Soluciones

### Bloque I — Qué es un kernel

**1.** El kernel actúa de **árbitro y traductor** entre las aplicaciones y el hardware: gestiona el acceso a la CPU, la memoria, los dispositivos y la red, garantizando que varios programas puedan coexistir sin corromperse entre sí y sin necesidad de que cada uno sepa hablar con el hardware directamente.

---

**2.** Los cinco servicios fundamentales son:
1. **Gestión de memoria**: espacio de direcciones virtual por proceso, page tables, swap.
2. **Gestión de procesos**: creación, planificación (scheduler), destrucción de procesos e hilos.
3. **Drivers y gestión de dispositivos**: interfaz uniforme sobre hardware heterogéneo.
4. **VFS (Virtual File System)**: abstracción de archivos y directorios sobre cualquier sistema de ficheros.
5. **Pila de red**: implementación de TCP/IP, sockets, routing.

---

**3.** Sin kernel, los problemas eran: (a) **colisión**: dos programas accediendo al disco simultáneamente corrompían datos; (b) **monopolio**: un programa en bucle ocupaba la CPU indefinidamente impidiendo ejecutar cualquier otro; (c) **inseguridad**: cualquier programa podía leer o machacar la memoria de cualquier otro; (d) **reescritura perpetua**: cada programador reescribía los mismos drivers para cada modelo de hardware.

---

**4.** La memoria virtual es una abstracción donde cada proceso cree que tiene para sí solo un espacio de memoria contiguo desde la dirección 0. El kernel mantiene una page table por proceso que mapea cada dirección virtual a una dirección física real. La garantía de seguridad es el **aislamiento**: el proceso A no tiene entradas en su page table que apunten a los frames del proceso B, por lo que no puede leer ni escribir su memoria. Un intento provoca un page fault que el kernel convierte en segfault matando el proceso infractor.

---

**5.** El VFS es una capa de abstracción dentro del kernel que define objetos comunes (`inode`, `dentry`, `file`) e interfaces (`open`, `read`, `write`, `close`) que todo sistema de ficheros debe implementar. Cuando la aplicación llama a `open(path)`, el VFS resuelve el path, identifica qué sistema de ficheros está montado en ese punto y delega al driver concreto (ext4, btrfs, NFS…). La aplicación nunca sabe qué driver está debajo: solo ve la interfaz uniforme del VFS.

---

**6.** Una syscall es la única forma legal de que un proceso de usuario pida servicios al kernel. Los tres pasos básicos: (1) el proceso señaliza al hardware qué operación quiere (pone el número de syscall y sus argumentos en registros); (2) emite una instrucción especial que hace que la CPU transfiera el control al kernel elevando el nivel de privilegio; (3) el kernel ejecuta la operación en nombre del proceso y devuelve el resultado, bajando de nuevo el nivel de privilegio para retornar al proceso.

---

**7 ★.** El kernel corre en **ring 0**. Las aplicaciones de usuario corren en **ring 3**.

---

**8.** Las page tables son estructuras de datos que el kernel mantiene por proceso para traducir cada dirección virtual a su dirección física correspondiente en la RAM. Cuando el procesador accede a una dirección virtual, la MMU consulta la page table activa (una jerarquía de hasta 4-5 niveles en x86-64) para obtener la dirección física. En Linux, la estructura por proceso se llama `mm_struct` y dentro de ella hay un puntero `pgd` que apunta a la tabla de páginas de nivel 4 (PGD/PML4).

---

**9.** Las mitigaciones de Spectre/Meltdown (principalmente KPTI para Meltdown) requieren cambiar la tabla de páginas activa al cruzar la frontera user/kernel en cada syscall. Cambiar la tabla de páginas invalida el TLB (Translation Lookaside Buffer), la caché hardware de traducciones de direcciones. Repoblar el TLB cuesta entre centenares y miles de ciclos de CPU. En cargas de trabajo con millones de syscalls por segundo, la degradación medida es de **5–30 %** de rendimiento.

---

**10.** Si una aplicación se ejecuta aislada por un hipervisor dentro de una VM, la frontera kernel/usuario no añade seguridad observable (ya hay aislamiento de la VM), solo añade el coste de los cambios de ring y los flushes de TLB en cada operación de I/O o red.

---

### Bloque II — Kernel monolítico

**11.** Un kernel monolítico ejecuta **todos los servicios del sistema operativo** (scheduler, gestión de memoria, VFS, pila de red, drivers, IPC) en un **único espacio de direcciones compartido** con **nivel de privilegio máximo (ring 0)**. No hay fronteras de proceso entre subsistemas dentro del kernel.

---

**12.** Tanenbaum argumentaba que un kernel monolítico mezcla todo el código del OS en un único bloque de ring 0, sin separación entre componentes. Si cualquier parte falla, el sistema entero cae. Los microkernels, en cambio, minimizan el código en ring 0 y sacan el resto a user space, mejorando la resiliencia y la verificabilidad. Para Tanenbaum, ese diseño era superior y Linux estaba repitiendo los errores de los años 70.

---

**13.** Un módulo `.ko` es un objeto de código que se carga dinámicamente con `insmod`/`modprobe` y puede descargarse con `rmmod` sin reiniciar. Un driver estático se compila dentro del bzImage del kernel y siempre está presente. **No afecta al aislamiento de fallos**: ambos corren en ring 0 una vez activos con acceso total al espacio del kernel. La modularidad es de desarrollo y distribución, no de seguridad.

---

**14.** El CFS (Completely Fair Scheduler) es el planificador por defecto de Linux desde la versión **2.6.23** (2007). Intenta dar a cada proceso una proporción justa del tiempo de CPU ponderada por su prioridad (nice value / cgroups), usando un árbol rojo-negro ordenado por tiempo virtual de ejecución acumulado.

---

**15.** El OOM killer (Out-Of-Memory killer) se activa cuando el sistema se queda sin memoria RAM y sin espacio de swap disponible. El kernel no puede satisfacer ninguna nueva petición de memoria. En ese momento, el OOM killer selecciona un proceso según una heurística (mayor consumo de memoria, menor importancia del sistema) y lo mata para liberar memoria y permitir que el sistema continúe funcionando.

---

**16.** En un kernel monolítico (ring 0), el atacante obtiene control total del procesador: puede leer y escribir cualquier dirección física, modificar page tables de cualquier proceso, instalar rootkits persistentes o deshabilitar mecanismos de seguridad del kernel. Control total de la máquina.

Si el driver corriera como proceso de usuario separado (microkernel), el exploit solo comprometería ese proceso con privilegios mínimos. La CPU impide físicamente acceder a memoria del kernel o de otros procesos, y ejecutar instrucciones privilegiadas. El sistema sigue funcionando con el daño contenido en ese proceso.

---

**17.** KASLR (Kernel Address Space Layout Randomization) aleatoriza la dirección base donde el kernel se carga en memoria en cada arranque. Dificulta los exploits que necesitan conocer la dirección exacta de funciones del kernel (como `commit_creds`) para saltar a ellas o sobreescribirlas. No lo elimina: los exploits con *info leaks* (vulnerabilidades que filtran información de memoria) permiten deducir la base en tiempo de ejecución y saltarse KASLR.

---

**18 ★.** Más del **60 %** del código de Linux son drivers (~22 millones de líneas de los ~36 millones totales en 2024). Un unikernel que solo necesita TCP/IP y acceso a disco puede descartar el 90 % de ese código: todos los drivers de USB, audio, GPU, Bluetooth, WiFi, puertos serie, protocolos de red alternativos, sistemas de ficheros no usados, etc. La imagen resultante puede ser de decenas de KB a pocos MB frente a los 30+ MB de Linux.

---

**19.** La ventaja es la **comunicación entre subsistemas mediante llamadas a función ordinarias** (`CALL`/`RET`). En Linux, el scheduler llama al MM con una instrucción `call`: sin cambio de contexto, sin cambio de ring, sin copia de datos. Cuesta nanosegundos. En un microkernel, el mismo intercambio requiere un round-trip IPC completo: serialización, cambio de contexto entre procesos, posible cambio de espacio de memoria. Cuesta microsegundos.

---

**20.** El page cache es una zona del espacio del kernel gestionada por el Memory Manager donde se guardan en RAM las páginas de datos de ficheros leídas del disco. Cuando una aplicación lee el mismo fichero dos veces, la segunda lectura se sirve desde RAM sin tocar el disco. Es el mayor consumidor de RAM en servidores porque Linux usa agresivamente la RAM libre para caché (principio "free RAM is wasted RAM"): en un servidor con 32 GB, puede haber 20+ GB de page cache.

---

### Bloque III — Microkernels

**21.** Los tres componentes son:

- **IPC**: debe estar en ring 0 porque es el mecanismo por el que todos los servidores se comunican; si estuviera en user space habría una dependencia circular irresoluble al arrancar.
- **Gestión básica de memoria**: mapear páginas físicas a espacios de direcciones virtuales requiere escribir `CR3` e interactuar con la MMU, operaciones solo accesibles desde ring 0.
- **Planificador**: responde a interrupciones de timer (hardware privilegiado) y ejecuta cambios de contexto que implican guardar/restaurar registros privilegiados y cambiar `CR3`.

---

**22.** Mach fue diseñado en la **Universidad Carnegie Mellon** (CMU) a mediados de los 80. Hoy es la base de **XNU**, el kernel de macOS e iOS de Apple.

---

**23.** XNU (X is Not Unix) es el kernel de **macOS, iOS, iPadOS, tvOS y watchOS**. No es un microkernel puro: usa Mach como núcleo para IPC, gestión de memoria y threads, pero le añade una capa BSD encima para la interfaz POSIX. El código BSD corre en el espacio del kernel (ring 0), no como servidor de usuario. Es un diseño **híbrido**: parte del legado del microkernel Mach pero con servicios del OS en ring 0 por rendimiento.

---

**24.** QNX Neutrino es un microkernel RTOS (Real-Time Operating System) desarrollado por QNX Software Systems (adquirido por BlackBerry). Se despliega principalmente en sistemas de **misión crítica**: (1) **automoción** — sistemas de infotainment de BMW, Audi, Ford, Porsche; (2) **aviónica** — sistemas de cabina en Boeing y Airbus; también en médico (marcapasos, equipos de imagen) e industrial (reactores nucleares, PLCs).

---

**25.** Formalmente verificado significa que se construyó una **prueba matemática completa**, sin huecos, de que el código C de seL4 (~8.700 líneas) es una refinación correcta de su especificación funcional abstracta. La herramienta usada fue el asistente de pruebas **Isabelle/HOL**. La prueba tiene ~200.000 líneas y cubre todos los estados posibles del sistema.

---

**26.** **Garantiza**: ausencia de deadlocks, ausencia de acceso a memoria no inicializada, que el sistema de capacidades no puede ser bypasseado (ningún proceso obtiene acceso a un recurso sin concesión explícita), y que el comportamiento del código C es exactamente el de la especificación abstracta.

**No garantiza**: bugs en el hardware, en el compilador GCC, en el bootloader, en los drivers del sistema (que corren fuera del kernel en user space), ni errores en la propia especificación abstracta.

---

**27.** Para leer un fichero, un microkernel necesita al menos cuatro saltos IPC: (1) la app → servidor de ficheros (IPC con cambio de contexto); (2) el servidor de ficheros → driver de disco (IPC con cambio de contexto); (3) el driver de disco hace la operación y responde al servidor de FS (IPC de vuelta); (4) el servidor de FS responde a la app (IPC de vuelta). Cada salto IPC implica serialización de mensaje, cambio de proceso y posiblemente cambio de espacio de memoria. En Linux monolítico, toda esa ruta es código en ring 0 conectado por llamadas a función ordinarias.

---

**28.** **Microkernel**: el servidor FS es un proceso de user space con su propio espacio de memoria. Al morir, el kernel lo marca como proceso muerto. El resto del sistema continúa: el kernel, los drivers de red, otras apps. Los procesos que esperaban respuesta del FS reciben un error de IPC. El servidor puede reiniciarse sin reiniciar el sistema.

**Linux**: el código del FS corre en ring 0 compartiendo espacio con todo el kernel. Una corrupción de memoria puede afectar estructuras del planificador, del MM u otros subsistemas. Resultado probable: kernel panic y caída completa del sistema.

---

**29.** Un Library OS es un sistema donde cada aplicación lleva su propio OS **enlazado como una biblioteca estática**. No hay ningún OS compartido entre aplicaciones: cada instancia tiene su propia pila TCP/IP, su propio VFS, sus propios drivers. El kernel subyacente solo multiplexa el hardware de forma segura.

Un microkernel tiene servidores de OS compartidos entre todas las aplicaciones (mismo servidor de ficheros, misma pila de red). Un Library OS elimina esa compartición: los servicios del OS son privados y la aplicación los lleva consigo.

---

**30.** **Acertó en**: los sistemas de misión crítica (aviónica, automoción, médico) son dominados por microkernels (QNX, VxWorks). La verificación formal se demostró posible (seL4). La brecha de rendimiento se cerró con microkernels de tercera generación (familia L4).

**Falló en**: Linux no se volvió obsoleto. Los servidores, escritorio y Android dominan el mercado de propósito general. El rendimiento fue el factor determinante y los microkernels tardaron décadas en ser competitivos en ese espacio.

---

### Bloque IV — User space vs kernel space

**31 ★.** De `0x0000000000000000` a `0x00007FFFFFFFFFFF` (mitad inferior canónica del espacio de 64 bits, con el bit 47 en 0 y los bits superiores todos a 0).

---

**32.** Los procesadores x86-64 actuales implementan solo **48 bits** de espacio de direcciones virtual aunque los punteros tienen 64 bits. El hardware exige que los bits 48–63 sean todos iguales al bit 47 (extensión de signo). Las direcciones que no cumplen esa regla son "no canónicas". Si se intenta desreferenciar una dirección no canónica, la CPU lanza una excepción de protección general (**#GP, vector 13**) que el kernel convierte normalmente en `SIGSEGV` para el proceso. Esto deja un agujero no utilizable entre `0x0000800000000000` y `0xFFFF7FFFFFFFFFFF`.

---

**33.** La instrucción es **`SYSCALL`** (x86-64 moderno; `int 0x80` es la forma de 32 bits). El número de syscall va en el registro **`RAX`**. Los argumentos van en **`RDI`, `RSI`, `RDX`, `R10`, `R8`, `R9`** (en ese orden, hasta seis argumentos).

---

**34.** El registro **`CR3`** (Control Register 3). Contiene la dirección física de la tabla de páginas de nivel 4 (PGD/PML4) del proceso activo. Cambiarlo hace que toda la MMU comience a traducir direcciones virtuales a través de las tablas del nuevo proceso. Es privilegiado porque si cualquier proceso de user space pudiera escribirlo, podría apuntar a una tabla de páginas maliciosa y obtener acceso a toda la memoria física de la máquina.

---

**35.** Aproximadamente **0 o 1** syscalls de las 200. La libc gestiona un pool de memoria interno: la primera llamada a `malloc` pide un bloque grande al kernel (una sola syscall `brk` o `mmap`) y lo subdivide internamente para satisfacer peticiones pequeñas sin syscall. 200 × 128 bytes = 25.600 bytes, un bloque pequeño que el pool inicial del heap cubre con creces. Solo cuando el pool se agota completamente la libc hace otra syscall.

---

**36.** Tres ejemplos:

- **`hlt`** (detiene la CPU hasta la próxima interrupción): desde ring 3 pararía todo el sistema, impidiendo que el planificador ejecute cualquier otro proceso.
- **`mov CR3, rax`** (cambia la tabla de páginas activa): permitiría que cualquier proceso apuntara a una tabla de páginas maliciosa con acceso a toda la memoria física.
- **`in`/`out`** (lee/escribe puertos de E/S): permitiría acceso directo a hardware (controladores de disco, tarjetas de red, etc.) sin pasar por el kernel, pudiendo leer datos privados o corromper dispositivos.

---

**37.** Sin KPTI, una syscall cuesta entre **~100 y ~300 ciclos** de overhead (cambio de ring, guardar/restaurar registros). Con KPTI activo, hay que añadir el cambio doble de tabla de páginas (entrada a kernel y salida a user space), que invalida el TLB cada vez. Repoblar el TLB puede costar entre **500 y 3.000 ciclos adicionales** dependiendo del tamaño del working set. La degradación total en cargas con alta frecuencia de syscalls es de **5–30 %**.

---

**38.** En un unikernel, el equivalente de una syscall es una **llamada a función ordinaria** (`CALL`/`RET`). El overhead es prácticamente **0 ciclos extra**: no hay cambio de ring, no hay cambio de tabla de páginas, no hay flush de TLB. La llamada cuesta lo mismo que cualquier otra función en el programa, ~1 ciclo de instrucción `call`.

---

**39.** `mm_struct` es la estructura de Linux que describe el **espacio de memoria virtual completo de un proceso**. Contiene: (a) la lista de VMAs (Virtual Memory Areas) — qué rangos de direcciones están mapeados y con qué permisos (lectura, escritura, ejecución); (b) el puntero `pgd` a la tabla de páginas de nivel 4 (se carga en `CR3` cuando el proceso se planifica); (c) los límites del heap (`brk`), la dirección base del stack, la dirección de los segmentos de código y datos. El scheduler escribe `CR3` con el valor de `mm->pgd` en cada cambio de proceso.

---

**40.** El aislamiento lo proporciona el **hipervisor** mediante las **tablas de páginas extendidas** (EPT en Intel / NPT en AMD). Cada instancia del unikernel corre en una VM separada. El hipervisor mantiene una tabla de mapeo adicional (EPT/NPT) que traduce las direcciones físicas "vistas" por la VM a las direcciones físicas reales del host. Estas tablas garantizan que la memoria física asignada a la VM A no puede ser leída ni escrita desde la VM B, ni siquiera desde el hypervisor de la VM B. El hardware (IOMMU + VMX/SVM) hace que este aislamiento sea imposible de bypassear sin comprometer el propio hipervisor.

---

## Conexión con la semana 2

| Concepto esta semana | Lo que verás en días 06-10 |
|---|---|
| Syscall como cambio de ring | Disección completa de `syscall`/`sysret` en x86-64, tabla de syscalls del kernel |
| Rings de CPU mencionados | Ring -1 (VMX root mode), instrucción `vmlaunch`, cómo KVM ejecuta un kernel invitado |
| Frontera user/kernel | Cómo el hipervisor intercepta instrucciones privilegiadas del kernel invitado (VM exits) |
| Aislamiento por hipervisor | KVM + QEMU: la pila de virtualización en Linux que usa la mayoría de clouds |

---

*← [Día 04 — User vs kernel space](leccion-04-user-vs-kernel-space.md) · [Día 06 — Syscalls](leccion-06-syscalls.md) →*
