# Apunte — Teoría 8: Paradigmas de interacción entre procesos

## La idea en una frase

Hay tres esquemas básicos de interacción (productor/consumidor, cliente/servidor y pares que interactúan), y combinándolos salen siete paradigmas que se repiten en los programas distribuidos. Esta clase es corta y conceptual: lo importante es reconocer cada paradigma, saber para qué sirve y dar un ejemplo.

## 1. Los siete paradigmas de un vistazo

| # | Paradigma | Idea | Ejemplo de la clase |
| --- | --- | --- | --- |
| 1 | Manager/worker | Implementación distribuida de *bag of tasks*: un manager reparte tareas a los workers | Multiplicación de matrices ralas |
| 2 | Heartbeat | Los procesos intercambian información periódicamente con send/receive | Topología de una red, grids, autómatas celulares |
| 3 | Pipeline | La información recorre una serie de procesos | Multiplicación de matrices en bloques |
| 4 | Probe/echo | Se recorre un grafo o árbol enviando *probes* y juntando *echoes* | Recorrer redes sin cantidad fija de nodos |
| 5 | Broadcast | Se difunde información a todos para lograr un conocimiento global | Semáforos distribuidos |
| 6 | Token passing | Un mensaje especial (token) da permiso o junta información global | Exclusión mutua distribuida, detección de terminación |
| 7 | Servidores replicados | Varias instancias de un servidor manejan un recurso | Filósofos distribuidos y descentralizados |

## 2. Manager/worker

El concepto de *bag of tasks* con memoria compartida es una "bolsa" de tareas independientes que comparten los workers. Cada worker saca una tarea, la ejecuta y posiblemente genera tareas nuevas que pone en la bolsa.

Con mensajes, la bolsa la implementa un proceso **manager**:

- Maneja las tareas.
- Se comunica con los workers.
- Detecta el fin de las tareas.

Es un esquema cliente/servidor: los workers son clientes que piden trabajo al manager.

Su mayor virtud es la **escalabilidad** y la facilidad para **equilibrar la carga**: un worker más rápido simplemente pide más tareas.

## 3. Heartbeat

Sirve para soluciones **iterativas** que se quieren paralelizar.

- Con un esquema *divide & conquer* se reparten los datos entre los workers; cada uno actualiza una parte.
- Los valores nuevos dependen de los que tienen los vecinos inmediatos.
- Cada paso debería ser un progreso hacia la solución.

El nombre viene de que el algoritmo se **expande** enviando información y después se **contrae** incorporando la nueva, como un latido.

```
process worker [i = 1 to numWorkers]
{ declaraciones e inicializaciones locales;
  while (no terminado)
    { send valores a los workers vecinos;
      receive valores de los workers vecinos;
      Actualizar valores locales;
    }
}
```

### Ejemplo: topología de una red

Los procesadores están conectados por canales bidireccionales. Cada uno conoce solo a sus vecinos. ¿Cómo aprende cada uno la topología completa?

- Cada procesador es un proceso `Nodo[p]`, y cada link es un canal.
- `vecinos[q]` es true si q es vecino de p.
- Hay que calcular `top`, la matriz de adyacencia: `top[p,q]` es true si p y q son vecinos.

**Versión 1: se conoce el diámetro D.** Después de r rondas, cada nodo conoce la topología a distancia r. Con D rondas conoce toda la red.

```
chan topologia[1:n] ([1:n,1:n] bool)

Process Nodo[p:1..n]
{ bool vecinos[1:n], bool nuevatop[1:n,1:n], top[1:n,1:n] = ([n*n] false);
  top[p,1..n] = vecinos;
  for (r = 0; r < D; r++)
    { for [q = 1 to n st vecinos[q]] send topologia[q](top);
      for [q = 1 to n st vecinos[q]]
        { receive topologia[p](nuevatop);
          top = top or nuevatop;
        }
    }
}
```

Problemas:

- Rara vez se conoce D.
- Hay un intercambio excesivo de mensajes: los nodos cercanos al "centro" conocen la topología antes y siguen intercambiando sin aprender nada.
- No siempre la terminación se puede determinar localmente.

**Versión 2: cada nodo detecta cuándo terminó.** La fila q de `top` tiene los vecinos de q. Un nodo hizo suficientes rondas cuando **cada fila de `top` tiene algún true**, porque ya conoce los vecinos de todos los nodos. Después hace una última ronda para pasarles la topología a sus vecinos.

```
chan topologia[1:n](emisor: int; listo: bool; top: [1:n,1:n] bool)

Process Nodo[p:1..n]
{ bool vecinos[1:n], activo[1:n] = vecinos, top[1:n,1:n] = ([n*n] false), nuevatop[1:n,1:n];
  bool qlisto, listo = false;
  int emisor;
  top[p,1..n] = vecinos;
  while (not listo)
    { for [q = 1 to n st activo[q]] send topologia[q](p, false, top);
      for [q = 1 to n st activo[q]]
        { receive topologia[p](emisor, qlisto, nuevatop);
          top = top or nuevatop;
          if (qlisto) activo[emisor] = false;
        }
      if (todas las filas de top tienen 1 entry true) listo = true;
    }
  for [q = 1 to n st activo[q]] send topologia[q](p, listo, top);
  for [q = 1 to n st activo[q]] receive topologia[p](emisor, d, nuevatop);
}
```

