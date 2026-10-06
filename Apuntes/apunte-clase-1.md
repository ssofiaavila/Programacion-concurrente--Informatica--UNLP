# Apunte — Teoría 1: Conceptos básicos de concurrencia

## La idea en una frase

Un programa concurrente son varios programas secuenciales (procesos) que se ejecutan intercalados y que necesitan comunicarse y sincronizarse. Casi todo lo difícil de la materia sale de que ese intercalado no está determinado de antemano.

La clase tiene seis partes:

1. Qué es la concurrencia y por qué hace falta.
2. Conceptos básicos: comunicación, sincronización, interferencia, deadlock.
3. Concurrencia a nivel de hardware.
4. Las instrucciones del lenguaje que se usa en la materia.
5. Acciones atómicas y la sentencia `await`.
6. Propiedades (seguridad y vida) y fairness.

## 1. Qué es la concurrencia

Concurrencia es la capacidad de ejecutar múltiples actividades en paralelo o simultáneamente. Está en todos lados: un navegador que carga una página mientras atiende al usuario, varios usuarios en un sistema de reservas, un teléfono que avisa una llamada mientras se habla.

### El ejemplo de los carteles

Mostrar un cartel ROJO cada 3 segundos es trivial en forma secuencial. Mostrar además uno AZUL cada 5 segundos, con un solo programa, obliga a llevar la cuenta de cuál toca primero:

```
Programa Carteles
    Proximo_Rojo = 3
    Proximo_Azul = 5
    Actual = 0
    Mientras (true)
        Si (Proximo_Rojo < Proximo_Azul)
            Demorar (Proximo_Rojo – Actual)
            Desplegar cartel ROJO
            Actual = Proximo_Rojo
            Proximo_Rojo = Proximo_Rojo + 3
        sino
            Demorar (Proximo_Azul – Actual)
            Desplegar cartel AZUL
            Actual = Proximo_Azul
            Proximo_Azul = Proximo_Azul + 5
    Fin mientras
Fin programa
```

El código queda complejo, impone un orden y empeora con cada cartel que se agrega. Lo natural es que cada cartel sea un proceso independiente:

```
Programa Cartel (color, tiempo)
    Mientras (true)
        Demorar (tiempo segundos)
        Desplegar cartel (color)
    Fin mientras
Fin programa
```

El costo es que ya no hay un orden preestablecido: aparece el **no determinismo**. Dos ejecuciones con la misma entrada pueden dar salidas distintas.

### Por qué es necesaria

- **Límite físico**: ya no hay más ciclos de reloj que ganar; la potencia viene de tener más núcleos (multicore).
- **Estructura más natural**: el mundo no es secuencial. Sirve para reaccionar a entradas asincrónicas, como sensores.
- **Mejor respuesta**: no se bloquea toda la aplicación por una E/S, y se aprovecha mejor el hardware.
- **Sistemas distribuidos**: una aplicación repartida en varias máquinas (cliente/servidor, P2P).

### Cómo se comportan los procesos

| Comportamiento | Qué pasa | Riesgo o necesidad |
| --- | --- | --- |
| Independientes | No interactúan | Raros y poco interesantes |
| Competencia | Comparten recursos (típico en SO y redes) | Deadlock e inanición |
| Cooperación | Se combinan para resolver una tarea común | Sincronización |

### Secuencial, concurrente y paralelo

El ejemplo es fabricar un objeto de N partes.

- **Secuencial**: una máquina, N pasos en orden estricto.
- **Paralelo**: N máquinas, cada una hace una parte al mismo tiempo. Menos tiempo, pero aparecen problemas: repartir la carga, compartir recursos, esperarse en puntos clave, comunicarse, tratar fallas.
- **Concurrente sin paralelismo**: una sola máquina dedica una parte del tiempo a cada componente. Tiene las mismas dificultades, más la de recuperar el estado de cada proceso al retomarlo. Es la multiprogramación: el SO reparte la CPU por *time slicing* y hace *context switch*.

La distinción que hay que saber:

- **Concurrencia**: concepto de software, no atado a una arquitectura ni a una cantidad de procesadores. Especificarla es especificar los procesos, su comunicación y su sincronización.
- **Paralelismo**: ejecución concurrente en múltiples procesadores, con el objetivo de reducir el tiempo de ejecución.

