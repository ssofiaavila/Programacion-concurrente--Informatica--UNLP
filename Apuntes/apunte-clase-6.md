# Apunte — Teoría 6: PMS y CSP

## La idea en una frase

En Pasaje de Mensajes Sincrónicos (PMS) el emisor se queda bloqueado hasta que el receptor recibe el mensaje. Esa es la única diferencia con PMA, y de ella salen todas las consecuencias de la clase.

La clase tiene dos partes:

1. **PMS en general**: qué cambia respecto de PMA y por qué trae menos concurrencia y más riesgo de deadlock.
2. **CSP**: el lenguaje de Hoare para PMS, con sus sentencias `!` y `?` y la comunicación guardada, que es la herramienta para esquivar esos problemas.

| | PMA | PMS |
| --- | --- | --- |
| Envío | `send` no bloqueante | `sync_send` bloqueante |
| Cola del canal | Ilimitada | A lo sumo 1 mensaje |
| Memoria | Más | Menos |
| Concurrencia | Mayor | Menor |
| Riesgo de deadlock | Menor | Mayor |
| Buffering | Implícito en el canal | Hay que programar un proceso buffer |

## 1. Qué es PMS

Los canales son de tipo *link* o punto a punto: 1 emisor y 1 receptor. El `receive` es igual que en PMA; lo que cambia es el envío.

- `send` (PMA): deja el mensaje en la cola y sigue.
- `sync_send` (PMS): deja el mensaje y **espera** hasta que el receptor lo tome.

Como el emisor no puede avanzar hasta que lo reciban, nunca hay más de un mensaje pendiente por canal. De ahí sale que PMS usa menos memoria.

El costo es la concurrencia: mientras el emisor espera, no hace nada útil.

```
chan valores(int);

Process Productor
{ int datos[n];
  for [i=0 to n-1]
  { #Hacer cálculos productor
    sync_send valores(datos[i]);   # se bloquea hasta que el consumidor reciba
  }
}

Process Consumidor
{ int resultados[n];
  for [i=0 to n-1]
  { receive valores(resultados[i]);
    #Hacer cálculos consumidor
  }
}
```

`send` y `sync_send` se parecen y a veces son intercambiables, pero la semántica es distinta. Cambiar uno por otro puede introducir un deadlock que antes no existía.

## 2. Las dos desventajas de PMS

### Menos concurrencia

Cada par send/receive se completa al ritmo del proceso más lento. El ejemplo de la clase: el productor es mucho más rápido que el consumidor en las primeras n/2 operaciones y mucho más lento en las otras n/2.

- **Con PMS**: en cada mitad, el rápido espera al lento. Si la relación es 10 a 1, el tiempo total se multiplica por 10.
- **Con PMA**: en la primera mitad el productor encola mensajes sin esperar. En la segunda, el consumidor "descuenta" tiempo consumiendo esa cola.

Para lograr el efecto de PMA usando PMS hay que interponer un proceso buffer entre los dos.

Lo mismo pasa en cliente/servidor. Un cliente que libera un recurso, o que pide escribir en un display o un archivo, no necesita respuesta y querría seguir de inmediato. Con PMS igual tiene que esperar a que el servidor reciba.

### Más probabilidad de deadlock

Todo `sync_send` necesita un `receive` que le haga *matching* en el otro proceso, y en el orden correcto. El caso típico es el intercambio de valores entre dos procesos.

Versión con deadlock: los dos envían primero.

```
chan in1(int), in2(int);

Process P1
{ int valorA = 1, valorB;
  sync_send in2(valorA);   # se bloquea esperando que P2 reciba
  receive in1(valorB);
}

Process P2
{ int valorA, valorB = 2;
  sync_send in1(valorB);   # se bloquea esperando que P1 reciba
  receive in2(valorA);
}
```

P1 espera que P2 haga `receive in2`, y P2 espera que P1 haga `receive in1`. Ninguno llega a su `receive`. Con PMA este mismo código funciona, porque los `send` no bloquean.

Versión correcta: se rompe la simetría, P2 recibe primero.

```
Process P2
{ int valorA, valorB = 2;
  receive in2(valorA);
  sync_send in1(valorB);
}
```

