# Apunte — Teoría 9: Pthreads y MPI

## La idea en una frase

Esta clase baja los mecanismos de la materia a dos librerías reales de C: **Pthreads** para memoria compartida (mutex, variables condición y semáforos, con los que se simulan monitores) y **MPI** para pasaje de mensajes (send/receive bloqueantes y no bloqueantes, y operaciones colectivas).

## 1. Pthreads

Un *thread* es un proceso liviano: tiene su propio contador de programa y su pila, pero no controla el "contexto pesado" (por ejemplo, las tablas de páginas).

Al principio los mecanismos de multithreading eran heterogéneos y poco portables. A mediados de los 90, POSIX impulsó una biblioteca estándar en C: **Pthreads**. Los conceptos valen igual para Java Threads, NT Threads o Solaris Threads.

### Creación y terminación

```c
#include <pthread.h>

int pthread_create (pthread_t *thread_handle, const pthread_attr_t *attribute,
                    void * (*thread_function)(void *), void *arg);
int pthread_exit (void *res);
int pthread_join (pthread_t thread, void **ptr);
int pthread_cancel (pthread_t thread);
```

El `main` debe esperar con `pthread_join` a que terminen todos los threads.

### Mutex: exclusión mutua

Un mutex tiene dos estados, *locked* y *unlocked*. Solo un thread a la vez puede tenerlo bloqueado, y el lock es atómico. Todos los mutex se inicializan desbloqueados.

```c
int pthread_mutex_init (pthread_mutex_t *mutex, const pthread_mutexattr_t *lock_attr);
int pthread_mutex_lock (pthread_mutex_t *mutex);
int pthread_mutex_unlock (pthread_mutex_t *mutex);
int pthread_mutex_trylock (pthread_mutex_t *mutex_lock);
```

| Tipo de mutex | Si el mismo thread hace lock dos veces |
| --- | --- |
| Normal | Deadlock |
| Recursive | Se permite: incrementa un contador |
| Error Check | Devuelve un error |

El tipo se elige con los atributos antes de inicializar el mutex (`PTHREAD_MUTEX_NORMAL_NP`, `PTHREAD_MUTEX_RECURSIVE_NP`, `PTHREAD_MUTEX_ERRORCHECK_NP`).

**Overhead**: los locks son puntos de serialización. Si la sección crítica es larga, el rendimiento cae. `pthread_mutex_trylock` intenta el lock y devuelve el control de inmediato, diciendo si pudo o no. Evita tiempos ociosos y es más barato porque no maneja colas de espera.

### Productor/consumidor solo con mutex

Buffer de tamaño 1. El productor reintenta hasta que el buffer esté vacío:

```c
pthread_mutex_t mutex;
int hayElemento;
tipo_elemento Buffer;

void *productor (void *datos)
{ tipo_elemento elem;
  int ok;
  while (true)
    { ok = 0;
      generar_elemento(&elem);
      while (ok == 0)
        { pthread_mutex_lock(&mutex);
          if (hayElemento == 0)
            { Buffer = elem;
              hayElemento = 1;
              ok = 1;
            }
          pthread_mutex_unlock(&mutex);
        }
    }
}

void *consumidor (void *datos)
{ int ok;
  tipo_elemento elem;
  while (true)
    { ok = 0;
      while (ok == 0)
        { pthread_mutex_lock(&mutex);
          if (hayElemento == 1)
            { elem = Buffer;
              hayElemento = 0;
              ok = 1;
            }
          pthread_mutex_unlock(&mutex);
        }
      procesar_elemento(elem);
    }
}
```

Funciona, pero es **busy waiting**: el thread entra y sale del mutex una y otra vez. Para eso están las variables condición.

### Variables condición

Permiten que un thread se bloquee hasta que se alcance cierto estado, **sin usar CPU**.

- Cada variable condición se asocia con un predicado. Se puede asociar con varios, pero eso complica el debug.
- **Siempre tiene un mutex asociado.** El thread toma el mutex, chequea el predicado y, si es falso, hace `wait`.

