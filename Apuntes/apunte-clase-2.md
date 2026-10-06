# Apunte — Teoría 2: Locks y barreras

## La idea en una frase

Esta clase resuelve dos problemas usando solo variables compartidas y *busy waiting*: la **sección crítica** (que entre uno solo por vez) y la **barrera** (que nadie siga hasta que lleguen todos). Las soluciones funcionan, pero son complejas e ineficientes, y eso motiva los semáforos de la clase siguiente.

En *busy waiting* un proceso chequea repetidamente una condición hasta que sea verdadera.

- **Ventaja**: se implementa con instrucciones de cualquier procesador.
- **Desventaja**: es ineficiente en multiprogramación, porque el proceso que espera consume procesador.
- **Aceptable** si cada proceso ejecuta en su propio procesador.

## 1. El problema de la sección crítica

```
process SC[i=1 to n]
 { while (true)
    { protocolo de entrada;
      sección crítica;
      protocolo de salida;
      sección no crítica;
    }
 }
```

Hay que diseñar los protocolos de entrada y salida. Tienen que cumplir cuatro propiedades.

| Propiedad | Qué exige | Tipo |
| --- | --- | --- |
| Exclusión mutua | A lo sumo un proceso está en su SC | Seguridad |
| Ausencia de deadlock (livelock) | Si dos o más tratan de entrar, al menos uno lo logra | Seguridad |
| Ausencia de demora innecesaria | Si uno trata de entrar y los demás están en su SNC o terminaron, no se lo impide | Seguridad |
| Eventual entrada | Un proceso que intenta entrar eventualmente lo hace | Vida |

La clasificación en seguridad y vida no está en las diapositivas de esta clase; sale de las definiciones de la Teoría 1.

### Relación con el `await`

Cualquier solución al problema de la SC sirve para implementar un `await`. Si `SCEnter` y `SCExit` son los protocolos:

| Sentencia | Implementación |
| --- | --- |
| `⟨S;⟩` | `SCEnter; S; SCExit` |
| `⟨await (B) S;⟩` | `SCEnter; while (not B) { SCExit; SCEnter; } S; SCExit;` |
| `⟨await (B);⟩` con B que cumple ASV | `while (not B) skip;` |

La segunda es correcta pero ineficiente: el proceso está saliendo y entrando de la SC continuamente. Para reducir la contención de memoria se agrega una demora: `{ SCExit; Delay; SCEnter; }`.

## 2. Solución por hardware: deshabilitar interrupciones

```
process SC[i=1 to n] {
   while (true) {
       deshabilitar interrupciones;     # protocolo de entrada
       sección crítica;
       habilitar interrupciones;        # protocolo de salida
       sección no crítica;
   }
}
```

- Es correcta en un **monoprocesador**.
- Durante la SC no hay multiprogramación, lo que penaliza el rendimiento.
- **No es correcta en un multiprocesador**: deshabilitar interrupciones en un procesador no frena a los otros.

## 3. Solución de grano grueso

Para dos procesos, con una variable por proceso. El invariante buscado es `MUTEX: ¬(in1 ∧ in2)`.

```
bool in1 = false, in2 = false;

process SC1
 { while (true)
    { <await (not in2) in1 = true;>
      sección crítica;
      in1 = false;
      sección no crítica;
    }
 }

process SC2
 { while (true)
    { <await (not in1) in2 = true;>
      sección crítica;
      in2 = false;
      sección no crítica;
    }
 }
```

Sin el `await` (poniendo solo `in1 = true`) no se asegura el invariante. Con el `await`, cumple las cuatro propiedades:

- **Exclusión mutua**: por construcción.
- **Ausencia de deadlock**: para que haya deadlock los dos tendrían que estar bloqueados en la entrada, con `in1` e `in2` en true. No puede pasar: las dos son false en ese punto.
- **Ausencia de demora innecesaria**: si SC1 está fuera de su SC, `in1` es false y SC2 puede entrar.
- **Eventual entrada**: se garantiza con una política de scheduling **fuertemente fair**.

### Generalización a n procesos

Se hace un cambio de variables: `lock = in1 ∨ in2`. Con una sola variable alcanza para cualquier cantidad de procesos.

```
bool lock = false;

process SC [i=1..n]
 { while (true)
    { <await (not lock) lock = true;>
      sección crítica;
      lock = false;
      sección no crítica;
    }
 }
```

