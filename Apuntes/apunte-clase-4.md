# Apunte — Teoría 4: Monitores

## La idea en una frase

Un monitor encapsula un recurso compartido junto con las operaciones para usarlo. La exclusión mutua es **implícita** (dentro del monitor ejecuta un proceso por vez) y la sincronización por condición es **explícita**, con variables condición.

La clase tiene dos partes:

1. Cómo funcionan los monitores: notación, variables condición y disciplinas de señalización.
2. Técnicas de programación con ejemplos: semáforos, SJN, buffer limitado, lectores y escritores, reloj lógico, peluquero dormilón y scheduling de disco.

## 1. Por qué monitores

Problemas de los semáforos:

- Son variables compartidas globales a los procesos.
- Las sentencias de control de acceso quedan dispersas en el código.
- Al agregar procesos hay que verificar de nuevo que se usen bien.
- Exclusión mutua y sincronización por condición son conceptos distintos, pero se programan de forma parecida.

Los monitores son módulos con más estructura, y se pueden implementar tan eficientemente como los semáforos.

| | Semáforos | Monitores |
| --- | --- | --- |
| Exclusión mutua | Explícita, con P y V | Implícita |
| Sincronización por condición | Con P y V, igual que la exclusión mutua | Explícita, con variables condición |
| Dónde está el código de sincronización | Disperso en los procesos | Dentro del monitor |

Un programa concurrente con monitores tiene **procesos activos y monitores pasivos**. Dos procesos interactúan invocando procedures del mismo monitor.

## 2. Notación

```
monitor NombreMonitor {
  declaraciones de variables permanentes;
  código de inicialización

  procedure op1 (par. formales1)
    { cuerpo de op1 }
  ...
  procedure opn (par. formalesn)
    { cuerpo de opn }
}
```

- Solo los nombres de los procedures son visibles desde afuera. Se llaman con `NombreMonitor.opi(argumentos)`.
- Los procedures acceden solo a las variables permanentes, sus variables locales y sus parámetros.
- El programador del monitor no puede conocer de antemano el orden de las llamadas.

### Ejemplo: contador compartido

Cinco empleados hacen productos y un coordinador consulta el total cada tanto. No hay condiciones que esperar: alcanza con la exclusión mutua implícita.

```
monitor TOTAL {
  int cant = 0;

  procedure incrementar ()
    { cant = cant + 1; }

  procedure verificar (R: out int)
    { R = cant; }
}

process empleado[id: 0..4] {
  while (true)
    { ...
      TOTAL.incrementar();
      ...
    }
}

process coordinador {
  int c;
  while (true)
    { ...
      TOTAL.verificar(c);
      ...
    }
}
```

## 3. Variables condición

Se declaran con `cond cv;`. El valor de `cv` es una cola de procesos demorados, que el programador no ve directamente.

| Operación | Qué hace |
| --- | --- |
| `wait(cv)` | El proceso se demora al final de la cola de `cv` y **deja el acceso exclusivo al monitor** |
| `signal(cv)` | Despierta al proceso que está al frente de la cola, si hay alguno. El despertado ejecuta recién cuando readquiere el acceso al monitor |
| `signal_all(cv)` | Despierta a todos los demorados en `cv` y la cola queda vacía |

Operaciones adicionales que **no se usan en la práctica**:

| Operación | Qué hace |
| --- | --- |
| `empty(cv)` | Devuelve true si la cola de `cv` está vacía |
| `wait(cv, rank)` | El proceso se demora en la cola en orden ascendente según `rank` |
| `minrank(cv)` | Devuelve el mínimo ranking de demora |

### Disciplinas de señalización

Dentro del monitor ejecuta uno solo. Cuando alguien hace `signal` hay dos procesos que podrían seguir, y hay que decidir cuál.

