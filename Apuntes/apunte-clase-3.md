# Apunte — Teoría 3: Semáforos

## La idea en una frase

Un semáforo es un entero no negativo con dos operaciones atómicas, P y V, que permite demorar procesos sin busy waiting. Con él se resuelven la exclusión mutua y la sincronización por condición, y la clase recorre las técnicas típicas: mutex, señalización, semáforos binarios divididos, contadores de recursos y passing the baton.

## 1. Por qué semáforos

Los defectos del busy waiting, que cierran la Teoría 2:

- Protocolos complejos, sin separación clara entre variables de sincronización y de cómputo.
- Difícil probar corrección.
- Ineficiente en multiprogramación: el que espera gasta procesador.

## 2. Qué es un semáforo

Es una instancia de un tipo de datos abstracto con solo dos operaciones atómicas. Lo describió Dijkstra en 1968.

| Operación | Qué hace | Para qué se usa |
| --- | --- | --- |
| `P(s)` | Espera a que `s > 0` y decrementa | Demorar a un proceso hasta que ocurra un evento |
| `V(s)` | Incrementa | Señalar que ocurrió un evento |

```
# Semáforo general (counting semaphore)
P(s): < await (s > 0) s = s - 1; >
V(s): < s = s + 1; >

# Semáforo binario
P(b): < await (b > 0) b = b - 1; >
V(b): < await (b < 1) b = b + 1; >
```

### Declaración

```
sem s;                   # NO: hay que inicializarlo sí o sí en la declaración
sem mutex = 1;
sem fork[5] = ([5] 1);
```

### Sobre el orden en que se despiertan

Si la demora de los P se implementa con una cola, las operaciones son fair. **En la materia no se puede suponer ese tipo de implementación**: no se puede asumir que los procesos se despiertan en orden de llegada. Cuando se necesita un orden, hay que programarlo (ver passing the baton y semáforos privados).

## 3. Sección crítica: exclusión mutua

Partiendo de la solución de grano grueso con `lock`, se cambia la variable por `free` (lo opuesto), se la representa con un entero y lo que queda es exactamente un semáforo:

```
sem free = 1;

process SC[i=1 to n]
{ while (true)
   { P(free);
     sección crítica;
     V(free);
     sección no crítica;
   }
}
```

Es mucho más simple que las soluciones con busy waiting.

La pregunta de la diapositiva: ¿y si se inicializa `free = 0`? Nadie puede pasar el primer `P` y todos quedan bloqueados para siempre.

## 4. Barreras: señalización de eventos

Un **semáforo de señalización** generalmente se inicializa en 0. Un proceso señala el evento con `V(s)` y otros lo esperan con `P(s)`.

Barrera para dos procesos:

```
sem llega1 = 0, llega2 = 0;

process Worker1
{ ...
  V(llega1); P(llega2);
  ...
}

process Worker2
{ ...
  V(llega2); P(llega1);
  ...
}
```

Cada uno avisa que llegó y después espera al otro.

La pregunta de la diapositiva: ¿qué pasa si primero hacen P y después V? Los dos se bloquean en el P esperando una señal que el otro todavía no mandó: deadlock.

## 5. Productores y consumidores: semáforos binarios divididos

Los semáforos binarios b1, ..., bn forman un **semáforo binario dividido** (SBS, *split binary semaphore*) si este es un invariante global:

```
SPLIT: 0 ≤ b1 + ... + bn ≤ 1
```

Se pueden ver como un único semáforo binario repartido en n. Sirven para exclusión mutua: cada proceso empieza con un P sobre uno y termina con un V sobre otro, y lo que queda entre el P y el V ejecuta con exclusión mutua.

Ejemplo: buffer de un solo lugar con M productores y N consumidores. Depositar y retirar deben alternarse.

```
typeT buf; sem vacio = 1, lleno = 0;

process Productor [i = 1 to M]
{ while (true)
   { ...
     producir mensaje datos
     P(vacio); buf = datos; V(lleno);        # depositar
   }
}

process Consumidor [j = 1 to N]
{ while (true)
   { P(lleno); resultado = buf; V(vacio);    # retirar
     consumir mensaje resultado
     ...
   }
}
```

`vacio` y `lleno` juntos forman un SBS: nunca suman más de 1.

## 6. Buffers limitados: contadores de recursos

