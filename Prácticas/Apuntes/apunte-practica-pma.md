# Explicación práctica pasaje de mensajes asincrónicos (PMA)
## A tener en cuenta
- Los programas se componen **solo** de procesos y canales (NO EXISTEN LAS VARIABLES COMPARTIDAS).
- Los canales actúan como "colas" de mensajes enviados y no recibidos. Son de tipo mailbox (todos los procesos los pueden usar para enviar o recibir mensajes).
- Los procesos interactúan entre ellos ÚNICAMENTE por medio del envío de emnsajes (tanto para comunicación como para sincronización por condición).
- No se requiere sincronización por exclusión mutua ya que no existen las variables compartidas.

## Sintaxis

**Declaración de canales** se deben declarar los canales indicando la estructura de datos de los mensajes, puede ser
1. Un canal
   ```
   chan nombreCanal(tipoDato);
   ```
2. Un arreglo (de una o más dimensiones) de canales
    ``` 
    chan nombreArreglo[1..m](tipoDato);
    ```

**Sentencias de comunicación (`send`/`receive`)**: el uso de cada canal es atómico, por lo que no se harán al mismo tiempo 2 operaciones (send y/p receive) sobre el mismo canal.
1. **Envio (send):** la operación es no bloqueante, deposita el mensaje al final del canal y continúa su ejecución.
    ```
    send nombreCanal (mensaje);
    send nombreArreglo[i](mensaje);
    ```
2. **Recepción (receive):** la operación es bloqueante, si el canal está vacío se demora hasta que haya al menos un mensaje en él, luego saca el primer mensaje del canal (el más viejo).
    ```
    receive nombreCanal(variables para el mensaje);
    receive nomreArreglo[i](variables para el mensaje);
    ```
**Consultas por mensajes pendientes (empty):** esta función retorna un booleano que indica si el canal está vacío o no. **Usar con cuidado cuando el canal tiene más de un posible receptor.
```
empty (nombreCanal);
empty(nombreArreglo[i]);
```
**Uso de sentencias de alternativa múltiple (IF no determinístico) y alternativas iterativa múltiple (DO no determinístico):** puede generar busy waiting que en PMA está permitido (igual hay que tratar de evitarlo).

## A tener en cuenta
- Lo primero es definir la estructura del programa: qué procesos y cómo se van a comunicar.
- Los canales actáun como COLAS de mensajes, por lo que mantienen el orden de los mismos.
- Los canales son compartidos por todos los procesos.
- Por ser PMA, el *send* no bloquea al emisor.
- Se puede utilizar el if/do no determinístico, donde cada opción es una condición booleana donde se puede preguntar por variables locales y/o por empty de canales.
```
    if (cond1) -> acciones 1;
        (cond 2) -> accuibes 2;
        (cond N) -> acciones N;
    end if;
    De todas las conficiones sea verdadera elige una en forma no determinística y ejecuta las acciones correspondientes. Si ninguna es verdadera, sale del if/do sin ejecutar acción alguna.
```
- Se debe evitar hacer ***bussy waiting*** siempre que sea posible.
- En todos los ejercicios el tiempo debe representarse con la función **delay**.

# Ejemplos
## Ejemplo 1
En una empresa de software hay N personas que prueban un nuevo producto para encontrar errores, cuando encuentran uno generan un reporte para que un empleado corrija el error (las personas no deben recibir ninguna respuesta). El empleado toma los reportes de acuerdo al orden de llegada, los evalúan y hace las correcciones necesarias.
```
chan Reportes(texto);

process Persona[id:0..N-1]{
    texto R;
    while (true){
        R= generarReporteConProblema();
        send Reportes(R);
    }
}

process Empleado {
    texto Rep;
    while (true){
        receive Reportes(Rep);
        resolver(Rep);
    }
}
```

## Ejemplo 2
En una empresa de software hay N personas que prueban un nuevo producto para encontrar errores, cuando encuentran uno generar un reporte para que un empleado corrija el error y ***esperan la respuesta del mismo***. El empleado toma los reportes de acuerdo al orden de llegada, los evalúan, hace las correcciones necesarias y ***le responde a la persona que hizo el reporte***.

