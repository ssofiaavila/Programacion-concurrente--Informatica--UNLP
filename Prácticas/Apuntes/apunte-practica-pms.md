# Explicación práctica pasaje de mensajes sincrónicos (PMS)

## A tener en cuenta
- Los programas se componen SÓLO de procesos (no existen las variables compartidas).
- Los canales son de tipo link (un único emisor y un único receptor) y son sincrónicos. Son estructuras implícitas y no se deben declarar.
- Los procesos interactúan entre ellos únicamente por medio del envío de mensajes (tanto para comunicación como para sincronización por condición).
- No se requere sincronización por exlusión mutua ya que no existen las variables compartidas.

## Sintaxis
### Sentencias de comunicación
- **Sentencias de comunicación:** no existen las estructuras de canales, la comunicación se realiza nombrando al procesos con el cual se realiza la comunicación.
  1. **Envío(!):** la operación es bloqueante y sincrónica, se demora hasta que la recepción haya terminado.
    ```
        destino!port(mensaje);
        destino[i]!port(mensaje);
    ```
  2. **Recepción(?):** la operación es bloqueante y sincrónica.
    ```
        origen?port(variable);
        origen[i]?port(variable);
        origen[*]?port(variable);
    ```
    **port** debe coincidir de etiqueta tanto en el emisor como receptor.
    **[*]** es solamente usado en la recepción, indica que el origen puede ser cualquier proceso del arreglo. Si quiere contestarle a ese mismo, deberé usar algún identificador.

### Comunicación guardada
- Uso de comunicación guardada (id y do): permite seleccionar de forma no determinística entre varias alternativas de comunicación (en caso de la práctica de la práctica SOLO RECEPCIONES) en base a las condiciones del proceso y los mensajes que están listos para ser recibidos.
- Guardas: cada alternativa es una guarda con la forma: `B;C -> S`.
    1. `B`: condición booleana que puede no estar (en ese caso se considera true), e indica si el proceso está o no en condiciones de procesar el mensaje recibido en `C`.
    2. `C`: sentencia de comunicación (en la práctica sólo RECEPCIÓN) que seguro debe estar.
    3. `S`: conjunto de sentencias que se ejecutarán en caso de ser elegida la guarda.
- Evaluación de guarda:
    - `Exito`: la condición booleana es Verdadera (o no la tiene) y la comunicación se puede realizar sin producir demora (el emisor está esperando hacer la comunicación).
    - `Fallo`: la condición booleana es Falsa, sin importar lo que ocurra con la sentencia de comunicación.
    - `Bloqueo`: la condición boolean es Verdadera (o no la tiene) pero la comunicación NO se puede realizar sin producir demora (el emisor aún no llegó a la sentencia de envío).
- En el if `guardado` se evalúan todas las guarda en base a eso:
    - Si una o más son EXITOSAS se selecciona una de ellas en forma NO DETERMINISTICA, se ejecuta la sentencia de recepción `C` que forma parte de la guarda, y posteriormente el conjunto de sentencias asociadas a la guarda `S`.
    - Si todas las guardas FALLAN no se selecciona ninguna y se sale del IF sin realizar ninguna acción.
    - Si no hay ninguna guarda EXITOSA pero hay una o más con estado BLOQUEO entonces el proceso se demora en el IF hasta que haya una guarda exitosa. En ese momento se ejecuta igual que el primer caso.
- El do **guardado** funciona de la misma manera que el IF, con la única diferencia que en lugar de hacerlo una vez repite el mecanismo hasta que todas las guardas FALLAN.


# Ejemplo 1
En una empresa de software hay un empleado *Testeo* que prueba un nuevo producto para encontrar errores, cuando encuentra uno generan un reporte para que otro empleado *Mantenimiento* corrija el error y le responda. El empleado *Mantenimiento* toma los reportes para evaluarlos, hacer las correcciones necesarias y responderle al empleado *Testeo*.

