# Apunte — Teoría 7: RPC, Rendezvous y ADA

## La idea en una frase

RPC y Rendezvous son mecanismos pensados para cliente/servidor: el cliente hace un `call` y queda demorado hasta recibir el resultado, como si llamara a un procedimiento. La diferencia está en quién atiende: en RPC un proceso **nuevo** por cada llamado; en Rendezvous un proceso **ya existente**, que decide cuándo aceptar.

La clase tiene tres partes:

1. RPC: módulos, procedimientos exportados y sincronización dentro del módulo.
2. Rendezvous: sentencia de entrada `in ... ni` con guardas.
3. ADA: el lenguaje con rendezvous que se usa en la práctica.

## 1. Por qué RPC y Rendezvous

El pasaje de mensajes se ajusta bien a filtros y pares, porque la comunicación es unidireccional. Para cliente/servidor es incómodo:

- La comunicación es bidireccional, así que hacen falta dos tipos de canales (requerimientos y respuestas).
- Cada cliente necesita un canal de respuesta distinto.

RPC y Rendezvous suponen un **canal bidireccional**. Combinan una interfaz "tipo monitor" (operaciones exportadas que se invocan con `call`) con mensajes sincrónicos: el llamador se demora hasta que la operación termina y se devuelven los resultados.

| | RPC | Rendezvous |
| --- | --- | --- |
| Quién atiende el llamado | Un proceso nuevo (al menos conceptualmente) por cada llamado | Un proceso ya existente |
| Cómo se declara la operación | Un `proc` por operación | Una sentencia de entrada (`in` o `accept`) |
| Atención de llamados | Concurrente | De a uno por vez |
| Servidor | Pasivo: un conjunto de procedimientos | Activo: decide cuándo y qué aceptar |
| Sincronización | Hay que programarla dentro del módulo | Viene incluida en el mecanismo |
| Ejemplo de lenguaje | Java (RMI) | Ada |

## 2. RPC

Los programas se descomponen en **módulos**, que tienen procesos y procedures y pueden residir en espacios de direcciones distintos.

- Los procesos de un módulo pueden compartir variables y llamar a procedures de ese módulo.
- Un proceso solo puede comunicarse con otro módulo invocando los procedimientos que ese módulo **exporta**.

```
module Mname
  headers de procedures exportados (visibles)
body
  declaraciones de variables
  código de inicialización
  cuerpos de procedures exportados
  procedures y procesos locales
end
```

```
op opname (formales) [returns result]             # header

proc opname (identif. formales) returns identificador resultado
  declaración de variables locales
  sentencias
end                                                # cuerpo

call Mname.opname (argumentos)                     # llamado
```

Los procesos locales se llaman *background*, para distinguirlos de las operaciones exportadas.

### Qué pasa en un llamado intermódulo

1. Un nuevo proceso server sirve el llamado. Los argumentos viajan como mensajes.
2. El llamador se demora mientras el server ejecuta el cuerpo del procedure.
3. El server envía los resultados al llamador y termina.
4. El llamador sigue.

Si el llamador y el procedure están en el mismo espacio de direcciones, se puede evitar crear el proceso. En general el llamado es remoto y hay que crear un server o tomarlo de un pool.

### Sincronización en módulos

**Por sí mismo, RPC es solo un mecanismo de comunicación.** La sincronización entre el llamador y su server es implícita, pero los procesos dentro de un módulo (servers de distintos llamados y procesos locales) también tienen que sincronizarse entre sí.

| Si los procesos del módulo ejecutan... | Entonces |
| --- | --- |
| Con exclusión mutua (uno por vez) | Las variables compartidas quedan protegidas; solo hay que programar la sincronización por condición |
| Concurrentemente | Hay que programar exclusión mutua y sincronización por condición, con cualquier método (semáforos, monitores, rendezvous) |

Se asume el caso más general: **ejecutan concurrentemente**.

### Ejemplo: Time Server

Un módulo que da servicios de tiempo. Exporta `get_time` y `delay`, y tiene un proceso interno `Clock` que incrementa la hora en cada interrupción del timer.

