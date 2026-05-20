# Día 04 — Espacio de usuario vs espacio de kernel

> **Fase:** 1 · **Semana:** 1 · **Tipo:** Lectura · **Duración:** 30 min

---

## Contexto y motivación

En los días anteriores estudiaste las arquitecturas de kernel desde un punto de vista organizativo: qué código vive en el kernel, qué vive fuera, cómo se comunican los componentes. Hoy bajas un nivel más: la CPU misma impone esa separación mediante mecanismos de hardware. No es una convención de software — es una restricción física grabada en silicio.

Entender esto es imprescindible para comprender los unikernels, porque la propuesta más radical de esa tecnología es exactamente esta: **eliminar esa separación**. Para apreciar qué se gana y qué se pierde, primero debes saber qué es lo que se elimina y por qué existe.

---

## Concepto principal

La CPU x86-64 opera en distintos **niveles de privilegio** llamados *rings*. El código de kernel corre en ring 0 (el más privilegiado); el código de aplicación corre en ring 3 (el menos privilegiado). El hardware se encarga de que el código en ring 3 no pueda ejecutar instrucciones peligrosas ni acceder a páginas de memoria que no le pertenecen.

Esto crea dos "espacios" lógicos bien diferenciados:

- **Kernel space:** memoria y CPU accesibles solo desde ring 0.
- **User space:** memoria accesible desde ring 3, con instrucciones y acceso a recursos limitados.

La frontera entre ambos se cruza de forma controlada a través de las *system calls*.

---

## Desarrollo (los 30 minutos)

### Parte 1 — El mapa de memoria virtual en x86-64 (~10 min)

En una arquitectura de 64 bits, los punteros tienen 64 bits, pero los procesadores actuales solo implementan 48 bits de espacio de direcciones virtual (extensible a 57 bits con la extensión LA57). Esto divide el espacio en dos mitades canónicas:

```
 Espacio de direcciones virtuales en x86-64 (48 bits canónicos)
 ────────────────────────────────────────────────────────────────

 0xFFFFFFFFFFFFFFFF ┐
        ...         │  Kernel space
 0xFFFF800000000000 ┘  (mitad superior — dirección canónica alta)

         ▲
         │  "agujero canónico" — direcciones no válidas
         │  (0x0000800000000000 – 0xFFFF7FFFFFFFFFFF)
         ▼

 0x00007FFFFFFFFFFF ┐
        ...         │  User space
 0x0000000000000000 ┘  (mitad inferior — dirección canónica baja)

 ────────────────────────────────────────────────────────────────
```