La regla práctica: en PMS, si un proceso arranca enviando, el otro tiene que arrancar recibiendo.

## 3. CSP: el lenguaje para PMS

CSP (Communicating Sequential Processes, Hoare 1978) introdujo dos ideas: PMS y comunicación guardada (pasaje de mensajes con *waiting* selectivo). OCCAM, ADA y SR se basan en CSP.

### Canales y sentencias

En CSP no se declaran canales globales tipo mailbox. El canal es un link directo entre dos procesos, *half-duplex* y nominado: cada proceso nombra al otro.

| Sentencia | Símbolo | Forma general | Se lee |
| --- | --- | --- | --- |
| Salida | `!` (shriek, bang) | `Destino ! port(e1, ..., en);` | "le envío a Destino" |
| Entrada | `?` (query) | `Fuente ? port(x1, ..., xn);` | "recibo de Fuente" |

- **Destino y Fuente** nombran un proceso simple o un elemento de un arreglo de procesos.
- **`Fuente[*]`** permite recibir de cualquier elemento del arreglo. Solo vale para la entrada.
- **port** es una etiqueta para distinguir clases de mensajes. Se puede omitir si hay una sola.

### Matching

Dos procesos se comunican cuando ejecutan sentencias que hacen matching: uno hace `B ! e` y el otro `A ? x`, con el mismo port y tipos compatibles. Recién ahí las dos se ejecutan, simultáneamente.

```
process A { ... B ! e; ... }
process B { ... A ? x; ... }
```

El efecto es una **asignación distribuida**: equivale a `x = e`, con `e` evaluada en A y `x` asignada en B.

### Ejemplo: filtro Copiar

Copia caracteres de Oeste a Este.

```
Process Oeste
{ char c;
  do true → Generar(c); Copiar ! (c); od
}

Process Copiar
{ char c;
  do true → Oeste ? (c); Este ! (c); od
}

Process Este
{ char c;
  do true → Copiar ? (c); Usar(c); od
}
```

### Ejemplo: servidor MCD

Calcula el máximo común divisor con el algoritmo de Euclides. Muestra para qué sirven los ports y `[*]`.

```
Process MCD
{ int id, x, y;
  do true →
     Cliente[*] ? args(id, x, y);      # recibe de cualquier cliente
     do x > y → x := x - y;
     □  x < y → y := y - x;
     od
     Cliente[id] ! resultado(x);       # responde a ese cliente
  od
}
```

El cliente `i` hace `MCD ! args(i, v1, v2);` y después `MCD ? resultado(r);`. Tiene que mandar su propio `i`, porque el servidor recibe con `[*]` y necesita saber a quién responder.

## 4. Comunicación guardada

Es el concepto central de la clase. Resuelve este problema: `?` y `!` son bloqueantes, y un proceso que se comunica con varios otros no sabe en qué orden van a querer hacerlo. Si elige mal el orden, se queda bloqueado en uno mientras otro lo está esperando.

La solución es ofrecer varias comunicaciones a la vez y ejecutar la que esté lista. Se usan los comandos guardados de Dijkstra, con una sentencia de comunicación dentro de la guarda:

```
B; C → S;
```

- **B**: condición booleana. Se puede omitir y se asume true.
- **C**: sentencia de comunicación (`?` o `!`).
- **B y C juntas** forman la guarda.
- **S**: lo que se ejecuta después de la comunicación.

### Los tres estados de una guarda

| Estado | Cuándo | Qué significa |
| --- | --- | --- |
| Éxito | B es true y C se puede ejecutar sin demora | Hay alguien del otro lado listo para hacer matching |
| Falla | B es falsa | Esa alternativa no se considera; C ni se mira |
| Se bloquea | B es true pero C no puede ejecutarse todavía | Hay que esperar a que el otro proceso llegue a su sentencia |

La diferencia entre fallar y bloquearse es la pregunta más probable de teoría sobre este tema. Fallar depende solo de B. Bloquearse depende de C.

### Cómo se ejecuta un `if`

```
if B1; comunicación1 → S1;
□  B2; comunicación2 → S2;
fi
```

