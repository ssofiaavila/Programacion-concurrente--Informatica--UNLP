# Explicación práctica 5 ADA/ Rendezvous
## Lenguaje ADA (Rendezvous)
Una aplicación ADA es un único programa con un cuerpo principal, y para que trabaje concurrentemente se desarrollan **TASK** dentro de ese programa que pueden ejecutar independientemente y contienen primitivas de sincronización/ comunicación.

Se pueden implementar diretamente `TASK` o se puede declarar un `TASK TYPE` para crear varias estancias iguales de ese tipo (arreglo, puntero, instancia simple).

Cada `TASK` o `TASK TYPE` tiene una especificación donde se declaran las operaciones exportadas (llamadas `ENTRYs`) y un `BODY` donde se implementa la tarea.

## Sintaxis - estructura del programa
```
procedure nombre is 
    -- Especificación de un TASK
    TASK nombreT1 is
        -- Declaración de los ENTRYs de la tarea
    END nombreT1;

    -- Especificación de un tipo TASK
    TASK TYPE nombreTipoT is
        -- Declaración de los ENTRYs del tipo tarea
    END nombreTipoT;

    -- Declaración de variables del tipo nombreTipoT

    -- Cuerpo de las tareas
    TASK BODY nombreT1 is
        -- Implementación de la tarea
    END nombreT1;

    TASK BODY nombreTipoT is
        -- Implementación del tipo tarea
    END nombreTipoT;

Begin
    -- Cuerpo principal del programa
END nombre;
```
## Sintaxis - especificación de las tareas
Las tareas pueden exportar operaciones o no, si lo hacen deben definir los ENTRYs correspondientes en la especificación.
Los ENTRYs pueden o no tener parámetros, si los tiene se debe declarar el nombre, el tipo de datos y el tipo de parámetro (IN, OUT o IN OUT).
```
-- Especificación de tarea y tipo tarea sin ENTRY;
    TASK nombreT1;
    TASK TYPE nombreTipo1;

-- Especificación de tareas con ENTRY sin parámetros
    TASK nombreT2 IS
        ENTRY nombreE1;
        ENTRY nombreE2;
    END nombre T2;

-- Espeficiación de tareas con ENTRYs con parámetros
    TASK nombreT3 IS
        ENTRY nombreE3(A, B: IN integer; C: OUT chat; D: IN OUT boolean);
    END nombreT3;

-- Delcaración de variables de tipo tarea (nombreTipo1)
    arreloT1: ARRAY(1..10) OF nombreTipo1;
    nombreVariable: nombreTipo1;    
```

## Sintaxis - cuerpo de las tareas
El cuerpo de las tareas es donde se implementa la tarea. En esta parte no hay diferencia entre las TASK y los TASK TYPE (no se pone TYPE).

```
    TASK BODY nombreT1 IS
        variables
    BEGIN 
        implementación de la tarea
    END nombreT1;
```

Si se hubiese definido un arreglo de un tipo tarea (por ejemplo `arregloT1` de la página anterior). se generarán varias tareas idénticas de ese tipo. La única manera de identificar una de la otra es por su posición en el arreglo, pero la tarea por si misma no conoce ese dato. Si lo necesita saber, otra tarea o el cuerpo principal del programa se lo deberá comunicar para que guarde ese "identificador" en una variable.

## Sintaxis - comunicación/ sincronización
- **Rendezvous** (encuentro) es el principal mecanismo de sincronización en ADA y también es el mecanismo de comunicación primario.
- ADA tiene otros mecanismos de comunicación QUE NO PUEDEN ser usados en la materia como variables compartidas.
- La comunicación (el encuentro) entre dos atarea se realiza por medio de un `Entry call` por parte del proceso que realiza el pedido y un `Accept` por parte del que sirve ese pedido. Ambas tareas deben esperar a que el **Accept** termine su ejecución para poder continuar.
- La comunicación es bidireccional, y es fundamental que esta característica se aproveche (siempre que sea posible y no reduzca la concurrecia) al resolver los problemas en ADA.

## Sintaxis - entry call
Hay 3 foromas opciones de `Entry call` que se diferencias por el tiempo de demora.
1. **Entry call** simple: la ejecución demora al llamador hasta que la otra tarea termine de ejecutar el **accept** correspondiente.
   ```
    Tarea2.nombreE1;
    Tarea3.nombreE3(5, x, y, z);
    -- Los parámetros de tipo IN pueden ser variables, constantes o expresiones
    -- Los de tipo OUT o IN OUT deben ser variables.
    ArregloT2(3).nombreE4;
    -- Siendo ArregloT2 un arreglo de tareas
   ```
2. **Entry call condicional (select- else):** no espera a que le acepten el pedido, si la otra tarea no está lista para realziar el **accept** a su pedido inmediatamnete, entonces lo cancela y realiza otra cosa.
3. **Entry call temporal (select- or delay):** espera a lo sumo un tiempo a que la otra tarea realice el **accept** a su pedido, pasado el tiempo cancela el entry call y realiza otra cosa.

### Sintaxis - entry call
**Entry call condicional:** no espera a que le acepten el pedido, si la otra tarea no está lista para realizar el accept a su pedido inmediatamente, entonces lo cancela y realiza otra cosa.
```
-- Ejemplos de Entry Call Condicional a una tarea
    select
        Tarea3.nombreE3(5, x, y, z);
        sentencias S1 a realizar sólo si se completó el entry call
    else 
        sentencias S2 que se harán si se canceló el entry call
    end select;
``` 
> Intenta hacer el entry call nombreE3 y si Tarea3 inmediatamente acepta su pedido, se demora hasta que se termine el accept; luego (ya terminado la sincronización con Tarea3) realiza el conjunto de sentencias S1 (puede no estar este conjunto de sentencias S1); al terminar finaliza el select.
> Si Tarea3 no acepta inmediatamente su pedido, cancela el entry call y realiza el conjunto de sentencias S2 que está después del else (si no hubiese que hacer nada en el else igual se debe poner alguna sentencia, en ese caso S2 sería sentencia nula null que no hace nada); al terminar finaliza el select.

