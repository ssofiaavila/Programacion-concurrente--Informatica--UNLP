# Apunte — Práctica 2: Semáforos (explicación de ejercicios)

Basado en el material "Explicación Práctica de Semáforos - Ejercicios" (6 ejercicios resueltos paso a paso). Complementa el apunte teórico de Teoría 3 (`claude/apuntes-teoria-3.md`) con la aplicación práctica: acá está el "cómo pensar" cada ejercicio, no solo la definición.

## Cómo usar este apunte para arrancar la práctica

1. Repasá la sintaxis (abajo) hasta que la puedas escribir de memoria.
2. Leé los 6 ejercicios en orden: son progresivos, cada uno agrega una dificultad nueva sobre la anterior (grano de la SC → recurso limitado → barrera/evento → cola de pedidos → múltiples servidores → orden de llegada sin busy waiting).
3. Para cada ejercicio nuevo del enunciado de la práctica 2, hacete siempre las mismas 4 preguntas (ver "Checklist" al final) antes de escribir código.

## Puente conceptual: de `<>`/`<await>` a `P`/`V`

Antes de la sintaxis, la idea que hay que tener firme: **no son dos herramientas que compiten, son dos niveles**.

- `<S>` y `<await B; S>` son una **especificación**: "quiero que esto se comporte como atómico" o "quiero esperar a que B sea verdad y ahí ejecutar S sin interferencias". Admiten cualquier condición B y cualquier secuencia S, pero justamente por eso una máquina real no los puede ejecutar tal cual (no hay forma eficiente de esperar una condición arbitraria sin busy waiting).
- El semáforo es la **herramienta real e implementable**: un contador entero + una cola de procesos bloqueados, con solo dos operaciones. Y de hecho, por definición, `P(s)` **es** `<await (s>0) s=s-1;>` y `V(s)` **es** `<s=s+1;>` — son casos particulares de lo anterior, restringidos a que B sea siempre "un contador > 0".

O sea: `<>`/`<await>` es el lenguaje en el que pensás y diseñás la solución; P/V es el único vocabulario con el que después hay que *construirla* de verdad, porque es lo único que se puede ejecutar sin espera activa.

**Guía de traducción**, bloque por bloque de tu solución con `<>`/`<await B; S>`:

1. **`<S>` sin condición** (solo necesito que nadie más entre mientras hago S) → exclusión mutua directa: `P(mutex); S; V(mutex);` con `mutex = 1`.
2. **`<await B; S>` con B tipo "hay recursos/elementos disponibles"** (un contador que sube y baja, ej. `cant > 0`) → ese contador se convierte directamente en el semáforo: `<await(cant>0); cant-->` pasa a ser `P(cant)`, y quien genera el recurso hace `V(cant)` en vez de incrementarlo a mano.
3. **`<await B; S>` con B que NO es "contador > 0"** (una igualdad, "llegó mi turno", "todos llegaron") → no hay traducción directa: hay que inventar un semáforo nuevo (casi siempre privado por proceso, inicializado en 0) que se vuelva disponible justo cuando B se cumple. Quien antes hacía que B fuera verdadera ahora hace `V()`; quien esperaba `<await B>` ahora hace `P()` sobre ese mismo semáforo.
4. **Regla de consistencia, siempre**: todo `P` necesita, en algún camino de ejecución, un `V` que lo libere — y nunca te quedás esperando en un segundo semáforo sin haber soltado antes un mutex que ya tenías tomado (si no, deadlock).

El caso 2 es el ejercicio 4 de este apunte (`pedidos` es un contador real → semáforo directo); el caso 3 es el ejemplo del docente/alumno de práctica 1 (`Actual == id` es una igualdad → semáforo privado inventado) y el ejercicio 3 de acá (semáforos privados `espera_chicos[C]`).

## Sintaxis de semáforos

Declaración — **siempre con inicialización obligatoria**, no se puede declarar sin valor inicial:

```
sem s;              // NO, está mal
sem mutex = 1;      // OK, semáforo binario para exclusión mutua
sem espera[5] = ([5] 1);   // OK, array de 5 semáforos, todos inicializados en 1
```

Operaciones (son las únicas dos, y son atómicas):

```
P(s)  →  < await (s > 0) s = s - 1; >     // se demora si s = 0, si no, resta 1
V(s)  →  < s = s + 1; >                    // nunca se demora, solo suma 1
```

Dos usos típicos de un semáforo, que conviene distinguir siempre:

- **Semáforo de exclusión mutua (mutex)**: inicializado en 1. Rodea una sección crítica con `P(mutex) ... V(mutex)`. Garantiza que un único proceso esté adentro a la vez.
- **Semáforo de sincronización / evento / contador de recursos**: inicializado en 0 (o en la cantidad de recursos disponibles). No protege una SC simétrica: un proceso hace `V` para "avisar" o "liberar un recurso" y otro proceso hace `P` para "esperar el aviso" o "tomar el recurso". El P y el V casi siempre están en procesos distintos.

Un error clásico para detectar en un parcial: si inicializás el mutex en 0 en vez de 1, **todos los procesos quedan bloqueados en el primer P** (deadlock total desde el arranque).

## Ejercicio 1 — Exclusión mutua básica

**Problema**: C chicos sacan caramelos de una bolsa infinita, uno a la vez, y llevan la cuenta con una variable compartida `cant`.

**Idea**: `cant = cant + 1` no es atómico (se descompone en grano fino, ver apunte de variables compartidas), así que si dos chicos lo hacen a la vez hay interferencia. Se protege con `sem mutex = 1` alrededor del incremento.

**El error a evitar (el más importante del ejercicio)**: no alcanza con proteger solo el incremento de `cant`. Si "tomar caramelo" (sacar físicamente el caramelo de la bolsa) queda afuera de la sección crítica, dos chicos pueden tomar caramelo al mismo tiempo aunque el conteo esté bien. Hay que preguntarse: *¿qué operaciones tocan el recurso compartido de verdad?* Ahí, además de `cant`, el recurso compartido es "la bolsa", así que `tomar caramelo` también debe entrar en la SC.

**La corrección de más**: dentro de la SC va solo lo que necesita exclusión mutua (tomar caramelo + incrementar cant); "comer caramelo" es una acción privada de cada chico y **debe quedar afuera** de la SC, para no perder concurrencia de más (todos pueden comer al mismo tiempo, solo no pueden tomar/contar al mismo tiempo).

```
int cant = 0;
sem mutex = 1;
Process Chico[id: 0..C-1]
{ while (true)
  { P(mutex);
    -- tomar caramelo
    cant = cant + 1;
    V(mutex);
    -- comer caramelo
  }
}
```

**Regla general que deja este ejercicio**: en la SC va sólo lo estrictamente necesario con exclusión mutua; todo lo demás se saca afuera para maximizar concurrencia.

## Ejercicio 2 — Recurso limitado (control del corte, cuidado con el `while`)

**Problema**: igual que el 1, pero la bolsa tiene sólo N caramelos.

Este ejercicio es una lección sobre **dónde exactamente poner el `while` y el `P`/`V`** cuando la condición de corte (`cant < N`) también es una variable compartida que hay que chequear con exclusión mutua.

Recorrido de errores típicos (van apareciendo en este orden, es bueno entenderlos todos porque son los que un parcial suele pedir "encontrar el error"):

1. **Poner el `while (cant < N)` afuera de la SC**: falla, porque si queda un solo caramelo y dos chicos chequean la condición al mismo tiempo, ambos la ven verdadera y los dos entran a la SC (de a uno, pero los dos entran) — el segundo saca "un caramelo que no existe" y `cant` termina en N+1. Conclusión: la condición de corte también es parte de lo que hay que proteger.
2. **Meter *todo* el `while` dentro de una única sección `P(mutex) ... V(mutex)`**: ahora sí funciona para el conteo, pero un solo chico se queda con el mutex tomado durante todas sus N iteraciones — nadie más puede entrar hasta que ese chico termine de vaciar la bolsa él solo. No hay concurrencia real.
3. **Liberar el mutex después de tomar el caramelo, para comer afuera de la SC, y volver a tomarlo antes de rechequear el `while`**: acá aparece el error más sutil — si sacás el `V(mutex)` de después del incremento y ponés el próximo `P(mutex)` después de "comer caramelo" pero *dentro* del cuerpo del while, el chico que encuentra `cant == N` sale del `while` **sin haber liberado el mutex** (porque el `P(mutex)` de re-entrada nunca se ejecutó para ese último chequeo fallido). Resultado: todos los demás quedan bloqueados para siempre.
4. **Solución correcta**: el `P(mutex)` inicial va *antes* del `while` (una sola vez), y el `V(mutex)` final va *después* del `while` (una sola vez, al salir). Adentro del while se libera y se vuelve a tomar el mutex en cada vuelta, así cada chico solo mantiene el mutex durante el instante de "tomar caramelo + incrementar cant", y lo libera para comer y para que otro chico entre.