| Disciplina | El que hace signal | El despertado |
| --- | --- | --- |
| **Signal and Continue** | Sigue usando el monitor | Pasa a competir por entrar de nuevo al monitor. Continúa en la instrucción siguiente al wait |
| Signal and Wait | Pasa a competir por entrar de nuevo al monitor | Ejecuta de inmediato, desde la instrucción siguiente al wait |

**En la materia se usa Signal and Continue.**

La consecuencia práctica: con Signal and Continue, entre que a un proceso lo despiertan y que vuelve a ejecutar, otro pudo haber entrado al monitor y cambiado el estado. Por eso muchas veces la condición se vuelve a chequear con `while` en lugar de `if`.

### Diferencia entre wait/signal y P/V

| | Monitores | Semáforos |
| --- | --- | --- |
| Demorar | `wait`: el proceso **siempre** se duerme | `P`: solo se duerme si el semáforo vale 0 |
| Despertar | `signal`: si hay dormidos despierta al primero; **si no hay, no tiene efecto** | `V`: incrementa el semáforo; el efecto queda guardado. No sigue ningún orden al despertar |

Un `signal` que no encuentra a nadie se pierde. Un `V` no.

### Ejemplo: A le pasa un valor a B

Sin variables condición hay que devolver un booleano y que el proceso reintente, lo que es busy waiting:

```
monitor Buffer {
  int dato;
  bool hayDato = false;

  procedure Enviar (D: in int; Ok: out bool)
    { Ok = not hayDato;
      if (Ok) { dato = D; hayDato = true; }
    }

  procedure Recibir (R: out int; Ok: out bool)
    { Ok = hayDato;
      if (Ok) { R = dato; hayDato = false; }
    }
}

process A {
  bool ok; int aux;
  while (true) { # genera valor a enviar en aux
      ok = false;
      while (not ok) → Buffer.Enviar(aux, ok);     # BUSY WAITING
  }
}
```

Con variables condición:

```
monitor Buffer {
  int dato;
  bool hayDato = false;
  cond P, C;

  procedure Enviar (D: in int)
    { if (hayDato) → wait(P);
      dato = D;
      hayDato = true;
      signal(C);
    }

  procedure Recibir (R: out int)
    { if (not hayDato) → wait(C);
      R = dato;
      hayDato = false;
      signal(P);
    }
}
```

Acá alcanza con `if` porque hay un solo proceso que envía y uno solo que recibe.

## 4. Técnicas de sincronización

| Técnica | Idea | Ejemplo de la clase |
| --- | --- | --- |
| Condición básica | Esperar con `while` y volver a chequear al despertar | Semáforo, buffer limitado |
| Passing the condition | El que señala deja el estado ya resuelto para el despertado | Semáforo FIFO, lectores y escritores |
| Wait con prioridad | La cola de la variable condición se ordena por un rank | SJN, reloj lógico |
| Variables condición privadas | Una variable condición por proceso, para despertar a uno puntual | SJN, reloj lógico |
| Broadcast signal | `signal_all` despierta a todos | Lectores y escritores |
| Covering condition | Despertar a todos y que cada uno chequee si le toca | Reloj lógico |
| Rendezvous | Dos procesos pasan por varias etapas de sincronización | Peluquero dormilón |

### Semáforo: condición básica

Primer intento, con `if`:

```
monitor Semaforo
{ int s = 1; cond pos;

  procedure P ()
    { if (s == 0) wait(pos);
      s = s - 1;
    };

  procedure V ()
    { s = s + 1;
      signal(pos);
    };
};
```

Está mal: el semáforo puede quedar con valor menor que 0. Con Signal and Continue, entre que V despierta a un proceso y este vuelve a ejecutar, otro puede entrar a P, ver `s == 1` y decrementar. Cuando el despertado sigue, decrementa de nuevo.

La corrección es volver a chequear:

```
  procedure P ()
    { while (s == 0) wait(pos);
      s = s - 1;
    };
```

Esta versión es correcta, pero no garantiza que los procesos pasen el P en orden de llegada: uno nuevo se le puede adelantar al despertado.

