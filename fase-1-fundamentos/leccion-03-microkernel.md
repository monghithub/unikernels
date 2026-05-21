# Día 03 — Microkernel: Mach, QNX y seL4

> **Fase:** 1 · **Semana:** 1 · **Tipo:** Lectura · **Duración:** 30 min

---

## Contexto y motivación

En la lección anterior viste que el kernel monolítico mete todo dentro del mismo espacio privilegiado: drivers, sistema de ficheros, red, gestión de memoria, planificador... Todo corre en ring 0. Eso tiene una consecuencia directa: un bug en el driver de una tarjeta de sonido puede tirar el sistema entero. Pregunta sencilla: **¿es necesario que el driver de audio tenga acceso irrestricto al hardware más crítico del sistema?**

La respuesta obvia es no. Y de esa respuesta nació el microkernel.

La idea es radical en su simplicidad: **mete en el kernel solo lo absolutamente imprescindible y saca el resto a espacio de usuario**. ¿Cuánto es lo mínimo? Muy poco: comunicación entre procesos (IPC), asignación básica de memoria física y un planificador. Todo lo demás —drivers, sistema de ficheros, pila de red, servicios de seguridad— vive fuera del kernel como procesos normales llamados *servidores*.

Este enfoque no es nuevo. Tiene décadas de historia, ha llegado a los bolsillos de millones de personas a través de macOS e iOS, aterriza en sistemas de misión crítica que no pueden fallar jamás, y tiene la distinción de ser el único diseño de kernel verificado matemáticamente de arriba abajo.

---

## Concepto principal

Un **microkernel** es un kernel reducido a su mínima expresión funcional. Solo ejecuta en modo privilegiado (ring 0) lo que no puede ejecutarse en otro sitio:

1. **IPC (Inter-Process Communication)**: el mecanismo por el que los servidores se hablan entre sí y con las aplicaciones. Es el corazón del diseño.
2. **Gestión básica de memoria**: mapeo de páginas físicas, espacios de direcciones. Sin políticas complejas; esas las implementan servidores.
3. **Planificador**: qué hilo se ejecuta en qué momento. Igual que la memoria, el schedulerpuede ser mínimo y extensible desde fuera.

Todo lo demás —el driver de disco, el sistema de ficheros ext4, la pila TCP/IP, el driver gráfico— es un proceso de usuario con privilegios restringidos. Si el servidor de ficheros tiene un bug y cae, el resto del sistema puede continuar funcionando. Puedes incluso reiniciar ese servidor en caliente.

El precio: para hacer algo tan simple como leer un fichero, el kernel monolítico ejecuta una llamada al sistema directa. El microkernel necesita varios saltos IPC: la app le habla al servidor de ficheros, que le habla al servidor de disco, que le habla al driver, que responde por el mismo camino. Cada salto tiene coste. Ese coste fue durante décadas el argumento definitivo en contra del microkernel.

---

## Desarrollo (los 30 minutos)

### Parte 1 — Historia: de Mach al núcleo de tu iPhone (~10 min)

**Mach** nació en Carnegie Mellon University a mediados de los 80, diseñado por Richard Rashid y Avie Tevanian. El objetivo era crear un microkernel portable, con un modelo de IPC basado en *puertos* y *mensajes*, y soporte nativo para multiprocesador en una época en que eso era ciencia ficción.

Mach no llegó a triunfar como microkernel puro en producción, pero sí como base de algo mucho más duradero. NeXT Inc., la empresa que Steve Jobs fundó tras dejar Apple en 1985, eligió Mach 2.5 como base de su sistema operativo NeXTSTEP. Cuando Apple compró NeXT en 1997, NeXTSTEP se convirtió en el cimiento de Mac OS X.

Hoy, el kernel de macOS e iOS se llama **XNU** (X is Not Unix). XNU es un híbrido: usa Mach como núcleo para IPC, gestión de memoria y threads, pero le añade una capa BSD encima para proporcionar la interfaz POSIX. No es un microkernel puro —demasiado código corre en espacio de kernel—, pero Mach es la razón por la que macOS tiene una gestión de memoria virtual tan robusta y por la que herramientas como `mach_msg` existen en el sistema.