## 4. Solución de grano fino: spin locks

El objetivo es hacer atómico ese `await` con instrucciones reales del procesador: Test & Set (TS), Fetch & Add (FA) o Compare & Swap.

```
bool TS (bool ok);
{ < bool inicial = ok;
    ok = true;
    return inicial; >
}
```

TS devuelve el valor que tenía la variable y la deja en true, todo atómicamente.

```
bool lock = false;

process SC[i=1 to n]
{ while (true)
   { while (TS(lock)) skip;
     sección crítica;
     lock = false;
     sección no crítica;
   }
}
```

Si `lock` estaba en false, TS devuelve false, el `while` termina y `lock` ya quedó en true. Si estaba en true, devuelve true y el proceso sigue iterando (*spinning*).

- Cumple las cuatro propiedades si el scheduling es **fuertemente fair**.
- En la práctica una política débilmente fair es aceptable, porque rara vez todos los procesos están tratando de entrar a la vez.

### Test-and-Test-and-Set

TS escribe siempre en `lock`, aunque el valor no cambie. Para reducir la contención de memoria, primero se mira y recién después se hace TS:

```
while (lock) skip;
while (TS(lock))
    while (lock) skip;
```

La contención se reduce pero no desaparece: cuando `lock` pasa a false, posiblemente todos intenten hacer TS.

## 5. Soluciones fair

Los spin locks no controlan el orden de los procesos demorados. Alguno podría no entrar nunca si el scheduling no es fuertemente fair. Las tres soluciones que siguen necesitan solo scheduling **débilmente fair**.

| Algoritmo | Idea | Instrucciones especiales | Problema |
| --- | --- | --- | --- |
| Tie-Breaker | Se demora al último en empezar el protocolo de entrada | No | Complejo y costoso para n procesos |
| Ticket | Cada proceso saca un número y espera su turno | Sí (Fetch & Add) | Los contadores crecen sin límite |
| Bakery | Cada proceso mira los números de los demás y toma uno mayor | No | Más complejo |

### Tie-Breaker (dos procesos)

Usa una variable por proceso (`in1`, `in2`) y una variable `ultimo` que indica quién fue el último en empezar a entrar. Ese es el que se demora.

Grano grueso:

```
bool in1 = false, in2 = false;
int ultimo = 1;

process SC1 {
  while (true) {
      ultimo = 1; in1 = true;
      <await (not in2 or ultimo == 2);>
      sección crítica;
      in1 = false;
      sección no crítica;
  }
}

process SC2 {
  while (true) {
      ultimo = 2; in2 = true;
      <await (not in1 or ultimo == 1);>
      sección crítica;
      in2 = false;
      sección no crítica;
  }
}
```

Grano fino:

```
process SC1 {
  while (true) {
      in1 = true; ultimo = 1;
      while (in2 and ultimo == 1) skip;
      sección crítica;
      in1 = false;
      sección no crítica;
  }
}
```

SC2 es simétrico, con `in2 = true; ultimo = 2;` y `while (in1 and ultimo == 2) skip;`.

SC1 espera solo si SC2 también quiere entrar **y** SC1 fue el último en llegar. Si los dos llegan a la vez, `ultimo` queda con el valor del que escribió segundo, y ese es el que espera.

**Para n procesos** el protocolo de entrada es un loop de n-1 etapas. En cada etapa se usan instancias del tie-breaker de dos procesos para decidir quiénes avanzan. Como a lo sumo uno pasa todas las etapas, a lo sumo uno está en la SC.

```
int in[1:n] = ([n] 0), ultimo[1:n] = ([n] 0);

process SC[i = 1 to n] {
  while (true) {
      for [j = 1 to n] {
          # el proceso i está en la etapa j y es el último
          in[i] = j; ultimo[j] = i;
          for [k = 1 to n st i <> k] {
              # espera si k está en una etapa más alta
              # y el proceso i fue el último en entrar a la etapa j
              while (in[k] >= in[i] and ultimo[j] == i) skip;
          }
      }
      sección crítica;
      in[i] = 0;
      sección no crítica;
  }
}
```

### Ticket

Como en una panadería con números: se reparte un número y se espera a que sea el turno.

Grano grueso:

```
int numero = 1, proximo = 1, turno[1:n] = ([n] 0);

process SC [i: 1..n]
{ while (true)
   { < turno[i] = numero; numero = numero + 1; >
     < await turno[i] == proximo; >
     sección crítica;
     < proximo = proximo + 1; >
     sección no crítica;
   }
}
```