```
module TimeServer
  op get_time() returns INT;
  op delay(INT interval, INT myid);
body
  INT tod = 0;
  SEM m = 1;
  SEM d[n] = ([n] 0);
  QUEUE of (INT waketime, INT id's) napQ;

  proc get_time () returns time
    { time := tod; }

  proc delay (interval, myid)
    { INT waketime = tod + interval;
      P(m);
      insert((waketime, myid), napQ);
      V(m);
      P(d[myid]);
    }

  Process Clock
    { Inicia timer por hardware;
      WHILE (true)
        { Esperar interrupción, luego rearrancar timer;
          tod := tod + 1;
          P(m);
          WHILE tod ≥ min(waketime, napQ)
            { remove((waketime, id), napQ);
              V(d[id]);
            }
          V(m);
        }
    }
end TimeServer;
```

- Varios clientes pueden llamar a la vez, así que hay varios servers ejecutando concurrentemente.
- `get_time` no necesita protección: solo lee `tod`.
- `delay` y `Clock` necesitan exclusión mutua porque manipulan `napQ`. Para eso está el semáforo `m`.
- `d[myid]` es un semáforo privado: cada cliente espera en el suyo.

### Ejemplo: caches en un sistema de archivos distribuido

Los programas de aplicación corren en workstations y los archivos están en un file server.

- **`FileCache`** (uno por workstation): exporta `read` y `write`. Mantiene en cache los bloques leídos recientemente. Si lo pedido no está, llama al `FileServer`.
- **`FileServer`** (en el servidor): exporta `readblk` y `writeblk`. Maneja el acceso a bloques de disco de 1024 bytes y contiene un proceso `DiskDriver`.

```
Module FileCache                 # ubicado en cada workstation
  op read (INT count; result CHAR buffer[*]);
  op write (INT count; CHAR buffer[*]);
body
  cache de N bloques; descripción de los registros de cada file;
  semáforos para sincronizar acceso al cache;

  proc read (count, buffer)
    { IF (los datos pedidos no están en el cache)
        { seleccionar los bloques del cache a usar;
          IF (se necesita vaciar parte del cache) FileServer.writeblk(....);
          FileServer.readblk(....);
        }
      buffer = número de bytes requeridos del cache;
    }

  proc write (count, buffer)
    { IF (los datos apropiados no están en el cache)
        { seleccionar los bloques del cache a usar;
          IF (se necesita vaciar parte del cache) FileServer.writeblk(....);
        }
      bloquedeCache = número de bytes desde buffer;
    }
end FileCache;
```

`FileCache` es server de los programas de aplicación y, a la vez, cliente de `FileServer`.

| Módulo | ¿Necesita sincronización interna? |
| --- | --- |
| `FileCache`, uno por programa de aplicación | No: solo un `read` o `write` puede estar activo |
| `FileCache` compartido por varios programas | Sí: semáforos para exclusión mutua |
| `FileServer` | Sí: atiende a múltiples `FileCache` y contiene al `DiskDriver` |

### Ejemplo: intercambio de valores entre pares

Si dos procesos de módulos distintos tienen que intercambiar valores, cada módulo exporta un procedimiento que el otro llama.

```
module Intercambio [i = 1 to 2]
  op depositar(int);
body
  int otrovalor;
  sem listo = 0;

  proc depositar (otro)
    { otrovalor = otro;
      V(listo);
    }

  process Worker
    { int mivalor;
      call Intercambio[3-i].depositar(mivalor);
      P(listo);
      .......
    }
end Intercambio
```

`3-i` da el otro módulo (para 1 da 2 y para 2 da 1). El `Worker` deposita su valor en el otro y espera en `listo` a que el otro deposite en él.

### RPC en Java: RMI

Java soporta RPC mediante la invocación de métodos remotos (RMI). Una aplicación tiene tres componentes: una **interfaz** que declara los headers de los métodos remotos, una **clase server** que la implementa, y uno o más **clientes** que llaman a esos métodos. Server y clientes pueden estar en máquinas distintas.

## 3. Rendezvous

Problemas de RPC que motivan el rendezvous:

- Solo da comunicación intermódulo; la sincronización hay que programarla aparte.
- A veces hacen falta procesos extra solo para manipular los datos comunicados.