Cada vez que abres una app en tu Mac, Mach está creando el espacio de direcciones. Cada vez que dos procesos se comunican a través de servicios del sistema en iOS, usan puertos Mach. El microkernel está en el bolsillo de mil millones de personas, aunque nadie lo sepa.

---

### Parte 2 — El debate de 1992 y dos implementaciones reales: QNX y seL4 (~15 min)

**El debate Torvalds-Tanenbaum (1992)**

El 29 de enero de 1992, Andrew Tanenbaum publicó en comp.os.minix un mensaje que empezaba así:

> *"Just as a comparison, minix is a microkernel-based design... Linux is already obsolete."*

Tanenbaum argumentaba que Linux, con su diseño monolítico, era un anacronismo. El futuro eran los microkernels. Linus Torvalds respondió con su famosa réplica directa:

> *"Your job is being a professor and researcher: That's great, because it means you can be impractical. [...] MINIX is a monolithic kernel... just in user space."*

La discusión técnica duró días en la lista y concentra todos los argumentos de ambos lados en forma cruda y sin filtros. Torvalds defendía que el overhead del IPC hacía a los microkernels imprácticamente lentos en hardware real. Tanenbaum respondía que el hardware mejoraría y que la fiabilidad importaba más que los ciclos de CPU.

Treinta años después, ambos tienen razón en algo: Linux domina el mundo en servidores y dispositivos, pero los sistemas donde nadie puede permitirse un fallo de software —aviones, coches, marcapasos— usan microkernels.

---

**QNX: el microkernel que vuela y conduce**

**QNX** (originalmente Quick Unix, desarrollado por QNX Software Systems, adquirido por BlackBerry en 2010) es el ejemplo más exitoso de microkernel en producción de misión crítica.

Su kernel, llamado *Neutrino*, implementa solo IPC, scheduling y gestión de memoria. Todo lo demás —drivers, sistema de ficheros, pila de red— son procesos de usuario. Si el driver de red falla, no se lleva al sistema. Puedes actualizarlo en caliente. Puedes matar y reiniciar el servidor de ficheros sin reiniciar el sistema.

Dónde lo encuentras:
- **Automoción**: es el sistema operativo que corre en los sistemas de infotainment de BMW, Audi, Ford, Porsche y muchos otros. La UE exige que el software de navegación y entretenimiento no pueda interferir con el control del vehículo; QNX lo garantiza por diseño.
- **Aviación**: sistemas de aviónica de cabina, incluyendo algunos sistemas del Boeing 737 y Airbus A380.
- **Médico**: marcapasos, equipos de diagnóstico por imagen, sistemas de radioterapia.
- **Industria**: PLCs, controladores de reactores nucleares.

El modelo de QNX es también notable por su compatibilidad POSIX. Puedes portar software Unix a QNX con mínimas modificaciones porque la capa de compatibilidad POSIX vive como servidor de usuario, no dentro del kernel.

---

**seL4: el primer kernel verificado formalmente**

En 2009, investigadores de NICTA (ahora Data61, Australia) publicaron en el simposio SOSP algo que muchos consideraban imposible: habían verificado formalmente, en el asistente de pruebas **Isabelle/HOL**, que la implementación en C del microkernel **seL4** es correcta con respecto a su especificación funcional abstracta.

Qué significa eso en concreto:
- Tomaron el código C de seL4 (unas 8.700 líneas).
- Escribieron una especificación formal abstracta de lo que el kernel *debe* hacer.
- Escribieron una especificación ejecutable como nivel intermedio.
- Probaron matemáticamente, sin huecos, que el código C refina la especificación abstracta.
- La prueba tiene aproximadamente **200.000 líneas** en Isabelle/HOL.

