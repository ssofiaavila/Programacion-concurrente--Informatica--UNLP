# Práctica de repaso- memoria distribuida
## Pasaje de mensajes
1. En una oficina existen 100 empleados que envían documentos para imprimir en 5 impresoras compartidas. Los pedidos de impresión son procesados por orden de llegada y se asignan la primera impresora que se encuentre libre.
   - a. Implemente un programa que permita resolver el problema usando PMA.
    ```
        chan imprimir(integer, text);
        chan impresiones[1..100](text);

        process Empleado[i=1..100]{
            text archivo, impresion;

            send imprmir(i, archivo);
            receive impresiones[i](impresion);
        }

        process Impresora[i=1..5]{
            text archivo, impresion;
            int idE;
            while(true){
                receive imprimir(idE, archivo);
                impresion= imprimir(archivo);
                send impresiones[idE](impresion);
            }
        }
    ```
   - b. Resuelva el mismo problema anterior pero ahora usando PMS.
    ```
        process Empleado[i=1..100]{
            text archivo, impresion;

            Admin!enviarArchivo(i, archivo);
            Impresora[*]?impresiones(impresion);
        }

        process Admin {
            text archivo;
            int idE, idI;
            queue cola;
            do Empleado[*]?enviarArchivo(idE, archivo) -> push(cola,(idE,archivo));
                not empty(cola); Impresora[*]?lista(idI) -> pop(cola,(idE,archivo));
                                                            Impresora[idI]!proximo(idE, archivo);
            od;
        }

        process Impresora[i=1..5]{
            text archivo, impresion;
            int idE;
            while (true){
                Admin!listo(i);
                Admin?proximo(idE,archivo);
                imprimir(archivo, impresion);
                Empleado[idE]!impresiones(impresion);
            }
        }
    ```
2. Resolver el siguiente problema con PMS. En la estación de trenes hay una terminal de SUBE que debe ser usada por P personas de acuerdo con el orden de llegada. Cuando la persona accede a la terminal, la usa y luego se retira para dejar al siguiente. Nota: cada Persona usa sólo una vez la terminal. 
   ```
   process Persona[i=1..P]{
        Admin!avisar(i);
        Terminal?pasar();
        usarTerminal();
        Terminal!liberar();
   }

   process Admin{
        queue cola;
        int total=0, idP;
        
        do total < P; Persona[*]?avisar(idP) -> push(cola, idP);
                                                total++;
            !empty(cola); Terminal?listo() -> pop(cola, idP);
                                                Terminal!siguiente(idP);
        od
   }

   process Terminal{
        int idP;
        for (1..P){
            Admin!listo();
            Admin?siguiente(idP);
            Persona[idP]!pasar();
            Persona[idP]?liberar();
        }
   }
   ```
3. Resolver el siguiente problema con PMA. En un negocio de cobros digitales hay P personas que deben pasar por la única caja de cobros para realizar el pago de sus boletas. Las personas son atendidas de acuerdo con el orden de llegada, teniendo prioridad aquellos que deben pagar menos de 5 boletas de los que pagan más. Adicionalmente, las personas embarazadas tienen prioridad sobre los dos casos anteriores. Las personas entregan sus boletas al cajero y el dinero de pago; el cajero les devuelve el vuelto y los recibos de pago.
   ```
    chan filaEmbarazada(integer, text, integer);
    chan filaMenos(integer, text, integer);
    chan atencion();
    chan filaNormal(integer, text, integer);
    chan recibos[i1..P](integer, text);

    process Persona[i=1..P]{
        boolean embarazada;
        int cantBoletas, dinero;
        text boletas, recibos;

        embarazada= estaEmbarazada();
        cantBoletas= boletasAPagar();

        if (embarazada){
            send filaEmbarazada(dinero, boletas, i);
        } else {
            if (cantBoletas < 5){
                send filaMenos(dinero, boletas, i);
            } else {
                send filaNormal(dinero, boletas,i);
            }
        }
        send atencion();
        receive recibosListos[i](dinero, recibos);
    }
   ```

## ADA
1. Resolver el siguiente problema. La página web del Banco Central exhibe las diferentes cotizaciones del dólar oficial de 20 bancos del país, tanto para la compra como para la venta. Existe una tarea programada que se ocupa de actualizar la página en forma periódica y para ello consulta la cotización de cada uno de los 20 bancos. Cada banco dispone de una API, cuya única función es procesar las solicitudes de aplicaciones externas. La tarea programada consulta de a una API por vez, esperando a lo sumo 5 segundos por su respuesta. Si pasado ese tiempo no respondió, entonces se mostrará vacía la información de ese banco.
   ```ada
    procedure Cotizaciones is
        task type Banco is
            entry obtenerCotizacion(c: OUT integer);
        end Banco;

        task Cotizador;
        bancos: array(1..20) of Banco;

        task body Banco is
            integer cotizacion;
        begin
            loop 
                accept obtenerCotizacion(c: OUT integer) do
                    c:= cotizacion;
                end obtenerCotizacion;
            end loop
        end Banco;

        task body Cotizador is
            cotizaciones[20]= (0[20]);
            int i, cotizacion;
        begin
            for i in 1..20 loop
                select bancos(i).obtenerCotizacion(cotizacion);
                or delay 5.0
                    cotizaciones(i):= 0;
                end select;
            end loop;
        end Cotizador;
        begin
            null;
        end Cotizaciones;
   ```
2. Resolver el siguiente problema. En un negocio de cobros digitales hay P personas que deben pasar por la única caja de cobros para realizar el pago de sus boletas. Las personas son atendidas de acuerdo con el orden de llegada, teniendo prioridad aquellos que deben pagar menos de 5 boletas de los que pagan más. Adicionalmente, las personas ancianas tienen prioridad sobre los dos casos anteriores. Las personas entregan sus boletas al cajero y el dinero de pago; el cajero les devuelve el vuelto y los recibos de pago. 
3. Resolver el siguiente problema. La oficina central de una empresa de venta de indumentaria debe calcular cuántas veces fue vendido cada uno de los artículos de su catálogo. La empresa se compone de 100 sucursales y cada una de ellas maneja su propia base de datos de ventas. La oficina central cuenta con una herramienta que funciona de la siguiente manera: ante la consulta realizada para un artículo determinado, la herramienta envía el identificador del artículo a las sucursales, para que cada una calcule cuántas veces fue vendido en ella. Al final del procesamiento, la herramienta debe conocer cuántas veces fue vendido en total, considerando todas las sucursales. Cuando ha terminado de procesar un artículo comienza con el siguiente (suponga que la herramienta tiene una función
generarArtículo() que retorna el siguiente ID a consultar). Nota: maximizar la concurrencia. Existe una función ObtenerVentas(ID) que retorna la cantidad de veces que fue vendido el artículo con identificador ID en la base de la sucursal que la llama.