### Procesos e hilos

- **Proceso**: tiene su propio espacio de direcciones y sus recursos.
- **Hilo (proceso liviano)**: tiene su propio contador de programa y su pila, pero comparte el espacio de direcciones y los recursos del proceso. Por eso hay que evitar interferencias entre hilos.

## 2. Conceptos básicos

### Comunicación

| Mecanismo | Cómo funciona | Qué exige |
| --- | --- | --- |
| Memoria compartida | Los procesos intercambian información sobre datos residentes en memoria común | Bloquear y liberar el acceso, porque no pueden operar a la vez sobre ella |
| Pasaje de mensajes | Se transmite información por un canal lógico o físico | Un protocolo, y que los procesos sepan cuándo hay mensajes para leer o enviar |

### Sincronización

Sincronizar es tener información sobre otro proceso para coordinar actividades. Hay dos formas:

- **Exclusión mutua**: asegurar que solo un proceso a la vez acceda a un recurso compartido. Evita que dos procesos estén en la misma sección crítica al mismo tiempo.
- **Por condición**: bloquear a un proceso hasta que se cumpla una condición dada.

El buffer limitado con productores y consumidores usa las dos: exclusión mutua sobre el buffer, y condición para no depositar si está lleno ni retirar si está vacío.

### Interferencia

Un proceso toma una acción que invalida las suposiciones hechas por otro.

```
int x, y, z;

process A1
 { ...
   y = 0;
   ...
 }

process A2
 { ...
   if (y <> 0) z = x/y;
   ...
 }
```

A2 chequea que `y` no sea 0, pero A1 puede ponerla en 0 justo entre el chequeo y la división. El otro ejemplo de la clase son dos procesos que hacen `Público = Público + 1` cien veces cada uno: `Público` debería terminar en 200 y puede terminar en menos.

### Prioridad y granularidad

- **Prioridad**: un proceso con más prioridad puede causar la suspensión (*preemption*) de otro, o quitarle un recurso.
- **Granularidad**: relación entre cómputo y comunicación. Grano fino es poco cómputo entre comunicaciones; grano grueso, mucho.

### Manejo de recursos

Lo deseable es **fairness**: equilibrio en el acceso a recursos compartidos. Lo que hay que evitar:

- **Inanición**: un proceso nunca logra acceder al recurso.
- **Overloading**: la carga asignada excede la capacidad del proceso.
- **Deadlock**: dos o más procesos esperan cada uno que el otro libere un recurso.

### Las cuatro condiciones del deadlock

Son necesarias y suficientes. Se preguntan seguido.

| Condición | Significa |
| --- | --- |
| Recursos reusables serialmente | Los procesos comparten recursos que usan con exclusión mutua |
| Adquisición incremental | Un proceso mantiene los recursos que tiene mientras espera adquirir otros |
| No-preemption | Los recursos no se quitan por la fuerza; solo se liberan voluntariamente |
| Espera cíclica | Hay un ciclo de procesos donde cada uno tiene un recurso que espera su sucesor |

Si se rompe cualquiera de las cuatro, no hay deadlock.

### Problemas de la programación concurrente

- Más complejidad, por la exclusión mutua y la sincronización.
- Procesos que pueden no estar "vivos" (pérdida de *liveness*).
- No determinismo en el intercalado, que dificulta interpretar y depurar.
- Posible pérdida de rendimiento por overhead de context switch, comunicación y sincronización.
- Más tiempo de desarrollo; es difícil paralelizar algoritmos secuenciales.

## 3. Concurrencia a nivel de hardware

Los procesadores llegaron a un límite físico de velocidad, así que la potencia se busca con más procesadores por chip.

| Arquitectura | Cómo interactúan | Observaciones |
| --- | --- | --- |
| Multiprocesador de memoria compartida | Modificando datos en la memoria común | Esquemas UMA (bus o crossbar, SMP) y NUMA para más procesadores. Hay problemas de sincronización y consistencia |
| Multiprocesador de memoria distribuida | Solo por pasaje de mensajes, a través de una red | Cada uno tiene memoria local; no hay problemas de consistencia |

En memoria distribuida, según el grado de acoplamiento: multicomputadores (fuertemente acoplados), memoria compartida distribuida, clusters y redes (débilmente acoplados).