**Entry call temporal (select- or delay):** espera a lo sumo un tiempo a que la otra tarea realice el accept a su pedido, pasado el tiempo cancela el entry call y realiza otra cosa.
```
-- Ejemplos de Entry Call Temporal a una tarea
    select
        Tarea3.nombreE3(5, x, y, z);
        sentencias S1 a realizar sólo si se completó el entry call
    or delay tiempo
        sentencias S2 que se harán si se canceló el entry call
    end select;
```
> Intenta hacer el entry call nombreE3 y espera a lo sumo `tiempo` a que Tarea3 acepte su pedido. Si antes de ese tiempo le aceptan el pedido, entonces se demora hasta que se termina el accept; luego (ya terminada la sincronización con Tarea3) realiza el conjunto de sentencias S1 (puede no estar este conjunto de sentencias S1); al terminar finaliza el select.
> Si pasado `tiempo` la Tarea3 aún no acepta su pedido, cancela el entry call y realiza el conjunto de sentencias S2 que está después del or delay (si no hubiese que hacer nada en el or delay igual se debe poner alguna sentencia, en ese caso S2 sería la sentencia nula null que no hace nada); al terminar finaliza el select.

## Sintaxis - accept
- Por cada entry declarado en la especificación de una tarea, en el cuerpo se debe hacer al menos un **accept** para dicho entry.
- Por cada entry existe en la tarea una **cola implícita** (no se puede manipular por la tarea) de entry calls pendientes.
- El accept puede tener un cuerpo, en cuyo caso hasta que no se termine de ejecutar el mismo no se pierde la sincronización con el entry call. O puede no tener un cuerpo, en cuyo caso inmediatamente termina la sincronización con el entry call.
- El accept demora la tarea hasta que haya al menos un entry call en la cola implícita asociada; saca al primero de ellos y, dependiendo del tipo de accept hace:
  - **Si no tiene cuerpo:** termina inmediatamente el accept y ambas tareas continúan con su ejecución.
  - **Si tiene cuerpo:** copia los parámetros reales del llamado en los parámetros formales (si el entry no tuviese parámetros no haría este paso); ejecuta las sentencias que están en el cuerpo; cuando termina, los parámetros formales de salida son copiados a los parámetros reales (si tuviese parámetros de tipo OUT o IN OUT). Luego ambas tareas continúan con su ejecución.

```
-- Ejemplo de accept sin cuerpo en la tarea nombreT2
    ACCEPT nombreE1;

-- Ejemplo de accept con cuerpo de un entry sin parámetros en la tarea nombreT2
    ACCEPT nombreE2 IS
        sentencias S1 que forman el cuerpo del accept
    END nombreE2;

-- Ejemplo de accept con cuerpo de un entry con parámetros en la tarea nombreT3
    ACCEPT nombreE3 (A, B: IN integer; C: OUT char; D: IN OUT boolean) IS
        sentencias S1 que forman el cuerpo del accept
    END nombreE3;
```
> Los parámetros en el accept se ponen completos (igual que en la especificación), y **SÓLO** son conocidos y se pueden usar dentro del cuerpo del accept, fuera de este los parámetros ya no existen. Si alguno de esos parámetros se debe usar fuera del cuerpo del accept, entonces dentro de él se debe copiar el valor en una variable de la tarea, y luego se usa esa variable.
>
> Las sentencias que forman el conjunto S1 pueden ser cualquier sentencia válida de ADA (incluso otros ACCEPT o ENTRY CALL de cualquiera de los 3 tipos).

### Select de accepts (wait selectivo)
ADA brinda la posibilidad de implementar **wait selectivos** por medio de comunicación guardada (como la de PMS). Select para los Accept. Permite tener varias alternativas de ACCEPT que pueden o no tener asociada una condición booleana (**no se pueden utilizar los parámetros del entry como parte de la condición**) utilizando la cláusula `when`.
```
    SELECT
        ACCEPT nombreE1;
        sentencias S1 a realizar sólo si se completó el accept nombreE1
    OR
        WHEN (condición) => ACCEPT nombreE2 IS
                                sentencias S2 que forman el cuerpo del accept
                            END nombreE2;
                            sentencias S3 a realizar sólo si se completó el accept nombreE2
    END SELECT;
```
> En este ejemplo el Select tiene dos alternativas de ACCEPT, la primera de ellas no tiene condición asociada (se considera TRUE) y el ACCEPT no tiene cuerpo. En cambio la segunda alternativa tiene una condición booleana y tiene un cuerpo (sentencias S2).
>
> Cuando la tarea llega a este select, espera hasta que al menos una de las alternativas sea exitosa (condición booleana TRUE y que haya entry calls pendientes para ese entry) y selecciona una de ellas (de las exitosas) de forma **no determinística** para ejecutarla. Si todas las alternativas fallan (todas tuviesen condición booleana y fuese falsa) termina el select sin ejecutar nada.
>
> Al elegir una alternativa para ejecutar (supongamos la segunda en este caso), primero ejecuta el ACCEPT (de la forma que se vio antes) y luego se ejecuta el conjunto de sentencias S3 (si lo tuviese), pero en este punto ya liberó a la tarea que hizo el entry call.

A su vez se le puede agregar al final un `ELSE` o un `OR DELAY` que actúa de forma similar al de los entry call:
- En el caso del **ELSE**, si no se puede ejecutar ninguna alternativa en forma inmediata, entonces se ejecuta el conjunto de sentencias asociadas al ELSE.
- En el caso del **OR DELAY**, espera a lo sumo un tiempo a poder aceptar alguna alternativa, si no lo logra ejecuta el conjunto de sentencias asociadas al OR DELAY.

```
    SELECT
        ACCEPT nombreE1;
        sentencias S1 a realizar sólo si se completó el accept nombreE1
    OR
        WHEN (condición) => ACCEPT nombreE2 IS
                                sentencias S2 que forman el cuerpo del accept
                            END nombreE2;
                            sentencias S3 a realizar sólo si se completó el accept nombreE2
    OR DELAY tiempo
        sentencias S4 a realizar si pasó tiempo sin poder aceptar ninguna alternativa
    END SELECT;
```

### Atributo `'count`
- Cada entry tiene asociado un atributo que puede ser consultado **SÓLO por la tarea a la que pertenece el entry** que indica la cantidad de entry calls pendientes (cantidad de elementos de la cola implícita asociada al entry): `NombreDelEntry'count`
- Este atributo puede ser usado en los `when`, por ejemplo para dar prioridad a un entry sobre otro (como se ve en algunos de los ejercicios).
- Algo importante a tener en cuenta es que **NO TIENE SENTIDO** usarlo en el when de un accept dentro de un select para comprobar que haya un entry call pendiente. Por ejemplo:
  ```
  when (nombreE1'count > 0) => ACCEPT nombreE1;
  ```
  Ya que si no hay un pedido pendiente (es decir que `nombreE1'count = 0`) esta alternativa no podrá ser elegida por el momento, ya que no se puede hacer el accept inmediatamente. Y además, esto podría llevar a producir **BUSY WAITING** si se hace lo mismo en todas las alternativas del select.