En este caso hay una interacción entre empleado y persona por lo cual habrá una sincronización. NO alcanza con un solo canal como en el ejemplo anterior.
Podría NO ser mi respuesta por lo que lo ideal es vectorizar las respuestas, quedando de la siguiente manera:
```
chan reportes(int, texto); 
chan Respuestas[N](texto);

process Persona[id:0..N-1]{
    texto R, res;
    while (true){
        R= generarReporteConProblema();
        send Reportes(id, R);
        receive Respuestas[id](res); // acá si no hay un mensaje, se queda esperando?
    }
}

process Empleado {
    texto, rep, res;
    int idP;
    while (true){
        receive Reportes(idP, rep);
        res= resolver(rep);
        send respuestas[idP](res);
    }
}
```

## Ejemplo 3
*Simular al ejercicio 2 pero ahora hay 3 empleados*
En una empresa de software hay N personas que prueban un nuevo producto para encontrar errores, cuando encuentran uno generan un reporte para que alguno de los 3 empleados corrija el error y esperan la respuesta del mismo. Los empleados toman los reportes de acuerdo al orden de llegada, los evalúan, hacen las correcciones necesarias y le responden a la persona que hizo el reporte.

> Las personas deben enviar sus reportes para que cualquiera de los empleados lo resuelva, se debe seguir usando un único canal para enviar los reportes. Sólo se debería usar un canal para cada empleado si la persona manda el reporte a UN empleado en particular, lo que no es este caso.
*La solución es la mis,a solo hay 3 empleados.*
```
chan reportes(int, texto);
chan respuestas[N](texto);

process Persona[id=0..N-1]{
    texto r, res;
    while (true){
        R= generarReporteConProblema();
        send reportes(id, r);
        receive respuestas[id](res);
    }
}

process Empleado[id=0..2]{
    texto rep, res;
    int idP;
    while (true){
        receive reportes(idP, rep);
        res= resolver(rep);
        send respuestas[idP](res);
    }
}
``` 

## Ejemplo 4
**Similar al ejemplo 1**
En una empresa de softrware hay N personas que prueban un nuevo producto para encontrar errores, cuando encuentran uno generan un reporte para que un empleado corrija el error (las personas no deben recibir ninguna respuesta). El empleado toma los reportes de acuerdo al orden de llegada, los evalúan y hace las correcciones necesarias; ***cuando no hay reportes para atender el empleado se dedica a leer durante 10 minutos.***

> En este caso el empleado no puede quedarse bloqueado haciendo un receive sobre el canal Reportes cuando está vacío.
> El empleado debe chequear antes de hacer el receive si hay algo en el canal, y si está vacio ponerse a leer por lo que se usará la función empty.

```
chan reportes(texto);

process Persona[id:0..N-1]{
    texto r;
    while (true){
        r= generarReporteConProblema();
        send reportes(r);
    }
}

process Empleado{
    texto rep;
    while (true){
        if (not empty(reportes)){
            receive reportes(rep);
            resolver(rep);
        } else {
            delay(600); // lee 10 minutos
        }
    }
}
```

## Ejemplo 5
**Se hace una modificación al ejercicio 4, donde habrá 3 empleados.**
En una empresa de software hay N personas que prueban un nuevo producto para encontrar errores, cuando encuentran uno generan un reporte para que *uno de los 3 empleados* corrija el error (las personas no deben recibir ninguna respuesta). Los empleados toman los reportes de acuerdo al orden de llegada, los evalúan y hacen las correcciones necesarias; cuando no hay reportes para atender los empleados se dedican a leer durante 10 minutos.


```
chan reportes(texto);

process Persona[id:0..N-1]{
    texto r;
    while (true){
        r= generarReporteConProblema();
        send reportes(r);
    }
}

process Empleado[id:0..2]{
    texto rep;
    while (true){
        if (not empty(reportes)){
            receive reportes(rep); // puede haber una posible demora
            resolver(rep);
        } else {
            delay(600); // lee 10 minutos
        }
    }
}
```