Rendezvous **combina comunicación y sincronización**:

- El cliente invoca con `call`, igual que en RPC.
- La operación la sirve un proceso existente, con una **sentencia de entrada**.
- Las operaciones se atienden **de a una por vez**.

El cuerpo del módulo es un único proceso que sirve las operaciones.

```
in opname (parámetros formales) → S; ni
```

La sentencia demora al server hasta que haya al menos un llamado pendiente de `opname`. Después elige el **más viejo**, copia los argumentos en los parámetros formales, ejecuta S y devuelve los resultados. Recién ahí los dos procesos continúan.

### Comunicación guardada

```
in op1 (formales1) and B1 by e1 → S1;
□ ...
□ opn (formalesn) and Bn by en → Sn;
ni
```

| Parte | Qué es | Detalle |
| --- | --- | --- |
| `and Bi` | Expresión de sincronización, opcional | Puede referenciar los parámetros formales |
| `by ei` | Expresión de scheduling, opcional | Entre los llamados que cumplen la condición, elige el de menor valor |

Que la condición pueda mirar los parámetros del llamado es lo que hace tan concisas las soluciones que siguen.

### Ejemplo: buffer limitado

```
module BufferLimitado
  op depositar (typeT), retirar (OUT typeT);
body
  process Buffer
    { queue buf;
      int cantidad = 0;
      while (true)
        { in depositar (item) and cantidad < n → push(buf, item);
                                                 cantidad = cantidad + 1;
          □ retirar (OUT item) and cantidad > 0 → pop(buf, item);
                                                  cantidad = cantidad - 1;
          ni
        }
    }
end BufferLimitado
```

### Ejemplo: filósofos centralizado

```
module Mesa
  op tomar(int), dejar(int);
body
  process Mozo
    { bool comiendo[5] = ([5] false);
      while (true)
        in tomar(i) and not (comiendo[izq(i)] or comiendo[der(i)]) → comiendo[i] = true;
        □ dejar(i) → comiendo[i] = false;
        ni
    }
end Mesa

module Persona [i = 0 to 4]
body
  process Filosofo
    { while (true)
        { call Mesa.tomar(i);
          come;
          call Mesa.dejar(i);
          piensa;
        }
    }
```

La condición usa el parámetro `i`: el mozo acepta el `tomar` de un filósofo solo si sus dos vecinos no están comiendo.

### Ejemplo: Time Server

A diferencia de la versión con RPC, acá `waketime` es directamente la hora a la que debe despertarse.

```
module TimeServer
  op get_time (OUT int);
  op delay (int);
  op tick ();
body TimeServer
  process Timer
    { int tod = 0;
      while (true)
        in get_time (OUT time) → time = tod;
        □ delay (waketime) and waketime <= tod by waketime → skip;
        □ tick () → tod = tod + 1; reiniciar timer;
        ni
    }
end TimeServer
```

El `delay` de un cliente recién se acepta cuando ya llegó su hora. No hacen falta cola ni semáforos privados.

### Ejemplo: alocador SJN

```
module Alocador_SJN
  op pedir(int), liberar();
body
  process SJN
    { bool libre = true;
      while (true)
        in pedir (tiempo) and libre by tiempo → libre = false;
        □ liberar () → libre = true;
        ni
    }
end SJN_Allocator
```

`by tiempo` hace que, entre los pedidos pendientes, se acepte el de menor tiempo. Es toda la política SJN en una línea.

## 4. ADA

Lo desarrolló el Departamento de Defensa de Estados Unidos como estándar para aplicaciones de defensa.

- Un programa tiene **tasks** (tareas) que ejecutan independientemente.
- Los puntos de invocación de una tarea son los **entrys**, declarados en la parte visible.
- Una tarea decide si acepta una comunicación con la primitiva **accept**.
- Se puede declarar un `task type` y crear varias instancias (arreglo, puntero, instancia simple).

```
TASK nombre IS
   declaraciones de ENTRYs
end;

TASK BODY nombre IS
   declaraciones locales
BEGIN
   sentencias
END nombre;
```

### Del lado del cliente: entry call

Se escribe `Tarea.entry(parámetros)`. Demora al llamador hasta que la operación terminó, abortó o alcanzó una excepción. Los parámetros pueden ser `IN`, `OUT` o `IN OUT`.