---

## Repaso: entendiendo el rendezvous (dudas resueltas)

> Sección agregada a partir de las dudas que fueron surgiendo al leer la explicación. La idea es conectar ADA con lo que ya vimos (semáforos, monitores, PMA, PMS).

### ¿Qué es el rendezvous, a partir de lo que ya conozco?
Pensalo como **un monitor, pero donde el procedimiento lo ejecuta un proceso activo que decide cuándo atender**.

- **Del monitor** toma la idea de **llamar a una operación con parámetros y recibir un resultado**: el cliente hace `Empleado.Pedido(datos, resultado)` y cuando vuelve ya tiene la respuesta.
- **De PMS** toma la idea de que **los dos procesos tienen que coincidir** (sincrónico): el que llama se bloquea hasta que lo atiendan, y el que atiende se bloquea hasta que alguien lo llame.

La gran diferencia con PMS: en PMS el mensaje va en **una sola dirección**, así que pedir y responder son **dos comunicaciones**:
```
// PMS: ida y vuelta = 2 mensajes
Cliente:  Empleado!pedido(datos);  Empleado?respuesta(res);
Empleado: Cliente[*]?pedido(d);    res = resolver(d);   Cliente[id]!respuesta(res);
```
En rendezvous es **un solo encuentro de ida y vuelta**:
```
-- ADA: 1 encuentro, los datos van (IN) y el resultado vuelve (OUT)
Cliente:   Empleado.Pedido("datos", Resultado);
Empleado:  accept Pedido (D: IN texto; R: OUT texto) do
               R := resolverPedido(D);
           end Pedido;
```
Por eso se insiste en "aprovechar la comunicación bidireccional": si el pedido se resuelve **dentro** del `do … end` del accept, se ahorra el segundo mensaje (es lo que corrige el Ejercicio 1).

### ¿Qué es el `accept`? (analogía de la ventanilla)
`accept` es la instrucción con la que una tarea dice **"estoy lista para atender un pedido de este tipo"**. Es la contraparte del entry call.

| Elemento | En el código | En la ventanilla del banco |
|---|---|---|
| **Entry** | `entry Pedido (...)` en la especificación | El **cartel de la ventanilla**: "acá se atienden pedidos" |
| **Entry call** | `Empleado.Pedido("datos", Resultado);` | El **cliente que se pone en la fila** |
| **Accept** | `accept Pedido (...) do ... end Pedido;` | El **empleado que dice "¡siguiente!"** y atiende al primero |

Qué hace el `accept` al ejecutarse:
1. Si **no hay nadie en la fila** del entry, la tarea **se queda esperando** ahí (bloqueada).
2. Si **hay alguien**, toma al primero (orden de llegada), recibe sus datos y ejecuta lo que está entre `do` y `end`.
3. Mientras se ejecuta ese cuerpo, **el que llamó sigue esperando**.
4. Al llegar a `end Pedido`, se devuelven los parámetros OUT y **ambos se liberan**.

Dos formas:
- **Con cuerpo** (`accept Pedido(...) do ... end Pedido;`): cuando hay que recibir datos o devolver una respuesta.
- **Sin cuerpo** (`accept Pedido;`): punto de encuentro puro, un "ya llegaste, seguí" (sincronización sin datos).

Comparado con PMS: el `accept` es como un **`receive`** (espera a que llegue algo), pero **tiene cuerpo** para procesar el pedido y **puede devolver datos** en el mismo encuentro.

### Leyendo el `task body` línea por línea
El `task body` es **todo lo que hace la tarea**, de principio a fin (como el código de un `process` en PMS).
```
task body Empleado is        -- acá empieza el código del Empleado
begin
   loop                      -- repetir para siempre (como while(true))
      accept Pedido (D: IN texto; R: OUT texto) do   -- atiendo al siguiente; si no hay nadie, espero
         R := resolverPedido(D);                      -- resuelvo (el cliente SIGUE esperando)
      end Pedido;                                     -- devuelvo R y el cliente queda libre
   end loop;                 -- vuelvo a atender al próximo
end Empleado;
```

Traza con dos clientes:

| Momento | Empleado | Cliente 1 | Cliente 2 |
|---|---|---|---|
| 1 | Llega al `accept`, no hay nadie: **espera** | — | — |
| 2 | Atiende al Cliente 1: `D = "datosC1"` | Llamó a `Empleado.Pedido("datosC1", Resultado)`: **espera** | — |
| 3 | Ejecuta `R := resolverPedido(D)` | Sigue esperando | Llama a `Empleado.Pedido(...)`: queda **en la fila** |
| 4 | `end Pedido`: le devuelve `R` | **Recibe `Resultado` y sigue** | Sigue en la fila |
| 5 | Nueva vuelta del `loop`: atiende al Cliente 2 | — | Lo atienden |

Del lado del cliente todo eso es **una sola línea**: `Empleado.Pedido("datos", Resultado);`. Cuando esa línea termina, `Resultado` ya tiene la respuesta.

### "Parece muy secuencial", ¿dónde está la concurrencia?
El código **de cada proceso** siempre fue secuencial (también en PMS, monitores, etc.). La concurrencia está **entre** las tareas:

- **Todas las tareas corren al mismo tiempo**: N clientes y el empleado ejecutan en paralelo.
- **Solo se sincronizan en el encuentro**: antes y después, cada uno va por su lado.
- **Lo que está fuera del `accept` corre en paralelo** con el cliente ya liberado:
```
loop
   accept Pedido (D: IN texto; R: OUT texto) do
      R := resolverPedido(D);       -- cliente esperando
   end Pedido;
   registrarAtencion;               -- cliente YA liberado: esto corre en paralelo con él
end loop;
```
**Regla práctica:** dentro del accept, solo lo que el cliente necesita esperar; todo lo demás, afuera (así no se reduce la concurrencia).

Un servidor que atiende de a uno se ve secuencial en cualquier mecanismo. Lo que cambia respecto de PMS es que la sincronización es **más fuerte**: el cliente queda bloqueado hasta que termina el cuerpo del accept, como al llamar a un procedimiento de un monitor.