También importa la jerarquía de memoria (cache L1, L2, memoria principal): a mayor tamaño, mayor tiempo de acceso.

## 4. Clases de instrucciones

La programación secuencial se expresa con asignación, alternativa e iteración. La concurrente agrega una instrucción para expresar concurrencia.

### Declaraciones y asignación

```
int x = 8; int z, y;
int a[10]; int c[3:10];
int b[10] = ([10] 2);          # 10 elementos inicializados en 2
int bb[5,5] = ([5] ([5] 2));

x = e
v1 :=: v2                       # swap
skip                            # termina inmediatamente, sin efecto
```

### Alternativa

```
if B1 → S1
□  B2 → S2
...
□  Bn → Sn
fi
```

- Las guardas se evalúan en un orden arbitrario.
- Si varias son verdaderas, la elección es **no determinística**.
- Si ninguna es verdadera, el `if` no tiene efecto.

Los tres ejemplos de la diapositiva quedan como preguntas. Mi resolución:

| Ejemplo | Guardas | Respuesta |
| --- | --- | --- |
| 1 | `p > 2`, `p < 2`, `p == 2` | Nunca termina sin efecto: las tres guardas cubren todos los casos |
| 2 | `p > 2`, `p < 2` | Con `p = 2` ninguna guarda es verdadera y el `if` no hace nada |
| 3 | `p > 2 → p*2`, `p < 6 → p+4`, `p == 4 → p/2` | Ver tabla siguiente |

| p inicial | Guardas verdaderas | Resultado posible |
| --- | --- | --- |
| 1 | `p < 6` | 5 |
| 2 | `p < 6` | 6 |
| 3 | `p > 2`, `p < 6` | 6 o 7 |
| 4 | las tres | 8 o 2 |
| 5 | `p > 2`, `p < 6` | 10 o 9 |
| 6 | `p > 2` | 12 |
| 7 | `p > 2` | 14 |

### Iteración

```
do B1 → S1
□  B2 → S2
...
□  Bn → Sn
od
```

Se evalúan y ejecutan sentencias guardadas hasta que **todas las guardas sean falsas**. Si más de una es verdadera, la elección es no determinística.

Los cuatro ejemplos de "¿cuándo termina?", con mi resolución:

| Ejemplo | Guardas | ¿Cuándo termina? |
| --- | --- | --- |
| 1 | `p > 0 → p-2`, `p < 0 → p+3`, `p == 0 → random` | Nunca: siempre hay una guarda verdadera |
| 2 | `p > 2 → p*2`, `p < 2 → p*3` | Solo si p vale 2. Si arranca en otro valor, no llega nunca a 2 |
| 3 | `p > 0 → p-2`, `p > 3 → p+3`, `p > 6 → p/2` | Cuando p ≤ 0. Con 0 termina ya; con 3 hace 3 → 1 → -1 y termina; con 6 y 9 depende de las elecciones y no está garantizado |
| 4 | `p == 1 → p*2`, `p == 2 → p+3`, `p == 4 → p/2` | Siempre: en a lo sumo dos pasos p llega a 5 o ya no cumple ninguna guarda |

El **for-all** es la forma general de repetición:

```
fa i := 1 to n → a[i] = 0 af                                    # inicializar un vector
fa i := 1 to n, j := i+1 to n → m[i,j] :=: m[j,i] af             # trasponer una matriz
fa i := 1 to n, j := i+1 to n st a[i] > a[j] → a[i] :=: a[j] af  # ordenar un vector
```

La cláusula `st` (such that) filtra los valores de las variables de iteración. También se usan `while (cond) S;` y `for [i = 1 to n] S;`.

### Concurrencia: `co` y `process`

```
co S1 // ... // Sn oc                      # ejecuta las Si concurrentemente
co [i=1 to n] { a[i]=0; b[i]=0 } oc        # crea n tareas concurrentes

process A { sentencias }                    # un proceso
process B [i=1 to n] { sentencias }         # n procesos
```

La diferencia: el `co` **espera** a que todas sus tareas terminen antes de seguir con la sentencia siguiente. El `process` ejecuta en *background*.

```
process imprime10
  { for [i=1 to 10] write(i); }

process imprime1 [i = 1..10]
  { write(i); }
```

