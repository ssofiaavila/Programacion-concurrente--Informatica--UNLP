# Apunte de patrones — Monitores (Práctica 3)

Sep 28, 2026 · @Sofia Agostina Avila

Mini-catálogo para repasar rápido antes del parcial: ante un enunciado nuevo, identificás qué patrón(es) de monitores aplican y recién ahí escribís código — mismo enfoque que el apunte de semáforos (Práctica 2), adaptado a `Monitor`/`cond`/`wait`/`signal`.

## Reglas fijas de la cátedra (no negociables)

Del enunciado de Práctica 3 — cualquier solución que las viole está mal aunque la lógica de sincronización sea correcta:

- Protocolo **signal and continue**: quien hace `signal` sigue ejecutando; el despertado vuelve a competir por el monitor, no entra automáticamente.
- Sobre una variable `cond` **solo** se puede hacer `wait`, `signal`, `signal_all` — nada de `empty(cv)` ni contar cuántos procesos están dormidos en ella (para eso hay que llevar vos mismo un contador entero aparte, ver Patrón 2).
- **No existe** el `wait` con prioridad.
- No hay variables globales — todo dato compartido entre procesos vive **dentro de un monitor**, y la única forma de pasar datos entre dos monitores (o entre un proceso y un monitor) es invocando un procedure.
- Maximizar la concurrencia siempre que se pueda, y aprovechar al máximo la exclusión mutua implícita del monitor — no agregues mutex propios, ya está dado gratis por estar dentro de un `Procedure`.
- Nunca busy waiting.
- El tiempo se representa con `delay(...)`.

## Cómo usar este apunte

1. Leé el enunciado nuevo y fijate cuál fila del índice de abajo aplica (casi siempre se combinan 2 o 3 patrones).
2. Andá al patrón correspondiente, copiá el esqueleto, y adaptalo.
3. Antes de entregar, pasá el checklist y la lista de errores típicos.

## Índice rápido: señal en el enunciado → patrón

- "solo un [recurso] a la vez", "de a uno" (sin orden) → **Patrón 1: Mutex implícito puro** (nada de `cond`, el monitor solo)
- "uno a la vez, respetando el orden de llegada" → **Patrón 2: Estado + cond + contador de espera** (passing the condition básico)
- "orden de llegada + prioridad" (el más viejo, el más grave, el de mayor edad...) → **Patrón 2b: cola ordenada + variable condición privada**
- "hasta N a la vez" (recurso múltiple) → **Patrón 3: contador de recursos dentro del monitor**
- "dos partes que se comunican entre sí y una tiene que esperar a la otra" (cliente-empleado, par de procesos en rendezvous) → **Patrón 4: dos monitores que interactúan**
- "todos deben esperar a que lleguen todos" → **Patrón 5: barrera con `signal_all`**
- "hay una cola de pedidos y un servidor que los resuelve" → **Patrón 6: productor/consumidor con cola + resultado por id**
- "más de un servidor/consumidor compitiendo por la misma cola" → Patrón 6 + **Patrón 7: recheck de condición con `while`**
- "todos los procesos deben terminar" → cortar con un `for` de cantidad fija de iteraciones, igual que en semáforos → nunca un `while(true)` sin salida.

## Patrones (esqueleto mínimo)

### 1. Mutex implícito puro

Cuando lo único que pide el enunciado es "uso exclusivo, no importa el orden": no hace falta ninguna variable condición. El monitor ya te da la exclusión mutua gratis — cualquier código adentro de un `Procedure` corre atómico respecto a los demás procedures del mismo monitor.

```
Monitor Recurso {
  Procedure usar ()
  { -- lo que haya que hacer con el recurso }
}
```

Ojo: acá el `Procedure` hace TODO el trabajo (incluido lo que tarda tiempo), porque no hay forma de "salir" del monitor a mitad de camino y volver. Si el enunciado pide después respetar orden, hay que pasar al Patrón 2 — no alcanza con este.

### 2. Estado + cond + contador de espera (orden de llegada, un solo servidor)