```c
int pthread_cond_init (pthread_cond_t *cond, const pthread_condattr_t *attr);
int pthread_cond_wait (pthread_cond_t *cond, pthread_mutex_t *mutex);
int pthread_cond_timedwait (pthread_cond_t *cond, pthread_mutex_t *mutex,
                            const struct timespec *abstime);
int pthread_cond_signal (pthread_cond_t *cond);
int pthread_cond_broadcast (pthread_cond_t *cond);
int pthread_cond_destroy (pthread_cond_t *cond);
```

`pthread_cond_wait` libera el mutex mientras espera y lo vuelve a tomar al despertarse. Por eso recibe los dos.

| Monitores (Teoría 4) | Pthreads |
| --- | --- |
| `wait(cv)` | `pthread_cond_wait(&cv, &mutex)` |
| `signal(cv)` | `pthread_cond_signal(&cv)` |
| `signal_all(cv)` | `pthread_cond_broadcast(&cv)` |

```c
pthread_cond_t vacio, lleno;
pthread_mutex_t mutex;
int hayElemento;
tipo_elemento Buffer;

void *productor (void *datos)
{ tipo_elemento elem;
  while (true)
    { generar_elemento(elem);
      pthread_mutex_lock(&mutex);
      while (hayElemento == 1)
          pthread_cond_wait(&vacio, &mutex);
      Buffer = elem;
      hayElemento = 1;
      pthread_cond_signal(&lleno);
      pthread_mutex_unlock(&mutex);
    }
}

void *consumidor (void *datos)
{ tipo_elemento elem;
  while (true)
    { pthread_mutex_lock(&mutex);
      while (hayElemento == 0)
          pthread_cond_wait(&lleno, &mutex);
      elem = Buffer;
      hayElemento = 0;
      pthread_cond_signal(&vacio);
      pthread_mutex_unlock(&mutex);
      procesar_elemento(elem);
    }
}
```

### Atributos

Un *attribute object* describe las propiedades de un thread, un mutex o una variable condición. Se arma y se le pasa a la función que inicializa la entidad. Mejora la modularidad y facilita modificar el código.

- Threads: `pthread_attr_init`, `pthread_attr_destroy`, y funciones `pthread_attr_set...` para el estado *detach*, el tamaño de pila y la política de scheduling.
- Mutex: `pthread_mutexattr_init` y `pthread_mutexattr_settype_np`.

## 2. Semáforos en Pthreads

Están en `semaphore.h`.

| Operación | Equivale a |
| --- | --- |
| `sem_t semaforo;` | Declaración, global a los threads |
| `sem_init(&semaforo, alcance, inicial)` | Inicialización. `alcance` es 0 si lo comparten los hilos de un solo proceso, distinto de 0 si lo comparten todos los procesos |
| `sem_wait(&semaforo)` | P |
| `sem_post(&semaforo)` | V |

Hay funciones extra (wait condicional, obtener el valor, destruir), pero **no se pueden usar en la práctica de la materia**.

```c
#include <pthread.h>
#include <semaphore.h>
#define SHARED 1

sem_t vacio, lleno;
int dato, numItems;

int main(int argc, char *argv[])
{ ...
  sem_init(&vacio, SHARED, 1);
  sem_init(&lleno, SHARED, 0);
  ...
  pthread_create(&pid, &attr, Productor, NULL);
  pthread_create(&cid, &attr, Consumidor, NULL);
  pthread_join(pid, NULL);
  pthread_join(cid, NULL);
}

void *Productor (void *arg)
{ int item;
  for (item = 1; item <= numItems; item++)
    { sem_wait(&vacio);
      dato = item;
      sem_post(&lleno);
    }
  pthread_exit();
}

void *Consumidor (void *arg)
{ int total = 0, item, aux;
  for (item = 1; item <= numItems; item++)
    { sem_wait(&lleno);
      aux = dato;
      sem_post(&vacio);
      total = total + aux;
    }
  printf("TOTAL: %d\n", total);
  pthread_exit();
}
```

