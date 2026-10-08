# Pasaje de mensajes asincrónico (PMA)
### Ejercicio 1
Suponga que N clientes llegan a la cola de un banco y que serán atendidos por sus empleados. Analice el problema y defina qué procesos, recursos y canales/comunicaciones
serán necesarios/convenientes para resolverlo. Luego, resuelva considerando las siguientes situaciones:
- a. Existe un único empleado, el cual atiende por orden de llegada.
  ```
  chan atencion(int);

  process Persona[i=1..N]{
    llegarAlBanco();
    send atencion(i);
  }

  process Empleado{
    int proximo;
    for (i=1..N){
        receive atencion(proximo);
        atender(proximo);
    }
  }
  ```
- b. Ídem a. pero considerando que hay 2 empleados para atender, ¿qué debe modificarse en la solución anterior?
    ```
    es igual que la resolución a. pero process Empleado[i=1..2] y un while

    chan atencion(int);

    process Persona[i=1..N]{
        llegarAlBanco();
        send atencion(i);
    }

    process Empleado[i=1..2]{
        int proximo;
        while(true){
            receive atencion(proximo);
            atender(proximo);
        }
    }
    ```
- c. Ídem b. pero considerando que, si no hay clientes para atender, los empleados realizan tareas administrativas durante 15 minutos. ¿Se puede resolver sin usar
procesos adicionales? ¿Qué consecuencias implicaría?
    ```
    chan atencion(int);
    chan siguiente[2](int);

    process Persona[i=1..N]{
        llegarBanco();
        send atencion(i);
    }

    process Coordinador{
        int proximo;
        int idE;
        while (true){
            receive pedido(idE);
            if (empty(atencion)){
                proximo= -1;
            } else {
                receive atencion(proximo);
            }
            send siguiente[idE](proximo);
        }
    }

    process Empleado[i=1..N]{
        int atender;
        while(true){
            send pedido(i);
            receive siguiente[i](atender);
            if (atender == -1){
                delay(600);
            } else {
                atender(atender);
            }
        }
    }
    ```
### Ejercicio 2
Se desea modelar el funcionamiento de un banco en el cual existen 5 cajas para realizar pagos. Existen P clientes que desean hacer un pago. Para esto, cada uno selecciona la caja
donde hay menos personas esperando; una vez seleccionada, espera a ser atendido. En cada caja, los clientes son atendidos por orden de llegada por los cajeros. Luego del pago, se les entrega un comprobante. Nota: maximizar la concurrencia.

```
```

### Ejercicio 3
Se debe modelar el funcionamiento de una casa de comida rápida, en la cual trabajan 2 cocineros y 3 vendedores, y que debe atender a C clientes. El modelado debe considerar
que:
- Cada cliente realiza un pedido y luego espera a que se lo entreguen.
- Los pedidos que hacen los clientes son tomados por cualquiera de los vendedores y se lo pasan a los cocineros para que realicen el plato. Cuando no hay pedidos para atender,
los vendedores aprovechan para reponer un pack de bebidas de la heladera (tardan entre 1 y 3 minutos para hacer esto).
- Repetidamente cada cocinero toma un pedido pendiente dejado por los vendedores, lo cocina y se lo entrega directamente al cliente correspondiente.
Nota: maximizar la concurrencia.

```
chan pedidos(text,int);
chan paquetes[C](text);
chan vendedorListo(int);
chan pedidosPorTomar[3];
chan pedidosPorHacer(text,int);

process Cliente[i=1..N]{
    text pedido= generarPedido();
    send pedidos(pedido,i);
    receive paquetes[i](paquete);
}

procedure Vendedor[i=1..3]{
    text prox;
    int cli;
    while (true){
        send vendedorListo(i);
        receive pedidosPorTomar[i](prox, cli);
        if (prox == 'VACIO'){
            delay(600);
        } else {
            pedidosPorHacer(prox, id);
        }
    }
}

procedure Coordinador{
    text pedido;
    int idV;
    int cli;
    while (true){
        receive vendedorListo[idV];
        if (empty pedidos){
            pedido=('VACIO');
            cli=-1;
        } else {
            receive pedidos(pedido,cli);
        }
        send pedidosPorTomar[idV](pedido,cli);
    }
}

process Cocinero[i=1..2]{
    text pedido;
    int cli;

    while(true){
        receive pedidosPorHacer(pedido,cli);
        pedido= HacerPedido(pedido);
        send paquetes[cli](pedido);
    }
}
```

### Ejercicio 4
Simular la atención en un locutorio con 10 cabinas telefónicas, el cual tiene un empleado que se encarga de atender a N clientes. Al llegar, cada cliente espera hasta que el empleado le indique a qué cabina ir, la usa y luego se dirige al empleado para pagarle. El empleado atiende a los clientes en el orden en que hacen los pedidos. A cada cliente se le entrega un ticket factura por la operación.

- a. Implemente una solución para el problema descrito.
  ```
  ```
- b. Modifique la solución implementada para que el empleado dé prioridad a los que terminaron de usar la cabina sobre los que están esperando para usarla.
  ```
  ```

### Ejercicio 5
Resolver la administración de 3 impresoras de una oficina. Las impresoras son usadas por N administrativos, los cuales están continuamente trabajando y cada tanto envían documentos a imprimir. Cada impresora, cuando está libre, toma un documento y lo imprime, de acuerdo con el orden de llegada.
- a. Implemente una solución para el problema descrito.
  ```
  ```
- b. Modifique la solución implementada para que considere la presencia de un director de oficina que también usa las impresoras, el cual tiene prioridad sobre los administrativos.
  ```
  ```
- c. Modifique la solución (a) considerando que cada administrativo imprime 10 trabajos y que todos los procesos deben terminar su ejecución.
  ``` 
  ```
- d. Modifique la solución (b) considerando que tanto el director como cada administrativo imprimen 10 trabajos y que todos los procesos deben terminar su ejecución.
  ```
  ```
- e. Si la solución al ítem (d) implica realizar Busy Waiting, modifíquela para evitarlo.
  ```
  ```
  



