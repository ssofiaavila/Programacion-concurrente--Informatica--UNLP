# Apunte — Práctica 3: Monitores (explicación de ejercicios)

Complementa el apunte teórico de Teoría 4 (`claude/apuntes-teoria-4.md`) con la aplicación práctica: acá está el "cómo pensar" cada ejercicio, no solo la definición. Sigue el mismo espíritu que `apuntes-practica-2.md` (semáforos).

## Reglas del enunciado de Práctica 3 (fijas para todos los ejercicios)

- Los monitores usan **signal and continue**.
- A una variable condition solo se le puede aplicar `wait`, `signal`, `signal_all`. **No hay `wait` con prioridades.**
- **No se puede usar ninguna operación que diga cuántos procesos hay encolados en una variable condition, ni si está vacía** (nada de `empty(cv)`). Sí se puede usar `empty`/`size` sobre colas de **datos** (estructuras que vos definís, no variables condition).
- Los monitores/procesos solo se comunican **por invocación a procedures**. No hay variables globales.
- Hay que **maximizar la concurrencia** y **aprovechar al máximo la exclusión mutua** que da el monitor.
- **Nada de busy waiting.**
- El tiempo se representa con `delay(...)`.

## Sintaxis básica

```
Monitor Nombre {
  // variables permanentes (solo accesibles desde adentro del monitor)
  cond vc;

  Procedure uno (params)
    { ....
      wait(vc);   // duerme al proceso en la cola de vc, libera el monitor
      ....
    }

  Procedure dos (params)
    { ....
      signal(vc);      // despierta al primero dormido en vc
      signal_all(vc);  // despierta a todos los dormidos en vc
      ....
    }
}

Process P[id: 0..N-1] {
  ....
  Nombre.uno();
  ....
}
```

- **Exclusión mutua implícita**: nunca hay dos procesos ejecutando procedures del mismo monitor al mismo tiempo. Se libera el monitor solo cuando el procedure termina o cuando el proceso hace `wait`.
- **Sincronización por condición explícita**: se hace con variables `cond`, a diferencia de la exclusión mutua que es automática.
- Un procedure puede llamar a un procedure de **otro** monitor, pero mientras esa llamada no termine, el monitor original queda ocupado (inaccesible para otros procesos).
- Cuando el monitor está libre, **todos** los procesos que llaman a algún procedure compiten por entrar; no entran en orden de llegada por sí solos — si se necesita orden, hay que construirlo con una variable condition.

## Signal and Continue: qué implica

Quien hace `signal(vc)` **sigue ejecutando** dentro del monitor hasta terminar su procedure (o hasta el siguiente `wait`). El proceso despertado **no entra de inmediato**: pasa de la cola de `vc` a competir por el acceso al monitor junto con cualquier otro proceso, y recién cuando le toca continúa después de su `wait`.

Consecuencia directa: **entre el `signal` y que el despertado retome el monitor, el estado puede cambiar** (otro proceso pudo entrar y modificar variables). Por eso:

- Al volver de un `wait`, siempre se re-chequea la condición con `while`, nunca con `if` — salvo que se use *passing the condition* (ver abajo), donde el que señala deja el estado ya actualizado para el despertado y no hace falta re-chequear.
- **Un `signal` sobre una variable condition sin nadie dormido simplemente no hace nada** (se pierde), a diferencia de un `V(sem)`, que sí queda "guardado" en el contador. Esto es la fuente de un bug clásico: si alguien hace `signal` antes de que el otro llegue a hacer su `wait`, ese aviso se pierde y el que hace `wait` después se queda dormido para siempre (deadlock). La solución típica es un booleano de estado (ver Ejercicio 5b/patrón Escritorio) que se chequea antes de decidir si hay que dormirse.

## Errores típicos y patrones (de más simple a más complejo)

### Patrón 1 — El monitor administra el acceso, no *es* el recurrso que se usa

Error típico: hacer que el trabajo pesado (usar el cajero, fotocopiar, generar un comprobante, resolver una consulta) se ejecute **dentro** del procedure del monitor. Mientras eso corre, el monitor está ocupado y **nadie más puede ni siquiera encolarse** — quedan todos compitiendo en la entrada del monitor, sin ningún orden, cuando se libera.