En un **contador de recursos** cada semáforo cuenta las unidades libres de un recurso. Es la técnica adecuada cuando se compite por recursos de múltiples unidades.

Con un productor y un consumidor:

```
typeT buf[n]; int ocupado = 0, libre = 0;
sem vacio = n, lleno = 0;

process Productor
{ while (true)
   { producir mensaje datos
     P(vacio); buf[libre] = datos; libre = (libre+1) mod n; V(lleno);
   }
}

process Consumidor
{ while (true)
   { P(lleno); resultado = buf[ocupado]; ocupado = (ocupado+1) mod n; V(vacio);
     consumir mensaje resultado
   }
}
```

`vacio` cuenta los lugares libres y `lleno` los ocupados. Depositar y retirar se asumen atómicas porque hay un solo productor y un solo consumidor.

### Con varios productores y consumidores

Depositar y retirar pasan a ser secciones críticas. Si no se protegen, podría retirarse dos veces el mismo dato o perderse datos por sobrescritura.

```
typeT buf[n]; int ocupado = 0, libre = 0;
sem vacio = n, lleno = 0;
sem mutexD = 1, mutexR = 1;

process Productor [i = 1..M]
{ while (true)
   { producir mensaje datos
     P(vacio);
     P(mutexD); buf[libre] = datos; libre = (libre+1) mod n; V(mutexD);
     V(lleno);
   }
}

process Consumidor [i = 1..N]
{ while (true)
   { P(lleno);
     P(mutexR); resultado = buf[ocupado]; ocupado = (ocupado+1) mod n; V(mutexR);
     V(vacio);
     consumir mensaje resultado
   }
}
```

Se usan dos mutex distintos (`mutexD` y `mutexR`) porque depositar y retirar tocan variables distintas. Así un productor y un consumidor pueden trabajar a la vez.

El orden importa: primero el contador (`P(vacio)`) y después el mutex. Al revés, un productor que toma el mutex con el buffer lleno se bloquea teniéndolo tomado.

## 7. Varios procesos, varios recursos: los filósofos

Cuando cada recurso está protegido por un lock y un proceso necesita varios, puede haber deadlock si compiten por conjuntos superpuestos. Es **exclusión mutua selectiva**.

En el problema de los filósofos cada tenedor es una SC: levantar es P, bajar es V. Cada filósofo necesita el izquierdo y el derecho.

Si todos hacen exactamente lo mismo (por ejemplo, tomar primero el izquierdo), cada uno puede quedarse con un tenedor esperando el otro: espera cíclica y deadlock. La solución es que uno lo haga en el orden inverso.

```
sem tenedores[5] = {1,1,1,1,1};

process Filosofos[i = 0..3]
{ while (true)
   { P(tenedor[i]); P(tenedor[i+1]);
     comer;
     V(tenedor[i]); V(tenedor[i+1]);
   }
}

process Filosofos[4]
{ while (true)
   { P(tenedor[0]); P(tenedor[4]);
     comer;
     V(tenedor[0]); V(tenedor[4]);
   }
}
```

El filósofo 4 toma primero el tenedor 0 y después el 4. Eso rompe el ciclo.

## 8. Lectores y escritores

Dos clases de procesos comparten una base de datos. Los escritores necesitan acceso exclusivo. Los lectores pueden leer a la vez entre ellos si no hay un escritor.

También es exclusión mutua selectiva: compiten clases de procesos. Hay dos enfoques.

### Como problema de exclusión mutua

La solución más simple, con un único `sem rw = 1` y `P(rw)` / `V(rw)` en todos, no deja leer a dos lectores a la vez.

La mejora: los lectores **como grupo** bloquean a los escritores. Solo el primer lector hace `P(rw)` y solo el último hace `V(rw)`.

```
int nr = 0;          # número de lectores activos
sem rw = 1;          # bloquea el acceso a la BD
sem mutexR = 1;      # bloquea el acceso de los lectores a nr

process Lector [i = 1 to M]
{ while (true)
   { ...
     P(mutexR);
       nr = nr + 1;
       if (nr == 1) P(rw);
     V(mutexR);
     lee la BD;
     P(mutexR);
       nr = nr - 1;
       if (nr == 0) V(rw);
     V(mutexR);
   }
}

process Escritor [j = 1 to N]
{ while (true)
   { ...
     P(rw);
     escribe la BD;
     V(rw);
   }
}
```

