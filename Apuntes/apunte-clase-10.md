# Apunte — Teoría 10: Introducción a la programación paralela

## La idea en una frase

Para que un programa paralelo rinda hay que entender la arquitectura donde corre, descomponer bien el problema y repartir las tareas entre los procesadores. El rendimiento se mide con **speedup** y **eficiencia**, y los problemas típicos se encuadran en unos pocos **paradigmas** con estructura de control conocida.

La clase tiene cuatro partes:

1. Arquitecturas paralelas y cómo se clasifican.
2. Diseño de algoritmos paralelos: descomposición, aglomeración y mapeo.
3. Métricas de rendimiento.
4. Paradigmas de programación paralela.

## 1. Arquitecturas paralelas

Se pueden clasificar por cuatro criterios:

- Por la organización del espacio de direcciones.
- Por la granularidad.
- Por el mecanismo de control.
- Por la red de interconexión.

La clase desarrolla tres. La granularidad se menciona, pero no tiene diapositivas propias.

### Por el espacio de direcciones

| | Memoria compartida | Memoria distribuida |
| --- | --- | --- |
| Interacción | Modificando datos en la memoria común | Solo por pasaje de mensajes |
| Esquemas | UMA (bus o crossbar, SMP) y NUMA para más procesadores | Multicomputadores, memoria compartida distribuida, clusters, redes |
| Problema | Sincronización y consistencia | No hay problema de consistencia: la memoria es local |
| Ejemplos | Multicore, GPU | Clusters |

En algunas plataformas conviven los dos mecanismos.

En memoria distribuida, según el acoplamiento: los **multicomputadores** son fuertemente acoplados (procesadores y red físicamente cerca, alto ancho de banda) y las **redes** son débilmente acopladas.

### Por el mecanismo de control: taxonomía de Flynn

Propuesta por Flynn en 1972. Clasifica según cómo se ejecutan las instrucciones sobre los datos.

| Clase | Instrucciones | Datos | Características | Ejemplos |
| --- | --- | --- | --- | --- |
| SISD | Una | Uno | Ejecución secuencial y determinística | Monoprocesadores |
| SIMD | Una | Muchos | Procesadores idénticos y simples ejecutan la misma instrucción sobre distintos datos, en forma sincrónica. Se pueden deshabilitar procesadores selectivamente | Array processors (CM-2, Maspar, Illiac IV). Procesamiento de imágenes |
| MISD | Muchas | Uno | Operación sincrónica (*lockstep*). No son de propósito general ("hipotéticas") | Varios filtros sobre una señal, varios algoritmos criptográficos sobre un mensaje |
| MIMD | Muchas | Muchos | Cada procesador ejecuta su propio programa a su ritmo. Con memoria compartida o distribuida | Multicores, clusters, nCube 2, CM-5 |

MIMD se subdivide en:

- **MPMD** (*multiple program multiple data*): cada procesador ejecuta su propio programa. Ejemplo: PVM.
- **SPMD** (*single program multiple data*): un único programa fuente y cada procesador ejecuta su copia. Ejemplo: MPI.

### Por la red de interconexión

| Tipo | Cómo se construye | Uso típico | Ejemplos |
| --- | --- | --- | --- |
| Estática | Links punto a punto | Máquinas de pasaje de mensajes | Arreglo lineal, anillo, estrella, árbol, mesh, toro, hipercubo, completamente conectada |
| Dinámica | Switches y enlaces | Máquinas de memoria compartida | Bus, crossbar, multistage, árbol dinámico |

Un hipercubo d-dimensional tiene p = 2^d procesadores.

El diseño de la red depende del ancho de banda, el tiempo de *startup*, si los caminos son estáticos o dinámicos, si opera sincrónica o asincrónicamente, la topología y el costo.

## 2. Diseño de algoritmos paralelos

La mejor solución paralela puede ser totalmente distinta de la secuencial. Conviene un enfoque metódico: los aspectos independientes de la máquina (como la concurrencia) se consideran temprano, y los específicos de la máquina se postergan.