**Regla:** si hace falta orden de llegada o simplemente maximizar concurrencia, el monitor solo debe *otorgar y liberar el turno* (procedures cortos tipo `pasar()`/`salir()`), y el trabajo real (`UsarCajero()`, `Fotocopiar()`, `AnalizarSec()`) lo hace el **proceso**, fuera del monitor.

```
Process Persona[id: 0..N-1]
{ Recurso.pasar();
  UsarRecurso();      // fuera del monitor
  Recurso.salir();
}
```

### Patrón 2 — Recurso con estado (libre/ocupado) + cola de espera, orden de llegada

Cuando hace falta respetar el orden de llegada para usar un único recurso:

- Una variable `libre` (o un contador `esperando`) que dice el estado real del recurso.
- Una variable condition (`cond cola`) donde se duerme el que no puede pasar.
- **Nunca alcanza con mirar si la cola está vacía** para decidir si el recurso está libre: la cola vacía solo dice que nadie está *esperando*, no que el recurso esté libre.
- Al liberar, **si hay alguien esperando, no se marca "libre"**: se le pasa el turno directamente al que sigue (*passing the condition*), sin volver a poner el recurso en estado libre, porque en la práctica el "dueño" cambia pero el recurso sigue ocupado.

```
Monitor Recurso {
  bool libre = true;
  cond cola;
  int esperando = 0;

  Procedure pasar ()
  { if (not libre) { esperando++; wait(cola); }
    else libre = false;
  }

  Procedure salir ()
  { if (esperando > 0) { esperando--; signal(cola); }
    else libre = true;
  }
}
```

Con prioridad (ej. edad) en vez de FIFO: se cambia la cola simple por una **cola ordenada** (insertar al frente o al final según el criterio), y conviene usar **variables condition privadas por proceso** (`cond espera[N]`) en vez de una sola, para poder despertar puntualmente al elegido sin que otros se enteren.

### Patrón 3 — Passing the condition (por qué hay que reservar el estado antes de señalar)

Con signal and continue, si al despertar a alguien el `signal` no deja el estado ya "reservado" para ese proceso, puede colarse un proceso nuevo en el intervalo entre el `signal` y que el despertado retome el monitor.

**Mal:**
```
Procedure salir ()
{ libre = true;      // un proceso nuevo puede ver esto y pasar
  signal(cola);       // el despertado todavía no coreligió el monitor
}
```

**Bien (passing the condition):** quien señala dice "el recurso ya es tuyo" *antes* de señalar, de forma que cualquier proceso nuevo que entre vea el estado correcto:
```
Procedure salir ()
{ if (esperando > 0) { esperando--; signal(cola); }  // nunca pongo libre = true acá
  else libre = true;
}
```

Esto se generaliza a cualquier variable de estado compartido: si el que despierta a alguien es quien "actualiza la cuenta" (sumar un peso, marcar un lugar ocupado, incrementar un contador de recursos en uso), hay que hacerlo **antes** del `signal`, no dejar que lo haga el despertado al volver. Ver Ejercicio 4 (puente) más abajo para un caso completo.

### Patrón 4 — Cliente/Servidor con cola de pedidos y respuesta privada

El servidor no puede hacer `pop` sobre una cola vacía. Como no se puede usar `empty` sobre una variable condition, se usa una **variable condition + cola de datos**: el cliente hace `push` a la cola y `signal` sobre la condition; el servidor hace `wait` (con `while`, por si dos servidores compiten) hasta que haya algo, y recién ahí hace `pop`.

```
Monitor Admin {
  Cola C;
  cond hayPedido;
  cond espera[N];       // uno por cliente, para la respuesta
  text resultados[N];

  Procedure Pedido (id: in int, S: in text, R: out text)
  { push(C, (id, S));
    signal(hayPedido);
    wait(espera[id]);
    R = resultados[id];
  }

  Procedure Sig (id: out int, S: out text)
  { while (empty(C)) wait(hayPedido);   // while: puede haber más de un servidor
    pop(C, (id, S));
  }

  Procedure Resultado (id: in int, R: in text)
  { resultados[id] = R;
    signal(espera[id]);
  }
}

Process Cliente[id: 0..N-1]
{ text S, res;
  while(true) { -- generar S; Admin.Pedido(id, S, res); }
}

Process Servidor
{ text sec, res; int aux;
  while(true)
  { Admin.Sig(aux, sec);
    res = AnalizarSec(sec);   // el trabajo pesado, fuera del monitor
    Admin.Resultado(aux, res);
  }
}
```