Es el semáforo binario dividido de la Teoría 3: `vacio` y `lleno` fuerzan la alternancia.

## 3. Monitores en Pthreads

Pthreads no tiene monitores, pero se simulan:

- La **exclusión mutua implícita** del monitor se hace con un **mutex por monitor**: se bloquea antes de la llamada al procedure y se desbloquea al terminar.
- La **sincronización por condición** se hace con variables condición.
- Cada llamada a un procedure del monitor se **reemplaza por el código** de ese procedure.

En la diapositiva 29 dice "Pthreads **no** permite manejar la Exclusión Mutua por medio de las variables mutex". Es un error de tipeo: sí lo permite, y es justamente lo que se usa.

### Ejemplo: lectores y escritores

Se parte del monitor con passing the condition de la Teoría 4.

**Paso 1**: un mutex para el monitor, que rodea cada llamada.

```c
void *lector (void *)
{ while (true)
    { pthread_mutex_lock(&mutex);
      Controlador.pedido_leer();
      pthread_mutex_unlock(&mutex);
      // Lee la BD
      pthread_mutex_lock(&mutex);
      Controlador.libera_leer();
      pthread_mutex_unlock(&mutex);
    }
}
```

**Paso 2**: se reemplaza cada llamada por el cuerpo del procedure.

```c
void *escritor (void *)
{ while (true)
    { pthread_mutex_lock(&mutex);
      if (nr > 0 OR nw > 0)
        { dw = dw + 1;
          pthread_cond_wait(&ok_escribir, &mutex);
        }
      else nw = nw + 1;
      pthread_mutex_unlock(&mutex);
      // Escribe la BD
      pthread_mutex_lock(&mutex);
      if (dw > 0)
        { dw = dw - 1;
          pthread_cond_signal(&ok_escribir);
        }
      else
        { nw = nw - 1;
          if (dr > 0)
            { nr = dr;
              dr = 0;
              pthread_cond_broadcast(&ok_leer);
            }
        }
      pthread_mutex_unlock(&mutex);
    }
  pthread_exit();
}

void *lector (void *)
{ while (true)
    { pthread_mutex_lock(&mutex);
      if (nw > 0)
        { dr = dr + 1;
          pthread_cond_wait(&ok_leer, &mutex);
        }
      else nr = nr + 1;
      pthread_mutex_unlock(&mutex);
      // Lee la BD
      pthread_mutex_lock(&mutex);
      nr = nr - 1;
      if (nr == 0 and dw > 0)
        { dw = dw - 1;
          pthread_cond_signal(&ok_escribir);
          nw = nw + 1;
        }
      pthread_mutex_unlock(&mutex);
    }
  pthread_exit();
}
```

El mutex se suelta mientras se lee o se escribe la BD. Si no, los lectores no podrían leer a la vez.

Dos observaciones sobre las diapositivas 31 a 34:

- `nr`, `nw`, `dr`, `dw`, las variables condición y el mutex están declarados **dentro del `main`**. Así los threads no los verían: tendrían que ser globales.
- Fuera de la materia, en Pthreads real un `pthread_cond_wait` puede despertar sin que nadie haya hecho signal (*spurious wakeup*). Por eso en código real se recomienda chequear con `while`. Esto no está en la teoría; para los parciales seguí lo que pide la cátedra.

## 4. Pasaje de mensajes: semántica del send

```
P0                      P1
a = 100;                receive(&a, 1, 0);
send(&a, 1, 1);         printf("%d\n", a);
a = 0;
```

La semántica del send exige que P1 imprima **100**, no 0, aunque P0 modifique `a` justo después del send.