1. Se evalúan las guardas.
    - Si **todas fallan**, el `if` termina sin efecto.
    - Si **al menos una tiene éxito**, se elige una de ellas en forma no determinística.
    - Si **algunas se bloquean** y ninguna tiene éxito, se espera hasta que alguna tenga éxito.
2. Se ejecuta la sentencia de comunicación de la guarda elegida.
3. Se ejecuta la sentencia `Si` correspondiente.

El `do` funciona igual, pero repite hasta que todas las guardas fallen.

## 5. Ejemplos con comunicación guardada

### Copiar, reescrito

La entrada pasa a la guarda. Hace lo mismo que la versión del punto 3.

```
Process Copiar
{ char c;
  do Oeste ? (c) → Este ! (c); od
}
```

### Copiar2: buffer de 2 caracteres

Después de recibir el primer carácter, el proceso puede recibir un segundo de Oeste o enviar el que tiene a Este. No sabe cuál va a pasar primero, así que ofrece las dos.

```
Process Copiar2
{ char c1, c2;
  Oeste ? (c1);
  do Oeste ? (c2) → Este ! (c1); c1 = c2;
  □  Este ! (c1)  → Oeste ? (c1);
  od
}
```

Al entrar al `do` siempre hay exactamente 1 carácter guardado, en `c1`.

- **Rama 1**: entra un segundo carácter (hay 2). Obligatoriamente saca `c1` y mueve `c2` a `c1`. Vuelve a haber 1.
- **Rama 2**: sale `c1` (hay 0). Obligatoriamente recibe uno nuevo. Vuelve a haber 1.

### Copiar con buffer limitado

Generaliza la idea a 80 caracteres. Acá aparecen las condiciones booleanas en las guardas.

```
Process Copiar
{ char buffer[80];
  int front = 0, rear = 0, cantidad = 0;
  do cantidad < 80; Oeste ? (buffer[rear]) → cantidad = cantidad + 1;
                                             rear = (rear + 1) MOD 80;
  □  cantidad > 0; Este ! (buffer[front])  → cantidad = cantidad - 1;
                                             front = (front + 1) MOD 80;
  od
}
```

- Si el buffer está lleno, la primera guarda **falla** y solo se puede sacar.
- Si está vacío, la segunda **falla** y solo se puede recibir.
- En el medio, las dos están habilitadas y gana la que tenga un proceso listo del otro lado.

Este proceso es justamente el "proceso buffer" que PMS necesita para imitar a PMA. Con PMA no hace falta, porque el buffering está implícito en el canal.

### Asignación de recursos

```
Process Alocador
{ int disponible = MaxUnidades;
  set unidades = valores iniciales;
  int indice, idUnidad;
  do disponible > 0; cliente[*] ? acquire(indice) →
          disponible = disponible - 1;
          remove(unidades, idUnidad);
          cliente[indice] ! reply(idUnidad);
  □  cliente[*] ? release(indice, idUnidad) →
          disponible = disponible + 1;
          insert(unidades, idUnidad);
  od
}
```

Usa un port por tipo de pedido y una rama del `do` para cada uno. Si no hay unidades, la guarda de `acquire` falla y esos clientes quedan esperando en su `!`. Por eso **no hace falta guardar los pedidos pendientes** en una cola, como sí pasaba con monitores o con PMA.

La rama de `release` no tiene condición booleana: siempre se puede devolver.

### Intercambio de valores, versión simétrica

Es el mismo problema del deadlock del punto 2, ahora resuelto sin romper la simetría.

```
Process P1
{ int valor1 = 1, valor2;
  if P2 ! (valor1) → P2 ? (valor2);
  □  P2 ? (valor2) → P2 ! (valor1);
  fi
}

Process P2
{ int valor1, valor2 = 2;
  if P1 ! (valor2) → P1 ? (valor1);
  □  P1 ? (valor1) → P1 ! (valor2);
  fi
}
```

No hay deadlock porque cada proceso ofrece las dos comunicaciones. El `!` de uno hace matching con el `?` del otro, se elige uno de los dos acoplamientos posibles y la segunda comunicación queda complementaria. Es simétrica, pero más compleja que la solución con PMA.