Puntos clave:
- El trabajo (`AnalizarSec`) queda fuera del monitor, así mientras se resuelve un pedido otros clientes pueden seguir encolando los suyos (Patrón 1).
- `resultados[]` indexado por id evita que el servidor tenga que esperar a que el cliente retire su resultado antes de atender al siguiente.
- Con **2+ servidores**, `if (empty(C))` no alcanza: hay que usar `while`, porque entre que un servidor se despierta y retoma el monitor, otro servidor pudo haber vaciado la cola de nuevo. Y como puede haber más de un cliente esperando resultado a la vez, conviene usar `cond espera[N]` (una condition por cliente) en vez de una sola, para no despertar de más.

### Patrón 5 — Barrera (todos llegan, todos siguen juntos)

Un contador que se incrementa con cada llegada; los primeros N-1 se duermen, el último despierta a todos con `signal_all`.

```
Monitor Cancha {
  int cant = 0;
  cond espera;

  Procedure llegada ()
  { cant++;
    if (cant == 22) signal(inicio);   // avisa que ya están todos (opcional, ver abajo)
    wait(espera);                      // TODOS se duermen, incluso el último
  }
}
```

**Error típico:** que el último en llegar no se duerma (haga solo el `signal_all` y siga de largo). Hay que asegurarse de que el que arma la barrera espere *también* a que el evento que depende de todos (ej. el partido) termine antes de que él continúe — normalmente separando "avisar que están todos" (un `signal` a un proceso coordinador aparte) de "esperar a que termine el evento" (`wait(espera)`), y todos —incluido el último— pasan por el `wait`.

Si se necesita ejecutar algo una sola vez cuando se completa la barrera (ej. `delay` del partido), conviene un **proceso coordinador separado** que se despierta con un `signal` cuando se junta el último, hace lo que corresponda, y al final hace el `signal_all` que libera a todos los jugadores.

**Variante — condition privada por grupo:** si hay *varias* barreras corriendo en paralelo sobre el mismo monitor (ej. 4 equipos que se arman independientemente), **una sola variable condition compartida rompe el orden**: un `signal` puede despertar a alguien del grupo equivocado, que todavía no está listo. Hay que usar un array de conditions, una por grupo (`cond espera[4]`), y despertar solo a los miembros del grupo que efectivamente se completó.

## Ejercicios resueltos (enunciado propio de la Práctica 3)

### Ejercicio 4 — Puente con límite de peso, orden de llegada estricto

**Problema**: N vehículos cruzan un puente que soporta hasta 50000 kg, y debe respetarse el orden de llegada.

Ideas clave (combina Patrones 2 y 3):
- Una sola `cond espera` alcanza — como el orden es estricto, siempre hay que despertar al primero de la fila, nunca hace falta elegir.
- Se necesita una **cola de datos** con los pesos de los que esperan, en el mismo orden que la cola de la variable condition (se llenan juntas: primero `push` a la cola de datos, después `wait`).
- **El error más común**: sumar el peso del vehículo despertado a `pesoTotal` recién cuando el despertado retoma el monitor (después de su `wait`). Eso dejaría una ventana donde `pesoTotal` no refleja la realidad y un vehículo nuevo podría colarse antes de que el despertado termine de "hacerse cargo" de su lugar. La solución (passing the condition): quien libera el puente **ya suma** el peso de cada vehículo que decide despertar, antes de hacer `signal`.
- El `while` en `irse` permite despertar a **más de uno** si se liberó peso suficiente (ej. se va un camión pesado y hay dos autos livianos esperando).