| Protocolo | Cómo asegura la semántica | Costo |
| --- | --- | --- |
| Bloqueante sin buffering | No devuelve el control hasta que el dato llegó al receptor | Ociosidad del emisor |
| Bloqueante con buffering | Copia el dato a un buffer y devuelve el control | Manejo del buffer |
| No bloqueante (con o sin buffering) | Devuelve el control de inmediato | Hay que chequear después que terminó. **La semántica queda en manos del programador** |

## 5. MPI

MPI (*Message Passing Interface*) define una librería estándar de pasaje de mensajes para C y Fortran.

- Más de 125 rutinas, pero con **6** alcanza para escribir programas: `MPI_Init`, `MPI_Finalize`, `MPI_Comm_size`, `MPI_Comm_rank`, `MPI_Send` y `MPI_Recv`.
- Usa el modelo **SPMD**: el mismo programa en todos los procesos.
- Todo tiene el prefijo `MPI_`. Las operaciones exitosas devuelven `MPI_SUCCESS`.

### Inicio, comunicadores e identificación

```c
MPI_Init (int *argc, char **argv)          // antes de cualquier otra rutina MPI
MPI_Finalize ()                            // la última llamada MPI
MPI_Comm_size (MPI_Comm comunicador, int *cantidad)
MPI_Comm_rank (MPI_Comm comunicador, int *rank)
```

- Un **comunicador** define un dominio de comunicación (un grupo de procesos). Un proceso puede estar en varios, con un rank distinto en cada uno.
- `MPI_COMM_WORLD` incluye a todos los procesos de la aplicación.
- Cada transferencia indica sobre qué comunicador se hace.

```c
#include <mpi.h>
main(int argc, char *argv[])
{ int cantidad, identificador;
  MPI_Init(&argc, &argv);
  MPI_Comm_size(MPI_COMM_WORLD, &cantidad);
  MPI_Comm_rank(MPI_COMM_WORLD, &identificador);
  printf("Soy %d de %d \n", identificador, cantidad);
  MPI_Finalize();
}
```

La diapositiva 43 dice que el rank está entre [0..cantidad]. En realidad va de 0 a cantidad-1.

Los tipos de datos son equivalentes a los de C: `MPI_CHAR`, `MPI_INT`, `MPI_FLOAT`, `MPI_DOUBLE`, etc., más `MPI_BYTE` y `MPI_PACKED`.

### Comunicación punto a punto bloqueante

| Rutina | Tipo |
| --- | --- |
| `MPI_Send` | Send básico |
| `MPI_Bsend` | Bloqueante con buffering |
| `MPI_Ssend` | Bloqueante sin buffering (sincrónico) |
| `MPI_Recv` | Receive bloqueante |

```c
MPI_Send (void *buf, int cantidad, MPI_Datatype tipoDato, int destino, int tag,
          MPI_Comm comunicador)
MPI_Recv (void *buf, int cantidad, MPI_Datatype tipoDato, int origen, int tag,
          MPI_Comm comunicador, MPI_Status *estado)
```

- El **tag** distingue clases de mensajes, como los ports de CSP.
- En el receive se pueden usar los comodines `MPI_ANY_SOURCE` y `MPI_ANY_TAG`.
- `MPI_Status` tiene `MPI_SOURCE`, `MPI_TAG` y `MPI_ERROR`. Sirve para saber de quién vino el mensaje cuando se usó un comodín.
- `MPI_Get_count(&estado, tipoDato, &cantidad)` da cuántos elementos llegaron.

### Ejemplo: intercambio de valores

Los dos procesos envían primero y después reciben:

```c
MPI_Comm_rank(MPI_COMM_WORLD, &id);
if (id == 0) { idAux = 1; valor = 14; }
else { idAux = 0; valor = 25; }
MPI_Send(&valor, 1, MPI_INT, idAux, 1, MPI_COMM_WORLD);
MPI_Recv(&otroValor, 1, MPI_INT, idAux, 1, MPI_COMM_WORLD, &estado);
```