**Por qué existe el agujero canónico:** los bits 48–63 deben ser todos iguales al bit 47. Si el bit 47 es 0, los bits superiores deben ser 0 (user space). Si el bit 47 es 1, los bits superiores deben ser todos 1 (kernel space). Cualquier dirección que viole esto es no canónica y el procesador lanza una excepción de protección general (#GP).

**Desglose típico de user space (Linux, x86-64):**

```
 0x00007FFFFFFFFFFF  ── tope del espacio de usuario
        stack          (crece hacia abajo)
        ...
        mmap / libs    (bibliotecas compartidas, archivos mapeados)
        ...
        heap           (crece hacia arriba desde el BSS)
        BSS            (datos no inicializados)
        data           (datos inicializados)
        text           (código ejecutable)
 0x0000000000400000  ── dirección base típica del ejecutable (sin PIE)
 0x0000000000000000  ── página nula (nunca mapeada, para detectar NULL)
```

**Desglose típico de kernel space (Linux, x86-64):**

```
 0xFFFFFFFFFFFFFFFF  ── tope absoluto
        módulos del kernel
        vmalloc / ioremap   (memoria virtual del kernel)
        memoria física mapeada directamente
        texto del kernel / datos / stack por CPU
 0xFFFF800000000000  ── inicio del kernel space
```

El kernel mapea su propio espacio en todas las tablas de páginas de todos los procesos — pero con las páginas marcadas como supervisor-only. Un proceso en ring 3 que intente leer esas páginas recibe un *page fault* con acceso denegado, no los datos.

> **Nota sobre KPTI (Kernel Page Table Isolation):** tras las vulnerabilidades Meltdown (2018), Linux separa casi completamente las tablas de páginas de usuario y kernel. Cuando el proceso corre en user space solo tiene mapeado un fragmento mínimo del kernel (el necesario para gestionar la entrada a ring 0). Esto añade overhead al cruzar la frontera, pero elimina el canal lateral de Meltdown.

---

### Parte 2 — Instrucciones privilegiadas y el mecanismo de protección (~15 min)

#### Instrucciones que solo puede ejecutar ring 0

El manual de Intel clasifica ciertas instrucciones como "privilegiadas" o "sensibles". Si ring 3 intenta ejecutarlas, la CPU lanza una excepción de protección general (#GP, vector 13) que el kernel captura y generalmente convierte en `SIGSEGV` o `SIGILL` para el proceso:

| Instrucción | Qué hace | Por qué es peligrosa en ring 3 |
|-------------|----------|-------------------------------|
| `in` / `out` | Lee/escribe puertos de E/S | Acceso directo a hardware (disco, red, etc.) |
| `hlt` | Detiene la CPU hasta la próxima interrupción | Pararía todo el sistema |
| `lidt` / `lgdt` | Carga la tabla de interrupciones/descriptores | Podría redirigir todas las interrupciones |
| `mov cr0` / `cr3` / `cr4` | Modifica registros de control | `cr3` cambia la tabla de páginas activa — control total de memoria |
| `wrmsr` / `rdmsr` | Escribe/lee MSRs (model-specific registers) | Configura comportamiento de la CPU a bajo nivel |
| `invlpg` | Invalida una entrada del TLB | Podría exponer datos de otros procesos |
| `clts` | Limpia el flag TS en CR0 | Relacionado con el estado de la FPU entre procesos |
| `sti` / `cli` | Habilita/deshabilita interrupciones | Podría bloquear el planificador indefinidamente |

#### Cómo funciona el cambio de contexto user → kernel (syscall)

Cuando una aplicación necesita algo que solo el kernel puede hacer, ejecuta una instrucción especial: `syscall` (la forma moderna en x86-64, que sustituyó a `int 0x80`).

El flujo es:

```
  Aplicación (ring 3)
      │
      │  1. Pone el número de syscall en RAX
      │     (ej: 9 = mmap, 12 = brk, 1 = write)
      │  2. Argumentos en RDI, RSI, RDX, R10, R8, R9
      │  3. Ejecuta la instrucción SYSCALL
      │
      ▼  ──── hardware cruza a ring 0 ────────────────────────────
      │
      │  4. CPU guarda RIP y RFLAGS en RCX y R11
      │  5. CPU carga RIP desde MSR LSTAR (entry point del kernel)
      │  6. CPU cambia CS a kernel code segment (ring 0)
      │
      ▼
  Kernel (ring 0)
      │
      │  7. El kernel guarda el estado del proceso
      │     (registros → struct pt_regs en la pila del kernel)
      │  8. Despacha al handler correcto según RAX
      │  9. Ejecuta la operación (con acceso total al hardware)
      │ 10. Pone el resultado en RAX
      │ 11. Ejecuta SYSRET
      │
      ▼  ──── hardware vuelve a ring 3 ───────────────────────────
      │
      │ 12. CPU restaura RIP desde RCX, RFLAGS desde R11
      │ 13. CPU cambia CS a user code segment (ring 3)
      │
      ▼
  Aplicación (ring 3)
      │
      │ 14. Lee el resultado desde RAX
      ▼
```

Este viaje de ida y vuelta cuesta entre **~100 y ~1000 ciclos de CPU** dependiendo de KPTI, hardware y estado del TLB. En aplicaciones que hacen miles de syscalls por segundo, este overhead es medible.

#### Qué pasa cuando tu aplicación llama a `malloc`

Este es un ejemplo concreto que desmitifica mucho:

```
  Tu código:   ptr = malloc(1024);
                     │
                     ▼
  libc (user space):
    ¿Hay espacio libre en el heap gestionado por la libc?
      ├── SÍ → la libc lo devuelve directamente
      │         (no hay syscall, todo en user space)
      │
      └── NO → necesita más memoria del kernel
                     │
                     ▼
            syscall brk() o mmap()
            (cruza a ring 0)
                     │
                     ▼
  Kernel (ring 0):
    Ajusta el límite del heap (brk) o crea un nuevo
    mapping anónimo (mmap). Actualiza las tablas de
    páginas del proceso. Devuelve la dirección.
                     │
                     ▼
  libc (user space):
    Recibe el bloque del kernel, lo gestiona
    internamente, devuelve un puntero a tu código.
                     │
                     ▼
  Tu código:   usa ptr normalmente
```

La clave: **la mayoría de los `malloc` no hacen syscall**. La libc gestiona un pool de memoria propio y solo pide más al kernel cuando se agota. Esto es eficiente precisamente porque cruzar la frontera tiene coste.

#### El contexto del proceso: dónde vive la información del espacio de memoria

El kernel mantiene para cada proceso una estructura `task_struct` (en Linux) que contiene, entre otras cosas, un puntero a `mm_struct` — la descripción completa del espacio de memoria virtual del proceso:

- Las VMAs (*Virtual Memory Areas*): qué rangos de direcciones están mapeados y con qué permisos.
- El puntero a la tabla de páginas de nivel 4 (PGD), que se carga en `CR3` cuando el proceso se planifica.
- El límite del heap (`brk`), la dirección de inicio del stack, etc.

Cuando el planificador cambia de proceso, lo fundamental que cambia es el valor de `CR3` — instrucción privilegiada de ring 0. Esto hace que toda la MMU (Memory Management Unit) empiece a usar las tablas de páginas del nuevo proceso. En nanosegundos, la vista de la memoria virtual cambia completamente.

---

### Parte 3 — La clave para unikernels: cuando la separación desaparece (~5 min)

Todo lo que acabas de aprender — rings de CPU, tablas de páginas separadas, syscalls como frontera controlada — es la arquitectura de seguridad fundamental de cualquier OS de propósito general.

**En un unikernel, todo esto desaparece.**

Un unikernel compila la aplicación y el código del kernel (drivers, protocolos de red, sistema de archivos) en **un único binario**. Ese binario corre en un único nivel de privilegio. No hay separación de espacios de direcciones. No hay syscalls — las llamadas a servicios del "kernel" son simples llamadas a función (`call`, no `syscall`). Todo comparte el mismo espacio de memoria virtual.

```
  OS convencional                   Unikernel
  ─────────────────────────         ─────────────────────────
  0xFFFF... Kernel space            Un único espacio plano
             ring 0                 ┌──────────────────────┐
  ─── frontera HW ──────            │  Stack               │
  0x0000... User space              │  Heap                │
             ring 3                 │  Código de la app    │
                                    │  Drivers / libs      │
  Cruzar = ~100-1000 ciclos         │  Stack del "kernel"  │
                                    └──────────────────────┘
                                    ring 0 (o ring 3 con VMM)
                                    Cruzar = 0 ciclos
```

**La fuerza:** eliminar el overhead de las syscalls y los cambios de tablas de páginas es un ahorro real en latencia y throughput, especialmente en aplicaciones de red. Los benchmarks de Mirage OS (unikernel OCaml) o Nanos muestran latencias de red considerablemente menores que Linux para las mismas cargas de trabajo.

**El riesgo:** si hay un bug en cualquier parte del código — ya sea en la aplicación o en el driver de red — el error tiene acceso completo a toda la memoria. No hay ring 3 que contenga el daño. Un desbordamiento de buffer en el parser HTTP tiene los mismos privilegios que el código que gestiona la memoria física. El aislamiento que tanto costó diseñar en hardware simplemente no está ahí.

Esta es la tensión fundamental de los unikernels: **compran rendimiento a cambio de aislamiento interno**. El aislamiento no desaparece del todo — lo proporciona el hipervisor (la VM sigue estando aislada de otras VMs) — pero dentro del unikernel no existe. Esa decisión de diseño lo impregna todo.

---

## Puntos clave

1. **La separación user/kernel es hardware, no software.** Los rings de CPU (ring 0 / ring 3) y las tablas de páginas son mecanismos del procesador. No hay forma de saltárselos desde user space sin que el hardware intervenga.

2. **El mapa de memoria virtual en x86-64** divide el espacio canónico en dos mitades: `0x0000...` a `0x00007FFF...` para user space, `0xFFFF8000...` a `0xFFFF...` para kernel space. El agujero canónico en el centro es físicamente no direccionable.

3. **Instrucciones como `in`/`out`, `hlt`, o escrituras a `CR3`** solo pueden ejecutarse en ring 0. Un intento desde ring 3 genera una excepción que el kernel convierte en señal de error para el proceso.

4. **`malloc` habitualmente no hace syscall.** La libc gestiona un pool propio. Solo cruza a ring 0 (via `brk` o `mmap`) cuando necesita más memoria del kernel. Ese cruce cuesta entre 100 y 1000 ciclos.

5. **El cambio de proceso cambia `CR3`.** Toda la vista de memoria virtual se reconstruye en nanosegundos cargando la nueva tabla de páginas en ese registro privilegiado.

6. **Los unikernels eliminan esta separación.** App y "kernel" comparten espacio de direcciones y nivel de privilegio. Las llamadas a servicios del kernel son `call`, no `syscall`. Esto elimina overhead pero también elimina aislamiento interno. El hipervisor pasa a ser la única barrera de contención.

---

## Preguntas de autoevaluación

1. ¿Cuál es el rango de direcciones virtuales reservado para user space en x86-64 con 48 bits? ¿Por qué existe un "agujero canónico"?

2. ¿Qué instrucción ejecuta una aplicación Linux en x86-64 para realizar una syscall? ¿Qué sucede en la CPU en el momento exacto de ejecutarla?

3. Un proceso llama a `malloc(64)` cien veces seguidas. ¿Cuántas de esas llamadas probablemente cruzarán al kernel? ¿Por qué?

4. ¿Qué registro de la CPU se carga al hacer un cambio de contexto de proceso? ¿Por qué ese registro es una instrucción privilegiada?

5. ¿Qué diferencia concreta hay entre un proceso Linux haciendo `write(fd, buf, n)` y un unikernel Nanos haciendo la misma operación? ¿Dónde está el ahorro de ciclos?

6. Si un unikernel no tiene separación user/kernel, ¿qué proporciona el aislamiento entre distintas instancias del mismo unikernel corriendo en el mismo servidor físico?

7. ¿Por qué KPTI (Kernel Page Table Isolation) incrementa el coste de las syscalls? ¿Qué vulnerabilidad intentaba mitigar?

---

## Referencias

- **Intel 64 and IA-32 Architectures Software Developer's Manual, Vol. 3A**, capítulos 2 (System Architecture Overview) y 5 (Protection). Fuente primaria sobre rings, descriptores y protección de memoria. Disponible en intel.com.

- **AMD64 Architecture Programmer's Manual, Vol. 2**, capítulo 4 (Paging). Complementa a Intel con detalles sobre la implementación AMD del espacio canónico.

- **Bovet, D. P. & Cesati, M. — *Understanding the Linux Kernel*, 3ª ed.** (O'Reilly, 2005). Capítulos 2 (Memory Addressing) y 10 (System Calls). Aunque antiguo, el modelo de memoria de Linux no ha cambiado en lo fundamental.

- **Love, R. — *Linux Kernel Development*, 3ª ed.** (Addison-Wesley, 2010). Capítulo 12 (Memory Management). Más accesible que Bovet/Cesati.

- **Meltdown paper:** Lipp et al., *Meltdown: Reading Kernel Memory from User Space*, USENIX Security 2018. Explica por qué el kernel space estaba mapeado en user space y por qué eso fue un problema.

- **Linux kernel source:** `arch/x86/entry/entry_64.S` — el entry point real de las syscalls en x86-64. `include/linux/mm_types.h` — definición de `mm_struct` y `vm_area_struct`.

- **Elphinstone, K. & Heiser, G. — *From L3 to seL4: What Have We Learnt in 20 Years of L4 Microkernels?*** (SOSP 2013). Contextualiza el coste real de cruzar la frontera user/kernel en distintos diseños.

---

## Notas personales

> Añade aquí tus notas al estudiar esta lección.

---
*← [Día 03 — Microkernel](leccion-03-microkernel.md) · [Día 05 — Repaso semana 1](leccion-05-repaso-semana-1.md) →*