*Habrá dos procesos: Testeo y Mantenimiento. Testeo le debe enviar el reporte a Mantenimiento y esperar a que le responda para continuar trabajando. Por lo tanto no se requiere modelar un buffer donde se vayan almacenando los reportes, ya que nunca habrá más de uno a la vez.*
```
process Testeo{
    texto R, Res;
    while (true){
        R= generarReporteConProblema;
        Mantenimiento!reporte(R);
        Mantenimiento?respuesta(Res);
    }
}

process Mantenimiento{
    texto Rep, Res;
    while(true){
        Testeo?reporte(Rep);
        Res= resolver(Rep);
        Testeo!respuesta(Res);
    }
}
```
Ambos procesos se comunican directamente entre ellos 2 veces por cada reporte generado, una con el *reporte* y otra con el *resultado*.

# Ejemplo 2
En una empresa de software hay un empleado *Testeo* que prueba un nuevo producto para encontrar errores, cuando encuentra uno generan un reporte para que otro empleado *Mantenimiento* corrija el error **(no requiere una respuesta para seguir trabajando) y continua.** El empleado *Mantenimiento* toma los reportes para evaluarlos y hacer las correcciones necesarias.

**Partimos de la solución del ejercicio 1, sin cambios se limitará la concurrencia porque el proceso Testeo esperará a que Mantenimiento termine en generar el reporte.** Por esa razón, habrá que usar un **Buffer**.

```
process Testeo {
    texto R, Res;
    while (true){
        R= generarReporteConProblema();
        Admin!reporte(R);
    }
}

process Mantenimiento {
    texto Rep, Res;
    while (true){
        Admin!pedido();
        Admin?reporte(Rep);
        Res= resolver(Rep);
    }
}

process Admin{
    cola Buffer;
    texto R;
    do Testeo?reporte(R) --> push(Buffer,R);
        not empty(Buffer); Mantenimiento?pedido() --> Mantenimiento!reporte(pop(Buffer));
}
```
# Ejemplo 3
En una empresa de software hay *N empleados Testeo* que prueban un nuevo producto para encontrar errores, cuando encuentran uno generan un reporte para que otro empleado *Mantenimiento* corrija el error y le responda. El empleado *Mantenimiento* toma los reportes para evaluarlos de acuerdo al orden de llegada, hace las correcciones necesarias y le responde al empleado *Testeo* correspondiente.

*Partimos de la solución del ejemplo 1 pero con N procesos Testeo.*
```
process Admin{
    cola Fila; texto R; int idT;
    do Testeo[*]?reporte(R, idT) --> push(fila, (R, idT));
        not empty(Fila); Mantenimiento?pedido() --> pop(Fila, (R, idT));
                                                    Mantenimiento!reporte(R, idT);
}

process Testeo[id: 1..N] {
    texto R, Res;
    while (true){
        R= generarReporteConProblema();
        Admin!reporte(R, id);
        Mantenimiento?respuesta(Res);
    }
}

process Mantenimiento {
    texto Rep, Res;
    int idT;
    while (true){
        Admin!pedido();
        Admin?reporte(Rep, idT);
        Res= resolver(Rep);
        Testeo[idT]!respuesta(Res);
    }
}
```

# Ejemplo 4
En una empresa de software hay N Empleados *Testeo* que prueban un nuevo producto para encontrar errores, cuando encuentra uno generan un reporte para que uno de los *3 empleados Mantenimiento* corrija el error y le responda. Los empleados *Mantenimiento* toma los reportes para evaluarlos de acuerdo al orden de llegada, hace las correcciones necesarias y le responde al empleado *Testeo* correspondiente.