### Tabla de equivalencias con lo que ya vimos
| Concepto en ADA | Lo más parecido que ya vimos |
|---|---|
| `TASK` / `TASK TYPE` | `process` / `process P[1..N]` |
| `ENTRY` (en la especificación) | Un procedimiento exportado del monitor / un canal de PMS |
| **Entry call** `Tarea.Entry(params)` | Llamar a `monitor.proc(params)`, o `send` + `receive` en un solo paso |
| **`accept`** | Un `receive` con cuerpo que además puede devolver datos |
| Cola implícita del entry | Cola de entrada del monitor / canal FIFO |
| `select … or … end select` con `when` | El `if` guardado de PMS (`cond; P?msg → …`) |
| `Entry'count` | Parecido a `empty(canal)` de PMA, pero **solo lo consulta la dueña del entry** |
| `select … or delay t` | Nuevo: esperar como máximo un tiempo |
| `select … else` | Nuevo: intentar sin esperar |

**Nombrado asimétrico:** el que llama tiene que saber a qué tarea llama (`Empleado.Pedido`), pero el que acepta **no sabe quién lo llamó**. Por eso, si hay que responderle después a alguien, hay que pasar su id como parámetro (ver Ejercicio 4 y el entry `Ident`).

### Tips de sintaxis que confunden al principio
- **IN**: entra a la tarea que acepta. **OUT**: vuelve al que llamó. **IN OUT**: va y vuelve.
- En el `accept` los parámetros se **repiten completos** igual que en el `entry`.
- Los parámetros del accept **solo existen dentro del `do … end`**: si se necesitan después, copiarlos a una variable de la tarea.
- En el `when` **no** se pueden usar parámetros del entry (la condición se evalúa antes de saber quién viene).
- Una tarea **solo puede hacer accept de sus propios entries** y solo ella puede consultar su `'count`.
- Si un `else` / `or delay` no tiene nada que hacer, igual va `null;`.
- El programa principal (`begin … end` del procedure) arranca cuando todas las tareas ya están corriendo; se usa para cosas como repartir ids (`arrClientes(i).Ident(i)`) o simplemente `null;`.

---

## Ejercicio 1
Se debe modelar la atención en un banco por medio de un único empleado. Los clientes llegan y son atendidos de acuerdo al orden de llegada.

Lo primero es definir la estructura del programa: qué tareas y tipos de tareas se necesitan. En este problema hay una TASK empleado y un arreglo de un TASK TYPE para los clientes.
```
Procedure Banco1 is
    Task Empleado ...
    Task Type Cliente ...
    arrClientes: array (1..N) of Cliente;
Begin
    ...
End Banco1;
```

En primera instancia vamos a realizar una solución donde el cliente envía el pedido y el empleado le responde.
```
Procedure Banco1 is
    Task empleado is
        Entry Pedido (Datos: IN texto);
    End empleado;

    Task type cliente is
        Entry Respuesta (Res: IN texto);
    End cliente;

    arrClientes: array (1..N) of Cliente;

    Task Body cliente is
        Resultado: texto;
    Begin
        Empleado.Pedido ("datos");
        Accept Respuesta (Res: IN texto) do
            Resultado := Res;
        End Respuesta;
    End cliente;

    Task Body Empleado is
        R, D: texto;
    Begin
        loop
            Accept Pedido (Datos: IN texto) do
                D := Datos;
            End Pedido;
            R := resolverPedido(D);
            arrClientes(...).Respuesta(R);
        end loop;
    End empleado;

Begin
    null;
End Banco1;
```
> ❌ **No aprovecha la comunicación bidireccional de rendezvous.**

El cliente no debe hacer nada entre que hace el pedido y recibe la respuesta. Por el otro lado, el empleado cuando acepta un pedido inmediatamente lo resuelve y le envía el resultado al cliente. Por lo tanto la resolución del pedido se debe hacer **EN EL CUERPO DEL ACCEPT**, y evitar así una comunicación.
```
Procedure Banco1 is
    Task empleado is
        Entry Pedido (D: IN texto; R: OUT texto);
    End empleado;

    Task type cliente;

    arrClientes: array (1..N) of Cliente;

    Task Body cliente is
        Resultado: texto;
    Begin
        Empleado.Pedido ("datos", Resultado);
    End cliente;

    Task Body empleado is
    Begin
        loop
            accept Pedido (D: IN texto; R: OUT texto) do
                R := resolverPedido(D);
            end Pedido;
        end loop;
    End empleado;

Begin
    null;
End Banco1;
```
> **¿Cómo se asegura la atención en orden de llegada?** Los entry call a un entry se almacenan en la cola implícita del mismo de acuerdo al orden de llegada.

## Ejercicio 2
Se debe modelar la atención en un banco por medio de un único empleado. Los clientes llegan y esperan a lo sumo 10 minutos a ser atendidos de acuerdo al orden de llegada. Pasado ese tiempo se retiran sin ser atendidos.

Partimos de la solución anterior. **¿Cómo resolvemos lo del tiempo de espera?** Para modelar la espera máxima de 10 minutos debemos utilizar un **entry call temporal** en el cliente.
```
Procedure Banco2 is
    Task empleado is
        Entry Pedido (D: IN texto; R: OUT texto);
    End empleado;

    Task type cliente;

    arrClientes: array (1..N) of Cliente;

    Task Body cliente is
        Resultado: texto;
    Begin
        SELECT
            Empleado.Pedido ("datos", Resultado);
        OR DELAY 600.0
            NULL;
        END SELECT;
    End cliente;

    Task Body empleado is
    Begin
        loop
            accept Pedido (D: IN texto; R: OUT texto) do
                R := resolverPedido(D);
            end Pedido;
        end loop;
    End empleado;

Begin
    null;
End Banco2;
```

## Ejercicio 3
Se debe modelar la atención en un banco por medio de un único empleado. Los clientes llegan y esperan a ser atendidos; pueden ser Regulares o Prioritarios. El empleado los atiende de acuerdo al orden de llegada pero dando prioridad a los clientes Prioritarios.

Partimos de la solución del ejercicio 1. **¿Cómo resolvemos la prioridad de los clientes?**