```
int cant = 0;
sem mutex = 1;
Process Chico[id: 0..C-1]
{ P(mutex);
  while (cant < N)
  { -- tomar caramelo
    cant = cant + 1;
    V(mutex);
    -- comer caramelo
    P(mutex);
  }
  V(mutex);
}
```

**Regla general que deja este ejercicio**: cuando la condición de corte de un `while` usa una variable compartida, el chequeo de la condición Y la actualización que la modifica tienen que quedar protegidos por el mismo mutex, sin agujeros — y hay que rastrear con cuidado que *toda* salida del bucle deje el mutex liberado.

## Ejercicio 3 — Barrera + evento coordinado (la abuela)

**Problema**: C chicos y una bolsa de N caramelos administrada por una abuela. Los chicos deben esperar a que **todos** hayan llegado (barrera) antes de que la abuela empiece a repartir; después la abuela llama de a uno (aleatoriamente) N veces.

Se resuelve por partes, cada una con su propio semáforo — esta es la idea más importante del ejercicio: **usar un semáforo distinto para cada tipo de espera**, no tratar de reusar uno solo para todo.

1. **Barrera entre los C chicos**: `contador` compartido, protegido con `mutex`. El último en llegar (`contador == C`) es quien "abre la barrera": hace un `V(barrera)` por cada uno de los C chicos que estaban esperando, y un `V(espera_abuela)` para despertar a la abuela (que estaba dormida en `P(espera_abuela)` inicializado en 0).
2. **Regla de oro de las barreras con semáforos**: siempre hay que liberar la SC (`V(mutex)`) *antes* de demorarse en el semáforo de la barrera (`P(barrera)`) — si no, el último en llegar no podría nunca hacer los V(barrera) porque los otros procesos seguirían bloqueados en el mutex... espera, ojo: en realidad el riesgo es al revés — si el `P(barrera)` estuviera *antes* de liberar el mutex, cada chico se bloquearía en la barrera sin soltar el mutex, y ningún otro chico (incluido el que tendría que incrementar `contador` hasta llegar a C) podría avanzar. Por eso siempre: primero salís de la SC (V(mutex)), después te demorás en la barrera (P(barrera)).
3. **Cómo la abuela llama a un chico específico**: no alcanza un solo semáforo compartido, porque un `V` sobre un semáforo compartido despertaría a "cualquiera" de los que están esperando, no al chico con el ID sorteado. Ahí aparece la técnica de **array de semáforos privados**: `sem espera_chicos[C] = ([C] 0)`. La abuela hace `V(espera_chicos[aux])` con el `id` sorteado, y cada chico se demora en `P(espera_chicos[id])`, su propio semáforo. Esta técnica de "semáforo privado por proceso" es muy reutilizable — aparece de nuevo en el ejercicio 4.
4. **Cómo la abuela sincroniza inicio y fin de cada turno**: la abuela no puede despertar al siguiente chico hasta que el anterior haya terminado de tomar su caramelo. Se agrega otro semáforo (`listo`) para que el chico avise a la abuela cuando terminó, antes de que la abuela sortee al próximo.
5. **Cómo se enteran los chicos de que no hay más caramelos**: una variable booleana `seguir`, que la abuela pone en `false` cuando se acaban los N caramelos. Como los chicos están dormidos esperando en su semáforo privado, la abuela tiene que "despertarlos a todos una vez más" (un `V(espera_chicos[aux])` final para cada uno) para que puedan salir de su `while(seguir)` al ver el nuevo valor.

**Regla general que deja este ejercicio**: cuando necesitás despertar a un proceso *específico* (no a cualquiera), usá un array de semáforos privados indexado por `id`, inicializados en 0. Cuando necesitás avisar a *todos*, hacés un `V` por cada uno en un `for`.

## Ejercicio 4 — Cliente/Servidor con cola de pedidos (orden de llegada + contador de recursos + respuesta privada)

**Problema**: N clientes mandan pedidos a 1 servidor, que los resuelve en orden de llegada y devuelve el resultado a cada cliente.

Se resuelve en capas, cada una agregando un semáforo:

1. **Mantener el orden de llegada**: se usa una `cola C` compartida (estructura FIFO). `push` y `pop` son operaciones sobre una estructura compartida, así que van protegidas con `sem mutex = 1` (exclusión mutua clásica).
2. **El problema de la cola vacía**: el servidor no puede hacer `pop` si no hay nada en la cola. La primera idea (chequear `if not empty(C)) pop(...)` dentro de la SC) es incorrecta porque si está vacía, el servidor tendría que salir de la SC y volver a preguntar en un bucle — eso es **busy waiting** (espera activa, consumiendo CPU en vano), que siempre hay que evitar cuando se pueda resolver con un semáforo.
3. **La solución**: un semáforo `pedidos = 0` que funciona como **contador de recursos disponibles** (acá "recurso" = "pedido esperando ser atendido"). Cada cliente, después de encolar su pedido, hace `V(pedidos)`. El servidor, antes de tocar la cola, hace `P(pedidos)` — si no hay pedidos, se bloquea ahí (sin gastar CPU) hasta que alguien haga el `V`.
4. **Cómo devolver el resultado al cliente correcto**: acá se repite exactamente la técnica del ejercicio 3 — un array de semáforos privados `espera[N] = ([N] 0)` y un array `resultados[N]`. El servidor escribe `resultados[aux]` y hace `V(espera[aux])`; el cliente que generó ese pedido se demora en `P(espera[id])` y al despertar lee `resultados[id]`.

```
sem mutex = 1, pedidos = 0, espera[N] = ([N] 0);
int resultados[N];
cola C;

Process Cliente[id: 0..N-1]
{ secuencia S;
  while (true)
  { -- generar secuencia S
    P(mutex); push(C, (id, S)); V(mutex);
    V(pedidos);
    P(espera[id]);
    -- ver resultado en resultados[id]
  }
}

Process Servidor
{ secuencia sec; int aux;
  while (true)
  { P(pedidos);
    P(mutex); pop(C, (aux, sec)); V(mutex);
    resultados[aux] = resolver(sec);
    V(espera[aux]);
  }
}
```

**Regla general que deja este ejercicio**: es el patrón "productor/consumidor con contador de recursos" aplicado a pedidos, combinado con "semáforo privado por proceso" para la respuesta. Este esqueleto (cola + mutex + semáforo contador + semáforos privados de respuesta) es probablemente el patrón más reusable de toda la práctica — aparece disfrazado en muchos ejercicios de exámenes.

## Ejercicio 5 — Extensión a 2 servidores con turnos (agregar un proceso Reloj)

**Problema**: mismo esquema del ejercicio 4, pero con 2 servidores que se turnan cada 5 horas (solo uno atiende a la vez).

**Idea clave**: los clientes **no se tocan** — no les importa quién los atiende, así que la solución del ejercicio 4 para `Process Cliente` queda igual. Lo que cambia es el servidor, y se agrega un proceso nuevo, `Process Reloj`, cuyo único trabajo es avisar cuándo se cumplieron las 5 horas.

Piezas nuevas:

- `sem turno[2] = (1, 0)`: cada servidor tiene su propio semáforo de turno. El servidor 0 arranca "habilitado" (turno[0]=1) y el servidor 1 arranca dormido (turno[1]=0). Es el mismo patrón de "semáforo privado por proceso" otra vez, pero acá para indicar *a quién le toca trabajar*, no para pedidos.
- `bool finTiempo`: la avisa el Reloj cuando pasaron las 5 horas.
- `sem inicio = 0`: el servidor que arranca a trabajar avisa al Reloj (`V(inicio)`) para que el Reloj recién ahí empiece a contar las 5 horas de esa tanda (si no, el reloj podría empezar a contar antes de que el servidor efectivamente arranque).
- El servidor activo espera en el **mismo** semáforo `pedidos` tanto los pedidos de clientes como el aviso de fin de turno — el reloj, al cumplirse las 5 horas, también hace `V(pedidos)` para "despertar" al servidor y que este chequee `finTiempo`. Así el servidor no necesita esperar en dos lugares distintos.
- Cuando el servidor activo detecta `finTiempo == true`, sale de su bucle interno y hace `V(turno[1-id])` para habilitar al otro servidor, y él mismo vuelve a demorarse en `P(turno[id])` esperando su próximo turno.

**Regla general que deja este ejercicio**: cuando agregás una restricción temporal/de turnos a una solución que ya funciona, conviene: (a) no tocar los procesos que no están involucrados en la restricción nueva (los clientes), y (b) meter la restricción como un proceso más (el Reloj) que se comunica con el resto solo a través de semáforos y variables, igual que cualquier otro proceso.