Para la primera acción atómica se usa Fetch-and-Add:

```
FA(var, incr): < temp = var; var = var + incr; return(temp) >
```

Grano fino:

```
process SC [i: 1..n]
{ while (true)
   { turno[i] = FA(numero, 1);
     while (turno[i] <> proximo) skip;
     sección crítica;
     proximo = proximo + 1;
     sección no crítica;
   }
}
```

- El `await` se implementa con busy waiting porque referencia una sola variable compartida.
- El incremento de `proximo` puede ser un load/store normal: a lo sumo un proceso está en el protocolo de salida.
- Los valores de turno son únicos, y de ahí sale la ausencia de deadlock y de demora innecesaria.
- **Problema**: `numero` y `proximo` crecen sin límite. En la práctica se resetean.
- **Problema**: si no existe FA hay que simularla con una SC, y la solución puede dejar de ser fair.

### Bakery

No usa un contador global. Cada proceso recorre los números de los demás y se asigna uno mayor. Después espera a que el suyo sea el menor de los que esperan.

Grano grueso:

```
int turno[1:n] = ([n] 0);

process SC[i = 1 to n]
{ while (true)
   { < turno[i] = max(turno[1:n]) + 1; >
     for [j = 1 to n st j <> i]
         < await (turno[j] == 0 or turno[i] < turno[j]); >
     sección crítica
     turno[i] = 0;
     sección no crítica
   }
}
```

No es implementable directamente por dos razones: calcular el máximo de n valores no es atómico, y el `await` referencia una variable compartida dos veces.

Grano fino:

```
process SC[i = 1 to n]
{ while (true)
   { turno[i] = 1;                     # indica que empezó el protocolo de entrada
     turno[i] = max(turno[1:n]) + 1;
     for [j = 1 to n st j != i]        # espera su turno
         while (turno[j] != 0) and ((turno[i], i) > (turno[j], j)) → skip;
     sección crítica
     turno[i] = 0;
     sección no crítica
   }
}
```

Como el máximo no se calcula atómicamente, dos procesos pueden quedar con el mismo número. Por eso se comparan los pares `(turno[i], i)`: si los turnos empatan, desempata el identificador del proceso.

## 6. Sincronización barrier

Una barrera es un punto de demora al que deben llegar todos los procesos antes de que cualquiera pueda continuar. En algoritmos iterativos se reutiliza en cada iteración.

| Solución | Cómo funciona | Problema |
| --- | --- | --- |
| Contador compartido | Cada uno incrementa `cantidad` y espera a que valga n | Necesita FA; no está claro cuándo resetear el contador |
| Flags y coordinador | Cada worker avisa por su flag; un coordinador los libera | Proceso extra, y su tiempo es proporcional a n |
| Árbol (*combining tree*) | Cada worker es también coordinador; arribos suben, continuar baja | Los procesos cumplen roles distintos |
| Simétrica (*butterfly*) | Barreras de dos procesos combinadas en log₂ n etapas | n debe ser potencia de 2 |

### Contador compartido

```
int cantidad = 0;

process Worker[i=1 to n]
{ while (true)
   { código para implementar la tarea i;
     FA(cantidad, 1);
     while (cantidad <> n) skip;
   }
}
```

La pregunta de la diapositiva es cuándo se reinicia `cantidad` en 0 para la siguiente iteración. No hay un buen momento: si un proceso la resetea, otro que todavía no chequeó `cantidad == n` se queda esperando para siempre.

### Flags y coordinador

Si no existe FA, se distribuye el contador en un arreglo `arribo[1..n]`. Sumar todos los elementos en cada chequeo reintroduce contención de memoria. La solución es agregar un proceso coordinador y un segundo arreglo, de modo que cada worker espere por un único valor.

```
int arribo[1:n] = ([n] 0), continuar[1:n] = ([n] 0);

process Worker[i=1 to n]
{ while (true)
   { código para implementar la tarea i;
     arribo[i] = 1;
     while (continuar[i] == 0) skip;
     continuar[i] = 0;
   }
}

process Coordinador
{ while (true)
   { for [i = 1 to n]
        { while (arribo[i] == 0) skip;
          arribo[i] = 0;
        }
     for [i = 1 to n] continuar[i] = 1;
   }
}
```