**Primer intento:** pasamos el tipo de cliente como parámetro en el entry call.
```
    Task empleado is
        Entry Pedido (Tipo: IN texto; D: IN texto; R: OUT texto);
    End empleado;

    Task Body cliente is
        Resultado: texto;
    Begin
        if ("es cliente prioritario") then
            Empleado.Pedido ("PRIORITARIO", "datos", Resultado);
        else
            Empleado.Pedido ("REGULAR", "datos", Resultado);
        end if;
    End cliente;

    Task Body empleado is
    Begin
        loop
            accept Pedido (Tipo: IN texto; D: IN texto; R: OUT texto) do
                R := resolverPedido(Tipo, D);
            end Pedido;
        end loop;
    End empleado;
```
> ❌ **NO RESPETARÁ LA PRIORIDAD.** Si el primer pedido es de un cliente Regular, lo atenderá aunque haya clientes Prioritarios en la cola del entry.
>
> Para manejar el tipo de cliente como un parámetro del entry deberíamos aceptar todos los pedidos para ver a cuál se debe atender. Y en realidad el empleado no debe perder tiempo en eso → en este caso es mejor **tener entrys diferentes para cada tipo de cliente**.

**Segundo intento:** el empleado tendrá 2 entrys, uno para cada tipo de cliente, y usará un select para determinar cuál de los dos entrys aceptar.
```
    Task empleado is
        Entry Prioritario (D: IN texto; R: OUT texto);
        Entry Regular (D: IN texto; R: OUT texto);
    End empleado;

    Task Body empleado is
    Begin
        loop
            SELECT
                accept Prioritario (D: IN texto; R: OUT texto) do
                    R := resolverPedidoPrioritario(D);
                end Prioritario;
            OR
                accept Regular (D: IN texto; R: OUT texto) do
                    R := resolverPedidoRegular(D);
                end Regular;
            END SELECT;
        end loop;
    End empleado;
```
> **¿Tiene más prioridad el primer accept por estar como primera alternativa del select?**
> **NO.** El orden de las alternativas no afecta en la elección a ejecutar. Para seleccionar el accept Regular se debe estar seguro de que no hay pedidos pendientes de Prioritario → usar el `when` en el accept Regular consultando por el `count` de Prioritario.

**Solución final:**
```
Procedure Banco3 is
    Task empleado is
        Entry Prioritario (D: IN texto; R: OUT texto);
        Entry Regular (D: IN texto; R: OUT texto);
    End empleado;

    Task type cliente;

    arrClientes: array (1..N) of Cliente;

    Task Body cliente is
        Resultado: texto;
    Begin
        if ("es cliente prioritario") then
            Empleado.Prioritario ("datos", Resultado);
        else
            Empleado.Regular ("datos", Resultado);
        end if;
    End cliente;

    Task Body empleado is
    Begin
        loop
            SELECT
                accept Prioritario (D: IN texto; R: OUT texto) do
                    R := resolverPedidoPrioritario(D);
                end Prioritario;
            OR
                when (Prioritario'count = 0) =>
                    accept Regular (D: IN texto; R: OUT texto) do
                        R := resolverPedidoRegular(D);
                    end Regular;
            END SELECT;
        end loop;
    End empleado;

Begin
    null;
End Banco3;
```

## Ejercicio 4
Se debe modelar la atención en un banco por medio de **DOS** empleados. Los clientes llegan y son atendidos de acuerdo al orden de llegada por cualquiera de los dos.

Partimos de la solución del ejercicio 1, pero Empleado será un tipo y declararemos dos variables: EmpleadoA y EmpleadoB.
```
Procedure Banco4 is
    Task Type empleado is
        Entry Pedido (D: IN texto; R: OUT texto);
    End empleado;

    EmpleadoA, EmpleadoB: empleado;

    Task type cliente;

    arrClientes: array (1..N) of Cliente;

    Task Body cliente is
        Resultado: texto;
    Begin
        Empleado.Pedido ("datos", Resultado);
    End cliente;

    Task Body empleado is
    Begin
        loop
            accept Pedido (D: IN texto; R: OUT texto) do
                R := resolverPedido(D);
            end Pedido;
        end loop;
    End empleado;

Begin
    null;
End Banco4;
```
> ❌ El entry call debe ser a una tarea en particular y no a un tipo tarea. **¿A cuál de los dos se lo hace?**

El cliente no determina qué empleado lo debe atender: cuando un empleado está libre debe atender al primer cliente que esté esperando. Necesitaremos una tarea **Administrador** que almacena los pedidos de los clientes en orden, y los empleados le piden el siguiente.

Comencemos con los clientes:
```
    Task Administrador ...
    Task Type empleado ...
    Task type cliente ...

    EmpleadoA, EmpleadoB: empleado;
    arrClientes: array (1..N) of Cliente;

    Task Body cliente is
        Resultado: texto;
    Begin
        Administrador.Pedido ("datos", Resultado);
    End cliente;
```
> ❌ En este caso el Administrador no puede resolver el pedido del cliente para devolverle el resultado.

Necesitamos **dos comunicaciones**: el cliente le hace el pedido al administrador, y luego espera a que un empleado le envíe la respuesta.
```
    Task Administrador is
        entry Pedido (D: IN texto);
    End Administrador;

    Task type cliente is
        entry Respuesta (R: IN texto);
    End cliente;

    arrClientes: array (1..N) of Cliente;

    Task Body cliente is
        Resultado: texto;
    Begin
        Administrador.Pedido ("datos");
        Accept Respuesta (R: IN texto) do
            Resultado := R;
        end Respuesta;
    End cliente;
```
> **¿Cómo sabrá a quién contestarle el empleado?** Debería saber su identificador (posición en el vector arrClientes).

Para esto el cliente debería conocer ese identificador y enviarlo como parámetro en el Pedido hecho al Administrador. Pero él no lo conoce, alguien se lo tiene que enviar por medio de un entry call, por ejemplo en el cuerpo principal del programa.
```
    Task type cliente is
        entry Ident (Pos: IN integer);
        entry Respuesta (R: IN texto);
    End cliente;

    arrClientes: array (1..N) of Cliente;

    Task Body cliente is
        Resultado: texto;
        id: integer;
    Begin
        Accept Ident (Pos: IN integer) do
            id := Pos;
        end Ident;
        Administrador.Pedido (id, "datos");
        Accept Respuesta (R: IN texto) do
            Resultado := R;
        end Respuesta;
    End cliente;

Begin
    for i in 1..N loop
        arrClientes(i).Ident(i);
    end loop;
End Banco4;
```