## Ejercicio 6 — Recurso único con orden de llegada, sin busy waiting

**Problema**: 30 escaladores deben usar un único paso, de a uno, en el orden en que llegaron. Solo hay procesos Escalador (no hay coordinador aparte).

Este ejercicio junta ideas de los anteriores (cola para el orden + evitar busy waiting) pero con la vuelta de tuerca de que el "servidor" no existe como proceso separado: son los mismos escaladores los que se coordinan entre sí.

Errores típicos, en orden (mismo estilo pedagógico que el ejercicio 2):

1. **Encolarse solo si la cola no está vacía, y entrar directo si está vacía**: el chequeo `if (not empty(C))` para decidir si hay que esperar, dentro de una sección protegida, tiene un defecto de fondo: que la cola esté vacía **no significa que el paso esté libre**, solo significa que nadie está *esperando*. Puede haber alguien usando el paso en ese momento sin que haya cola.
2. **Agregar una variable `libre` para llevar el estado real del recurso**, pero sin protegerla: si dos escaladores llegan casi al mismo tiempo y los dos leen `libre == true` antes de que ninguno la ponga en `false`, los dos creen que tienen vía libre → interferencia clásica sobre una variable compartida (exactamente el tema de variables compartidas de la práctica 1).
3. **Solución correcta**: proteger con el mismo `mutex` tanto la variable `libre` como la cola `C`, porque están relacionadas (dos formas de describir el mismo estado del recurso). Cada escalador, al llegar, entra a la SC: si `libre` está en true la toma para sí (`libre = false`) y sigue; si no, se encola y se demora en su semáforo privado `espera[id]` (de nuevo, el patrón de semáforo privado). Al liberar el paso, vuelve a entrar a la SC: si la cola está vacía deja `libre = true`; si no, saca al primero de la cola y le hace `V(espera[aux])` — le "pasa la posta" directamente a él, sin volver a poner `libre` en true (porque el recurso sigue "ocupado", solo cambia de dueño).

```
cola C;
sem mutex = 1, espera[30] = ([30] 0);
boolean libre = true;

Process Escalador[id: 0..29]
{ int aux;
  -- llega al paso
  P(mutex);
  if (libre) { libre = false; V(mutex); }
  else { push(C, id); V(mutex); P(espera[id]); }
  // Usa el paso con Exclusión Mutua
  P(mutex);
  if (empty(C)) libre = true;
  else { pop(C, aux); V(espera[aux]); }
  V(mutex);
}
```

**Regla general que deja este ejercicio**: cuando el "recurso" tiene un estado propio (libre/ocupado) además de una cola de espera, ambos se protegen con el mismo mutex porque son parte del mismo estado compartido. Y la liberación nunca debe generar busy waiting: si hay alguien esperando, se le pasa el control directamente a él por su semáforo privado, nunca se lo deja "reintentando".

## Ejercicios del enunciado de Práctica 2 (resueltos en tutoría)

Estos son ejercicios del enunciado propio de la práctica (no de la PDF explicativa de arriba). Numeración según el enunciado.

### Ejercicio 10 — Coordinador con orden de llegada estricto + concurrencia limitada por tipo

**Problema**: llegan camiones de trigo (T) y de maíz (M) a una cerealera. Se pueden descargar hasta 7 camiones a la vez, pero como máximo 5 de cada tipo. En el ítem (a) además deben ser atendidos respetando el orden de llegada.

**(b) — sin restricción de orden**, cada camión pide sus propios permisos, respetando primero el semáforo específico (`trigo`/`maiz`) y después el general (`total`) — mismo motivo que en el ejercicio 4 (BD con altas/bajas): pedir primero lo específico evita que un camión bloqueado en `total` le gane el lugar a otro de distinto tipo cuando en realidad hay cupo de sobra en el general.

```
sem total = 7, trigo = 5, maiz = 5;

process CamionTrigo[i: 1..T]
{ P(trigo); P(total);
  descargar();
  V(total); V(trigo);
}

process CamionMaiz[i: 1..M]
{ P(maiz); P(total);
  descargar();
  V(total); V(maiz);
}
```

**(a) — con orden de llegada estricto**: acá no alcanza con que cada camión pida sus propios semáforos, porque un semáforo no garantiza que despierte en orden FIFO a quien esperó primero. La solución es agregar un proceso **Coordinador** que es el único que pide `trigo`/`maiz`/`total`, uno por uno, en el orden exacto en que fueron llegando — como es un único proceso secuencial, si se bloquea pidiendo recursos para el camión k, no puede "saltarse" al k+1: el orden queda garantizado por la propia estructura del proceso, sin semáforos extra de por medio.