Las propiedades verificadas incluyen ausencia de deadlocks, ausencia de ejecución de memoria no inicializada, integridad de los derechos de acceso (ningún proceso puede obtener acceso a un recurso sin que el sistema de capacidades se lo haya concedido explícitamente) y confidencialidad (no hay canales de filtrado de información no autorizados).

seL4 no verifica la ausencia de bugs en el hardware, ni en el compilador GCC, ni en el bootloader. Pero el kernel en sí mismo es, dentro de sus supuestos, matemáticamente correcto.

Aplicaciones reales:
- Drones militares de la DARPA (proyecto HACMS).
- Vehículos autónomos (como base para separar dominios de seguridad).
- La Fundación seL4 mantiene hoy el proyecto como open source.

El modelo de capacidades de seL4 también fue influyente en diseños posteriores: los *capabilities* como primitive de seguridad aparecen en sistemas como Barrelfish, Redox OS y varios unikernels modernos.

---

### Parte 3 — Microkernel vs monolítico y el paso siguiente: Library OS (~5 min)

**Tabla comparativa**

| Dimensión | Kernel monolítico | Microkernel |
|---|---|---|
| **Rendimiento IPC** | Alta (llamadas directas en kernel) | Menor (saltos de contexto IPC) |
| **Tamaño del TCB** | Grande (millones de líneas en ring 0) | Pequeño (miles de líneas en ring 0) |
| **Superficie de ataque** | Alta (todo en ring 0 es crítico) | Baja (solo el microkernel en ring 0) |
| **Fiabilidad** | Un bug en un driver puede tumbar el sistema | Los servidores de usuario pueden fallar y reiniciarse |
| **Verificabilidad formal** | Prácticamente imposible a escala | Factible (seL4 lo demostró) |
| **Portabilidad de drivers** | Los drivers se integran en el árbol del kernel | Los drivers son procesos portables |
| **Rendimiento general** | Excelente (Linux, histórico ganador) | Bueno en microkernels modernos (L4, seL4) |
| **Casos de uso típicos** | Servidores, escritorio, móvil de consumo | Sistemas críticos, aviónica, automoción, seguridad |

La brecha de rendimiento que Torvalds señalaba en 1992 se ha reducido drásticamente. Los microkernels de tercera generación (familia L4, que incluye seL4) lograron tiempos de IPC del orden de nanosegundos. El argumento del rendimiento ya no es tan definitivo.

**El paso siguiente: Library OS**

El microkernel lleva el razonamiento de minimización al límite de lo que tiene sentido a nivel de sistema operativo compartido. Pero hay una pregunta más radical que todavía no hemos hecho:

Si el objetivo es aislar aplicaciones y darles solo lo que necesitan, **¿por qué hay que compartir el OS en absoluto?**

Un microkernel sigue siendo un sistema operativo compartido entre todas las aplicaciones que corren encima. Todas comparten el mismo servidor de ficheros, la misma pila de red, el mismo driver de disco. Si una aplicación necesita una versión diferente del sistema de ficheros, o una pila TCP/IP con comportamiento distinto, no puede tenerla sin afectar al resto.

La respuesta a esta pregunta es el **Library OS**: en lugar de un OS compartido, cada aplicación lleva su propio sistema operativo enlazado como una biblioteca. El kernel solo hace una cosa: multiplexar el hardware de forma segura. Todo lo demás —FS, red, drivers— es código de la aplicación.

Y del Library OS al unikernel hay un paso corto: compilar aplicación y Library OS juntos en un único binario que arranca directamente sobre un hypervisor o sobre metal desnudo.

Eso es exactamente lo que verás en las próximas lecciones.

---

## Puntos clave

- Un microkernel ejecuta en ring 0 solo IPC, gestión básica de memoria y planificador. Todo lo demás son servidores en espacio de usuario.
- **Mach** es la base de XNU, el kernel de macOS e iOS. Cada iPhone lleva un microkernel.
- **QNX Neutrino** es el estándar de facto en sistemas de misión crítica: automoción, aviónica, médico.
- **seL4** es el primer y único kernel con verificación formal completa de su implementación en C, probada en Isabelle/HOL.
- El debate Torvalds-Tanenbaum de 1992 resume todos los argumentos técnicos de ambos enfoques.
- La brecha de rendimiento se ha cerrado mucho con microkernels de tercera generación (familia L4).
- El microkernel lleva la minimización al OS compartido. La pregunta siguiente —¿y si cada app lleva su propio OS?— conduce al Library OS y al unikernel.