Seguimos con los empleados: cada empleado le debe pedir al administrador el siguiente pedido (cliente) a atender, luego resuelve el pedido y le da la respuesta al cliente correspondiente.
```
    Task Administrador is
        entry Pedido (IdC: IN integer; D: IN texto);
        entry Siguiente (Id: OUT integer; DC: OUT texto);
    End Administrador;

    Task Type Cliente is
        entry Ident (Pos: IN integer);
        entry Respuesta (R: IN texto);
    End Cliente;

    Task Type Empleado;

    EmpleadoA, EmpleadoB: Empleado;
    arrClientes: array (1..N) of Cliente;

    Task Body Empleado is
        Res, Dat: texto;
        idC: integer;
    Begin
        loop
            Administrador.Siguiente (idC, Dat);
            Res := resolverPedido(Dat);
            arrClientes(idC).Respuesta(Res);
        end loop;
    End Empleado;
```

Seguimos con el Administrador: para evitar tener que encolar en forma explícita los pedidos de los clientes, podemos esperar hasta que haya un pedido de un empleado para aceptar el pedido del cliente (**accept anidado**).
```
    Task Body Administrador is
    Begin
        loop
            Accept Siguiente (Id: OUT integer; DC: OUT texto) do
                Accept Pedido (IdC: IN integer; D: IN texto) do
                    Id := IdC;
                    DC := D;
                End Pedido;
            End Siguiente;
        end loop;
    End Administrador;
```

**Solución completa:**
```
Procedure Banco4 is
    Task Administrador is
        entry Pedido (IdC: IN integer; D: IN texto);
        entry Siguiente (Id: OUT integer; DC: OUT texto);
    End Administrador;

    Task Type Cliente is
        entry Ident (Pos: IN integer);
        entry Respuesta (R: IN texto);
    End Cliente;

    arrClientes: array (1..N) of Cliente;

    Task Type Empleado;

    EmpleadoA, EmpleadoB: Empleado;

    Task Body Administrador is
    Begin
        loop
            Accept Siguiente (Id: OUT integer; DC: OUT texto) do
                Accept Pedido (IdC: IN integer; D: IN texto) do
                    Id := IdC;
                    DC := D;
                End Pedido;
            End Siguiente;
        end loop;
    End Administrador;

    Task Body Empleado is
        Res, Dat: texto;
        idC: integer;
    Begin
        loop
            Administrador.Siguiente (idC, Dat);
            Res := resolverPedido(Dat);
            arrClientes(idC).Respuesta(Res);
        end loop;
    End Empleado;

    Task Body cliente is
        Resultado: texto;
        id: integer;
    Begin
        Accept Ident (Pos: IN integer) do
            id := Pos;
        end Ident;
        Administrador.Pedido (id, "datos");
        Accept Respuesta (R: IN texto) do
            Resultado := R;
        end Respuesta;
    End cliente;

Begin
    for i in 1..N loop
        arrClientes(i).Ident(i);
    end loop;
End Banco4;
```

## Ejercicio 5
Se debe modelar un buscador para contar la cantidad de veces que aparece un número dentro de un vector distribuido entre las N tareas contador. Además existe un administrador que decide el número que se desea buscar y se lo envía a los N contadores para que lo busquen en la parte del vector que poseen, y calcula la cantidad total.

**Primer intento:**
```
Procedure ContadorOcurrencias is
    Task Admin;

    Task type Contador is
        entry Contar (num: in integer; res: out integer);
    End Contador;

    ArrC: array (1..N) of Contador;

    Task body Admin is
        num: integer := elegirNumero;
        parcial, total: integer := 0;
    Begin
        for i in 1..N loop
            ArrC(i).Contar (num, parcial);
            total := total + parcial;
        end loop;
    End Admin;

    Task body Contador is
        vec: array (1..V) of integer := InicializarVector;
        cant: integer := 0;
    Begin
        Accept Contar (num: in integer; res: out integer) do
            for i in 1..V loop
                if (vec(i) = num) then
                    cant := cant + 1;
                end if;
            end loop;
            res := cant;
        end Contar;
    End Contador;

Begin
    null;
End ContadorOcurrencias;
```
> ❌ **SECUENCIAL.** No puede indicarle qué número buscar a un Contador hasta que el anterior no le devolvió su resultado.

**Segundo intento:** se debe separar el envío del número de la recepción de los resultados.
```
Procedure ContadorOcurrencias is
    Task Admin is
        entry Resultado (res: in integer);
    End Admin;

    Task type Contador is
        entry Contar (num: in integer);
    End Contador;

    ArrC: array (1..N) of Contador;

    Task body Admin is
        num: integer := elegirNumero;
        total: integer := 0;
    Begin
        for i in 1..N loop
            ArrC(i).Contar (num);
        end loop;
        for i in 1..N loop
            accept Resultado (res: in integer) do
                total := total + res;
            end Resultado;
        end loop;
    End Admin;

    Task body Contador is
        vec: array (1..V) of integer := InicializarVector;
        valor, cant: integer := 0;
    Begin
        Accept Contar (num: in integer) do
            valor := num;
        end Contar;
        for i in 1..V loop
            if (vec(i) = valor) then
                cant := cant + 1;
            end if;
        end loop;
        Admin.Resultado(cant);
    End Contador;

Begin
    null;
End ContadorOcurrencias;
```
> ⚠️ Posible demora si hay demora en una de las tareas (el Admin le asigna el número a los contadores en orden).
> La solución es **ACEPTABLE**, pero se puede mejorar evitando las demoras mencionadas.

**Solución mejorada:** para evitar asignar en orden el número a buscar (generando demoras si uno está demorado), se puede optar por la opción donde el Contador pide el número cuando está listo (**intercambiamos quién hace el entry call y quién el accept**).
```
Procedure ContadorOcurrencias is
    Task Admin is
        entry Valor (num: out integer);
        entry Resultado (res: in integer);
    End Admin;

    Task type Contador;

    ArrC: array (1..N) of Contador;

    Task body Contador is
        vec: array (1..V) of integer := InicializarVector;
        valor, cant: integer := 0;
    Begin
        Admin.Valor(valor);
        for i in 1..V loop
            if (vec(i) = valor) then
                cant := cant + 1;
            end if;
        end loop;
        Admin.Resultado(cant);
    End Contador;

    Task body Admin is
        numero: integer := elegirNumero;
        total: integer := 0;
    Begin
        for i in 1..2*N loop
            select
                accept Valor (num: out integer) do
                    num := numero;
                end Valor;
            or
                accept Resultado (res: in integer) do
                    total := total + res;
                end Resultado;
            end select;
        end loop;
    End Admin;

Begin
    null;
End ContadorOcurrencias;
```