## 6. Criba de Eratóstenes

El problema es generar todos los primos entre 2 y n. El algoritmo secuencial toma el primer número (2), borra sus múltiplos, pasa al siguiente que quedó (3), borra los suyos, y así. Los que sobreviven son primos.

Para paralelizarlo se arma un **pipe de procesos filtro**. Cada proceso recibe un stream de números de su predecesor y le pasa un stream a su sucesor.

- El **primer número** que recibe un proceso es un primo: se lo queda en `p`.
- De ahí en más, **deja pasar** los que no son múltiplos de `p` y descarta los que sí.

```
Process Criba[1]
{ int p = 2;
  for [i = 3 to n by 2] Criba[2] ! (i);    # manda solo los impares
}

Process Criba[i = 2 to L]
{ int p, proximo;
  Criba[i-1] ? (p);                         # el primero que llega es primo
  do Criba[i-1] ? (proximo) →
       if ((proximo MOD p) <> 0) and (i < L) → Criba[i+1] ! (proximo);
  od
}
```

Seguimiento con n = 15. Criba[1] se queda con 2 y manda 3, 5, 7, 9, 11, 13, 15.

| Proceso | Se queda con p | Descarta | Deja pasar |
| --- | --- | --- | --- |
| Criba[2] | 3 | 9, 15 | 5, 7, 11, 13 |
| Criba[3] | 5 | nada | 7, 11, 13 |
| Criba[4] | 7 | nada | 11, 13 |
| Criba[5] | 11 | nada | 13 |
| Criba[6] | 13 | nada | nada |

Dos detalles que se preguntan:

- **L** (cantidad de procesos) tiene que ser suficientemente grande para garantizar que se generan todos los primos hasta n. En el seguimiento hacen falta 6.
- **Terminación**: salvo Criba[1], los procesos no terminan. Quedan bloqueados esperando un mensaje de su predecesor. Cuando el programa se detiene, los valores de `p` son los primos. Se puede arreglar con centinelas.

## 7. Ordenación de un arreglo

### Con dos procesos: compare-and-exchange

Hay n valores (n par) y dos procesos, P1 y P2, cada uno con n/2 valores ya ordenados (`a1` y `a2`). Se quiere que P1 termine con la mitad más chica y P2 con la más grande.

La idea es repetir intercambios: P1 manda su mayor, P2 manda su menor. Se sigue hasta que `a1[mayor] < a2[menor]`.

```
Process P1
{ int nuevo, a1[1:n/2]; const mayor = n/2;
  ordenar a1 en orden no decreciente
  P2 ! (a1[mayor]);
  P2 ? (nuevo);
  do a1[mayor] > nuevo →
       poner nuevo en el lugar correcto en a1, descartando el viejo a1[mayor]
       P2 ! (a1[mayor]);
       P2 ? (nuevo);
  od
}

Process P2
{ int nuevo, a2[1:n/2]; const menor = 1;
  ordenar a2 en orden no decreciente
  P1 ? (nuevo);
  P1 ! (a2[menor]);
  do a2[menor] < nuevo →
       poner nuevo en el lugar correcto en a2, descartando el viejo a2[menor]
       P1 ? (nuevo);
       P1 ! (a2[menor]);
  od
}
```

Ojo con la diapositiva 24: la última línea de P2 dice `P1 ? (a2[menor])`. Por el resto del código y por la explicación de la diapositiva 25, debería ser `P1 ! (a2[menor])`, que es como está arriba. Conviene confirmarlo con la cátedra.

La solución es **asimétrica** a propósito. Como `!` y `?` son bloqueantes, P1 hace salida y después entrada, y P2 hace entrada y después salida. Si los dos arrancaran enviando, habría deadlock.

Se puede hacer simétrica con comunicación guardada, pero es más costosa de implementar:

```
P1: ... if P2 ? (nuevo) → P2 ! (a1[mayor])
        □  P2 ! (a1[mayor]) → P2 ? (nuevo)
        fi ...
```