| Variante | Sintaxis | Comportamiento |
| --- | --- | --- |
| Simple | `Tarea.entry(params);` | Espera lo que haga falta |
| Condicional | `select entry call; sentencias; else sentencias; end select;` | Si no lo aceptan de inmediato, ejecuta el `else` |
| Temporal | `select entry call; sentencias; or delay tiempo sentencias; end select;` | Espera como máximo ese tiempo |

### Del lado del servidor: accept

```
accept nombre (parámetros formales) do sentencias end nombre;
```

Demora a la tarea hasta que haya una invocación, copia los parámetros reales en los formales y ejecuta las sentencias. Al terminar, copia los parámetros de salida. Después ambos procesos continúan.

El cliente queda demorado **solo durante el cuerpo del accept**, entre `do` y `end`. Lo que la tarea hace después ya es concurrente con el cliente.

### Wait selectivo

```
select when B1 => accept E1; sentencias1
or     ...
or     when Bn => accept En; sentenciasn
end select;
```

- Cada línea es una **alternativa**. Las cláusulas `when` son opcionales.
- Puede incluir una alternativa `else`, `or delay` u `or terminate`.
- Atributos del entry: `count` (cantidad de llamados pendientes) y `calleable`.

### Diferencias con el rendezvous general

Comparando las soluciones de la clase, en ADA no se puede hacer lo que hacían `and B` y `by e` del rendezvous general:

- El `when` **no puede mirar los parámetros** del llamado. Se evalúa antes de aceptar.
- **No hay expresión de scheduling**. Los llamados a un entry se atienden en orden de llegada.

Por eso las soluciones en ADA son más largas. Las técnicas para compensarlo aparecen en los ejemplos: un entry por cada caso, o aceptar siempre, guardar el pedido y avisarle después al cliente por un entry propio.

### Ejemplo: mailbox para un mensaje

```
TASK TYPE Mailbox IS
   ENTRY Depositar (msg: IN mensaje);
   ENTRY Retirar (msg: OUT mensaje);
END Mailbox;

A, B, C : Mailbox;

TASK BODY Mailbox IS
   dato: mensaje;
BEGIN
   LOOP
      ACCEPT Depositar (msg: IN mensaje) DO dato := msg; END Depositar;
      ACCEPT Retirar (msg: OUT mensaje) DO msg := dato; END Retirar;
   END LOOP;
END Mailbox;
```

Los dos `accept` en secuencia fuerzan la alternancia. Se usa con `A.Depositar(x1); C.Retirar(x3);`.

### Ejemplo: mailbox para N mensajes

Con una cola:

```
TASK Mailbox IS
   ENTRY Depositar (msg: IN mensaje);
   ENTRY Retirar (msg: OUT mensaje);
END Mailbox;

TASK BODY Mailbox IS
   buf: queue;
   cantidad: integer := 0;
BEGIN
   LOOP
      SELECT
         WHEN cantidad < N => ACCEPT Depositar (msg: IN mensaje) DO
                                 push(buf, msg);
                                 cantidad := cantidad + 1;
                              END Depositar;
      OR
         WHEN cantidad > 0 => ACCEPT Retirar (msg: OUT mensaje) DO
                                 pop(buf, msg);
                                 cantidad := cantidad - 1;
                              END Retirar;
      END SELECT;
   END LOOP;
END Mailbox;
```

Acá el `when` alcanza porque la condición depende solo del estado del servidor, no de los parámetros.

La diapositiva 37 muestra la misma solución con un arreglo circular e índices `pri` y `ult`. Ojo: ahí `Depositar` hace `ult := (ult+1) MOD N` **antes** de guardar, con `pri` y `ult` inicializados en 0. El primer mensaje queda en la posición 1 y el primer `Retirar` lee la posición 0. Creo que el incremento debería ir después de `datos[ult] := msg`. Conviene confirmarlo con la cátedra.

### Ejemplo: lectores y escritores