Esta solución da **preferencia a los lectores** y no es fair: mientras sigan llegando lectores, un escritor puede no entrar nunca.

### Como problema de sincronización por condición

Se cuentan los procesos de cada clase con `nr` y `nw` y se restringen los valores:

```
int nr = 0, nw = 0;

process Lector [i = 1 to M]
{ while (true)
   { ...
     < await (nw == 0) nr = nr + 1; >
     lee la BD;
     < nr = nr - 1; >
   }
}

process Escritor [j = 1 to N]
{ while (true)
   { ...
     < await (nr == 0 and nw == 0) nw = nw + 1; >
     escribe la BD;
     < nw = nw - 1; >
   }
}
```

Los estados malos son los que tienen lectores y escritores a la vez, o más de un escritor. El problema es implementar esos `await`: las guardas se superponen (el escritor necesita `nr == 0` y `nw == 0`; el lector, solo `nw == 0`) y ningún semáforo solo puede distinguir entre ellas. Hace falta passing the baton.

## 9. Passing the baton

Es una técnica general para implementar sentencias `await` arbitrarias con semáforos.

La idea: quien está dentro de una SC tiene el *baton* (testimonio), que es el permiso para ejecutar. Al salir, se lo pasa a otro proceso. Si nadie lo espera, lo libera para el próximo que quiera entrar.

Elementos:

- **`e`**: semáforo binario, inicialmente 1. Controla la entrada a las sentencias atómicas.
- **`bj`**: un semáforo por cada guarda distinta Bj, inicialmente 0. Demora a los que esperan que Bj sea true.
- **`dj`**: un contador por cada guarda, inicialmente 0. Cuenta los demorados sobre `bj`.

`e` y los `bj` forman un SBS: a lo sumo uno vale 1, y cada camino de ejecución empieza con un P y termina con un único V.

| Sentencia | Traducción |
| --- | --- |
| `⟨Si⟩` | `P(e); Si; SIGNAL;` |
| `⟨await (Bj) Sj⟩` | `P(e); if (not Bj) { dj = dj + 1; V(e); P(bj); } Sj; SIGNAL;` |

```
SIGNAL: if (B1 and d1 > 0) { d1 = d1 - 1; V(b1) }
        □ ...
        □ (Bn and dn > 0) { dn = dn - 1; V(bn) }
        □ else V(e);
        fi
```

SIGNAL hace exactamente un V: despierta a uno cuya condición ya se cumple, o libera la entrada.

El detalle clave: el proceso despertado **no hace `P(e)` de nuevo**. Recibe el baton directamente, así nadie puede colarse entre que lo despiertan y que ejecuta.

### Lectores y escritores con passing the baton

Con los SIGNAL ya simplificados (en cada punto se sabe qué condiciones pueden ser verdaderas):

```
int nr = 0, nw = 0, dr = 0, dw = 0;
sem e = 1, r = 0, w = 0;

process Lector [i = 1 to M]
{ while (true)
   { P(e);
     if (nw > 0) { dr = dr + 1; V(e); P(r); }
     nr = nr + 1;
     if (dr > 0) { dr = dr - 1; V(r); }
     else V(e);
     lee la BD;
     P(e);
     nr = nr - 1;
     if (nr == 0 and dw > 0) { dw = dw - 1; V(w); }
     else V(e);
   }
}

process Escritor [j = 1 to N]
{ while (true)
   { P(e);
     if (nr > 0 or nw > 0) { dw = dw + 1; V(e); P(w); }
     nw = nw + 1;
     V(e);
     escribe la BD;
     P(e);
     nw = nw - 1;
     if (dr > 0) { dr = dr - 1; V(r); }
     elseif (dw > 0) { dw = dw - 1; V(w); }
     else V(e);
   }
}
```

Esta versión da preferencia a los lectores. La ventaja de la técnica es que la política se cambia tocando los `if`: por ejemplo, haciendo que un lector nuevo se demore si hay escritores esperando (`nw > 0 or dw > 0`), o que el escritor que sale despierte primero a otro escritor.

## 10. Alocación de recursos y scheduling

El problema general: decidir cuándo se le da a un proceso acceso a un recurso.

```
request(parámetros): < await (request puede ser satisfecho) tomar unidades; >
release(parámetros): < retornar unidades; >
```