---

# Patrones y situaciones de la práctica 5 (ADA / Rendezvous)

Resumen de lo que se repitió en la explicación (Ej. 1 a 5 del apunte) y en los ejercicios 1 al 9 de la práctica. Los nombres son genéricos (Cliente, Servidor, Admin, Worker): en cada ejercicio se reemplazan por los del enunciado.

## 1. ¿Qué patrón uso? (decisión rápida)

| El enunciado dice... | Patrón | Apareció en |
|---|---|---|
| cliente pide algo y espera la respuesta | **A.** Pedido con respuesta en un solo encuentro | Apunte 1, Práctica 2a, 5 |
| recurso con capacidad / condición, con distintos tipos de procesos | **B.** Un entry por tipo + `when` | Práctica 1a |
| "dar prioridad a..." | **C.** Prioridad con `'count` | Apunte 3, Práctica 1b, 4 |
| "espera a lo sumo X minutos" | **D.** Entry call temporal (`or delay`) | Apunte 2, Práctica 2b, 3, 4, 5 |
| "si no lo atienden inmediatamente" | **E.** Entry call condicional (`else`) | Práctica 2c, 3, 4 |
| "vuelve a intentarlo" / "hasta N veces" | **F.** Reintentos con bandera | Práctica 2d, 3, 4, 5, 9 |
| "durante X minutos recibe solo de..." | **G.** Timer aparte | Práctica 3 |
| "esperan hasta que estén todos" | **H.** Barrera en dos fases | Práctica 6 |
| un coordinador reparte trabajo y junta los resultados | **I.** Coordinador / Workers | Apunte 5, Práctica 7, 8 |
| varios servidores atienden por orden de llegada / hay que responderle a alguien después | **J.** Admin + ids (`Ident`) | Apunte 4, Práctica 9 |
| alguien deja algo y sigue sin esperar al que lo procesa | **K.** Tarea buffer con cola | Práctica 4 (Escritorio) |
| "esperar a lo sumo X a que **lo** llamen" (el que espera es quien acepta) | **L.** `select` de accept + `or delay` | Práctica 9 |

## 2. Reglas que se repiten en todos los ejercicios
1. **No hay variables compartidas.** Todo dato compartido vive como variable local de una tarea (Puente, Empresa, Escritorio) y los demás lo tocan solo por entries.
2. **Aprovechar la comunicación bidireccional:** si el que llama necesita una respuesta, va como `OUT` en el mismo entry y se calcula dentro del accept. No hacer dos comunicaciones cuando alcanza con una.
3. **Dentro del accept, solo lo que el otro necesita esperar.** El trabajo pesado (cruzar el puente, recolectar, procesar una señal, sumar un vector) va **fuera** del rendezvous. Si no, se pierde concurrencia.
4. **En el `when` no se pueden usar parámetros del entry.** Si la decisión depende de "quién es" o "de qué tipo es", hace falta **un entry distinto por tipo**.
5. **`'count` lo consulta solo la tarea dueña del entry**, y se usa para prioridades (`when (Prioritario'count = 0)`). No tiene sentido `when (X'count > 0) => accept X`.
6. **Nombrado asimétrico:** el que llama sabe a quién llama; el que acepta no sabe quién lo llamó. Conviene que **las muchas tareas llamen a la única** (así nadie necesita ids). Si alguien tiene que llamar a una instancia de un arreglo, esa instancia necesita conocer su id (entry `Ident` desde el programa principal).
7. **Una tarea que solo llama no necesita entries:** `task type Auto;`. Una tarea puede tener entries y además hacer entry calls a otras (Equipo, Admin).
8. **El `select` elige una alternativa solo si su `when` es verdadero y hay entry calls pendientes.** Si ninguna cumple, espera en el `select` completo. Un `accept` suelto, en cambio, espera ahí hasta que llamen.
9. **`or delay t` vs `else`:** "a lo sumo t" es `or delay t`; "inmediatamente" es `else`. En un mismo `select` va uno u otro, nunca los dos. Un `delay t;` suelto hace dormir a la tarea (otra cosa).
10. **Cuando alguien tiene que esperar a que pase algo con otros (todos llegaron, todos los resultados están), separar en dos fases con dos entries:** una para contar/recibir, otra para liberar/responder.
11. **Cada `accept` necesita su entry call (en la misma cantidad).** Si no, una tarea queda trabada para siempre.
12. **Si se sabe cuántas veces se repite algo, `for i in 1..N loop`.** Si puede cortar antes, `while` con bandera (o `exit`).

## 3. Patrones

### A. Pedido con respuesta en un solo encuentro (Apunte 1, Práctica 2a)
```ada
task Servidor is
   entry Pedido (D: IN texto; R: OUT texto);
end Servidor;

task body Cliente is
   res: texto;
begin
   Servidor.Pedido("datos", res);
end Cliente;

task body Servidor is
begin
   loop
      accept Pedido (D: IN texto; R: OUT texto) do
         R := resolver(D);
      end Pedido;
   end loop;
end Servidor;
```
El orden de llegada lo da la cola implícita del entry (FIFO).

### B. Un entry por tipo + `when` (Práctica 1a)
**Idea:** el recurso es una tarea con su estado local; cada tipo de proceso tiene su propio entry y su condición. La salida no lleva `when`.
```ada
task body Recurso is
   ocupado: integer := 0;
begin
   loop
      select
         when (ocupado + 1 <= MAX) => accept EntraTipo1 do ocupado := ocupado + 1; end EntraTipo1;
      or
         when (ocupado + 3 <= MAX) => accept EntraTipo3 do ocupado := ocupado + 3; end EntraTipo3;
      or
         accept Sale (cant: IN integer) do ocupado := ocupado - cant; end Sale;
      end select;
   end loop;
end Recurso;
```
El proceso hace `Recurso.EntraTipoX;` → usa el recurso (fuera del accept) → `Recurso.Sale(x);`.