```
Procedure Lectores-Escritores is

  Task Sched IS
     Entry InicioLeer;
     Entry FinLeer;
     Entry InicioEscribir;
     Entry FinEscribir;
  End Sched;

  Task type Lector;
  Task body Lector is
  Begin
     Loop
        Sched.InicioLeer; ... Sched.FinLeer;
     End loop;
  End Lector;

  Task type Escritor;
  Task body Escritor is
  Begin
     Loop
        Sched.InicioEscribir; ... Sched.FinEscribir;
     End loop;
  End Escritor;

  VecLectores: array (1..cantL) of Lector;
  VecEscritores: array (1..cantE) of Escritor;

  Task body Sched is
     numLect: integer := 0;
  Begin
     Loop
        Select
           When InicioEscribir'Count = 0 =>
              accept InicioLeer;
              numLect := numLect + 1;
        or accept FinLeer;
           numLect := numLect - 1;
        or When numLect = 0 =>
              accept InicioEscribir;
              accept FinEscribir;
              For i in 1..InicioLeer'count loop
                 accept InicioLeer;
                 numLect := numLect + 1;
              End loop;
        End select;
     End loop;
  End Sched;

Begin
   Null;
End Lectores-Escritores
```

- Un lector entra solo si **no hay escritores esperando** (`InicioEscribir'Count = 0`). Eso evita que los escritores queden postergados.
- El escritor entra si no hay lectores. Mientras escribe, `Sched` queda bloqueado en `accept FinEscribir`, así que nadie más entra.
- Cuando el escritor termina, se deja pasar a todos los lectores que estaban esperando.

### Ejemplo: filósofos en ADA

En el rendezvous general la condición miraba el parámetro `i`. En ADA no se puede, y la clase muestra dos formas de resolverlo.

**Solución 1: múltiples entry.** Un entry por filósofo, así cada `when` sabe de quién se trata.

```
TASK Mesa IS
   ENTRY Tomar0; ENTRY Tomar1; ENTRY Tomar2; ENTRY Tomar3; ENTRY Tomar4;
   ENTRY Dejar (id: IN integer);
END Mesa;

TASK BODY Mesa IS
   Comiendo: array (0..4) of bool := (0..4 => false);
BEGIN
   For i in 0..4 loop
      Filosofos(i).identificacion(i);
   end loop;
   LOOP
      SELECT
         when (not (comiendo(4) or comiendo(1))) => ACCEPT Tomar0; comiendo(0) := true;
      OR when (not (comiendo(0) or comiendo(2))) => ACCEPT Tomar1; comiendo(1) := true;
      OR when (not (comiendo(1) or comiendo(3))) => ACCEPT Tomar2; comiendo(2) := true;
      OR when (not (comiendo(2) or comiendo(4))) => ACCEPT Tomar3; comiendo(3) := true;
      OR when (not (comiendo(3) or comiendo(0))) => ACCEPT Tomar4; comiendo(4) := true;
      OR ACCEPT Dejar (id: IN integer) do
            comiendo(id) := false;
         end Dejar;
      END SELECT;
   END LOOP;
END Mesa;
```

Cada filósofo recibe su id por el entry `Identificacion` y después llama al entry que le corresponde (`Mesa.Tomar0`, `Mesa.Tomar1`, ...). No escala: hay que escribir un entry y una alternativa por filósofo.

**Solución 2: encolar pedidos.** Se acepta siempre el pedido. Si no puede comer, se anota. Cuando alguien deja los tenedores, se revisa quién puede comer y se le avisa por un entry propio del filósofo.