> Puede haber demora innecesaria ya que si dos o más empleados chequean por si el canal está vacío (supongamos que hay un solo mensaje) con la función empty, a todos les devolverá que no está vaciío, por lo que más de un empleado intentará hacer el receive y uno lo podrá hacer y el resto se bloqueará en el receive cuando en realidad deberían leer por 10 minutos.

Este problema se da porque estamos usando la función empty sobre un canal con múltiples receptores. Para evitar la demora innecesaria debemos:
  - **No usar el empty:** NO se puede evitar porque sino el empleado se quedaría si o si dormido en el receive y nunca leería.
  - **El canal no tenga múltiples receptores:** podría poner un canal Reportes para cada empleado, pero esto obligaría a que la persona le entregue su reporte a UN empleado en paticular (que no es lo pedido).

La única opción que queda es poner un proceso intermedio Coordinador que se encargue de recibir los reportes por el único canal Reportes y que los empleados le pidan a ese proceso un reporte para resolver.
Para las personas estos cambios son "invisibles", por lo que esos procesos no se modifican.

> Un empleado le "pide" el siguiente reporte al Coordinador por medio de un canal Pedido, el cual devolverá el reporte a antender por un canal Siguiente.
```
chan reportes(texto);
chan pedido(int);
chan siguiente(texto);

process Empleado[id:0..2]{
    texto rep;
    while(true){
        send pedido(id);
        if (not empty(siguiente)){
            receive siguiente(rep); // sigue generando demora innecesaria
            resolver(rep);
        } else { delay(600)};
    }
}

process Coordinador {
    texto rep;
    int idE;
    while (true){
        receive Pedido(idE);
        if (not empty(reportes)){
            receive reportes(rep);
            send sigueinte(rep);
        }
    }
}
```
> Debería usar un canal privado para que cada empleado reciba la respuesta que es sólo para él.
```
chan reportes(texto);
chan pedido(int);
chan siguiente[3](texto);

process Empleado[id:0..2]{
    texto  rep;
    while(true){
        send pedido(id);
         // probablemente el Coordinador aún no alcanzó a antender su pedido por lo que el canal estará vaciío aunque haya reportes pendientes
        if (not empty(siguiente[id])){
            receive siguiente[id](rep);
            resolver(rep); 
        } else delay(600);
    }
}

procedure Coordinador{
    texto rep;
    int idE;
    while (true){
        receive pedido(idE);
        if (not empty(reportes)){
            receive reportes(rep);
            send siguiente[idE](rep);
        }
    }
}
```
> El coordinador tendrá que responderle siempre al empleado, haya o no reportes pendientes. El empleado tendrá que esperar la respuesta del Coordinador para saber si efectivamente hay o no reportes pendientes
```
chan reportes(texto);
chan pedido(int);
chan siguiente[3](texto);

process Empleado[id:0..2]{
    text rep;
    while (true){
        send Pedido(id);
        receive siguiente[id](rep);
        if (rep <> "VACIO"){
            resolver(rep);
        } else {delay(600)}        
    }
}

process Coordinar{
    texto rep;
    int idE;
    while (true){
        receive pedido(idE);
        if (empty(reportes)){
            rep= "VACIO";
        } else {
            receive reportes(rep);
        }
        send siguiente[idE](rep);
    }
}
```

La resolución completa quedaría:
```
chan reportes(texto);
chan pedido(int);
chan siguiente[3](texto);

process Persona[id:0..N-1]{
    texto r;
    while (true){
        r= generarReporteConProblema;
        send reportes(r);
    }
}

process Empleado[id:0..2]{
    text rep;
    while (true){
        send Pedido(id); // avisa al coordinador que quiere un pedido
        receive siguiente[id](rep);
        if (rep <> "VACIO"){
            resolver(rep);
        } else {delay(600)}        
    }
}

process Coordinar{
    texto rep;
    int idE;
    while (true){
        receive pedido(idE); // recibe qué empleado quiere un pedido
        if (empty(reportes)){
            rep= "VACIO"; // si no hay lo asigna como vacío
        } else {
            receive reportes(rep);
        }
        send siguiente[idE](rep); // envía la información al empleado que solicitó el pedido
    }
}
```

