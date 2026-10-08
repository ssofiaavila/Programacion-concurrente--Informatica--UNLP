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