Si `MPI_Send` se comporta como `MPI_Ssend`, es el mismo deadlock del intercambio con PMS de la Teoría 6: los dos quedan bloqueados en el send esperando que el otro reciba.

La solución es romper la simetría:

```c
if (id == 0)
  { valor = 14;
    MPI_Send(&valor, 1, MPI_INT, 1, 1, MPI_COMM_WORLD);
    MPI_Recv(&otroValor, 1, MPI_INT, 1, 1, MPI_COMM_WORLD, &estado);
  }
else
  { valor = 25;
    MPI_Recv(&otroValor, 1, MPI_INT, 0, 1, MPI_COMM_WORLD, &estado);
    MPI_Send(&valor, 1, MPI_INT, 0, 1, MPI_COMM_WORLD);
  }
```

### Comunicación punto a punto no bloqueante

Empiezan la operación y devuelven el control de inmediato. No aseguran que la comunicación haya terminado.

```c
MPI_Isend (void *buf, int cantidad, MPI_Datatype tipoDato, int destino, int tag,
           MPI_Comm comunicador, MPI_Request *solicitud)
MPI_Irecv (void *buf, int cantidad, MPI_Datatype tipoDato, int origen, int tag,
           MPI_Comm comunicador, MPI_Request *solicitud)
MPI_Test (MPI_Request *solicitud, int *flag, MPI_Status *estado)   // ¿terminó?
MPI_Wait (MPI_Request *solicitud, MPI_Status *estado)              // bloquea hasta que termine
```

- **Ventaja**: se solapa cómputo con comunicación y se evita el overhead de buffering.
- **Desventaja**: asegurar que la comunicación sea correcta queda en manos del programador.

La pregunta de la diapositiva 50 es si alcanza con cambiar `MPI_Send` por `MPI_Isend`. No alcanza. Después del send, el proceso 0 pone `dato[i] = 0`. Con `Isend` el envío podría no haber terminado, y el receptor recibiría ceros. Hay que hacer `MPI_Wait` antes de tocar el buffer:

```c
if (id == 0)
  { cant = atoi(argv[1]);
    // INICIALIZA dato
    MPI_Isend(dato, cant, MPI_INT, 1, 1, MPI_COMM_WORLD, &req);
    // TRABAJA (sin tocar dato)
    MPI_Wait(&req, &estado);
    for (i = 0; i < 100; i++) dato[i] = 0;
  }
```

Del lado del receptor, con `MPI_Irecv` y `MPI_Test` se puede trabajar mientras se espera:

```c
MPI_Irecv(dato, 100, MPI_INT, 0, 1, MPI_COMM_WORLD, &req);
MPI_Test(&req, &flag, &estado);
while (!flag)
  { // Trabaja mientras espera
    MPI_Test(&req, &flag, &estado);
  }
// PROCESA LOS DATOS
```

### Consultar mensajes pendientes

Sirven para conocer origen, tag y cantidad de elementos de un mensaje **antes** de recibirlo.

| Rutina | Comportamiento |
| --- | --- |
| `MPI_Probe(origen, tag, comunicador, &estado)` | Bloquea hasta que llegue un mensaje que cumpla con origen y tag |
| `MPI_Iprobe(origen, tag, comunicador, &flag, &estado)` | Chequea si llegó, sin bloquear |

La diapositiva pregunta cuándo y por qué usar cada uno. Mi respuesta: `MPI_Probe` cuando no hay nada más que hacer y se necesita saber el tamaño o el origen para preparar el buffer antes del `MPI_Recv`. `MPI_Iprobe` cuando el proceso puede seguir trabajando, o para decidir entre varios orígenes o tags según cuál llegó.

### Comunicaciones colectivas

Operan sobre todos los procesos de un comunicador, y **todos tienen que llamar a la rutina**. Son más eficientes que armar lo mismo con punto a punto, porque se implementan en etapas (por ejemplo, en árbol: con 8 procesos, un broadcast se hace en 3 pasos).