### Por qué es difícil

- Decidir la granularidad óptima de las tareas.
- Mapear tareas y datos a procesadores, en forma estática o dinámica.
- Manejar comunicación y sincronización.
- Asegurar corrección, evitar deadlocks y desbalances.
- Lograr tolerancia a fallos.
- Manejar la heterogeneidad.
- Lograr escalabilidad.
- Consumo energético.

### Pasos

1. Identificar porciones de trabajo concurrentes (tareas).
2. Mapear tareas a procesos en distintos procesadores.
3. Distribuir los datos de entrada, intermedios y de salida.
4. Manejar el acceso a datos compartidos.
5. Sincronizar los procesos.

Los dos pasos fundamentales son la **descomposición en tareas** y el **mapeo de procesos a procesadores**.

### Descomposición en tareas

Al principio se busca una descomposición de **grano fino**: muchas tareas chicas, para tener la mayor flexibilidad. Después se revisa.

| Descomposición | Primero divide... | Después... | Tipo de paralelismo |
| --- | --- | --- | --- |
| De datos (dominio) | Los datos, en general en partes iguales | Asocia a cada parte el cómputo que opera sobre ella | De datos: tareas con el mismo código |
| Funcional | El cómputo, en tareas disjuntas | Trata los datos | Funcional: tareas con distinto código |

- En la descomposición de datos, si una operación necesita datos de varias tareas, aparece comunicación.
- En la funcional, si los datos que necesitan las tareas se superponen mucho, probablemente convenga descomponer por dominio.
- La funcional también sirve para estructurar el programa y reducir la complejidad del diseño.

### Aglomeración

El algoritmo que sale de la descomposición es abstracto: no está pensado para una máquina real. En esta etapa se decide si conviene **combinar tareas** en otras más grandes, y si conviene **replicar** datos o cómputo.

Tres objetivos que a veces se contradicen:

- **Aumentar la granularidad**: combinar tareas relacionadas para reducir la comunicación.
- **Preservar la flexibilidad**: juntar demasiado limita la escalabilidad. Poder crear una cantidad variable de tareas es clave para un programa portable y escalable.
- **Reducir costos de ingeniería de software**: evitar cambios extensos, por ejemplo reutilizando rutinas.

### Características de las tareas que afectan el rendimiento

- Cómo se generan.
- Su tamaño.
- Si se conoce su tamaño.
- El volumen de datos asociado a cada una.

### Mapeo de tareas a procesadores

Se decide dónde ejecuta cada tarea. No existe en monoprocesadores ni en máquinas de memoria compartida con scheduling automático.

El objetivo es minimizar el tiempo de ejecución, con dos estrategias que pueden chocar:

- Poner tareas que pueden ejecutar concurrentemente en **procesadores distintos**, para aumentar la concurrencia.
- Poner tareas que se comunican mucho en el **mismo procesador**, para aumentar la localidad.

En el caso general el problema es **NP-completo**; hay heurísticas para clases de problemas. Normalmente hay más tareas que procesadores.

Criterios:

1. Mapear tareas independientes a procesadores distintos.
2. Darles prioridad a las tareas que están en el **camino crítico**.
3. Asignar tareas que interactúan mucho al mismo procesador.

El criterio 3 puede llevar a no paralelizar. Hay que buscar un equilibrio: **el mapping determina la eficiencia del algoritmo**.

## 3. Métricas de rendimiento

Un sistema paralelo es un algoritmo más la arquitectura donde se implementa. El tiempo de ejecución depende del tamaño de la entrada, de la arquitectura y de la cantidad de procesadores.

### Speedup

Es el cociente entre el tiempo del **mejor algoritmo secuencial conocido** y el tiempo del algoritmo paralelo:

```latex
S = \frac{T_S}{T_P}
```

El speedup óptimo depende de la arquitectura. Si es homogénea, es P (la cantidad de procesadores). Si es heterogénea, se suman las potencias de cálculo relativas al mejor procesador:

```latex
S_{\text{óptimo}} = \frac{\sum_{i=1}^{P} \text{PotenciaCalculo}(i)}{\text{PotenciaCalculo}(\text{mejor})}
```

En general el speedup va entre 0 y el óptimo. Se habla de speedup **lineal o perfecto** (igual al óptimo), **sublineal** (menor) y **superlineal** (mayor).

### Eficiencia

```latex
E = \frac{S}{S_{\text{óptimo}}}
```

Mide la fracción del tiempo en que los procesadores hacen cómputo útil. Va de 0 a 1; vale 1 con speedup perfecto.

### Escalabilidad

Es la capacidad de usar eficientemente una cantidad creciente de procesadores. Es muy difícil extrapolarla desde pruebas chicas:

- Con **pocos datos**, los resultados están afectados por la localidad en el manejo de la memoria.
- Con **pocos procesadores**, no se ven los costos de comunicación por la distancia entre procesadores ni la caída del ancho de banda efectivo.

### Qué limita el speedup

- Alto porcentaje de código secuencial (**ley de Amdahl**).
- Mucha entrada/salida respecto del cómputo.
- Algoritmo no adecuado.
- Contención de memoria excesiva.
- Problema chico, o de tamaño fijo que no crece con p.
- **Desbalance de carga**, que genera esperas ociosas.
- **Overhead paralelo**: crear procesos, sincronizar, etc.

La clase nombra la ley de Amdahl pero no la desarrolla. La idea general es que si una fracción del programa es inherentemente secuencial, el speedup nunca puede superar 1 dividido esa fracción, por más procesadores que se agreguen.

## 4. Paradigmas de programación paralela

Un paradigma es una clase de algoritmos que resuelven problemas distintos pero tienen la **misma estructura de control**. Para cada uno se puede escribir un esqueleto, y los patrones de comunicación son muy parecidos en todos los casos.

| Paradigma | Estructura | Problema típico |
| --- | --- | --- |
| Cliente/servidor | Servidores atienden pedidos de muchos clientes | El esquema dominante en procesamiento distribuido |
| Master/worker | Un master reparte datos y junta resultados | El master puede ser cuello de botella |
| Pipeline / sistólico | Los datos pasan por una secuencia de etapas | Balance entre etapas |
| Dividir y conquistar | División recursiva en subproblemas | Combinar las soluciones |
| SPMD | Un único programa sobre porciones de los datos | Distribución de datos irregular |

### Cliente/servidor

Comunicación bidireccional. El servidor atiende a un cliente por vez, o a varios con multithreading. Los mecanismos de invocación son variados: rendezvous, RPC, monitores. El soporte puede ser una LAN o la web.

### Master/worker (master/slave)

El master envía datos iterativamente a los workers y recibe sus resultados.

- **Cuello de botella** posible, por tareas muy chicas o workers muy rápidos. Hay que elegir bien el grano.
- Según la dependencia entre iteraciones:
    - **Dependientes**: el master necesita los resultados de todos para generar el siguiente conjunto de datos.
    - **Independientes**: los datos llegan al master, que no necesita resultados anteriores.
- Distribución de datos **estática** (se reparte todo según una política) o **dinámica** (bajo demanda).
- Es una variación de SPMD con dos programas en lugar de uno.
- Complicaciones: procesadores heterogéneos (balance de carga), trabajo en fases (sincronización), y generalización a varios niveles.

### Pipeline y algoritmos sistólicos

El problema se divide en una secuencia de pasos y el flujo de datos pasa por los procesos, cada uno con su tarea. Ejemplo: filtrado, etiquetado y análisis de escena en imágenes. Se mapea naturalmente a un arreglo lineal de procesadores.

Extensiones: procesadores especializados distintos, más de un procesador por etapa, y flujos que no son una línea simple (varias líneas que se combinan), que es el **procesamiento sistólico**.

### Dividir y conquistar

Paralelismo **recursivo**. El problema se divide repetidamente en subproblemas más chicos (dividir), se resuelven de forma independiente (conquistar) y las soluciones se combinan (combinar). Cada proceso recibe una fracción de datos: si puede, la procesa; si no, crea hijos y les reparte los datos.