| Caso | Intercambios | Por qué |
| --- | --- | --- |
| Mejor | 1 par de valores | Ya estaban bien repartidos; el primer intercambio lo confirma |
| Peor | n/2 + 1 | n/2 para llevar cada valor al proceso correcto, más 1 para detectar la terminación |

### Con b procesos: odd/even exchange sort

Hay b procesos `P[1:b]`, cada uno con n/b valores. Cada uno ordena los suyos y después se aplica compare-and-exchange entre vecinos, en rondas.

- **Rondas impares**: los procesos impares hacen de P1 y los pares de P2. Parejas (1,2), (3,4), ...
- **Rondas pares**: los procesos pares hacen de P1 y los impares de P2. Parejas (2,3), (4,5), ...

Ejemplo con b = 4 y un valor por proceso:

| Ronda | P[1] | P[2] | P[3] | P[4] | Parejas que intercambian |
| --- | --- | --- | --- | --- | --- |
| 0 | 8 | 7 | 6 | 5 | estado inicial |
| 1 | 7 | 8 | 5 | 6 | (1,2) y (3,4) |
| 2 | 7 | 5 | 8 | 6 | (2,3) |
| 3 | 5 | 7 | 6 | 8 | (1,2) y (3,4) |
| 4 | 5 | 6 | 7 | 8 | (2,3) |

### Cómo saber que ya está ordenado

Un proceso solo no puede saberlo: conoce su porción y la del vecino con el que intercambió. Hay dos soluciones.

| Opción | Cómo funciona | Costo |
| --- | --- | --- |
| Coordinador separado | Después de cada ronda, cada proceso le avisa si cambió algo en su porción | 2b mensajes de overhead por ronda |
| Cantidad fija de rondas | Cada proceso ejecuta las rondas suficientes para garantizar el orden: en general, al menos b | Hasta n/b + 1 mensajes por proceso y por ronda |

Con la segunda opción el algoritmo completo requiere hasta `b² · (n/b + 1)` intercambios de mensajes. Sale de multiplicar b procesos, por b rondas, por n/b + 1 mensajes por ronda.

## 8. Trampas y autoevaluación

### Trampas típicas

- **Confundir falla con bloqueo.** Una guarda falla si B es falsa. Se bloquea si B es true pero la comunicación todavía no puede hacerse.
- **Creer que el `if` siempre espera.** Si todas las guardas fallan, termina sin hacer nada. Solo espera si hay alguna bloqueada.
- **Usar `[*]` en una salida.** Solo la entrada puede nombrar "cualquiera" (`Cliente[*] ? ...`). Para responder hay que nombrar a un proceso puntual, por eso el cliente manda su id.
- **Dos procesos que arrancan enviando.** Con PMA anda; con PMS es deadlock.
- **Decir que PMS "no tiene cola".** Lo que dice la teoría es que la cola se reduce a 1 mensaje.
- **Olvidar el proceso buffer.** En PMS el buffering no viene con el canal: hay que programarlo.

### Preguntas para chequear que se entendió

1. ¿Cuál es la única diferencia entre PMA y PMS, y qué dos desventajas trae?
2. ¿Por qué el intercambio de valores con dos `sync_send` primero produce deadlock y con `send` no?
3. ¿Qué significa que `!` y `?` producen una "asignación distribuida"?
4. ¿Para qué sirve un port? ¿Cuándo se puede omitir?
5. Dada una guarda `B; C → S`, ¿cuándo tiene éxito, cuándo falla y cuándo se bloquea?
6. ¿Cuándo termina un `do` con comunicación guardada?
7. En el Alocador, ¿por qué no hace falta guardar los pedidos `acquire` pendientes?
8. En la Criba, ¿cómo terminan los procesos y cómo se podría mejorar?
9. ¿Por qué la ordenación con dos procesos es asimétrica? ¿Cuántos intercambios hace en el mejor y en el peor caso?
10. En odd/even exchange sort, ¿por qué un proceso no puede saber solo que la lista está ordenada? ¿Qué dos soluciones hay y cuánto cuesta cada una?

## Material

- Diapositivas: `Clases teóricas/teoria-6.pdf`. La diapositiva 2 tiene los links a los dos videos con audio (PMS y CSP).