# Patrones y cosas a tener en cuenta (aprendidas en la práctica 4 - PMA)

## Cómo encarar un ejercicio
1. Identificar **quién produce** mensajes y **quién los consume** (clientes, empleados, impresoras, etc.).
2. Preguntarse: ¿el emisor **espera una respuesta**? → necesita un canal privado de respuesta.
3. Preguntarse: ¿los consumidores hacen **otra cosa si no hay trabajo**? → hay `empty` con varios receptores → probablemente necesite un **Coordinador**.
4. Preguntarse: ¿hay **prioridades** entre distintos tipos de pedidos? → patrón de canal de aviso.
5. Preguntarse: ¿los procesos **deben terminar**? → contar mensajes y mandar un mensaje de fin.
6. Recién ahí declarar los canales y escribir los procesos.

## Patrón 1: canal único compartido (cualquiera atiende)
Cuando el pedido lo puede atender **cualquier** consumidor, se usa **un solo canal** que todos leen. Pasar de 1 a K consumidores no cambia nada más que la cantidad de procesos (Ej. 1a → 1b, Ej. 5a).
```
chan trabajos(text);

process Productor[i=1..N]{ ... send trabajos(t); ... }
process Consumidor[i=1..K]{
    while (true){
        receive trabajos(t);
        procesar(t);
    }
}
```
> Si el consumidor tiene que atender una cantidad fija conocida, usar `for`; si es indefinido, `while (true)`.

## Patrón 2: respuesta privada (canal vectorizado por id)
Si el emisor espera una respuesta, manda **su id junto al pedido** y espera en **su propio canal** `respuesta[id]` (Ej. 2 comprobantes, Ej. 3 paquetes).
```
chan pedidos(int, text);
chan respuestas[N](text);

// cliente
send pedidos(id, pedido);
receive respuestas[id](res);

// servidor
receive pedidos(idC, pedido);
send respuestas[idC](resultado);   // ojo: usar el id RECIBIDO, no el índice del proceso
```

## Patrón 3: Coordinador para el `empty` con múltiples receptores
Si varios consumidores tienen que hacer otra tarea cuando no hay trabajo (Ej. 1c, Ej. 3), **no** hacer `if not empty(canal)` en cada consumidor (demora innecesaria). Se agrega un Coordinador que es el **único** que consulta el `empty`:
- El trabajador manda `pedido(id)` y se bloquea en `siguiente[id]`.
- El Coordinador **siempre responde**: un valor real o un valor centinela (`'VACIO'`, `-1`).
```
process Trabajador[i=1..K]{
    while (true){
        send pedido(i);
        receive siguiente[i](t);
        if (t == 'VACIO') delay(...);   // tarea alternativa
        else procesar(t);
    }
}

process Coordinador{
    while (true){
        receive pedido(idT);
        if (empty(trabajos)) t = 'VACIO';
        else receive trabajos(t);
        send siguiente[idT](t);
    }
}
```
> Los productores no cambian: el Coordinador es "invisible" para ellos.

## Patrón 4: prioridad entre canales con canal de aviso (sin busy waiting)
Cuando hay dos (o más) canales y uno tiene prioridad (Ej. 5b, Ej. 4b), el servidor no puede bloquearse en un solo canal, y hacer `if/do` sobre `empty` en un loop genera **busy waiting**. Solución: cada emisor, **después** de mandar su dato, manda una señal a un canal común de aviso.
```
// emisor prioritario
send prioritarios(dato);
send hayPendiente();          // primero el dato, DESPUÉS el aviso

// emisor normal
send normales(dato);
send hayPendiente();

// servidor
receive hayPendiente();       // se bloquea hasta que haya algo (no hay busy waiting)
if (not empty(prioritarios)) receive prioritarios(dato);
else receive normales(dato);
```
- El orden **dato → aviso** es clave: así cuando el servidor recibe el aviso, el dato ya está en algún canal y el `receive` posterior nunca se bloquea.
- Hay un aviso por cada mensaje, entonces la cantidad de `receive hayPendiente()` coincide con la de datos.
- La misma idea sirve para un **Admin que atiende distintos tipos de pedido** (Ej. 2: asignación de caja / salida de caja): se puede mandar el tipo de operación en el aviso (`send aviso('salida')`) y según eso hacer el `receive` del canal correspondiente.