El patrón que más se repite. Se usa cuando hay que garantizar el orden de llegada sobre un recurso de "pasar y usar, después avisar que salgo" (el cliente hace `Pasar()` / usa el recurso / `Salir()` como llamadas separadas al monitor, no un solo Procedure).

Clave: **no alcanza con reabrir el recurso poniendo `libre = true` a secas en `Salir()`** — si hay alguien dormido esperando, eso dejaría pasar a un proceso que ni siquiera estaba en la cola. Por eso hace falta:

- un booleano de estado (`libre`)
- una `cond` para dormir a los que esperan
- un contador entero propio (`esperando`), porque la cátedra prohíbe usar `empty(cv)` para saber si hay alguien dormido en la variable condición

```
Monitor Recurso {
  bool libre = true;
  cond cola;
  int esperando = 0;

  Procedure Pasar ()
  { if (not libre) { esperando++; wait(cola); }
    else libre = false;
  }

  Procedure Salir ()
  { if (esperando > 0) { esperando--; signal(cola); }
    else libre = true;
  }
}
```

Por qué importa: si no hay nadie durmiendo, un `signal` no tiene efecto (a diferencia de un `V` de semáforo, que queda "guardado" para el próximo `P`) — pero si SÍ hay alguien durmiendo, dejar `libre = true` de yapa le abriría la puerta a otro proceso que llega recién en ese instante y todavía no estaba en la cola, rompiendo el orden.

### 2b. Con prioridad / orden distinto al de llegada

Mismo patrón, pero con una cola ordenada explícita en vez de la FIFO implícita, y **variable condición privada por proceso** en vez de una `cond` compartida — porque una `cond` compartida solo garantiza despertar al primero que se durmió (orden de llegada al `wait`), y con prioridad el orden de despertar no es ese.

```
Monitor Recurso {
  bool libre = true;
  cond espera[N];          // una por proceso
  colaOrdenada fila;        // estructura propia, ordenada por prioridad
  int esperando = 0;

  Procedure Pasar (id, prioridad: in int)
  { if (not libre) { insertar(fila, id, prioridad); esperando++; wait(espera[id]); }
    else libre = false;
  }

  Procedure Salir ()
  { int idAux;
    if (esperando > 0) { esperando--; sacar(fila, idAux); signal(espera[idAux]); }
    else libre = true;
  }
}
```

### 3. Recurso múltiple (contador de N instancias)

Igual que el Patrón 2 pero con un contador en vez de un booleano — el equivalente en monitores del semáforo contador de la Práctica 2:

```
Monitor Recurso {
  int libres = N;
  cond cola;
  int esperando = 0;

  Procedure Pedir ()
  { if (libres == 0) { esperando++; wait(cola); }
    else libres--;
  }

  Procedure Devolver ()
  { if (esperando > 0) { esperando--; signal(cola); }
    else libres++;
  }
}
```

### 4. Dos monitores que interactúan (rendezvous entre dos procesos)

Cuando dos procesos de distinto tipo (cliente/empleado, jugador/árbitro) tienen que encontrarse e intercambiar datos en los dos sentidos. Necesitás un monitor por "puesto de encuentro" (uno por empleado, por ejemplo) con dos variables condición, una por cada lado.

La trampa clásica es el **deadlock por orden de llegada**: si el que llega primero hace `signal` a ciegas y se va a dormir, y el otro todavía no llegó a hacer su `wait`, esa señal se pierde (a diferencia de un semáforo, `signal` sobre una `cond` sin nadie esperando NO se guarda). Solución: un flag booleano que registra si el otro ya llegó.

```
Monitor PuestoDeEncuentro {
  cond ladoA, ladoB;
  dato compartido;
  bool listo = false;

  Procedure LadoA (in: dato)
  { compartido = dato; listo = true; signal(ladoB);
    wait(ladoA);              // espera la respuesta
  }

  Procedure LadoB (out: dato)
  { if (not listo) wait(ladoB);
    dato = compartido;
    listo = false;            // reseteo para el próximo
    signal(ladoA);
  }
}
```

