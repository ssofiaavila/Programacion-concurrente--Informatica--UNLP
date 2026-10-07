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