### Semáforo: passing the condition

Para respetar el orden, V no incrementa `s` si hay alguien esperando: le pasa la condición directamente al despertado.

```
monitor Semaforo
{ int s = 1; cond pos;

  procedure P ()
    { if (s == 0) wait(pos)
      else s = s - 1;
    };

  procedure V ()
    { if (empty(pos)) s = s + 1
      else signal(pos);
    };
};
```

Como `s` sigue en 0, ningún proceso nuevo puede colarse. El despertado no toca `s`: el V ya "decrementó por él".

Como `empty` no se puede usar en la práctica, se lleva la cuenta a mano:

```
monitor Semaforo
{ int s = 1, espera = 0; cond pos;

  procedure P ()
    { if (s == 0) { espera++; wait(pos); }
      else s = s - 1;
    };

  procedure V ()
    { if (espera == 0) s = s + 1
      else { espera--; signal(pos); }
    };
};
```

### Alocación SJN

Con wait con prioridad:

```
monitor Shortest_Job_Next
{ bool libre = true;
  cond turno;

  procedure request (int tiempo)
    { if (libre) libre = false;
      else wait(turno, tiempo);
    };

  procedure release ()
    { if (empty(turno)) libre = true
      else signal(turno);
    };
}
```

- El wait con prioridad ordena a los demorados por el tiempo que van a usar el recurso.
- El `wait` no va en un loop porque quien decide cuándo sigue un proceso es el que libera el recurso. Es passing the condition: `libre` queda en false.

Sin wait con prioridad, se maneja el orden a mano con una cola ordenada y **variables condición privadas**:

```
monitor Shortest_Job_Next
{ bool libre = true;
  cond turno[N];
  cola espera;

  procedure request (int id, int tiempo)
    { if (libre) libre = false
      else { insertar_ordenado(espera, id, tiempo);
             wait(turno[id]);
           };
    };

  procedure release ()
    { if (empty(espera)) libre = true
      else { sacar(espera, id);
             signal(turno[id]);
           };
    };
}
```

Acá `empty(espera)` se aplica a la cola propia, no a una variable condición, así que es válido en la práctica.

### Buffer limitado

```
monitor Buffer_Limitado
{ typeT buf[n];
  int ocupado = 0, libre = 0, cantidad = 0;
  cond not_lleno, not_vacio;

  procedure depositar (typeT datos)
    { while (cantidad == n) wait(not_lleno);
      buf[libre] = datos;
      libre = (libre + 1) mod n;
      cantidad++;
      signal(not_vacio);
    }

  procedure retirar (typeT &resultado)
    { while (cantidad == 0) wait(not_vacio);
      resultado = buf[ocupado];
      ocupado = (ocupado + 1) mod n;
      cantidad--;
      signal(not_lleno);
    }
}
```

No hacen falta mutex: la exclusión mutua la da el monitor.

### Lectores y escritores: broadcast signal

El monitor arbitra el acceso a la BD pero **no la contiene**. Los procesos avisan cuándo quieren acceder y cuándo terminaron, y por eso hay cuatro procedures.

```
monitor Controlador_RW
{ int nr = 0, nw = 0;
  cond ok_leer, ok_escribir

  procedure pedido_leer ()
    { while (nw > 0) wait(ok_leer);
      nr = nr + 1;
    }

  procedure libera_leer ()
    { nr = nr - 1;
      if (nr == 0) signal(ok_escribir);
    }

  procedure pedido_escribir ()
    { while (nr > 0 OR nw > 0) wait(ok_escribir);
      nw = nw + 1;
    }

  procedure libera_escribir ()
    { nw = nw - 1;
      signal(ok_escribir);
      signal_all(ok_leer);
    }
}
```

Si la lectura estuviera dentro del monitor, los lectores no podrían leer a la vez.

Al terminar un escritor se despierta a un escritor y a todos los lectores. Cada uno vuelve a chequear su condición en el `while`.