Ejemplos: *sorting by merging*, cálculo de raíces de funciones continuas, problema del viajante.

**Ejemplo: problema de la cuadratura.** Aproximar la integral de f(x) entre a y b.

Solución secuencial iterativa con el método trapezoidal:

```
double fl = f(a), fr, area = 0.0;
double dx = (b-a)/ni;
for [x = (a+dx) to b by dx]
  { fr = f(x);
    area = area + (fl+fr) * dx / 2;
    fl = fr;
  }
```

Versión recursiva adaptiva: se calcula el área del trapecio entero y la suma de las dos mitades. Si difieren en más de un error e, se repite para cada mitad.

```
double quad (double l, r, fl, fr, area) {
  double m = (l+r)/2;
  double fm = f(m);
  double larea = (fl+fm)*(m-l)/2;
  double rarea = (fm+fr)*(r-m)/2;
  if (abs((larea+rarea)-area) > e) {
    co larea = quad(l, m, fl, fm, larea);
    // rarea = quad(m, r, fm, fr, rarea);
    oc
  }
  return (larea+rarea);
}
```

La versión secuencial es igual pero sin `co ... oc`. Los dos llamados recursivos son independientes y pueden ejecutar en paralelo. Se usa así:

```
area = quad(a, b, f(a), f(b), (f(a)+f(b)) * (b-a) / 2)
```

En la figura de la diapositiva 51 el punto medio figura como m = (b-a)/2. Debería ser (a+b)/2, que es lo que usa el código de la diapositiva 52.

### SPMD

Un único programa que cada nodo ejecuta sobre su porción de datos. Las condiciones hacen que cada nodo tome caminos distintos.

Tiene dos fases:

1. **Elegir la distribución de datos.** La carga de cada nodo es proporcional a los datos asignados. Es difícil con cómputos irregulares o máquinas heterogéneas.
2. **Generar el programa paralelo**, que depende de esa distribución.

Suele implicar paralelismo **iterativo**: cada proceso es un programa con uno o más loops.

**Ejemplo: multiplicación de matrices.** Cada `c[i,j]` es el producto interno de la fila i de `a` por la columna j de `b`. Cada producto interno es independiente: es una aplicación *embarrassingly parallel*.

```
for [i = 1 to n]
  { for [j = 1 to n]
     { c[i,j] = 0;
       for [k = 1 to n]
         c[i,j] = c[i,j] + (a[i,k]*b[k,j]);
     }
  }
```

| Paralelización | Código | Procesos |
| --- | --- | --- |
| Por fila | `co [i = 1 to n] { for [j = 1 to n] ... }` | n |
| Por columna | `co [j = 1 to n] { for [i = 1 to n] ... }` | n |
| Por celda | `co [i = 1 to n, j = 1 to n] ...` o dos `co` anidados | n² |
| Por strips | `process worker [w = 1 to P]`, cada uno con n/P filas | P |

Si hay menos de n procesadores, se divide la matriz resultado en **strips** (grupos de filas o columnas) y se usa un worker por strip. Elegir el tamaño óptimo del strip es balancear costo de procesamiento con costo de comunicación.

```
process worker [w = 1 to P]
{ int primera = (w-1)*(n/P) + 1;
  int ultima = primera + (n/P) - 1;
  for [i = primera to ultima]
    { for [j = 1 to n]
        { c[i,j] = 0;
          for [k = 1 to n]
            c[i,j] = c[i,j] + (a[i,k]*b[k,j]);
        }
    }
}
```

## 5. Ejercicio de la diapositiva 59

El enunciado:

- a) Con P = 8 y n = 120, ¿cuántas asignaciones, sumas y productos hace cada procesador?
- b) P1 a P7 son iguales; una asignación tarda 1, una suma 2 y un producto 3. P8 es 2 veces más lento. ¿Cuánto tarda el proceso total? ¿Cuál es el speedup? ¿Qué se puede hacer para mejorarlo?