```
sem mutexC = 1, pedidos = 0;
cola turnos;                          // guarda pares (tipo, id)
sem camionesTrigo[T] = ([T] 0);
sem camionesMaiz[M]  = ([M] 0);
sem total = 7, trigo = 5, maiz = 5;

process CamionTrigo[i: 1..T]
{ P(mutexC);
    turnos.push((TRIGO, i));
  V(mutexC);
  V(pedidos);

  P(camionesTrigo[i]);      // espera a que el Coordinador confirme el paso

  descargaCamion();

  V(total);
  V(trigo);
}

process CamionMaiz[j: 1..M]
{ P(mutexC);
    turnos.push((MAIZ, j));
  V(mutexC);
  V(pedidos);

  P(camionesMaiz[j]);

  descargaCamion();

  V(total);
  V(maiz);
}

process Coordinador
{ tipo tipo; int id;
  for (k = 1; k <= T+M; k++)
  { P(pedidos);
    P(mutexC);
      (tipo, id) = turnos.pop();
    V(mutexC);

    if (tipo == TRIGO) P(trigo); else P(maiz);
    P(total);

    if (tipo == TRIGO) V(camionesTrigo[id]);
    else V(camionesMaiz[id]);
  }
}
```

Detalles clave de este diseño:
- La cola `turnos` guarda `(tipo, id)`, no solo `id` — porque los rangos `1..T` y `1..M` se pisan, y el Coordinador necesita saber a cuál de los dos arrays de semáforos privados mirar.
- `pedidos` evita que el Coordinador haga busy waiting mientras la cola está vacía (mismo patrón que el ejercicio 4/6 de la PDF explicativa).
- El camión nunca pide `trigo`/`maiz`/`total` por su cuenta en la versión (a) — solo el Coordinador lo hace. Esto es lo que garantiza el orden, y de paso evita necesitar un semáforo de "confirmación de vuelta" del camión hacia el Coordinador.
- La concurrencia real (hasta 7 camiones descargando a la vez) no se pierde: el Coordinador solo serializa el momento de *dar el pase*, no la descarga en sí — una vez que un camión tiene su `V(camionesTipo[id])`, sigue su curso en paralelo con los demás.
- Como T y M son cantidades fijas y conocidas, el Coordinador puede cortar con un `for` de `T+M` iteraciones — no hace falta ningún contador de "terminados" para saber cuándo se acabó.

**Regla general que deja este ejercicio**: cuando un enunciado pide preservar el orden de llegada entre procesos (y un semáforo común no lo garantiza), la solución típica es centralizar la decisión de admisión en un proceso Coordinador único y secuencial — el orden queda garantizado gratis por ser un solo hilo de control, sin necesidad de mecanismos de sincronización adicionales entre los procesos que compiten.

## Checklist para encarar cualquier ejercicio nuevo de la práctica 2

Antes de escribir código, para cada ejercicio nuevo preguntate:

1. **¿Cuáles son las variables/estructuras realmente compartidas** (leídas y escritas por más de un proceso), y qué operaciones sobre ellas necesitan exclusión mutua? → esas van dentro de un `P(mutex)/V(mutex)`, y sólo esas (no de más, no de menos — ver ejercicio 1).
2. **¿Hay una condición de espera** ("esperar a que haya algo", "esperar mi turno", "esperar que termine el otro")? Si es así, ¿te conviene resolverla con un semáforo (evita busy waiting) en vez de un `while` con chequeo repetido de una variable compartida?
3. **¿Necesitás despertar a alguien en particular**, o a cualquiera que esté esperando, o a todos? → semáforo privado por proceso (array), semáforo compartido, o un `V` en loop sobre todos, respectivamente.
4. **¿Toda salida posible del código** (cada `while`, cada `if`) deja los semáforos en un estado consistente? → especialmente: nunca demorarte en un semáforo "de espera" mientras todavía tenés tomado el mutex (siempre `V(mutex)` antes de cualquier otro `P` de espera), y verificar que cada `P` tenga, en algún camino de ejecución, un `V` correspondiente.

## Ver también
- Apunte de Teoría 3 (Semáforos — teoría completa): `claude/apuntes-teoria-3.md`
- Enunciado de Práctica 1 (Variables compartidas): archivo `enunciadopractica1.pdf` del proyecto