| Rutina | Qué hace |
| --- | --- |
| `MPI_Barrier(comunicador)` | Barrera |
| `MPI_Bcast` | Un proceso envía el mismo dato a todos, incluido él |
| `MPI_Reduce` | Combina los datos de todos (incluido el destino) con una operación, y deja el resultado en uno |
| `MPI_Gather` / `MPI_Gatherv` | Junta los vectores de todos, concatenados en orden, en un proceso. La versión `v` admite tamaños distintos |
| `MPI_Scatter` / `MPI_Scatterv` | Reparte un vector entre todos, incluido el dueño. La versión `v` admite cantidades distintas |

```c
MPI_Bcast (void *buf, int cantidad, MPI_Datatype tipoDato, int origen, MPI_Comm comunicador)
MPI_Reduce (void *sendbuf, void *recvbuf, int cantidad, MPI_Datatype tipoDato, MPI_Op operacion,
            int destino, MPI_Comm comunicador)
MPI_Gather (void *sendbuf, int cantEnvio, MPI_Datatype tipoDatoEnvio, void *recvbuf, int cantRec,
            MPI_Datatype tipoDatoRec, int destino, MPI_Comm comunicador)
MPI_Scatter (void *sendbuf, int cantEnvio, MPI_Datatype tipoDatoEnvio, void *recvbuf, int cantRec,
             MPI_Datatype tipoDatoRec, int origen, MPI_Comm comunicador)
```

Scatter y Gather son inversos: uno reparte, el otro junta.

### Cómo minimizar el overhead de comunicación

- Maximizar la localidad de datos.
- Minimizar el volumen de datos intercambiado.
- Minimizar la cantidad de comunicaciones.
- Considerar el costo de cada bloque de datos intercambiado.
- Replicar datos cuando convenga.
- Solapar cómputo y comunicación.
- En lo posible, usar comunicaciones asincrónicas.
- Usar comunicaciones colectivas en lugar de punto a punto.

## 6. Trampas y autoevaluación

### Trampas típicas

- **Olvidar el mutex de la variable condición.** `pthread_cond_wait` siempre lo recibe.
- **Usar `sem_getvalue` u otras funciones extra de semáforos** en la práctica. No están permitidas.
- **Dejar el mutex tomado mientras se usa la BD** al simular un monitor.
- **Declarar el estado compartido dentro del `main`.** Tiene que ser global.
- **Que los dos procesos hagan `MPI_Send` primero.** Deadlock si el send es sincrónico.
- **Tocar el buffer después de `MPI_Isend` sin `MPI_Wait`.**
- **Llamar a una colectiva solo desde algunos procesos.** Todos los del comunicador deben llamarla.

### Preguntas para chequear que se entendió

1. ¿Qué diferencia hay entre los mutex Normal, Recursive y Error Check?
2. ¿Para qué sirve `pthread_mutex_trylock`?
3. ¿Por qué el productor/consumidor solo con mutex es busy waiting?
4. ¿Por qué `pthread_cond_wait` recibe un mutex?
5. ¿Cómo se simula un monitor en Pthreads?
6. ¿Qué exige la semántica del send y cómo la aseguran los distintos protocolos?
7. ¿Cuáles son las seis rutinas básicas de MPI?
8. ¿Qué es un comunicador?
9. ¿Por qué el primer ejemplo de intercambio puede producir deadlock?
10. ¿Alcanza con cambiar `MPI_Send` por `MPI_Isend`? ¿Por qué?
11. ¿Qué diferencia hay entre `MPI_Probe` y `MPI_Iprobe`?
12. ¿Qué hacen `MPI_Bcast`, `MPI_Reduce`, `MPI_Gather` y `MPI_Scatter`?

## Material

- Diapositivas: `Clases teóricas/teoria-9.pdf`. La diapositiva 2 tiene los links a los tres videos con audio (Pthreads, semáforos y monitores en Pthreads, y MPI).