Esta es mi resolución. Cuento solo las operaciones del cálculo de `c`, sin los incrementos de los índices de los `for`.

**a)** Cada worker tiene n/P = 15 filas, o sea 15 × 120 = 1800 celdas. Por celda:

- Asignaciones: 1 (`c[i,j] = 0`) + 120 (una por iteración de k) = 121.
- Sumas: 120.
- Productos: 120.

| Operación | Por celda | Por procesador (1800 celdas) |
| --- | --- | --- |
| Asignaciones | 121 | 217.800 |
| Sumas | 120 | 216.000 |
| Productos | 120 | 216.000 |

**b)** Costo de una celda en P1 a P7: 121 × 1 + 120 × 2 + 120 × 3 = 721.

| Procesador | Tiempo |
| --- | --- |
| P1 a P7 | 1800 × 721 = 1.297.800 |
| P8 (2 veces más lento) | 2 × 1.297.800 = 2.595.600 |

El proceso total termina cuando termina el más lento: **2.595.600**.

Para el speedup, el tiempo secuencial es el de todo el cálculo en un procesador rápido: 14.400 celdas × 721 = 10.382.400.

```latex
S = \frac{10.382.400}{2.595.600} = 4
```

El speedup óptimo con esta arquitectura es 7 + 0,5 = 7,5, porque P8 tiene la mitad de potencia. La eficiencia es 4 / 7,5 ≈ 0,53.

**Cómo mejorarlo**: el problema es el desbalance de carga. P1 a P7 terminan en la mitad del tiempo y esperan a P8. Hay que repartir las filas en proporción a la potencia: si P8 recibe la mitad que los demás, 7x + x/2 = 120, entonces x = 16.

| Procesador | Filas | Tiempo |
| --- | --- | --- |
| P1 a P7 | 16 cada uno | 16 × 120 × 721 = 1.384.320 |
| P8 | 8 | 2 × 8 × 120 × 721 = 1.384.320 |

```latex
S = \frac{10.382.400}{1.384.320} = 7{,}5
```

Ahora el speedup es el óptimo y la eficiencia es 1.

## 6. Trampas y autoevaluación

### Trampas típicas

- **Usar cualquier algoritmo secuencial para el speedup.** Tiene que ser el más rápido conocido.
- **Suponer que el speedup óptimo siempre es P.** Solo si la arquitectura es homogénea.
- **Sumar los tiempos de los procesadores** para el tiempo paralelo. El tiempo paralelo es el del que termina último.
- **Confundir MIMD con SIMD.** En SIMD todos ejecutan la misma instrucción a la vez.
- **Confundir descomposición de datos con funcional.**
- **Extrapolar el rendimiento** desde pruebas con pocos procesadores o pocos datos.

### Preguntas para chequear que se entendió

1. ¿Cuáles son los cuatro criterios para clasificar arquitecturas paralelas?
2. Explicar las cuatro clases de Flynn con un ejemplo de cada una.
3. ¿Qué diferencia hay entre MPMD y SPMD?
4. ¿Qué diferencia hay entre redes de interconexión estáticas y dinámicas?
5. ¿Cuáles son los pasos fundamentales del diseño de un algoritmo paralelo?
6. ¿Qué diferencia hay entre descomposición de datos y funcional?
7. ¿Qué tres objetivos guían la aglomeración?
8. ¿Qué criterios se usan para el mapeo y por qué pueden contradecirse?
9. Definir speedup, speedup óptimo y eficiencia.
10. ¿Por qué es difícil extrapolar la escalabilidad?
11. Nombrar cinco factores que limitan el speedup.
12. ¿En qué casos master/worker se vuelve un cuello de botella?
13. ¿Por qué la cuadratura adaptiva es un ejemplo de dividir y conquistar?
14. ¿De cuántas formas se puede paralelizar la multiplicación de matrices y cuántos procesos usa cada una?

## Material

- Diapositivas: `Clases teóricas/teoria-10.pdf`. La diapositiva 2 tiene los links a los cuatro videos con audio (arquitecturas, diseño de algoritmos, métricas y paradigmas).