No son equivalentes. `imprime10` es un solo proceso e imprime 1 a 10 en orden. `imprime1` son diez procesos: imprimen los mismos números en cualquier orden.

## 5. Acciones atómicas y sincronización

### Atomicidad de grano fino

- Una **acción atómica** hace una transformación de estado indivisible: los estados intermedios no son visibles para otros procesos.
- La ejecución de un programa concurrente es un **intercalado** (*interleaving*) de las acciones atómicas de cada proceso.
- Una **historia** (*trace*) es una ejecución con un intercalado particular. La cantidad de historias posibles es enorme y no todas son válidas.

Una asignación no es atómica. `A = B` son dos instrucciones de máquina: `Load PosMemB, reg` y `Store reg, PosMemA`.

**Ejemplo 1.** Con `x = 0; y = 4; z = 2` se ejecutan en paralelo `x = y + z`, `y = 3` y `z = 4`. La primera se descompone en load de y, add de z y store en x.

| Orden de ejecución | Valor final de x |
| --- | --- |
| (1)(2)(3) o (1)(3)(2) | 6 |
| (2)(1)(3) | 5 |
| (3)(1)(2) | 8 |
| (2)(3)(1) o (3)(2)(1) | 7 |
| (1.1)(2)(1.2)(1.3)(3) | 6 |
| (1.1)(3)(1.2)(1.3)(2) | 8 |

**Ejemplo 2.** Con `x = 2; y = 2` se ejecutan en paralelo `z = x + y` y `x = 3; y = 4`. `z` puede valer 4, 5, 6 o 7. Lo llamativo es que `z` puede valer 6 aunque nunca exista un estado en el que `x + y` sea 6.

**Ejemplo 3: interleaving extremo.** Dos procesos hacen N veces `X = X + 1`, con X inicial en 0.

- **2N**: si no se pisan nunca.
- **N**: si en cada iteración los dos leen el mismo valor antes de que el otro escriba.
- **2**: P1 lee 0; P2 hace N-1 iteraciones; P1 escribe 1; P2 lee 1; P1 hace el resto de sus iteraciones; P2 escribe 2.

La conclusión de la clase: no se puede confiar en la intuición para analizar un programa concurrente. Además el tiempo absoluto se ignora; solo importan las secuencias, y el programa tiene que ser correcto para todos los intercalados.

### Referencia crítica y "a lo sumo una vez" (ASV)

Una **referencia crítica** en una expresión es una referencia a una variable que es modificada por otro proceso.

Una asignación `x = e` cumple ASV si pasa una de estas dos cosas:

1. `e` contiene a lo sumo una referencia crítica y `x` no es referenciada por otro proceso.
2. `e` no contiene referencias críticas, y entonces `x` puede ser leída por otro proceso.

Dicho corto: puede haber a lo sumo una variable compartida, y puede ser referenciada a lo sumo una vez. Si se cumple, la ejecución **parece atómica**.

| Programa (con `x = 0, y = 0`) | ¿Cumple ASV? | Resultado |
| --- | --- | --- |
| `co x=x+1 // y=y+1 oc` | Sí, no hay referencias críticas | Siempre x = 1, y = 1 |
| `co x=y+1 // y=y+1 oc` | Sí: el primero tiene una referencia crítica, el segundo ninguna | y = 1; x = 1 o 2 |
| `co x=y+1 // y=x+1 oc` | No, ninguna de las dos | x = 1, y = 2 o x = 2, y = 1. Pero puede dar x = 1, y = 1, que es un error |

### La sentencia `await`

Cuando algo no cumple ASV hay que ejecutarlo atómicamente. Para eso se construyen acciones atómicas de **grano grueso**.

- `⟨e⟩`: la expresión e se evalúa atómicamente.
- `⟨await (B) S;⟩`: se demora hasta que B sea verdadera y ejecuta S atómicamente. B es true cuando empieza S, y ningún estado interno de S es visible.

| Forma | Ejemplo | Sirve para |
| --- | --- | --- |
| Await general | `⟨await (s > 0) s = s - 1;⟩` | Exclusión mutua y condición juntas |
| Solo exclusión mutua | `⟨x = x + 1; y = y + 1⟩` | Acción atómica **incondicional** |
| Solo condición | `⟨await (count > 0)⟩` | Acción atómica **condicional** |