### Lectores y escritores: passing the condition

```
monitor Controlador_RW
{ int nr = 0, nw = 0, dr = 0, dw = 0;
  cond ok_leer, ok_escribir

  procedure pedido_leer ()
    { if (nw > 0) { dr = dr + 1; wait(ok_leer); }
      else nr = nr + 1;
    }

  procedure libera_leer ()
    { nr = nr - 1;
      if (nr == 0 and dw > 0)
         { dw = dw - 1;
           signal(ok_escribir);
           nw = nw + 1;
         }
    }

  procedure pedido_escribir ()
    { if (nr > 0 OR nw > 0) { dw = dw + 1; wait(ok_escribir); }
      else nw = nw + 1;
    }

  procedure libera_escribir ()
    { if (dw > 0)
         { dw = dw - 1;
           signal(ok_escribir);
         }
      else { nw = nw - 1;
             if (dr > 0)
                { nr = dr;
                  dr = 0;
                  signal_all(ok_leer);
                }
           }
    }
}
```

El que libera actualiza los contadores en nombre de los que despierta. Por eso los `wait` van con `if`, y el despertado no toca nada al volver. Cuando un escritor le pasa el turno a otro escritor, `nw` queda en 1 sin modificarse.

### Reloj lógico

Un timer que permite a los procesos dormirse una cantidad de unidades de tiempo. Tiene dos operaciones: `demorar(intervalo)` y `tick`, que es llamada periódicamente por un proceso de alta prioridad.

**Covering condition**: se despierta a todos y cada uno verifica si ya le toca.

```
monitor Timer
{ int hora_actual = 0;
  cond chequear;

  procedure demorar (int intervalo)
    { int hora_de_despertar;
      hora_de_despertar = hora_actual + intervalo;
      while (hora_de_despertar > hora_actual) wait(chequear);
    }

  procedure tick ()
    { hora_actual = hora_actual + 1;
      signal_all(chequear);
    }
}
```

Es ineficiente: en cada tick se despiertan todos, y la mayoría vuelve a dormirse.

**Wait con prioridad**:

```
monitor Timer
{ int hora_actual = 0;
  cond espera;

  procedure demorar (int intervalo)
    { int hora_de_despertar;
      hora_de_despertar = hora_actual + intervalo;
      wait(espera, hora_de_despertar);
    }

  procedure tick ()
    { hora_actual = hora_actual + 1;
      while (minrank(espera) <= hora_actual) signal(espera);
    }
}
```

**Variables condición privadas**:

```
monitor Timer
{ int hora_actual = 0;
  cond espera[N];
  colaOrdenada dormidos;

  procedure demorar (int intervalo, int id)
    { int hora_de_despertar;
      hora_de_despertar = hora_actual + intervalo;
      Insertar(dormidos, id, hora_de_despertar);
      wait(espera[id]);
    }

  procedure tick ()
    { int aux, idAux;
      hora_actual = hora_actual + 1;
      aux = verPrimero(dormidos);
      while (aux <= hora_actual)
         { sacar(dormidos, idAux)
           signal(espera[idAux]);
           aux = verPrimero(dormidos);
         }
    }
}
```

### Peluquero dormilón: rendezvous

Una peluquería con un peluquero y unas pocas sillas. Si no hay clientes, el peluquero duerme. Un cliente que llega lo despierta o, si está ocupado, espera en una silla. Al terminar el corte, el peluquero abre la puerta de salida y la cierra cuando el cliente se fue.

El monitor tiene tres procedures:

- `corte_de_pelo`: lo llaman los clientes; retornan después de recibir el corte.
- `proximo_cliente`: lo llama el peluquero para esperar a que un cliente se siente.
- `corte_terminado`: lo llama el peluquero para que el cliente deje la peluquería.