```
process Admin {
    cola Fila; 
    texto R;
    int idT, idM;
    do Testeo[*]?reporte(R, idT) --> push( Fila, (R, idT));
        not empty(Fila); Mantenimiento[*]?pedido(idM) --> pop( Fila, (R, idT));
                                                            Mantenimiento[idM]!reporte(R,idT);
    od
}

process Testeo[id:1..N]{
    texto R, Res;
    while (true){
        R= generarReporteConProblema();
        Admin!reporte(R,id);
        Mantenimiento[*]?respuesta(Res);
    }
}

process Mantenimiento[id:1..3]{
    texto Rep, Res;
    int idT;
    while (true){
        Admin!pedido(id);
        Admin?reporte(Rep,idT);
        Res= resolver(Rep);
        Testeo[idT]!respuesta(Res);
    }
}
```

# Explicación simple del `do` del Admin (Ejemplo 4)
```
do Testeo[*]?reporte(R, idT) --> push(Fila, (R, idT));
   not empty(Fila); Mantenimiento[*]?pedido(idM) --> pop(Fila, (R, idT));
                                                     Mantenimiento[idM]!reporte(R, idT);
od
```

## La idea con un ejemplo de la vida real
Pensar al Admin como una **secretaria con una bandeja de papeles** (la `Fila`). La secretaria atiende dos tipos de personas:
- **Testeo:** viene a **dejar** un reporte.
- **Mantenimiento:** viene a **pedir** un reporte para trabajar.

La secretaria se queda todo el día en su escritorio repitiendo lo mismo: *"¿quién viene? lo atiendo y vuelvo a esperar"*. Ese "repetir siempre" es el `do ... od`.

## Línea 1: alguien de Testeo viene a dejar un reporte
```
Testeo[*]?reporte(R, idT) --> push(Fila, (R, idT));
```
- "Si algún Testeo (cualquiera, por eso el `*`) viene a dejar un reporte, lo recibo y lo pongo en la bandeja."
- No tiene condición antes del `;` porque **siempre** se aceptan reportes nuevos.
- Como con `*` no se sabe quién mandó el mensaje, el Testeo manda también su `idT`. Así, más adelante, Mantenimiento sabe a quién responderle.

## Línea 2: alguien de Mantenimiento viene a pedir trabajo
```
not empty(Fila); Mantenimiento[*]?pedido(idM) --> pop(Fila, (R, idT));
                                                  Mantenimiento[idM]!reporte(R, idT);
```
- "Si **hay papeles en la bandeja** y **algún Mantenimiento vino a pedir**, saco el primer reporte y se lo doy a ese Mantenimiento."
- `not empty(Fila)` evita atender a Mantenimiento cuando no hay nada para darle. Si la bandeja está vacía, esta línea ni se considera.
- Mantenimiento manda su `idM` para que el Admin sepa a cuál de los 3 contestarle.

## ¿Y si pasan las dos cosas a la vez?
Si al mismo tiempo hay un Testeo esperando para dejar un reporte y un Mantenimiento esperando para pedir uno (con la bandeja no vacía), el Admin **elige uno cualquiera** (no determinismo). Al otro lo atiende en la próxima vuelta del `do`.

## ¿Y si no viene nadie?
El Admin **se queda esperando** en el `do` hasta que llegue alguien que pueda atender. No gasta tiempo preguntando.

## ¿Cuándo termina?
Nunca. Un `do` termina solo cuando **todas** las condiciones son falsas, y la línea 1 no tiene condición (siempre es verdadera). Es lo que se busca para un administrador.

## ¿Por qué Mantenimiento tiene que "pedir" en vez de que el Admin le mande directamente?
Porque en la práctica, **en las guardas sólo se puede recibir (`?`), nunca enviar (`!`)**. Entonces el Admin no puede tener una guarda que diga "le mando a Mantenimiento". La solución es que Mantenimiento avise que está libre (`Admin!pedido(id)`) y el Admin, al recibir ese aviso, le responde con el reporte.

Ventaja: el Admin nunca se queda trabado esperando a un proceso en particular. Si los Mantenimiento están ocupados, sigue juntando reportes en la bandeja.

## Resumen en una frase
> El Admin repite siempre: *"si llega un reporte, lo guardo; si tengo reportes guardados y un Mantenimiento me pide, le doy uno"*.