---

## Preguntas de autoevaluación

1. ¿Cuáles son los tres componentes que un microkernel mantiene en ring 0 y por qué solo esos tres?

2. Si el servidor de sistema de ficheros de un sistema basado en microkernel tiene un bug de memoria y cae, ¿qué ocurre con el resto del sistema? ¿Y en un kernel monolítico?

3. Tanenbaum argumentaba en 1992 que los microkernels eran el futuro. ¿Qué parte de su predicción acertó y qué parte no?

4. ¿Qué significa exactamente que seL4 esté "verificado formalmente"? ¿Qué garantiza y qué no garantiza?

5. QNX se usa en sistemas críticos donde Linux domina en servidores. ¿Qué propiedad arquitectónica de QNX lo hace preferible en ese dominio?

6. ¿Cuál es la diferencia fundamental entre un microkernel y un Library OS en términos de qué comparten las aplicaciones?

---

## Respuestas de autoevaluación

**Pregunta 1.** Los tres componentes son **IPC**, **gestión básica de memoria** y **planificador**. Solo esos tres porque:

- **IPC**: es el mecanismo por el que todos los servidores del sistema se hablan entre sí y con las aplicaciones. Si el IPC estuviera en user space habría una dependencia circular: para invocar al servidor de IPC necesitarías IPC previo. Debe ser la capa más primitiva, siempre disponible.
- **Gestión básica de memoria**: mapear páginas físicas a espacios de direcciones virtuales requiere escribir en `CR3` y configurar la MMU, operaciones que solo pueden hacerse en ring 0.
- **Planificador**: responde a interrupciones de timer (hardware privilegiado) y ejecuta cambios de contexto que implican guardar/restaurar registros privilegiados y cambiar `CR3`. Debe estar en ring 0.

Todo lo demás (drivers, FS, red, seguridad) puede vivir en user space y comunicarse mediante el IPC que proporciona el microkernel.

---

**Pregunta 2.** **Microkernel**: el servidor FS es un proceso de user space con su propio espacio de memoria. Al morir (segfault, corrupción de memoria…), el kernel lo detecta como proceso muerto. El resto del sistema —kernel, drivers de red, otras aplicaciones— continúa funcionando porque no comparten espacio de memoria con ese servidor. Los procesos que intentaban operaciones de FS recibirán un error de IPC, pero el sistema no cae. El servidor puede reiniciarse.

**Kernel monolítico**: el código del FS corre en ring 0 compartiendo el mismo espacio de memoria que el scheduler, el MM y todos los drivers. Una corrupción de memoria puede sobrescribir estructuras del kernel. El resultado más probable es un kernel panic inmediato y el sistema se cae por completo, perdiendo todos los procesos en ejecución.

---

**Pregunta 3.** **Acertó en**: los sistemas de misión crítica (aviónica, automoción, médico, industrial) son dominados por microkernels: QNX, VxWorks, PikeOS. La verificación formal demostró ser posible y útil en producción (seL4 en drones DARPA, vehículos autónomos). Los microkernels de tercera generación (familia L4) cerraron gran parte de la brecha de rendimiento que Torvalds señalaba.

**Falló en**: Linux no se volvió obsoleto. Los servidores, el escritorio y Android dominan el mercado de propósito general. El rendimiento fue el factor determinante en ese mercado y los microkernels tardaron décadas en ser competitivos. Treinta años después, Linux sigue ganando en benchmarks de servidores.

---