```
Monitor Puente {
  const MAX = 50000;
  int pesoTotal = 0;
  cond espera;
  cola pesos;   // pesos de los que esperan, en orden de llegada
  // primero(pesos): lee el frente de la cola sin sacarlo (operación asumida)

  Procedure ingresar(peso: in int)
  { if (empty(pesos) AND (pesoTotal + peso <= MAX))
      pesoTotal = pesoTotal + peso;
    else
    { push(pesos, peso);
      wait(espera);
      // al volver, irse() ya sumó mi peso: no hay nada más que hacer acá
    }
  }

  Procedure irse(peso: in int)
  { int aux;
    pesoTotal = pesoTotal - peso;
    while (not empty(pesos) AND (pesoTotal + primero(pesos) <= MAX))
    { pesoTotal = pesoTotal + primero(pesos);  // reservo el lugar ANTES de signal
      pop(pesos, aux);
      signal(espera);
    }
  }
}

Process Vehiculo[id: 1..N]
{ int peso = ...;
  Puente.ingresar(peso);
  delay(tiempoCruce);       // cruza fuera del monitor
  Puente.irse(peso);
}
```

### Ejercicio 5a — Corralón con un único empleado (cliente/servidor con dos niveles)

**Problema**: N clientes son atendidos en orden de llegada por un empleado; cada cliente entrega una lista y espera su comprobante.

Es el Patrón 4, pero con **dos** interacciones distintas: el cliente necesita (1) esperar su turno, y (2) comunicarle sus datos al empleado y recibir el comprobante. Con un único empleado alcanza con un monitor, pero conceptualmente conviene separar las dos responsabilidades:

- El **monitor administra el turno**: solo el empleado que llama decide a quién atender (orden de llegada mediante cola + condition), nunca genera el comprobante adentro (Patrón 1).
- La **comunicación de datos** entre cliente y empleado necesita un booleano de estado (`listo`) para evitar el deadlock del `signal` perdido: si el cliente llega antes que el empleado la pida, y hace `signal` antes de que el empleado llegue a su `wait`, ese aviso se pierde. El booleano permite que el empleado, al llegar, chequee si el dato "ya está" antes de decidir si dormirse.

```
Monitor Admin {
  cola pedidos;         // listas de compra en orden de llegada
  cond esperaTurno;
  cond esperaComprobante[N];
  text comprobantes[N];

  Procedure entregarPapeles(id: in int, lista: in text)
  { push(pedidos, (id, lista));
    signal(esperaTurno);
    wait(esperaComprobante[id]);
    -- lee comprobantes[id]
  }

  Procedure pedirCliente(id: out int, lista: out text)
  { while (empty(pedidos)) wait(esperaTurno);
    pop(pedidos, (id, lista));
  }

  Procedure entregarComprobante(id: in int, comp: in text)
  { comprobantes[id] = comp;
    signal(esperaComprobante[id]);
  }
}

Process Cliente[id: 0..N-1]
{ text lista = ...;
  Admin.entregarPapeles(id, lista);
}

Process Empleado
{ int id; text lista, comp;
  while (true)
  { Admin.pedirCliente(id, lista);
    comp = generarComprobante(lista);   // fuera del monitor
    Admin.entregarComprobante(id, comp);
  }
}
```

Para el punto **b** (E empleados que no terminan): alcanza con declarar `E` procesos `Empleado` iguales, todos llamando a los mismos procedures — el `while` en `pedirCliente` ya contempla que compitan varios.

Para el punto **c** (los empleados terminan cuando se atendió a todos): hace falta que el monitor sepa cuántos clientes ya fueron atendidos en total, y que `pedirCliente` le indique al empleado que puede terminar en vez de quedarse esperando para siempre — cuidado con no dejar bloqueado a un empleado que se durmió justo antes de que se atienda al último cliente (puede necesitar un `signal_all` al momento de detectar que ya no van a llegar más pedidos).

### Ejercicio 8 — Fútbol: 4 equipos de 5, se enfrenta el primer par que se completa

**Problema**: 20 jugadores forman 4 equipos de 5. Los dos primeros equipos en completarse juegan en la cancha 1; los otros dos, en la cancha 2. Se arma una barrera de 10 (dos equipos) por cancha.