Para decidir QUIÉN atiende a quién (a qué empleado va cada cliente) se necesita además un monitor "administrador" separado con una cola de servidores libres — ahí reaparece el Patrón 2/2b, con el mismo cuidado: la condición de "hay alguien libre" tiene que ser una variable propia (por ejemplo `cantLibres`), nunca directamente el tamaño de la cola de libres, porque un proceso que NO estaba dormido también puede ver esa cola y robarle el lugar al que sí esperaba en orden.

### 5. Barrera (todos esperan a que lleguen todos)

```
Monitor Barrera {
  int cant = 0;
  cond espera;

  Procedure Llegada ()
  { cant++;
    if (cant < N) wait(espera);
    else signal_all(espera);
  }
}
```

Si además hay que disparar *un único evento* cuando se completa el grupo (arrancar un partido, repartir un enunciado) y ese evento no lo puede ejecutar cualquiera de los que llegaron —porque el último en llegar, si hace `signal_all` y sigue de largo, "se va sin esperar su turno"— conviene separar en dos `cond` y sumar un procedure disparador aparte:

```
Monitor Barrera {
  int cant = 0;
  cond espera, inicio;

  Procedure Llegada ()
  { cant++;
    if (cant < N) wait(espera); else signal(inicio);
  }

  Procedure Iniciar ()
  { if (cant < N) wait(inicio); }

  Procedure Terminar ()
  { signal_all(espera); }
}
```

### 6. Productor/consumidor con cola de pedidos (cliente-servidor)

Cuando hay pedidos que se encolan y un servidor los resuelve por afuera del monitor — el trabajo pesado (resolver, analizar, calcular) **nunca** va adentro del monitor, o bloqueás a todos los demás clientes mientras se resuelve uno solo.

```
Monitor Admin {
  Cola pedidos;
  cond hayPedido;
  cond espera[N];        // una por cliente, para no perder resultados
  text resultado[N];

  Procedure Pedido (id: in int, dato: in text, res: out text)
  { push(pedidos, (id, dato));
    signal(hayPedido);
    wait(espera[id]);
    res = resultado[id];
  }

  Procedure Siguiente (id: out int, dato: out text)
  { while (empty(pedidos)) wait(hayPedido);
    pop(pedidos, (id, dato));
  }

  Procedure Resultado (id: in int, res: in text)
  { resultado[id] = res;
    signal(espera[id]);
  }
}

Process Servidor
{ while (true)
  { Admin.Siguiente(id, dato);
    res = resolver(dato);       // el trabajo pesado, AFUERA del monitor
    Admin.Resultado(id, res);
  }
}
```

Nota: acá `empty(pedidos)` sí se puede usar porque `pedidos` es una cola propia (estructura de datos dentro del monitor), no una variable condition — la prohibición de `empty()` es solo sobre `cond`.

### 7. Más de un servidor compitiendo por la misma cola: `while`, no `if`

Con un solo servidor, `if (empty(pedidos)) wait(hayPedido);` alcanza. Apenas hay **dos o más** servidores, hay que re-chequear la condición al volver del `wait`, porque entre que te despiertan y volvés a entrar al monitor, el otro servidor pudo haber entrado antes y vaciado la cola de nuevo.

Regla general (vale para cualquier monitor, no solo este ejemplo): **si más de un proceso puede competir por la misma condición al volver de un `wait`, siempre `while`.** Con `if` asumís que nada cambió entre que te despertaron y que te tocó ejecutar — válido solo cuando hay un único posible consumidor de esa señal.

## Checklist antes de escribir código