**Pregunta 4.** Verificación formal significa que se construyó una **prueba matemática completa**, sin huecos, de que el código C de seL4 (~8.700 líneas) es una refinación correcta de su especificación abstracta. La herramienta usada fue el asistente de pruebas **Isabelle/HOL**. La prueba tiene ~200.000 líneas y cubre todos los estados posibles del sistema, sin casos no probados.

**Garantiza**: ausencia de deadlocks, ausencia de acceso a memoria no inicializada, que el sistema de capacidades (capability system) no puede ser bypasseado (ningún proceso obtiene acceso a un recurso sin concesión explícita), y que el comportamiento del código C es exactamente el descrito en la especificación.

**No garantiza**: bugs en el hardware, en el compilador GCC, en el bootloader, en los drivers de sistema (que corren fuera del microkernel en user space), ni errores en la propia especificación abstracta (si la especificación describe el comportamiento incorrecto, la implementación lo implementa correctamente).

---

**Pregunta 5.** **Aislamiento de fallos garantizado por hardware**. En QNX Neutrino, cada driver es un proceso de usuario con su propio espacio de memoria y privilegios mínimos. Si el driver de infotainment falla o es comprometido, la MMU impide físicamente que acceda a la memoria del driver de control del motor. La separación entre dominios de seguridad (entretenimiento vs. control crítico del vehículo) es una propiedad del hardware, no una convención de software que alguien podría romper con un bug. En Linux monolítico, un bug en el subsistema de USB o audio corre en ring 0 y puede corromper la memoria de cualquier otro subsistema incluyendo los críticos.

---

**Pregunta 6.** **Microkernel**: todas las aplicaciones comparten los mismos servidores de OS. El servidor de ficheros, la pila TCP/IP y el driver de disco son procesos únicos que todas las aplicaciones del sistema utilizan. Los servidores están en user space pero son compartidos.

**Library OS**: cada aplicación lleva su propio OS como biblioteca enlazada estáticamente. No hay ningún código de OS compartido entre aplicaciones: cada instancia tiene su propia pila TCP/IP, su propio VFS, sus propios drivers. Los servicios del OS son completamente privados a cada aplicación y las versiones pueden diferir entre instancias. El kernel subyacente solo hace multiplexado mínimo de hardware.

---

## Referencias

- **Accesible online**: Debate original Torvalds-Tanenbaum (29 ene 1992) — archivo en `groups.google.com/g/comp.os.minix`
- Klein, G. et al. *seL4: Formal Verification of an OS Kernel*. SOSP 2009. [doi:10.1145/1629575.1629596](https://doi.org/10.1145/1629575.1629596)
- Liedtke, J. *On µ-Kernel Construction*. SOSP 1995. — El paper fundacional de los microkernels de tercera generación (L4).
- Rashid, R. & Robertson, G. *Accent: A Communication Oriented Network Operating System Kernel*. SOSP 1981. — El precursor directo de Mach.
- Heiser, G. & Elphinstone, K. *L4 Microkernels: The Lessons from 20 Years of Research and Deployment*. ACM TOCS, 2016.
- Documentación técnica de QNX Neutrino: [www.qnx.com/developers/docs/](https://www.qnx.com/developers/docs/)
- Sitio oficial de seL4: [sel4.systems](https://sel4.systems/)
- Documentación de XNU (kernel de macOS/iOS): [github.com/apple-oss-distributions/xnu](https://github.com/apple-oss-distributions/xnu)

---

## Notas personales

> Añade aquí tus notas, conexiones con experiencia previa y dudas al estudiar esta lección.

---
---

## Lectura complementaria

> **[📐 Anexo 03 — Arquitecturas de OS: del Kernel Monolítico al Unikernel](anexo-03-arquitecturas-os.md)**  
> Tres niveles de profundidad (básico · intermedio · avanzado), diagramas C4 y Mermaid de cada arquitectura, flujos de syscall e IPC comparados, y glosario completo de términos técnicos.

---

*← [Día 02 — Kernel monolítico](leccion-02-kernel-monolitico.md) · [Día 04 — User vs kernel space](leccion-04-user-vs-kernel-space.md) →*