Combina el Patrón 5 (barrera) con la variante de condition privada por grupo, más una capa de emparejamiento:

- `cond jugadores[4]`: una condition por equipo, para no despertar jugadores de un equipo que todavía no está listo.
- Una cola `equiposListos` donde se anota un equipo cuando llega a 5 jugadores.
- Cuando hay 2 equipos en `equiposListos`, se sacan ambos, se les asigna cancha (`canchaAsignada[equipo]`, **antes** de despertar a nadie — mismo principio de reservar el estado que en el Patrón 3), y se despierta a los 5 de cada uno.
- El jugador que llega quinto a su equipo también puede tener que esperar (si todavía no hay rival), así que se duerme igual que sus compañeros, no sigue de largo.
- La barrera de 10 en sí (juntarse los dos equipos en la cancha y arrancar el partido) es un monitor `Cancha` aparte, igual al Patrón 5 puro.

```
Monitor Coordinador {
  int equipo[4] = ([4] 0);
  int canchaAsignada[4] = ([4] -1);
  cond jugadores[4];
  cola equiposListos;
  int canchaLibre = 1;

  Procedure avisarLlegada(nro: in int; cancha: out int)
  { int rival;
    equipo[nro] = equipo[nro] + 1;
    if (equipo[nro] < 5)
      wait(jugadores[nro]);
    else
    { push(equiposListos, nro);
      if (size(equiposListos) >= 2)
      { pop(equiposListos, nro);
        pop(equiposListos, rival);
        canchaAsignada[nro] = canchaLibre;
        canchaAsignada[rival] = canchaLibre;
        canchaLibre = canchaLibre + 1;
        for (i = 1..4) signal(jugadores[nro]);
        for (i = 1..4) signal(jugadores[rival]);
      }
      else
        wait(jugadores[nro]);   // 5to del equipo, pero sin rival todavía
    }
    cancha = canchaAsignada[nro];
  }
}

Process Persona[id: 0..19]
{ int nro, canchaNro;
  nro = DarEquipo();
  Coordinador.avisarLlegada(nro, canchaNro);
  Cancha[canchaNro].jugarPartido();
  delay(50);
}

Monitor Cancha[id: 1..2]
{ int esperando = 0;
  cond llegada;

  Procedure jugarPartido()
  { esperando = esperando + 1;
    if (esperando < 10) wait(llegada);
    else signal_all(llegada);
  }
}
```

## Checklist para encarar un ejercicio nuevo

1. **¿Qué procesos hay?** ¿Son todos simétricos (todos hacen lo mismo, ej. "N personas") o hay roles distintos (cliente/servidor, empleado/persona)?
2. **¿El monitor administra el acceso o hace el trabajo?** El trabajo pesado siempre va en el proceso, nunca en el procedure del monitor (Patrón 1).
3. **¿Hace falta orden de llegada?** Si sí, necesitás una cola de datos (para guardar lo que cada uno trae) y probablemente condition(s) que respeten ese orden.
4. **¿Una condition alcanza, o necesito una por proceso/grupo?** Si siempre hay que despertar "al primero de la fila", una sola alcanza. Si hay que despertar a alguien específico (por id, por grupo), hace falta un array de conditions.
5. **¿Quién actualiza el estado compartido al despertar a alguien?** Por signal and continue, conviene que sea **quien señala** el que deja el estado ya reservado para el despertado (passing the condition), no el despertado al volver.
6. **¿Hay más de un consumidor de la misma condition?** Si sí, `wait` va con `while`, no con `if`, porque la condición puede volver a ser falsa antes de que el despertado retome el monitor.
7. **¿Puede perderse un `signal`?** Si un proceso puede señalar antes de que el otro llegue a su `wait`, hace falta una variable de estado (booleano o contador) que se chequee antes de decidir si dormirse.
8. **¿Hay que evitar busy waiting?** Nunca usar un `if`/`while` con `skip` o reintentos; siempre `wait` sobre una condition.
9. **¿Los procesos deben terminar?** Si el enunciado lo pide, hay que asegurarse de que ningún proceso quede esperando una señal que nunca va a llegar (revisar el último caso: el último cliente, el último pedido, etc.).