Peluquero y cliente atraviesan varias etapas de sincronización. Empiezan con un **rendezvous**, parecido a una barrera de dos procesos: los dos tienen que llegar antes de que cualquiera siga.

```
monitor Peluqueria {
  int peluquero = 0, silla = 0, abierto = 0;
  cond peluquero_disponible, silla_ocupada, puerta_abierta, salio_cliente;

  procedure corte_de_pelo () {
      while (peluquero == 0) wait(peluquero_disponible);
      peluquero = peluquero - 1;
      signal(silla_ocupada);
      wait(puerta_abierta);
      signal(salio_cliente);
  }

  procedure proximo_cliente () {
      peluquero = peluquero + 1;
      signal(peluquero_disponible);
      wait(silla_ocupada);
  }

  procedure corte_terminado () {
      signal(puerta_abierta);
      wait(salio_cliente);
  }
}
```

| Etapa | Quién espera | En qué variable condición | Quién lo despierta |
| --- | --- | --- | --- |
| 1 | El cliente, a que el peluquero esté libre | `peluquero_disponible` | El peluquero, en `proximo_cliente` |
| 2 | El peluquero, a que el cliente se siente | `silla_ocupada` | El cliente, en `corte_de_pelo` |
| 3 | El cliente, a que termine el corte | `puerta_abierta` | El peluquero, en `corte_terminado` |
| 4 | El peluquero, a que el cliente se vaya | `salio_cliente` | El cliente, al final de `corte_de_pelo` |

## 5. Ejemplo integrador: scheduling de disco

El tiempo de acceso a disco depende del *seek time* (mover la cabeza al cilindro), el *rotational delay* y el tiempo de transmisión. Como el seek time es mucho mayor, conviene minimizar el movimiento de la cabeza.

| Política | Cómo elige | Fair | Observación |
| --- | --- | --- | --- |
| Shortest-Seek-Time (SST) | El pedido del cilindro más cercano al actual | No | Un pedido lejano puede no atenderse nunca |
| SCAN / LOOK (ascensor) | Sirve en una dirección y después invierte | Sí | Gran varianza del tiempo de espera |
| CSCAN / CLOOK | Sirve en una sola dirección | Sí | Reduce la varianza del tiempo de espera |

### Solución 1: monitor separado

El monitor solo hace de scheduler. El proceso usuario sigue un protocolo de tres pasos:

```
Scheduler_Disco.pedir(cil)  →  accede al disco  →  Scheduler_Disco.liberar()
```

```
monitor Scheduler_Disco
{ int posicion = -1, v_actual = 0, v_proxima = 1;
  cond scan[2];

  procedure pedir (int cil)
    { if (posicion == -1) posicion = cil;
      elseif (cil > posicion) wait(scan[v_actual], cil);
      else wait(scan[v_proxima], cil);
    }

  procedure liberar ()
    { if (!empty(scan[v_actual])) posicion = minrank(scan[v_actual]);
      elseif (!empty(scan[v_proxima]))
         { v_actual :=: v_proxima;
           posicion = minrank(scan[v_actual]);
         }
      else posicion = -1;
      signal(scan[v_actual]);
    }
}
```

- `posicion` es el cilindro que se está accediendo; -1 significa disco libre.
- Hay dos colas con prioridad: los pedidos que se sirven en la pasada actual (`scan[v_actual]`) y los de la próxima (`scan[v_proxima]`).
- Un pedido a un cilindro mayor que la posición entra en la pasada actual; uno menor o igual, en la próxima.
- Al liberar se toma el de menor cilindro de la pasada actual. Si no hay, se intercambian las colas.

### Solución 2: monitor intermedio

Problemas de la solución anterior:

- El scheduler es visible para el proceso usuario: si se saca, los procesos cambian.
- Todos tienen que seguir el protocolo. Si uno no lo hace, el scheduling falla.
- Después de obtener el acceso, el proceso todavía tiene que comunicarse con el driver.