Es muy expresiva, pero implementar el `await` general es costoso. Si B cumple ASV, `⟨await (B)⟩` se puede implementar con *busy waiting*: `while (not B);`.

```
cant: int = 0;
Buffer: cola;

process Productor
 { while (true)
     Generar Elemento
     <await (cant < N); push(buffer, elemento); cant++ >
 }

process Consumidor
 { while (true)
     <await (cant > 0); pop(buffer, elemento); cant-- >
     Consumir Elemento
 }
```

## 6. Propiedades y fairness

Una propiedad es un atributo verdadero en todas las historias del programa.

| Clase | Idea | Ejemplos |
| --- | --- | --- |
| Seguridad (*safety*) | Nada malo ocurre: los estados son consistentes | Exclusión mutua, ausencia de interferencia, *partial correctness* |
| Vida (*liveness*) | Eventualmente ocurre algo bueno: el programa progresa | Terminación, que un pedido sea atendido, que un proceso alcance su SC |

Las propiedades de vida dependen de la política de scheduling. La *total correctness* (la pregunta de la diapositiva) combina las dos: es *partial correctness* más terminación.

### Fairness

Fairness trata de garantizar que los procesos tengan chance de avanzar sin importar lo que hagan los demás. Una acción atómica es **elegible** si es la próxima a ejecutar en su proceso.

| Tipo | Garantiza que se ejecute | Problema |
| --- | --- | --- |
| Incondicional | Toda acción atómica incondicional elegible | No dice nada de las condicionales |
| Débil | Lo anterior, más toda acción condicional cuya guarda se vuelve true **y permanece true** | No alcanza si la guarda cambia de valor mientras el proceso está demorado |
| Fuerte | Lo incondicional, más toda acción condicional cuya guarda es true **con infinita frecuencia** | Difícil de lograr en la práctica |

```
bool continue = true;
co while (continue); // continue = false; oc
```

Si la política es darle el procesador a un proceso hasta que termine o se demore, y arranca el `while`, el programa no termina nunca. Round-robin es incondicionalmente fair en un monoprocesador.

```
bool continue = true, try = false;
co while (continue) { try = true; try = false; }
 // <await (try) continue = false>
oc
```

Con fairness fuerte termina, porque `try` es true con infinita frecuencia. Con fairness débil puede no terminar, porque `try` no permanece en true. Round-robin es práctica pero no es fuertemente fair.

## 7. Trampas y autoevaluación

### Trampas típicas

- **Confundir concurrencia con paralelismo.** La concurrencia es un concepto de software; el paralelismo necesita varios procesadores.
- **Creer que una asignación es atómica.** Son varias instrucciones de máquina.
- **Olvidar que en el `if` con guardas, si ninguna es verdadera, no pasa nada.** Y que en el `do` esa es justamente la condición de salida.
- **Confundir `co` con `process`.** El `co` espera a que terminen sus tareas.
- **Mezclar fairness débil y fuerte.** Débil: la guarda queda en true. Fuerte: es true infinitas veces aunque cambie.

### Preguntas para chequear que se entendió

1. ¿Qué diferencia hay entre concurrencia y paralelismo?
2. ¿Cuáles son las cuatro condiciones necesarias y suficientes para el deadlock?
3. ¿Qué es la interferencia? Dar un ejemplo.
4. ¿Qué diferencia hay entre sincronización por exclusión mutua y por condición?
5. ¿Qué es una acción atómica? ¿Qué es una historia?
6. Dos procesos hacen N veces `X = X + 1`. ¿Qué valores finales son posibles y por qué?
7. ¿Qué dice la propiedad de "a lo sumo una vez" y para qué sirve?
8. ¿Qué garantiza `⟨await (B) S;⟩`?
9. ¿Qué diferencia hay entre una propiedad de seguridad y una de vida?
10. ¿Qué diferencia hay entre fairness incondicional, débil y fuerte?

## Material

- Diapositivas: `Clases teóricas/teoria-1-introduccion.pdf`. La diapositiva 6 tiene los links a los seis videos con audio.
- Libro base: *Foundations of Multithreaded, Parallel, and Distributed Programming*, Gregory Andrews.
