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
chan clienteEsperando(int);
chan nroCajaAsignada[P](int);
chan comprobantes[P](text);
chan salidaCaja(int);
chan clienteAvisa(text);

process Cliente[i=1..P]{
    int cajaAsignada;
    text comprobante;

    send clienteEsperando(i); // avisa que está esperando para que le asignen una caja
    send clienteAvisa('asignacion');
    receive nroCajaAsignada[i](cajaAsignada); // obtiene qué caja usará
    send avisarLlegada[cajaAsignada](i); // avisa a la caja asignada que llegó
    receive comprobantes[i](comprobate); // obtiene su comprobante
    send salidaCaja[cajaAsignada](cajaAsignada);
    send clienteAvisa('salida');
}

process Admin{
    int idC;
    cantEsperando int[5];
    text operacion;

    while (true){
        receive clienteAvisa(operacion);
        if (operacion= 'asignacion'){
            receive salidaCaja(cajaLiberada);
            cantEsperando[cajaLiberada]--;
        } else {
            receive clienteEsperando(idC);
            cajaMenor= obtenerMenorEsperando(cantEsperando); // función para saber qué caja asignarle
            cantEsperando[cajaMenor]++;
            send nroCajaAsignada[i](cajaMenor);
        }   

    }        
}


process Caja[i=1..5]{
    int siguiente;

    while (true){
        receive avisarLlegada[i](siguiente);
        comprobante= generarComprobate(siguiente);
        send comprobantes[siguiente](comprobante);
    }
}
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
            delay(600); // podría ser un random entre 1 y 3
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
        receive vendedorListo(idV);
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
  chan archivos(text);

  procedure Administrativo[i=1..N]{
    text archivo;

    while (true){
        archivo= generarArchivo();
        send archivo(archivos);
    }
  }

  process Impresora[i=1..3]{
    text archivo;

    while (true){
        receive archivos(archivo);
        imprimir(archivo);
    }
  }
  ```
- b. Modifique la solución implementada para que considere la presencia de un director de oficina que también usa las impresoras, el cual tiene prioridad sobre los administrativos.
  ```
  chan archivos(text);
  chan archivosDirector(text);
  chan pendienteImpresion();

  process Administrativo[i=1..N]{
    text archivo;
    while (true){
        archivo= generarArchivo();
        send archivos(archivo);
        send pendienteImpresion();

    }
  }

  process Director{
    text archivo;
    while (true){
        archivo= generarArchivo();
        send archivosDirector(archivo);
        send pendienteImpresion();
    }
  }

    procedure Coordinador{
        text archivo;
        int idI;
        while (true){
            receive impresoraLista(idI);
            receive pendienteImpresion(); // estructura de aviso cuando hay varios canales y necesitamos cierta sincronización !!!
            if (not empty(archivoDirector)){
                receive archivosDirector(archivo);
            } else {
                receive archivos(archivo);
            }
            send pendientesImprimir[idI](archivo);
        }
    }
  process Impresora[i=1..3]{
    text archivo;
    while (true){
        send impresoraLista(i);
        receive pendientesImprimir[i](archivo);
        imprimir(archivo);
    }
  }
  ```
- c. Modifique la solución (a) considerando que cada administrativo imprime 10 trabajos y que todos los procesos deben terminar su ejecución.
   ``` 
  process Administrativo[i=1..N]{
    text archivo;
    for (i=1..10){
        archivo= generarArchivo();
        send archivos(archivo);
    }
  }
  procedure Coordinador{
        text archivo;
        int idI, cant= N*10;
        for (i=1..cant){
            receive impresoraLista(idI);
            receive archivos(archivo);
            send pendientesImprimir[idI](archivo);
        }
        for(i=1..3){
            send pendientesImprimir[i]('FINALIZAR');
        }
  }

  process Impresora[i=1.3]{
        text archivo;
        boolean quedanDocs;

        while(quedanDocs){
                send impresoraLista(i);
                receive pendientesImprimir[i](archivo);
                if (archivo === 'FINALIZAR'){
                    quedanDocs= false;
                } else {
                    imprimir(archivo);
                }
        }
    }
  ```
- d. Modifique la solución (b) considerando que tanto el director como cada administrativo imprimen 10 trabajos y que todos los procesos deben terminar su ejecución.
  ```
  chan archivos(text);
  chan archivosDirector(text);
  chan pendienteImpresion();

  process Administrativo[i=1..N]{
    text archivo;
    for (i=1..10){
        archivo= generarArchivo();
        send archivos(archivo);
        send pendienteImpresion()
    }
  }

  process Director{
    text archivo;
    for (i=1..10){
        archivo= generarArchivo();
        send archivosDirector(archivo);
        send pendienteImpresion()
    }
  }

    procedure Coordinador{
        text archivo;
        int idI, cant= (N+1)*10;
        for(i=1..cant){
            receive impresoraLista(idI);
            receive pendienteImpresion();
            if (not empty(archivoDirector)){
                receive archivosDirector(archivo);
            } else {
                receive archivos(archivo);
            }
            send pendientesImprimir[idI](archivo);
        }
        for(i=1..3){
            send pendientesImprimir[i]('FINALIZAR');
        }
    }

  process Impresora[i=1..3]{
        text archivo;
        boolean quedanDocs;

        while(quedanDocs){
            send impresoraLista(i);
            receive pendientesImprimir[i](archivo);
            if (archivo === 'FINALIZAR'){
                quedanDocs= false;
            } else {
                imprimir(archivo);
            }
        }
  }
  ```

- e. Si la solución al ítem (d) implica realizar Busy Waiting, modifíquela para evitarlo.
  ```
  No hay busy waiting en la solución d.
  ```
  
#### Todos los ejercicios están corregidos con ayudante y están ok