## Patrón 5: combinar Coordinador + "worker listo"
Si además de prioridades hay **varios servidores** (3 impresoras), el servidor avisa que está libre y el Coordinador le asigna el trabajo por su canal privado:
```
process Impresora[i=1..3]{
    while (...){
        send impresoraLista(i);
        receive asignado[i](doc);
        imprimir(doc);
    }
}

process Coordinador{
    while (...){
        receive impresoraLista(idI);
        receive hayPendiente();
        if (not empty(prioritarios)) receive prioritarios(doc);
        else receive normales(doc);
        send asignado[idI](doc);
    }
}
```

## Patrón 6: terminación de todos los procesos
Cuando se pide que todos terminen (Ej. 5c, 5d):
- Los productores usan `for` con la cantidad fija de mensajes.
- El Coordinador **conoce el total** de mensajes (ej. `N*10`, `(N+1)*10` si se suma el director) y hace un `for` con esa cantidad.
- Al terminar, manda un **mensaje de fin** (`'FINALIZAR'`) a **cada** trabajador por su canal privado.
- El trabajador itera con un `while` sobre un booleano que pone en `false` al recibir el mensaje de fin.
```
process Trabajador[i=1..K]{
    boolean seguir = true;     // ¡inicializar!
    while (seguir){
        send listo(i);
        receive asignado[i](t);
        if (t == 'FINALIZAR') seguir = false;
        else procesar(t);
    }
}
```
> Los trabajadores que se bloquean directamente en el canal compartido (sin Coordinador) no pueden recibir un fin "dirigido"; en ese caso hay que mandar K mensajes de fin al canal compartido, uno por trabajador.

## Busy waiting en PMA
- **Hay** busy waiting si un proceso pregunta `empty` en un loop sin ningún `receive` bloqueante que lo frene.
- **No hay** busy waiting si en cada vuelta el proceso se bloquea en algún `receive` (como en los patrones 3, 4 y 5).
- Aunque en PMA está permitido, siempre intentar evitarlo.

## Checklist de errores comunes antes de entregar
- [ ] **Declarar todos los canales** que se usan (con el tipo de dato de los mensajes, también los arreglos `canal[K]`).
- [ ] Sintaxis: `send canal(dato)` / `receive canal(variable)` (el nombre del canal va afuera del paréntesis).
- [ ] Usar `process`, no `procedure`, para los procesos.
- [ ] Al responder, usar el **id recibido** en el mensaje (`respuestas[idC]`) y no el índice del proceso servidor.
- [ ] No reutilizar como variable de un `for` el mismo nombre que el id del proceso (`process A[i=1..N]` con `for (i=1..10)` pisa el id → usar `j`).
- [ ] Inicializar las variables de control (booleanos, contadores, arreglos de cantidades).
- [ ] La cantidad de procesos debe coincidir con el enunciado (si son C clientes, `Cliente[i=1..C]` y `chan respuestas[C]`).
- [ ] Tiempos en `delay` coherentes con el enunciado (10 min = `delay(600)`, 15 min = `delay(900)`; "entre 1 y 3 minutos" = `delay(random(60,180))`).
- [ ] Revisar que las ramas de los `if` del Admin/Coordinador hagan el `receive` del canal que corresponde a esa operación.
- [ ] Si el enunciado dice "maximizar la concurrencia", que ningún proceso haga trabajo que podría hacer otro en paralelo (ej. el cocinero entrega directo al cliente, sin pasar por el vendedor).