Con passing the baton:

```
request(parámetros): P(e);
                     if (request no puede ser satisfecho) DELAY;
                     tomar las unidades;
                     SIGNAL;

release(parámetros): P(e);
                     retornar unidades;
                     SIGNAL;
```

### Shortest-Job-Next (SJN)

Un recurso de una sola unidad. Al liberarse, se le da al proceso demorado con el menor `tiempo`.

- **Minimiza** el tiempo promedio de ejecución.
- Es **unfair**: un proceso con tiempo largo puede no ser atendido nunca si siguen llegando procesos más cortos. Se mejora con *aging*.

Cada proceso tiene una condición de demora distinta (su posición en la lista), así que cada uno espera en su propio semáforo, `b[id]`.

Un **semáforo privado** es uno sobre el que exactamente un proceso ejecuta P. Sirve para despertar a un proceso puntual.

```
bool libre = true;
Pares = set of (int, int) = ∅;
sem e = 1, b[n] = ([n] 0);

Process Cliente [id: 1..n]
{ int sig;
  # Trabaja
  tiempo = # determina el tiempo de uso del recurso
  P(e);
  if (! libre) { insertar (tiempo, id) en Pares;
                 V(e);
                 P(b[id]);
               }
  libre = false;
  V(e);
  # USA EL RECURSO
  P(e);
  libre = true;
  if (Pares ≠ ∅) { remover el primer par (tiempo, sig) de Pares;
                   V(b[sig]);
                 }
  else V(e);
}
```

`Pares` se mantiene ordenado por tiempo, así que el primero es el de menor tiempo.

Las dos preguntas finales de la clase, con mi respuesta:

- **Generalizar a múltiples unidades**: reemplazar `libre` por un contador de unidades disponibles, y que el pedido indique cuántas necesita.
- **Respetar el orden de llegada**: usar una cola FIFO en lugar del conjunto ordenado por tiempo.

## 11. Resumen de técnicas

| Técnica | Inicialización | Para qué |
| --- | --- | --- |
| Mutex | 1 | Exclusión mutua: `P` al entrar, `V` al salir |
| Señalización | 0 | Avisar un evento: uno hace `V`, otro espera con `P` |
| Semáforo binario dividido | Suman 1 | Alternancia y exclusión mutua entre etapas |
| Contador de recursos | Cantidad de unidades | Recursos de múltiples unidades |
| Passing the baton | `e = 1`, el resto 0 | Implementar cualquier `await`, controlando el orden |
| Semáforo privado | 0 | Despertar a un proceso específico |

## 12. Trampas y autoevaluación

### Trampas típicas

- **Declarar un semáforo sin inicializar.** No está permitido.
- **Asumir que los procesos se despiertan en orden de llegada.** En la materia no se puede.
- **Tomar el mutex antes que el contador.** Puede bloquear a todos con el mutex tomado.
- **Proteger depositar y retirar con el mismo mutex** cuando no hace falta: reduce la concurrencia.
- **Hacer `P(e)` después de ser despertado en passing the baton.** El baton ya se recibió.
- **Hacer más de un V en un SIGNAL.** Rompe el SBS y la exclusión mutua.
- **Que todos los filósofos tomen los tenedores en el mismo orden.** Deadlock.

### Preguntas para chequear que se entendió

1. ¿Qué diferencia hay entre un semáforo general y uno binario?
2. ¿Qué pasa si un mutex se inicializa en 0?
3. En la barrera de dos procesos, ¿por qué el V va antes que el P?
4. ¿Qué es un semáforo binario dividido y qué garantiza?
5. ¿Por qué con varios productores hace falta `mutexD` además de `vacio`?
6. ¿Cómo se evita el deadlock en los filósofos?
7. En lectores y escritores como exclusión mutua, ¿quién hace `P(rw)` y `V(rw)`? ¿Por qué no es fair?
8. ¿Por qué los `await` de lectores y escritores no se pueden implementar con un semáforo simple?
9. Explicar el rol de `e`, `bj` y `dj` en passing the baton. ¿Qué hace SIGNAL?
10. ¿Qué es un semáforo privado y por qué SJN los necesita? ¿Por qué SJN es unfair?

## Material

- Diapositivas: `Clases teóricas/teoria-3.pdf`. La diapositiva 2 tiene el link al video con audio.