Cada flag lo pone en 1 un proceso y lo vuelve a 0 el que lo esperaba. Eso evita el problema del reseteo.

### Árbol

Para no necesitar un proceso y un procesador extra, se combinan los roles: los workers se organizan en árbol, las señales de arribo suben y las de continuar bajan. Es más eficiente para n grande.

### Barrera simétrica y butterfly

Se construye a partir de barreras simples para dos procesos:

```
W[i]:: < await (arribo[i] == 0); >
       arribo[i] = 1;
       < await (arribo[j] == 1); >
       arribo[j] = 0;

W[j]:: < await (arribo[j] == 0); >
       arribo[j] = 1;
       < await (arribo[i] == 1); >
       arribo[i] = 0;
```

Si n es potencia de 2, se combinan en una **butterfly barrier**:

- Hay log₂ n etapas, y cada worker sincroniza con uno distinto en cada etapa.
- En la etapa s, un worker sincroniza con otro a distancia 2^(s-1).
- Cuando cada worker pasó las log₂ n etapas, todos pueden seguir.

```
int E = log(N);
int arribo[1:N] = ([N] 0);

process P[i=1..N]
{ int j;
  while (true)
    { # Sección de código anterior a la barrera
      for (etapa = 1; etapa <= E; etapa++)
         { j = (i-1) XOR (1 << (etapa-1));   # calcula el proceso con el cual sincronizar
           while (arribo[i] == 1) → skip;
           arribo[i] = 1;
           while (arribo[j] == 0) → skip;
           arribo[j] = 0;
         }
      # Sección de código posterior a la barrera
    }
}
```

Con 8 workers: en la etapa 1 sincronizan los pares (1,2), (3,4), (5,6), (7,8); en la etapa 2, (1,3), (2,4), (5,7), (6,8); en la etapa 3, (1,5), (2,6), (3,7), (4,8).

Ojo con el cálculo de `j` de la diapositiva 35: como los procesos van de 1 a N, `(i-1) XOR (1 << (etapa-1))` da un valor entre 0 y N-1. Para i = 1 en la etapa 1 da 1, que es el mismo proceso. Para que coincida con los pares de arriba creo que falta sumar 1 al resultado. Conviene confirmarlo con la cátedra.

## 7. Defectos del busy waiting

- Los protocolos son **complejos**, y no hay separación clara entre las variables de sincronización y las que se usan para computar resultados.
- Es **difícil probar que son correctos**, y más cuando crece la cantidad de procesos.
- Es **ineficiente en multiprogramación**: un procesador ocupado por un proceso que hace spinning podría usarlo otro proceso.

De ahí la necesidad de herramientas para diseñar protocolos de sincronización: los semáforos.

## 8. Trampas y autoevaluación

### Trampas típicas

- **Decir que deshabilitar interrupciones sirve siempre.** Solo en monoprocesador.
- **Olvidar qué fairness necesita cada solución.** Spin locks: fuerte. Tie-breaker, ticket y bakery: débil.
- **Confundir Ticket con Bakery.** Ticket usa un contador global y necesita FA. Bakery compara contra los demás procesos y no necesita instrucciones especiales.
- **Invertir el orden en el tie-breaker de grano fino.** Primero `in1 = true`, después `ultimo = 1`.
- **Olvidar el desempate en Bakery.** Dos procesos pueden tener el mismo turno; por eso se compara el par `(turno, id)`.

### Preguntas para chequear que se entendió

1. ¿Cuáles son las cuatro propiedades que debe cumplir una solución al problema de la SC?
2. ¿Por qué deshabilitar interrupciones no es correcto en un multiprocesador?
3. ¿Qué hace exactamente Test & Set y cómo se usa para implementar un lock?
4. ¿Qué mejora Test-and-Test-and-Set respecto de TS?
5. ¿Por qué un spin lock puede dejar a un proceso sin entrar nunca?
6. ¿Cómo decide el tie-breaker quién espera cuando los dos quieren entrar?
7. ¿Qué problema tiene el algoritmo Ticket y qué instrucción necesita?
8. ¿Por qué la versión de grano grueso de Bakery no es implementable directamente?
9. ¿Qué problema tiene la barrera con contador compartido?
10. ¿Cuántas etapas tiene una butterfly barrier y con quién sincroniza cada worker en la etapa s?

## Material

- Diapositivas: `Clases teóricas/teoria-2.pdf`. La diapositiva 2 tiene el link al video con audio.