1. **¿Ya me da el monitor la exclusión mutua que necesito?** Si el enunciado solo pide "uso exclusivo sin más", no agregues mutex ni variable condición de más (Patrón 1) — no hace falta simular nada.
2. **¿Hay una condición de espera?** ¿Es un estado simple (libre/ocupado), un contador de recursos, o una cola de pedidos? → elegí Patrón 2, 3 o 6.
3. **¿Necesito recordar el orden de llegada?** Nunca alcanza con un booleano suelto — hace falta una estructura propia (contador `esperando`, cola ordenada), porque la cátedra prohíbe consultar el estado interno de una `cond`.
4. **¿Puede haber más de un proceso compitiendo por la misma condición al volver de un `wait`?** → `while`, no `if` (Patrón 7).
5. **¿Dos procesos tienen que encontrarse (rendezvous)?** Cuidado con quién llega primero: usá un flag booleano, nunca asumas que el otro ya está esperando cuando hacés `signal` (Patrón 4).
6. **¿El trabajo pesado (calcular, resolver, tardar tiempo) está afuera del monitor?** Metido adentro de un `Procedure` bloquea a todos los demás mientras dura.
7. **¿Estoy llamando a un monitor desde dentro de otro monitor?** El primero queda ocupado (nadie más entra) hasta que el segundo termine POR COMPLETO su procedure — evitalo si genera un cuello de botella que el enunciado no pide.
8. **¿Usé `empty()` o algún conteo sobre una variable `cond`?** Prohibido — reemplazalo por un contador entero propio.
9. **¿Todos los procesos que tienen que terminar, terminan?** Si el enunciado dice "todos deben terminar su ejecución", cortá con un `for` de cantidad conocida de iteraciones, no un `while(true)`.

## Errores típicos a repasar antes del parcial

Esta lista se va a ir completando a medida que corrijamos ejercicios tuyos. Por ahora, los que el propio material de la cátedra marca explícitamente como trampas:

- Marcar el recurso como libre en `Salir()` sin fijarse si hay alguien esperando — deja pasar a alguien que no estaba en la cola y rompe la exclusión mutua o el orden.
- Usar el tamaño de una cola de "recursos libres" como condición de espera directa, en vez de un contador propio — un proceso que no estaba dormido puede robarle el lugar al que sí esperaba en orden (Patrón 4).
- Hacer `signal` a ciegas antes de saber si el otro proceso ya está esperando → señal perdida → deadlock (Patrón 4, flag `listo`).
- Usar `if` en vez de `while` al volver de un `wait` cuando hay más de un consumidor posible de esa señal (Patrón 7).
- Resolver el trabajo pesado ("AnalizarSec", "Fotocopiar", etc.) dentro del monitor en vez de en el proceso — bloquea a todos los demás mientras tanto.
- Intentar usar `empty(cv)` o contar procesos dormidos en una variable condition — prohibido explícitamente por la cátedra.
- Olvidarse de resetear un flag de estado (`listo = false`) después de usarlo, dejando el monitor en un estado que rompe la próxima interacción.

## Semáforos vs Monitores: diferencias que importan para el parcial

|  | Semáforos (P2) | Monitores (P3) |
| --- | --- | --- |
| Exclusión mutua | Hay que programarla con `P(mutex)/V(mutex)` | Implícita: dos `Procedure` del mismo monitor nunca corren a la vez |
| Señal sin nadie esperando | `V` incrementa el contador igual — queda "guardada" para el próximo `P` | `signal`/`signal_all` sin nadie durmiendo en esa `cond` **se pierde** |
| "¿Hay alguien esperando?" | Se lee directamente el valor del semáforo | Prohibido consultar una `cond` — hay que llevar un contador propio |
| Volver de una espera | `P(s)` nunca necesita re-chequeo extra | Tras un `wait`, si puede haber más de un competidor, hay que re-chequear con `while` |
| Datos compartidos | Variables compartidas + semáforos, sueltos | Todo encapsulado dentro del monitor; se accede solo llamando a sus procedures |
| Reordenar el acceso | Con un proceso Coordinador centralizador (ver apunte P2) | Con estructura propia (cola/contador) dentro del mismo monitor — Coordinador aparte solo si el enunciado pide un evento disparador (Patrón 5) |

Ver el apunte de Teoría 4 para la fundamentación completa de por qué existen los monitores (motivación, modelo de las 3 regiones, Signal and Continue vs Signal and Wait).