```
TASK TYPE Filosofo IS
   ENTRY Identificacion (ident: IN integer);
   ENTRY Comer;
END Filosofo;

TASK Mesa IS
   ENTRY Tomar (id: IN integer);
   ENTRY Dejar (id: IN integer);
END Mesa;

Filosofos: array (0..4) of Filosofo;

TASK BODY Filosofo IS
   id: integer;
BEGIN
   ACCEPT Identificacion (ident: IN integer) do id := ident; End Identificacion;
   LOOP
      Mesa.Tomar(id);
      Accept Comer;
      -- Come
      Mesa.Dejar(id);
      -- Piensa
   END LOOP;
END Filosofo;

TASK BODY Mesa IS
   Comiendo: array (0..4) of bool := (0..4 => false);
   QuiereC: array (0..4) of bool := (0..4 => false);
   aux: integer;
BEGIN
   For i in 0..4 loop Filosofos(i).identificacion(i); end loop;
   LOOP
      SELECT
         ACCEPT Tomar (id: IN integer) do aux := id; END Tomar;
         if (not (comiendo((aux+1) mod 5) or comiendo((aux-1) mod 5))) then
            comiendo(aux) := true;
            Filosofos(aux).Comer;
         else QuiereC(aux) := true; end if;
      OR ACCEPT Dejar (id: IN integer) do aux := id; end Dejar;
         comiendo(aux) := false;
         for i in 0..4 loop
            if (QuiereC(i) and not (comiendo((i+1) mod 5) or comiendo((i-1) mod 5))) then
               comiendo(i) := true;
               QuiereC(i) := false;
               Filosofos(i).Comer;
            end if;
         end loop;
      END SELECT;
   END LOOP;
END Mesa;
```

El filósofo hace `Mesa.Tomar(id)`, que vuelve enseguida, y queda esperando en `Accept Comer` hasta que la mesa lo llame. El `accept` solo copia el parámetro; el resto se hace afuera para no demorar al cliente.

En las diapositivas 40 y 41 el cuerpo de la tarea `Filosofo` cierra con `END Mesa;`. Es un error de tipeo: va `END Filosofo;`.

### Ejemplo: Time Server en ADA

```
PROCEDURE DESPERTADORES IS

  Task TimeServer is
     entry get_time (hora: OUT int);
     entry delay (hd, id: IN int);
     entry tick;
  End TimeServer;

  Task Reloj;

  Task Type Cliente Is
     entry Identificar (identificacion: IN integer);
     entry seguir;
  End Cliente;

  ArrClientes: array (1..C) of Cliente;

  Task Body Cliente Is
     id: integer; hora: integer;
  BEGIN
     ACCEPT Identificar (identificacion: IN integer) do id := identificacion; End Identificar;
     TimeServer.get_time(hora);
     TimeServer.delay(hora + ....., id);
     ACCEPT seguir;
  End Cliente;

  Task Body Reloj is
  BEGIN
     loop delay(1); TimeServer.tick; end loop;
  End Reloj;

  Task Body TimeServer is
     actual: integer := 0;
     dormidos: colaOrdenada;
     auxId, auxHora: integer;
  BEGIN
     LOOP
        SELECT
           when (tick'count = 0) => ACCEPT get_time (hora: OUT integer) do hora := actual; END get_time;
        OR when (tick'count = 0) => ACCEPT delay (hd, id: IN integer) do agregar(dormidos, (id, hd)); END delay;
        OR ACCEPT tick;
           actual := actual + 1;
           while (not empty(dormidos)) and then (VerHoraPrimero(dormidos) <= actual) loop
              sacar(dormidos, (auxId, auxHora));
              ArrClientes(auxId).seguir;
           end loop;
        END SELECT;
     END LOOP;
  End TimeServer;

BEGIN
   for i in 1..C loop
      ArrClientes(i).identificacion(i);
   end loop;
END DESPERTADORES;
```

- `when (tick'count = 0)` le da **prioridad al tick**: si hay un tick pendiente, no se aceptan `get_time` ni `delay`.
- Como no existe `by`, el orden se maneja a mano con la cola ordenada `dormidos`.
- El cliente llama a `delay`, que vuelve enseguida, y queda esperando en `ACCEPT seguir`.

### Ejemplo: alocador SJN en ADA