La mejora es que el monitor sea un **intermediario** entre los usuarios y el driver del disco. El usuario hace una sola llamada, `usar_disco`, y el scheduling queda transparente.

El monitor tiene tres procedures: `usar_disco` (lo llaman los usuarios) y `buscar_proximo_pedido` y `transferencia_terminada` (los llama el driver).

```
monitor Interfaz_al_disco
{ int posicion = -2, v_actual = 0, v_proxima = 1, args = 0, resultados = 0;
  cond scan[2];
  cond args_almacenados, resultados_almacenados, resultados_recuperados;
  argType area_arg; resultadoType area_resultado;

  procedure usar_disco (int cil; argType params_transferencia; resultType &params_resultado)
    { if (posicion == -1) posicion = cil;
      elseif (cil > posicion) wait(scan[v_actual], cil);
      else wait(scan[v_proxima], cil);
      area_arg = parametros_transferencia;
      args = args + 1; signal(args_almacenados);
      wait(resultados_almacenados);
      parametros_resultado = area_resultado;
      resultados = resultados - 1;
      signal(resultados_recuperados);
    }

  procedure buscar_proximo_pedido (argType &parametros_transferencia)
    { int temp;
      if (!empty(scan[v_actual])) posicion = minrank(scan[v_actual]);
      elseif (!empty(scan[v_proxima]))
         { v_actual :=: v_proxima;
           posicion = minrank(scan[v_actual]);
         }
      else posicion = -1;
      signal(scan[v_actual]);
      if (args == 0) wait(args_almacenados);
      parametros_transferencia = area_arg; args = args - 1;
    }

  procedure transferencia_terminada (resultType valores_resultado)
    { area_resultado := valores_resultado;
      resultados = resultados + 1;
      signal(resultados_almacenados);
      wait(resultados_recuperados);
    }
}
```

El usuario y el driver hacen un rendezvous en dos tiempos: el usuario deja los argumentos y espera los resultados; el driver toma los argumentos, hace la transferencia, deja los resultados y espera a que el usuario los retire.

## 6. Trampas y autoevaluación

### Trampas típicas

- **Usar `if` donde hace falta `while`.** Con Signal and Continue el estado puede cambiar entre el signal y la reanudación. El `if` solo vale si se hace passing the condition.
- **Creer que el `signal` se acumula como el `V`.** Si no hay nadie esperando, se pierde.
- **Olvidar que el `wait` libera el monitor.** Si no lo hiciera, nadie podría entrar a hacer el signal.
- **Usar `empty`, `minrank` o wait con prioridad en la práctica.** No están permitidos; se reemplazan con contadores, colas propias y variables condición privadas.
- **Poner el uso del recurso dentro del monitor** cuando varios deberían usarlo a la vez, como la lectura de la BD.
- **Agregar mutex dentro de un monitor.** La exclusión mutua ya es implícita.

### Preguntas para chequear que se entendió

1. ¿Qué diferencias hay entre monitores y semáforos?
2. ¿Qué hacen `wait`, `signal` y `signal_all`?
3. ¿Qué diferencia hay entre Signal and Continue y Signal and Wait? ¿Cuál se usa en la materia?
4. ¿Qué diferencia hay entre `wait`/`signal` y `P`/`V`?
5. ¿Por qué el semáforo implementado con `if` puede quedar con valor negativo?
6. ¿Qué es passing the condition y por qué permite usar `if`?
7. ¿Cómo se resuelve SJN sin wait con prioridad?
8. ¿Por qué el monitor de lectores y escritores no contiene la BD?
9. ¿Qué es una covering condition y por qué es ineficiente en el reloj lógico?
10. ¿Qué etapas de sincronización hay en el peluquero dormilón?
11. ¿Qué ventajas tiene el monitor intermedio sobre el monitor separado en el scheduling de disco?

## Material

- Diapositivas: `Clases teóricas/teoria-4.pdf`. La diapositiva 2 tiene los links a los dos videos con audio.