### C. Prioridad con `'count` (Apunte 3, Práctica 1b, 4)
**Idea:** a las alternativas de **menor** prioridad se les agrega `and (EntryPrioritario'count = 0)`. La prioritaria y la salida no cambian.
```ada
select
   accept Prioritario (...) do ... end Prioritario;
or
   when (Prioritario'count = 0) => accept Regular (...) do ... end Regular;
or
   when (Prioritario'count = 0) and (Regular'count = 0) => accept Tarea_cuando_esta_libre ...;
end select;
```
El orden de las alternativas **no** da prioridad.

### D / E. Entry call temporal y condicional (Práctica 2b, 2c)
```ada
select                                   select
   Servidor.Pedido(d, r);                   Servidor.Pedido(d, r);
   -- solo si lo atendieron                 -- solo si lo atendieron
or delay 600.0                           else
   null;   -- se va                         null;   -- se va
end select;                              end select;
```
El `or delay` cuenta hasta que el servidor **empieza** el accept.

### F. Reintentos con bandera (Práctica 2d, 3, 4, 5)
```ada
atendido := false;
intentos := 0;
while (not atendido) and (intentos < 3) loop
   select
      Servidor.Pedido(d, r);
      atendido := true;                    -- solo si lo atendieron
   or delay 300.0
      intentos := intentos + 1;
      if (intentos < 3) then delay 600.0; end if;   -- espera antes de reintentar
   end select;
end loop;
```
- Reintentar **con el mismo dato** → generarlo fuera del `while` (Práctica 3, Periférico 2; Práctica 5).
- "Si no es inmediato, espera 1 min y reintenta" → `else delay 60.0;`.
- Combinación (Práctica 2d): un `select … else` **dentro** del `or delay` de otro `select`.

### G. Timer aparte (Práctica 3)
**Idea:** para "durante X minutos" no sirve `or delay X` en un loop (se reinicia en cada vuelta). Una tarea Timer cuenta el tiempo en paralelo y avisa con un entry.
```ada
task body Timer is
begin
   loop
      accept Iniciar;
      delay 180.0;
      Central.FinTiempo;
   end loop;
end Timer;

-- en la Central:
Timer.Iniciar;
fin := false;
while (not fin) loop
   select
      accept Senal2 (s: IN texto) do senal := s; end Senal2;
      procesar(senal);
   or
      accept FinTiempo;
      fin := true;
   end select;
end loop;
```

### H. Barrera en dos fases (Práctica 6)
```ada
task type Grupo is
   entry Llegada;
   entry Comenzar;
end Grupo;

-- Integrante:
grupos(miGrupo).Llegada;
grupos(miGrupo).Comenzar;      -- queda bloqueado hasta que estén todos

-- Grupo:
for i in 1..4 loop accept Llegada; end loop;
for i in 1..4 loop accept Comenzar; end loop;
```
Con un solo entry no funciona: el accept libera al que llega apenas lo atiende.

### I. Coordinador / Workers (Apunte 5, Práctica 7, 8)
**Idea:** separar **la orden / el dato de entrada** (accept rápido) de **la entrega del resultado**. Los workers calculan en paralelo, fuera de cualquier accept.
```ada
-- Worker:
loop
   Coordinador.PedirDato(dato);          -- dato: OUT
   resultado := calcular(dato);          -- en paralelo con los demás
   Coordinador.Resultado(resultado);
end loop;

-- Coordinador (un ciclo):
for i in 1..N loop
   accept PedirDato (d: OUT tipo) do d := dato; end PedirDato;
end loop;
for i in 1..N loop
   accept Resultado (r: IN tipo) do
      if (r > max) then max := r; end if;      -- o total := total + r
   end Resultado;
end loop;
```
- Primero los N `PedirDato` y después los N `Resultado`: así un worker rápido no toma dos veces el dato del mismo ciclo.
- **Trampa:** llamar a cada worker con el cálculo **dentro** del accept (`workers(i).Calcular(res)`) hace los N cálculos de a uno.
- No hace falta saber de qué worker viene cada resultado: un solo entry aceptado N veces.

### J. Admin + ids (Apunte 4, Práctica 9)
**Idea:** el cliente no elige quién lo atiende. Un Admin guarda los pedidos (en su cola implícita, o en arreglos/colas locales) y los servidores le piden el siguiente. Como el servidor después tiene que llamar al cliente, el cliente necesita su id.
```ada
-- Programa principal:
for i in 1..N loop clientes(i).Ident(i); end loop;

-- Cliente:
accept Ident (pos: IN integer) do id := pos; end Ident;
Admin.Pedido(id, datos);
accept Respuesta (r: IN texto) do res := r; end Respuesta;

-- Servidor:
loop
   Admin.Siguiente(idC, datos);
   r := resolver(datos);
   clientes(idC).Respuesta(r);
end loop;

-- Admin con accept anidado (sin cola explícita):
loop
   accept Siguiente (id: OUT integer; d: OUT texto) do
      accept Pedido (idC: IN integer; datos: IN texto) do
         id := idC;
         d := datos;
      end Pedido;
   end Siguiente;
end loop;
```
Si el Admin tiene que elegir con un criterio ("el que más reclamos hizo"), guarda contadores en arreglos locales y acepta `Siguiente` con `when (hayPendientes)`. Marcar a los ya asignados para no mandarle dos servidores al mismo (Práctica 9).

### K. Tarea buffer con cola (Práctica 4, Escritorio)
**Idea:** el que produce deja el dato y sigue; el que consume lo pide cuando puede. El buffer nunca entrega si está vacío.
```ada
task body Buffer is
   elems: cola;
begin
   loop
      select
         accept Dejar (e: IN texto) do push(elems, e); end Dejar;
      or
         when (not empty(elems)) =>
            accept Sacar (e: OUT texto) do pop(elems, e); end Sacar;
      end select;
   end loop;
end Buffer;
```
Si el consumidor final es una tarea que no puede salir a buscar (el Médico solo acepta), se agrega un intermediario (Secretario) que hace `Buffer.Sacar(e); Medico.Procesar(e);` en un loop.

### L. `select` de accept + `or delay` (Práctica 9)
**Idea:** el `or delay` también existe del lado del que **acepta**: "espero a lo sumo X a que me llamen".
```ada
while (not atendida) loop
   Empresa.Reclamo(miId);
   select
      accept Llegada;            -- me llamó el camión
      atendida := true;
   or delay 900.0
      null;                      -- vuelvo a reclamar
   end select;
end loop;
```