```
PROCEDURE SchedulerSJN IS

  Task Alocador_SJN is
     entry pedir (tiempo, id: IN integer);
     entry liberar;
  End Alocador_SJN;

  Task Type Cliente Is
     entry Identificar (identificacion: IN integer);
     entry usar;
  End Cliente;

  ArrClientes: array (1..C) of Cliente;

  Task Body Cliente Is
     id: integer; tiempo: integer;
  BEGIN
     ACCEPT Identificar (identificacion: IN integer) do
        id := identificacion;
     End Identificar;
     loop
        -- trabaja y determina el valor de tiempo
        Alocador_SJN.pedir(tiempo, id);
        Accept usar;
        -- Usa el recurso
        Alocador_SJN.liberar;
     end loop;
  End Cliente;

  Task Body Alocador_SJN is
     libre: boolean := true;
     espera: colaOrdenada;
     tiempo, aux: integer;
  Begin
     loop
        aux := -1;
        select
           accept Pedir (tiempo, id: IN integer) do
              if (libre) then libre := false; aux := id;
              else agregar(espera, (id, tiempo)); end if;
           end Pedir;
        or accept liberar;
           if (empty(espera)) then libre := true;
           else sacar(espera, (aux, tiempo)); end if;
        end select;
        if (aux <> -1) then ArrClientes(aux).usar; end if;
     end loop;
  End Alocador_SJN;

BEGIN
   for i in 1..C loop
      ArrClientes(i).identificacion(i);
   end loop;
END SchedulerSJN;
```

`aux` guarda a quién hay que darle el recurso en esta vuelta, o -1 si a nadie. Al liberar, si hay alguien esperando, `libre` queda en false y el recurso pasa directo al siguiente.

En la diapositiva 47 el cliente llama `Alocador_SJN.pedir(id, tiempo)`, pero el entry está declarado como `pedir(tiempo, id)`. Arriba lo dejé en el orden de la declaración.

## 5. Resumen: el mismo problema en cada mecanismo

| Problema | Rendezvous general | ADA |
| --- | --- | --- |
| Buffer limitado | `and cantidad < n` | `when cantidad < N =>` (igual de simple) |
| Filósofos | `and not (comiendo[izq(i)] or ...)` | Un entry por filósofo, o encolar pedidos y avisar por un entry `Comer` |
| Time Server | `and waketime <= tod by waketime` | Cola ordenada propia y entry `seguir` en el cliente |
| Alocador SJN | `and libre by tiempo` | Cola ordenada propia y entry `usar` en el cliente |

El patrón que se repite en ADA cuando la condición depende de los parámetros: aceptar el pedido, guardarlo, y más tarde llamar a un entry del cliente para despertarlo. Para eso el servidor necesita conocer el id del cliente, que se le pasa al inicio por un entry de identificación.

## 6. Trampas y autoevaluación

### Trampas típicas

- **Decir que RPC resuelve la sincronización.** Solo comunica; dentro del módulo hay que sincronizar a mano.
- **Confundir quién atiende.** RPC: un proceso nuevo por llamado. Rendezvous: un proceso existente, de a uno por vez.
- **Usar parámetros del entry en un `when` de ADA.** No se puede; el `when` se evalúa antes de aceptar.
- **Poner todo dentro del `accept ... do ... end`.** El cliente queda demorado durante todo ese bloque. Conviene que el cuerpo sea mínimo.
- **Olvidar el entry de identificación.** Las tareas de un arreglo no conocen su índice si nadie se lo pasa.
- **Olvidar el `Accept` del lado del cliente** en el patrón de encolar pedidos. Sin él, el cliente sigue sin esperar su turno.

### Preguntas para chequear que se entendió

1. ¿Por qué el pasaje de mensajes es incómodo para cliente/servidor?
2. ¿En qué se diferencian RPC y Rendezvous?
3. ¿Qué pasos ocurren en un llamado intermódulo con RPC?
4. ¿Por qué en el Time Server con RPC `get_time` no necesita exclusión mutua y `delay` sí?
5. ¿Qué hace la sentencia `in opname(formales) → S; ni`?
6. ¿Para qué sirven `and B` y `by e` en una operación guardada?
7. ¿Qué diferencia hay entre un entry call simple, condicional y temporal?
8. ¿Durante qué tramo queda demorado el cliente en un rendezvous de ADA?
9. ¿Qué devuelve el atributo `count` y para qué se usa en lectores y escritores y en el Time Server?
10. ¿Por qué el alocador SJN ocupa una línea en rendezvous general y necesita una cola en ADA?
11. Explicar las dos soluciones de filósofos en ADA y la desventaja de la primera.

## Material

- Diapositivas: `Clases teóricas/teroria-7.pdf`. La diapositiva 2 tiene los links a los cuatro videos con audio (introducción, RPC, Rendezvous y ADA).