`activo[q]` dice si el vecino q todavía está participando. Cuando un vecino avisa que terminó (`qlisto`), se deja de intercambiar con él. Así no se le manda nada a un proceso que ya no va a recibir, y nadie se queda esperando un mensaje que no llega.

## 4. Pipeline

Un arreglo lineal de procesos filtro: cada uno recibe datos por un canal de entrada y entrega resultados por uno de salida. Los workers pueden estar en procesadores que operan en paralelo.

| Esquema | Cómo se conecta |
| --- | --- |
| Abierto | W1 recibe la entrada (INPUT) y Wn produce la salida (OUTPUT) |
| Circular | Wn se conecta con W1. Sirve para procesos iterativos o que no se resuelven en una sola pasada |
| Cerrado | Un proceso coordinador maneja la realimentación entre Wn y W1 |

Los filtros de la Teoría 5 (merge network) y la criba de Eratóstenes de la Teoría 6 son pipelines.

## 5. Probe/echo

Es el análogo concurrente de DFS para recorrer árboles y grafos. Se usa en búsquedas web, bases de datos, sistemas expertos y juegos.

- Un nodo envía un mensaje (**probe**) a sus sucesores, **en paralelo** a todos.
- Después espera el mensaje de respuesta (**echo**) de cada uno.

Es especialmente útil para recorrer redes donde no hay, o no se conoce, una cantidad fija de nodos activos, como las redes móviles.

## 6. Broadcast

En la mayoría de las LAN cada procesador se conecta directamente con los otros, y suele existir la primitiva:

```
broadcast ch(m);
```

- Los mensajes de un mismo emisor se encolan en el orden de envío.
- **Broadcast no es atómico**: mensajes enviados por A y por B pueden llegarles a otros procesos en órdenes distintos.

Sirve para diseminar información o para sincronización distribuida. El ejemplo son los **semáforos distribuidos**, que se basan en un orden total de los eventos de comunicación usando **relojes lógicos**.

## 7. Token passing

Un mensaje especial, el **token**, circula entre los procesos y se usa para:

- **Dar un permiso**: por ejemplo, exclusión mutua distribuida. Solo el que tiene el token entra a la sección crítica.
- **Juntar información global**: por ejemplo, detectar que una computación distribuida terminó.

La sección crítica aparece sobre todo en memoria compartida, pero también en programas distribuidos cuando hay un recurso que solo puede usar un proceso a la vez. Suele ser parte de un problema mayor, como mantener la consistencia de una base de datos.

| Solución para exclusión mutua distribuida | Características |
| --- | --- |
| Monitor activo que da permiso | Centralizado. Ejemplo: locks en archivos |
| Semáforos distribuidos | Usan broadcast; gran intercambio de mensajes |
| Token ring | Descentralizado y fair |

## 8. Servidores replicados

Un servidor se replica cuando hay varias instancias de un recurso: cada servidor maneja una. También sirve para darles a los clientes la sensación de un único recurso cuando en realidad hay varios.

Ejemplo con los filósofos:

| Modelo | Mozos | Con quién habla cada filósofo | ¿Los mozos se comunican? |
| --- | --- | --- | --- |
| Centralizado | 1 | Con el único mozo | No hay otros |
| Distribuido | 5, uno por tenedor | Con 2 mozos: el izquierdo y el derecho | No |
| Descentralizado | 5, uno por filósofo | Con su propio mozo | Sí, cada uno con sus 2 vecinos |

## 9. Trampas y autoevaluación

### Trampas típicas

- **Confundir pipeline con heartbeat.** En el pipeline los datos fluyen en una dirección; en heartbeat los vecinos intercambian en las dos y repiten.
- **Decir que broadcast es atómico.** Dos emisores distintos pueden llegar en órdenes distintos.
- **Confundir distribuido con descentralizado** en los filósofos. En el distribuido los mozos no se hablan; en el descentralizado sí.
- **Usar la versión de topología con D** como si D siempre se conociera.

### Preguntas para chequear que se entendió

1. ¿Cuáles son los tres esquemas básicos de interacción?
2. Nombrar los siete paradigmas y dar un ejemplo de cada uno.
3. ¿Por qué manager/worker facilita el balance de carga?
4. ¿Qué forma general tiene un worker en heartbeat?
5. En la topología de red, ¿cómo sabe un nodo que ya conoce toda la red sin conocer D?
6. ¿Para qué sirve `activo[]` en la segunda versión?
7. ¿Qué diferencia hay entre un pipeline abierto, circular y cerrado?
8. ¿Por qué probe/echo sirve para redes móviles?
9. ¿Qué dos usos tiene un token?
10. Comparar los modelos centralizado, distribuido y descentralizado de los filósofos.

## Material

- Diapositivas: `Clases teóricas/teoria-8.pdf`. La diapositiva 2 tiene el link al video con audio.
