# Parcial 2 | Enunciados viejos
# PMA
 1. En un estadio de fútbol se dará un recital al que asistirán P personas. Para ello, cada asistente debe retirar su entrada por alguna de las 3 boleterías del estadio presentando su DNI (cada persona retira una única entrada). Los asistentes son atendidos por orden de llegada por alguno de los 3 empleados del club (cada empleado atiende una boletería). Implemente una solución usando PMA, considerando que los empleados atienden infinitamente. Notas: maximizar la concurrencia. La función obtenerEntrada(dni) le retorna al empleado la entrada para el DNI provisto por argumento; la función obtenerDNI() retorna el DNI para la persona que la invoca.
```
        chan comprarEntrada(integer, integer);
        chan entradas[1..P](text);

        process Persona[i=1..P]{
            int dni;
            text entrada;

            send comprarEntrada(dni,i);
            receive entradas[i](entrada);
        }

        process Empleado [i=1..3]{
            int dni, aux;
            text entrada;
            
            while (true){
                receive comprarEntrada(dni, aux);
                entrada= obtenerEntrada(dni);
                entradas[aux](entrada);
            }
        }
```
> **A tener en cuenta acá***
> En un principio lo había resuelto con un proceso intermedio admin pero NO es necesario porque usan un canal común para ir tomando las peticiones de los clientes.
> Un admin se usa cuando
> - Hay prioridades
> - El empleado tiene que hacer otra cosa si no hay nadie esperando
> - Hay una terminación
> - Otro orden que no sea el de llegada
> - Saber qué recurso usar
 2. En un estadio de fútbol se dará un recital al que asistirán P personas. Para ello, cada asistente debe retirar su entrada por alguna de las 3 boleterías del estadio presentando su DNI (cada persona retira una única entrada). Los asistentes son atendidos por orden de llegada por alguno de los 3 empleados del club (cada uno atiende una boletería). Implemente una solución usando PMA, considerando que los empleados se retiran cuando ya no hay personas por atender. Notas: maximizar la concurrencia. La función obtenerEntrada(dni) le retorna al empleado la entrada para el DNI provisto por argumento; la función obtenerDNI() retorna el DNI para la persona que la invoca. Todos los procesos deben terminar.
```
    Ídem solución 1.
```

 3. En un gimnasio hay tres máquinas iguales que pueden ser utilizadas para hacer ejercicio o rehabilitación. Hay E personas que quieren usar cualquiera de esas máquinas para hacer ejercicio. y R personas que las quieren usar para hacer rehabilitación. Siempre tienen prioridad aquellas que la quieran usar para realizar rehabilitación. Cuando una persona toma una máquina la usa por 10 minutos y se retira.
```
        chan usarEjercicio(integer);
        chan usarRehabilitacion(integer);
        chan pedirUso();
        chan liberarMaquina();
        chan usarMaquinaE[1..E](integer);
        chan usarMaquinaR[1..R](integer);

        process PersonaE[i=1..E]{
            send usarEjercicio(i);
            send pedirUso();
            receive usarMaquinaE[i];
            delay 600.0;
            send liberarMaquina();
            send pedirUso();
        }

        process personaE[i=1..R]{
            send usarRehabilitacion(i);
            send pedirUso();
            receive usarMaquinaR[i];
            delay 600.0;
            send liberarMaquina();
            send pedirUso();
        }

        process Admin {
            int disponibles= 3, aux;
            queue colaR, colaE;

            while (true) {
                receive pedirUso();
                if (not empty usarRehabilitacion){
                    receive usarRehabilitacion(aux);
                    if (disponibles > 0){
                        disponibles--;
                        send usarMaquinaR[aux]();
                    }
                    else {
                        push(colaR, aux);
                    }
                } else {
                    if (not empty usarEjercicio){
                        receive usarEjercicio(aux);
                        if (disponibles > 0){
                            disponibles--;
                            send usarMaquinaE[aux]();
                        } else {
                            push(colaE, aux);
                        }
                    } else {
                        receive liberarMaquina();
                        disponibles++;
                        if (not empty colaR){
                            pop(colaR, aux);
                            send usarMaquinaR[aux]();
                        } else {
                            if (not empty colaE){
                                pop(colaE, aux);
                                send usarMaquinaE[aux]();
                            }
                        }
                    }
                }
            }
        }

```

 4.  En una oficina hay 3 empleados y P personas que van a ser atendidas para realizarun trámite. Cuando una persona llega espera hasta ser atendida por cualquiera de los empleados, le indica el trámite a realizar y espera a que le den el resultado. Los empleados atienden las solicitudes en orden de llegada, si no hay personas esperando, durante 5 minutos resuelven trámites pendientes. Nota: No generar demora innecesaria, cada persona hace un solo pedido y termina, los empleados no necesitan terminar su ejecución. 
```
        chan avisarLlegada(integer);
        chan tramites[1..E](text);
        chan avisarDisponible(integer);
        chan proximo[1..3](integer);

        process Persona[i=1..P]{
            text tramite;

            send avisarLlegada(i);
            receive tramites[i](tramite);
        }

        process Empleado[i=1..3]{
            int idP;
            text tramite;

            while (true){
                send avisarDisponible(i);
                receive proximo[i](idP);
                if (idP == -1){
                    delay 300.0;
                } else {
                    tramite= resolver();
                    send tramites[idP](tramite);
                }
            }
        }    

        process Admin{
            int idE, idP;

            while (true){
                receive avisarDisponible(idE);
                if (not empty avisarLlegada){
                    receive avisarLlegada(idP);
                    send proximo[idE](idP);
                } else {
                    send proximo[idE](-1);
                }
            }
        }
```

 5.  En un negocio hay 5 personas que atienden a N personas que van a pedir un presupuesto, de acuerdo al orden de llegada. Cuando el cliente sabe qué empleado lo va a atender le entrega el listado de productos que necesita y luego el empleado le entrega el presupuesto del mismo. Nota: Maximizar la concurrencia. Existe una función HacerPresupuesto(lista) que simula la elaboración del presupuesto por parte de los empleados. 
```
    chan avisarLlegada(integer);
    chan empleadoAsignado[1..N](integer);
    chan enviarListado[1..5](text,integer);
    chan presupuesto[1..N](text);

    process Cliente[i=1..N]{
        text listado, presupuesto;
        int idE;

        send avisarLlegada(i);
        receive empleadoAsignado[i](idE);
        send enviarListado[idE](listado);
        receive presupuestos[i](presupuesto);
    }

    process Empleado[i=1..5]{
        int idC;
        text listado, presupuesto;

        while (true){
            receive avisarLlegada(idC);
            send empleadoAsignado[idC](i);
            receive enviarListado[i](listado);
            presupuesto= HacerPresupuesto(listado);
            send presupuestos[idC](presupuesto);
        }
    }
```

 6. Hay E estudiantes que rinden un examen final y un profesor que cunado todos los estudiantes han llegado les entrega el enunciado. Luego el profesor va corrigiendo los exámenes y enviando la nota de acuerdo con orden en que se van entregando. Cada alumno realiza el examen, lo entrega y espera a que el profesor le indique la nota. Maximizar la concurrencia, no realizar busy waiting, todos los procesos deben terminar.
```
    chan avisarLlegada();
    chan recibirEnunciado[1..E](text);
    chan avisarFin(integer, text);
    chan recibirNota[1..E](integer);

    process Estudiante[i=1..E]{
        text enunciado, presupuesto;
        int nota;

        send avisarLlegada();
        receive recibirEnunciado(enunciado);
        examen= resolver(enunciado);
        send avisarFin(i, text);
        receive recibirNota[i](nota);
    }

    process Profesor{
        int idE, nota;
        text examen, enunciado;

        for (i=1..E){
            receive avisarLlegada();
        }
        for (i=1..E){
            send recibirEnunciado[i](enunciado);
        }
        for (i=1..E){
            receive avisarFin(idE, examen);
            nota= corregir(examen);
            send recibirNota[idE](nota);
        }
    }
```

 7. En una empresa de software hay N personas que prueban un nuevo producto para encontrar errores, cuando encuentran uno generan un reporte para que un empleado corrija el error y esperan la respuesta del mismo. El empleado toma los reportes de acuerdo al orden de llegada, los evalúan, hace las correcciones necesarias y le responde a la persona que hizo el reporte.
```
    chan enviarReporte(text, integer);
    chan respuestas[1..N](text);

    process Persona[i=1..N]{
        text reporte, respuesta;

        while (true){
            reporte= probarProducto();
            send enviarReporte(reporte, i);
            receive respuestas[i](respuesta);
        }
    }

    process Empleado{
        text respuesta, reporte;
        int idP;

        while (true){
            receive enviarReporte(reporte, idP);
            respuesta= procesar(reporte);
            send respuestas[idP](respuesta);
        }
    }
```

 8. En una empresa de software hay 3 programadores que deben arreglar erroes informados por N clientes. Los clientes continuamente están trabajando, y cuando encuentran un error envían un reporte a la empresa para que lo corrija (no tienen que esperar a que se resuelva). Los programadores resuelven los reclamos de acuerdo al orden de llegada, y si no hay reclamos pendientes, trabajan durante una hora en otros programas. Nota: los procesos no deben terminal (trabajan en un loop infinito); suponga que hay una función ResolverError que simula que un programador está resolviendo un reporte de un clinete, y otra Programar que simula que está trabajando en otro programa.
```
    chan enviarReporte(text);
    chan listo(integer);
    chan siguiente[1..3](text);

    process Cliente[i=1..N]{
        text reporte;
        while (true){
            reporte= generarReporte();
            send enviarReporte(reporte);
        }
    }

    process Admin{
        int idP;
        text reporte;

        while (true){
            receive listo(idP);
            if (not empty enviarReporte){
                receive enviarReporte(reporte);
                send siguiente[idP](reporte);
            } else {
                send siguiente[idP]("VACIO");
            }
        }
    }

    process Programador[i=1..3]{
        text reporte;
        while (true){
            send listo(i);
            receive siguiente[i](reporte);
            if (reporte == "VACIO"){
                delay 3600.0;
            } else {
                ResolverError(reporte);
            }
        }
    }
```
> Enviar algún texto si es de tipo text el canal para indicar que no hay nada en el canal, no un -1

 9.  Resolver con PMA el siguiente problema. Se debe modelar el funcionamiento de una casa de venta de repuestos automotores, en la que trabajan V vendedores y que debe atender a C clientes. El modelado debe considerar que: (a) cada cliente realiza un pedido y luego espera a que se lo entreguen; y (b) los pedidos que hacen los clientes son tomados por cualquiera de los vendedores. Cuando no hay pedidos para atender, los vendedores aprovechan para controlar el stock de los repuestos (tardan entre 2 y 4 minutos para hacer esto). Nota: maximizar la concurrencia.
```
    chan realizarPedido(integer, text);
    chan entregas[1..C](text);
    chan siguiente[1..V](integer, text);
    chan listo(integer);
    
    process Cliente[i=1..C]{
        text pedido, entrega;

        send realizarPedido(i, pedido);
        receive entregas[i](entrega);
    }

    process Admin {
        int idC, idV;
        text pedido;

        while (true){
            receive listo(idV);
            if (not empty realizarPedido){
                receive realizarPedido(idC, pedido);
                send siguiente[idV](idC, pedido);
            } else {
                send siguiente[idV](-1,null);
            }
        }
    }

    process Vendedor[i=1..V]{
        int idC;
        text pedido, entrega;

        while (true){
            send listo(i);
            receive siguiente[i](idC, pedido);
            if (idC == -1){
                delay random(120, 240);
            } else {
                entrega= resolverPedido(pedido);
                send entregas[idC](entrega);
            }
        }
    }
```
> Usar función random si dice entre un tiempo y otro

 10. En una oficina existen 100 empleados que envían documentos para imprimir en 5 impresoras compartidas. Los pedidos de impresión son procesados por orden de llegada y se asignan a la primera impresora que se encuentre libre.
```
    chan pedidos(text);

    process Empleado[i=1..100]{
        text doc;

        while (true){
            doc= generarDocumento();
            send pedidos(doc);
        }
    }

    process Impresora[i=1..5]{
        text doc;

        while (true){
            receive pedidos(doc);
            imprimir(doc);
        }
    }
``` 

 11. Se debe simular la atención en un peaje con 7 cabinas para atender a N vehículos (algunos de ellos son ambulancias). Cuando el vehículo llega al peaje se dirige a la cabina con menos vehículos esperando y se queda ahí hasta que lo terminan de atender y le dan el ticket de pago. Las cabinas atienden a los vehículos que van a ella de acuerdo al orden de llegada pero dando prioridad a las ambulancias; cuando terminan de atender a un vehículo le dan el ticket de pago. Nota: maximizar la concurrencia.
```
    chan avisarLlegada(integer);
    chan atencion();
    chan atencionC[1..7]();
    chan cabinaAsignada[1..N](integer);
    chan tickets[1..N](text);
    chan atencionAmbulancia[1..7](integer);
    chan atencionVehiculo[1..7](integer);
    chan avisarSalida(integer);

    process Vehiculo[i=1..N]{
        text tipo, ticket;
        int idC;

        tipo= obtenerTipo();
        send avisarLlegada(i);
        send atencion();
        receive cabinaAsignada[i](idC);
        if (tipo == 'Ambulancia'){
            send atencionAmbulancia[idC](i);
        } else {
            send atencionVehiculo[idC](i);
        }
        send atencionC[idC]();
        receive tickets[i](ticket);
        send avisarSalida(idC);
        send atencion();
    }

    process Admin {
        int idC, idV;
        int cant_esperando[1..7]= (0[7]);

        while (true){
            receive atencion();
            if (not empty avisarSalida){
                receive avisarSalida(idC);
                cant_esperando[idC]--;
            } else {
                receive avisarLlegada(idV);
                idC= obtenerMenorCant(cant_esperando);
                cant_esperando[idC]++;
                send cabinaAsignada[idV](idC);
            }
        }
    }

    process Cabina[i=1..7]{
        int idV;
        text ticket;

        while (true){
            receive atencionC[i]();
            if (not empty atencionAmbulancia[i]){
                receive atencionAmbulancia[i](idV);
            } else {
                receive atencionVehiculo[i](idV);
            } 
            ticket= generarTicket();
            send tickets[idV](ticket);
        }
    }
```

 12. Se debe simular la atención en una planta para la VTV con 5 puestos para atender a N vehículos (que pueden ser sanitarios o comunes). Cuando el vehículo llega a la planta se dirige al puesto con menos vehículos esperando y se queda ahí hasta que le dan el comprobante de la verificación. Los puestos atienden a los vehículos que van a él de acuerdo al orden de llegada pero dando prioridad a los vehículos sanitarios; cuando terminan de atender a un vehículo le debe entregar un comprobante con los detalles de la verificación. Nota: maximizar la concurrencia.
```
    chan avisarLlegada(integer);
    chan atencion();
    chan atencionS[1..5](integer);
    chan atencionC[1..5](integer);
    chan puestos[1..N](integer);
    chan avisarPuesto[1..5]();
    chan verificaciones[1..N](text);
    chan avisarSalida(integer);

    process Vehiculo[i=1..N]{
        text tipo, verificacion;
        int idP;'
        
        tipo= obtenerTipoVehiculo();
        send avisarLlegada(i);
        send atencion();
        receive puestos[i](idP);
        if (tipo == 'Sanitario'){
            send atencionS[idP](i);
        } else {
            send atencionC[idP](i);
        }
        send avisarApuesto[idP]();
        receive verificaciones[i](verificacion);
        send avisarSalida(idP);
        send atencion();
    }

    process Puesto[i=1..5]{
        int idV;
        text verificacion;

        while (true){
            receive avisarPuesto[i]();
            if (not empty atencionS[i]){
                receive atencionS[i](idV);
            } else {
                receive atencionC[i](idV);
            }
            verificacion= generarV();
            send verificaciones[idV](verificacion);
        }
    }

    process Admin{
        int idV, idP;
        cant[1..5]=(0[5]);

        while (true){
            receive atencion();
            if (not empty avisarLlegada){
                receive avisarLlegada(idV);
                idP= menorOcupado(cant);
                cant[idP]++;
                send puestos[idV](idP);
            } else {
                receive avisarSalida(idP);
                cant[idP]--;
            }
        }
    }
```

 13.  Se debe simular la atención en un banco con 3 cajas para atender a N clientes que pueden ser especiales (son las embarazadas y los ancianos) o regulares. Cuando el cliente llega al banco se dirige a la caja con menos personas esperando y se queda ahi hasta que lo terminan de atender y le dan el comprobante de pago. Las cajas atienden a las personas que van a ella de acuerdo al orden de llegada pero dando prioridad a los clientes especiales; cuando terminan de atender a un cliente le debe entregar un comprobante de pago. Nota: maximizar la concurrencia.
```
    chan avisarLlegada(integer);
    chan atencion();
    chan cajaAsignada[1..N](integer);
    chan atencionE[1..3](integer);
    chan atencionC[1..3](integer);
    chan atencionCaja[1..3]();
    chan comprobantes[1..N](text);
    chan avisarSalida(integer);

    process Cliente[i=1..N]{
        text tipo, comprobante;
        int idC;

        tipo= obtenerTipo();
        send avisarLlegada(i);
        send atencion();
        receive cajaAsignada[i](idC);
        if (tipo == 'Especial'){
            send atencionE[idC](i);
        } else {
            send atencionC[idC](i);
        }
        send atencionCaja[idC]();
        receive comprobantes[i](comprobante);
        send avisarSalida(idC);
        send atencion();
    }

    process Caja[i=1..3]{
        int idC;
        text comprobante;

        while (true){
            receive atencionCaja[i]();
            if (not empty atencionE){
                receive atencionE(idC);
            } else {
                receive atencionC(idC);
            }
            comprobante= generarC();
            send compronbates[idC](comprobante);
        }
    }

    process Admin {
        int idC, idCaja;
        cant[1..3]=(0[3]);

        while (true){
            receive atencion();
            if (not empty avisarLlegada){
                receive avisarLlegada(idC);
                idCaja= menorOcupada(cant);
                cant[idCaja]++;
                send cajaAsignada[idC](idCaja);
            } else {
                receive avisarSalida(idCaja);
                cant[idCaja]--;
            }
        }
    }
```
   
 14.  En un negocio hay 5 empleados que atienden a N personas que van a pedir un presupuesto, de acuerdo al orden de llegada. Cuando el cliente sabe qué empleado lo va a atender le entrega el listado de productos que necesita, y luego el empleado le entrega el presupuesto del mismo. Nota: maximizar la concurrencia. Existe una función HacerPresupuesto(lista) que simula la elaboración del presupuesto por parte de los empleados.
```
    chan avisarLlegada(integer);
    chan avisarE[1..5](text);
    chan empleadosAsignados[1..N](integer);
    chan presupuestos[1..N](text);

    process Cliente[i=1..N]{
        text lista, presupuesto;
        int idE;

        send avisarLlegada(i);
        receive empleadosAsignados[i](idE);
        send avisarE[idE](lista);
        receive presupuestos[i](presupuesto);
    }

    process Empleado[i=1..5]{
        int idC;
        text presupuesto, lista;

        while (true){
            receive avisarLlegada(idC);
            send empleadosAsignados[idC](i);
            receive avisarE[i](lista);
            presupuesto= HacerPresupuesto(lista);
            send presupuestos[idC](presupuesto);
        }
    }
```

> **A tener en cuenta acá (ejercicios 11, 12 y 13)**
> Cuando el Admin mantiene un contador con llegadas y salidas, conviene preguntar primero por la salida en el `if (not empty ...)`. No importa el orden en que se declaran los canales, sino cuál se consulta primero.
> - Si hay una llegada y una salida pendientes, la salida ya ocurrió: procesarla primero deja el contador actualizado antes de elegir el puesto con menos gente.
> - Evita que las salidas queden postergadas si siguen llegando vehículos. Al revés no pasa, porque solo puede haber tantas salidas como asignaciones ya hechas.
> - Las dos versiones son correctas y ninguna se bloquea, porque cada `atencion` se envía después de su mensaje. Con la salida primero, la elección es más precisa.


# PMS
 1.  En un gimnasio hay una bicicleta de alta gama (que sólo puede ser usada por una persona a la vez) y un entrenador que controla su uso. También hay P deportistas que deben usar la bicicleta una única vez. Implemente una solución usando PMS considerando que el entrenador da acceso según el orden de llegada.
   ```
    process Deportista[i=1..P]{
        Admin!avisarLlegada(i);
        Entrenador?usarMaquina();
        delay usarMaquina;
        Entrenador!avisarSalida();
    }
    
    process Admin{
        queue esperando;
        int idE;
        do Deportista[*]?avisarLlegada(idE) -> push (esperando, idE);
            not empty(esperando); Entrenador?listo() -> pop(esperando, idE);
                                                        Entrenador!siguiente(idE);
        od
    }

    process Entrenador{
        int idE;
        for (i=1..P){
            Admin!listo();
            Admin?siguiente(idE);
            Deportista[idE]!usarMaquina();
            Deportista[idE]?avisarSalida();
        }
    }
   ```

 2. En un gimnasio hay una bicicleta de alta gama (que sólo puede ser usada por una persona a la vez) y un entrenador que controla su uso. También hay P deportistas que deben usar la bicicleta una única vez. Implemente una solución usando PMS considerando que el entrenador da acceso según el identificador del deportista (hasta que el deportista i no lo haya usado, el deportista i+1 debe esperar).
   ```
    process Deportista[i=1..P]{
        Empleado!avisarLlegada();
        Empleado?usarMaquina();
        delay usoMaquina;
        Empleado!avisarSalida();
    }

    process Empleado{
        for i=1..P{
            Deportista[i]?avisarLlegada();
            Deportista[i]!usarMaquina();
            Deportista[i]?avisarSalida();
        }
    }
   ```

 3. En una carrera hay C corredores y 3 coordinadores. Al llegar los corredores deben dirigirse a los coordinadores para que cualquiera de ellos le dé el número de chaleco con el que van a correr y luego se va. Los coordinadores atienden a los corredores de acuerdo al orden de llegada. Cuando un coordinador está libre atiende al primer corredor que está esperando. Nota: Maximizar la concurrencia. 
   ```
    process Corredor[i=1..C]{
        int nroAsignado;
        Admin!avisarLlegada(i);
        Coordinador[*]?asignacion(nroAsignado);
    }

    process Admin{
        queue esperando;
        int idC, idCoordi;
        do Corredor[*]?avisarLlegada(idC) -> push(esperando, idC);
            (not empty (esperando)); Coordinador[*]?listo(idCoordi) -> pop(esperando, idC);
                                                                        Coordinador[idCoordi]!siguiente(idC);
        od;
    }

    process Coordinador[i=1..3]{
        int idC, nro;
        while (true){
            Admin!listo(i);
            Admin?siguiente(idC);
            nro= asignarNro(idC);
            Corredor[idC]!asignacion(nro);
        }
    }
   ```
 4. Simular la atención de una estación de servicio con un único surtidor que tiene un empleado que atiende a los N clientes de acuerdo al orden de llegada. Cada cliente espera hasta que el empleado lo atienda y le indique qué y cuánto cargar, espera hasta que termine de cargarle combustible y se retira. Nota: Cada cliente carga combustible solo una vez, todos los procesos deben terminar. 
   ```
    process Cliente[i=1..N]{
        text combustible;
        int cant;
        Admin!avisarLlegada(i);
        Empleado!infoCombustible(combustible, cant);
        Empleado?fin();
    }

    process Admin{
        int cant=0, idC; 
        queue esperando;
        do cant < N; Cliente[*]?avisarLlegada(idC) -> push(esperando, idC);
                                                        cant++;
            not empty esperando; Empleado?listo() -> pop(esperando, idC);
                                                        Empleado!siguiente(idC);
        od
    }

    process Empleado {
        text combustible;
        int cant, idC;
        for i=1..N{
            Admin!listo();
            Admin?siguiente(idC);
            Cliente[idC]?infoCombustible(combustible, cant);
            delay carga;
            Cliente[idC]!fin();
        }
    }
   ```
 5. En una exposición aeronáutica hay un simulador de vuelo (que debe ser usado con exclusión mutua) y un empleado encargado de administrar el uso del mismo. A su vez hay P personas que van a la exposición y solicitan usar el simulador, cada una de ellas espera a que el empleado lo deje acceder, lo usa por un rato y se retira para que el empleado deje pasar a otra persona. El empleado deja usar el simulador a las personas respetando el orden en que hicieron la solicitud. Nota: Cada persona usa una vez el simulador. 
   ```
    process Persona[i=1..P]{
        Admin!avisarLlegada(i);
        Empleado?usar();
        delay usarSimulador;
        Empleado!finUso();
    }

    process Admin {
        queue esperando;
        int idP;

        do Persona[*]?avisarLlegada(idP) -> push(esperando, idP);
            not empty esperando; Empleado?listo() -> pop(esperando, idP);
                                                        Empleado!siguiente(idP);
        od;
    }

    process Empleado{
        int idC;
        for (i=1..P){
            Admin!listo();
            Admin?siguiente(idC);
            Persona[idC]!usar();
            Persona[idC]?finUso();
        }
    }
   ```

 6. En un consultorio hay un médico para atender a 15 pacientes de acuerdo con el orden de llegada. Cada paciente al llegar espera a que el médico lo atienda y le indique su tratamiento. Nota: Existe la función atender() que simula que el médico está atendiendo a un paciente, todos los procesos deben terminar. 
   ```
    process Paciente[i=1..15]{
        int tratamiento;
        Admin!avisarLlegada(i);
        Medico?atencion(tratamiento);
    }

    process Admin {
        queue esperando;
        int idP, cant= 0;

        do cant < 15; Paciente[*]?avisarLlegada(idP) -> push(esperando, idP);
                                                        cant++;
            not empty esperando; Medico?libre() -> pop(esperando, idP);
                                                        Medico!siguiente(idP);
        od;
    }

    process Medico{
        int idC, tratamiento;
        for (i=1..15){
            Admin!libre();
            Admin?siguiente(idC);
            tratamiento= atender();
            Paciente[idC]!atencion(tratamiento);
        }
    }
   ```

 7. En la estación de trenes hay una terminal de SUBE que debe ser usada por P personas de acuerdo con el orden de llegada. Cuando la persona accede a la terminal, la usa y luego se retira para dejar al siguiente. Cada persona usa solo una vez la terminal.
   ```
    process Persona[i=1..P]{
        Admin!avisarLlegada(i);
        Terminal?usar();
        usarTerminal();
        Terminal!avisarSalida();
    }

    process Admin{
        queue esperando;
        int idP;

        do Persona[*]?avisarLlegada(idP) -> push(esperando, idP);
            not empty esperando; Terminal?listo() -> pop(esperando, idP);
                                                    Terminal!siguiente(idP);
        od
    }

    process Terminal{
        int idP;
        while (true){
            Admin!listo();
            Admin?siguiente(idP);
            Persona[idP]!usar();
            Persona[idP]?avisarSalida();
        }
    }
   ```

 8. En una empresa de software hay un empleado Testeo que prueba un nuevo producto para encontrar errores, cuando encuentra uno generan un reporte para que otro empleado Mantenimiento corrija el error (no requiere una respuesta para seguir trabajando) y continúa trabajando. El empleado Mantenimiento toma los reportes para evaluarlos y hacer las correcciones necesarias.
   ```
    process Testeo{
        boolean tieneError;
        text reporte;
        
        while(true){
            reporte= analizarSitio();
            tieneError= revisar(reporte);
            if(tieneError){
                Admin!reportar(reporte);
            }
        }
    }

    process Admin{
        queue reportes;
        text reporte;

        do Testeo?reportar(reporte) -> push(reportes, reporte);
            not empty reportes; Mantenimiento?listo() -> pop(reportes, reporte);
                                                            Mantenimiento!siguiente(reporte);
        od
    }

    process Mantenimiento{
        text reporte;

        while(true){
            Admin!listo();
            Admin?siguiente(reporte);
            resolver(reporte);
        }
    }
   ```

 9.  En una excursión hay una tirolesa que debe ser usada por 20 turistas. Para esto hay un guía y un empleado. El empelado espera a que todos los turistas hayan llegado para darles una charla explicando las medidas de seguridad. Cuando termina la charla los turistas piden usar la tirolesa y esperan a que el guía les vaya dando el permiso de tirarse. El guía deja usar la tirolesa a un cliente a la vez y de acuerdo al orden en que lo van solicitando. Nota: todos los procesos deben terminar; suponga que el empleado tienen una función DarCharla() que simula que el empleado está dando la charla, y los turistas tienen una función UsarTitolesa() que simula que está usando la tirolesa. 
   ```
    process Turista[i=1..20]{
        Empleado!avisarLlegada();
        Empleado?avanzar();
        Admin!avisarEspera(i);
        Guia?ingresar();
        UsarTirolesa();
        Guia!avisarSalida();
    }

    process Admin{
        int cant=0, idT;
        queue espera;

        do cant < 20; Turista[*]?avisarEspera(idT) -> push(espera, idT);
                                                        cant++;
            not empty espera; Guia?listo() -> pop(espera, idT);
                                                Guia!siguiente(idT);
        od 
    }

    process Empleado{
        for (i=1..20){
            Turista[*]?avisarLlegada();
        }
        DarCharla();
        for (i=1..20){
            Turista[i]!avanzar();
        }
    }

    process Guia{
        int idT;
        for(i=1..20){
            Admin!listo();
            Admin?siguiente(idT);
            Turista[idT]!ingresar();
            Turista[idT]?avisarSalida();
        }
    }
   ```

 10. En un comedor estudiantil hay un horno microondas que debe ser usado por E estudiantes de acuerdo con el orden de llegada. Cuando el estudiante accede al horno, lo usa y luego se retira para dejar al siguiente. Nota: cada Estudiante una sólo una vez el horno.
   ```
    process Estudiante[i=1..E]{
        Admin!avisarLlegada(i);
        Horno?usar();
        delay usoHorno();
        Horno!avisarSalida();
    }

    process Admin{
        queue espera;
        int idE;

        do Estudiante[*]?avisarLlegada(idE) -> push(espera, idE);
            not empty (espera); Horno?listo() -> pop(espera, idE);
                                                Horno!siguiente(idE);
        od
    }

    process Horno {
        int idE;
        while (true){
            Admin!listo();
            Admin?siguiente(idE);
            Estudiante[idE]!usar();
            Estudiante[idE]?avisarSalida();
        }
    }
   ```

 11. En un torneo de programación hay 1 organizador, N competidores y S supervisores. El organizador comunica el desafío a resolver a cada competidor. Cuando un competidor cuenta con el desafío a resolver, lo hace y lo entrega para ser evaluado. A continuación, espera a que alguno de los supervisores lo corrija y le indique si está bien. En caso de tener errores, el competidor debe corregirlo y volver a entregar, repitiendo la misma metodología hasta que llegue a la solución esperada. Los supervisores corrigen las entregas respetando el orden en que los competidores van entregando. Nota: maximizar la concurrencia y no generar demora innecesaria.
```
    process Competidor[i=1..N]{
        text enunciado, solucion;
        boolean tieneError= true;
        
        Organizador!avisarLlegada(i);
        Organizador?enunciados(enunciado);
        solucion= resolver(enunciado);
        while (tieneError){
            Admin!entregar(solucion, i);
            Supervisor[*]?resultado(tieneError);
            if(tieneError){
                solucion= corregir(solucion);
            }
        }
    }

    process Admin{
        queue espera;
        int idS, idC;
        text solucion;

        do Competidor[*]?entregar(solucion, idC) -> push(espera, (solucion, idC));
            not empty espera; Supervisor[*]!listo(idS) -> pop(espera, (solucion, idC));
                                                            Supervisor[idS]!siguiente(solucion, idC);
        od;
    }

    process Organizador{
        text enunciado;
        int idC;
        while (true){
            Competidor[*]?avisarLlegada(idC);
            Competidor[idC]!enunciados(enunciado);
        }
    }

    process Supervisor[i=1..3]{
        text solucion;
        boolean tieneError;
        int idC;

        while (true){
            Admin!listo(i);
            Admin?siguiente(solucion, idC);
            tieneError= corregir(solucion);
            Compediro[idC]!resultado(tieneError);
        }
    }
```
 12. En una carrera hay C corredores y 3 Coordinadores. Al llegar los corredores deben dirigirse a los coordinadores para que cualquiera de ellos le dé el número de "chaleco" con el que van a correr y luego se va. Los coordinadores atienden a los corredores de acuerdo al orden de llegada (cuando un coordinador está libre atiende al primer corredor que está esperando). Nota: maximizar la concurrencia.
   ```
    process Corredor[i=1..C]{
        int nro;

        Admin!avisarLlegada(i);
        Coordinador[*]?asignado(nro);
    }

    process Admin{
        queue espera;
        int idC, idCoord;

        do Corredor[*]?avisarLlegada(idC) -> push(espera, idC);
            not empty espera; Coordinador[*]?listo(idCoord) -> pop(espera, idC);
                                                                Coordinador[idCoord]!siguiente(idC);
        od
    }

    process Coordinador[i=1..3]{
        int idC, nro;
        
        while(true){
            Admin!listo(i);
            Admin?siguiente(idC);
            nro= asignarNro(idC);
            Corredor[idC]!asignado(nro);
        }
    }
   ```

 13. Simular la atención de una estación de servicio con un único surtidor que tiene un empleado que atiende a los N clientes de acuerdo al orden de llegada. Cada cliente espera hasta que el empleado lo atienda y le indica qué y cuánto cargar; espera hasta que termina de cargarle combustible y se retira. Nota: cada cliente carga combustible sólo una vez; todos los procesos deben terminar.
```
    process Cliente [i=1..N]{
        int cant;
        text combustible;
        cant= cantPedido();
        combustible= tipoCombustible();
        Admin!avisarLlegada(i);
        Empleado?atencion();
        Empleado!pedido(cant, combustible);
        Empleado?avisarFin();
        Empleado!avisarSalida();
    }

    process Admin{
        int idC, cant= 0;
        queue esperando;
        do cant < N; Cliente[*]?avisarLlegada(idC) -> push(esperando, idC);
                                                        cant++;
            not empty esperando; Empleado?listo() -> pop(esperando, idC);
                                                        Empleado!siguiente(idC);
        od;
    }

    process Empleado{
        int idC, cant;
        text combustible;
        for (i=1..N){
            Admin!listo();
            Admin?siguiente(idC);
            Cliente[idC]!atencion();
            Cliente[idC]?pedido(cant, combustible);
            delay carga;
            Cliente[idC]!avisarFin();
            Cliente[idC]?avisarSalida();
        }
    }
```
 14. En la estación de trenes hay una terminal de SUBE que debe ser usada por P personas de acuerdo con el orden de llegada. Cuando la persona accede a la terminal, la usa y luego se retira para dejar al siguiente. Noda: cada Persona usa solo una vez la terminal.
```
    process Persona[i=1..P]{
        Admin!avisarLlegada(i);
        Terminal?usar();
        UsarTerminal();
        Terminal!avisarSalida();
    }

    process Admin{
        queue esperando;
        int idP;
        do Persona[*]?avisarLlegada(idP) -> push(esperando, idP);
            not empty esperando; Terminal?listo() -> pop(esperando, idP);
                                                    Terminal!siguiente(idP);
        od;
    }

    process Terminal{
        int idP;
        while (true){
            Admin!listo();
            Admin?siguiente(idP);
            Persona[idP]!usar();
            Persona[idP]?avisarSalida();
        }
    }
```

> **Errores que tuve (para revisar antes de entregar)**
>
> **PMS**
> - **El Admin no inicia la comunicación.** Si en una guarda de llegada hago `Empleado!siguiente(...)` sin que el empleado lo haya pedido con `listo()`, los dos quedan esperando al otro: deadlock. La guarda de llegada solo encola.
> - **`!` envía y `?` recibe.** En las guardas el Admin siempre *recibe* (`Coordinador[*]?listo(id)`), nunca `Coordinador[*]!listo`.
> - **No existe enviar a `[*]`.** `Proceso[*]` solo sirve para recibir de cualquiera. Para enviar necesito el id: `Turista[i]!avanzar()`.
> - **Usar el índice cuando sé con quién hablo.** Si ya tengo el id, es `Deportista[idE]?avisarSalida()`, no `Deportista[*]`.
> - **No hace falta un mensaje de "atención" extra.** El `!` ya bloquea al emisor hasta que el otro recibe, así que ese momento es la atención.
> - **Terminación con contador.** Si se pide que todo termine: `do cant < N; ...?llegada -> push; cant++;` (¡no olvidar el `cant++` y que el límite sea el total real!).
> - **Bucles con condición.** Inicializar la variable antes del `while` (`tieneError= true`) y, si hay que reintentar, cambiar lo que se reenvía (corregir la solución).
> - **Llamar a las funciones que da el enunciado** (`DarCharla()`, `Programar()`, `atender()`...).
>
> **PMA**
> - **Cada `send atencion()` va después del mensaje real**, y nunca hay que olvidarlo: si falta, el Admin no se entera de la llegada y todo se bloquea.
> - **Si mantengo un contador, sumarlo al asignar y restarlo al salir**, y consultar primero la salida.
> - **El cliente espera su respuesta** (`receive tickets[i](...)`) antes de avisar la salida.
> - **Revisar tipos de canales**: no mandar `-1` por un canal `text`; usar un centinela del tipo correcto.
> - **No olvidar el `send`** delante del canal, ni los índices `[i]` en canales por proceso.
>
> **En los dos:** usar siempre el mismo nombre para un mensaje o canal en el que envía y en el que recibe, y respetar los tiempos y cantidades del enunciado (1 hora = 3600, rangos con `random`, `1..S` y no `1..3`).

# ADA
 1. Implemente una solución para el siguiente problema usando ADA. En un aeropuerto internacional hay una mesa de informes que atiende las consultas de los pasajeros. Existen P pasajeros que consultan una única vez, cada uno con una prioridad diferente: los pasajeros con vuelos inmediatos tienen prioridad sobre los que viajan más tarde; mientras que los pasajeros con movilidad reducida tienen prioridad sobre todos los demás. Cada pasajero entrega su código de vuelo al empleado de informes, quien responde con la puerta de embarque y la hora estimada de salida. Notas: Cada pasajero obtiene su prioridad mediante la función obtenerPrioridad(), que retorna 0 para movilidad reducida, 1 para vuelos inmediatos y 2 para el resto; mientras que la función obtenerVuelo() le retorna el código de vuelo. El empleado dispone de las funciones obtenerPuerta(vuelo) y obtenerHora(vuelo) que retornan la información de salida para el vuelo recibido. El empleado de informes atiende un pasajero por vez, según la prioridad indicada. Todas las tareas deben finalizar.
```ada
    procedure Aeropuerto is
        task Empleado is
            entry filaMovilidadReducida(cod: IN integer; puerta, hora: OUT integer);
            entry filaInmediatos(cod: IN integer; puerta, hora: OUT integer);
            entry filaComun(cod: IN integer; puerta, hora: OUT integer);
        end Empleado;
        task type Persona;
        personas: array(1..P) of Persona;

        task body Persona is
            integer prioridad, codVuelo, nroPuerta, horaVuelo;
        begin
            prioridad:= obtenerPrioridad();
            codVuelo:= obtenerCodVuelo();
            if prioridad = 0 then
                Empleado.filaMovilidadReducida(codVuelo, nroPuerta, horaVuelo);
            elsif prioridad = 1 then
                Empleado.filaInmediatos(codVuelo, nroPuerta, horaVuelo);
                else 
                    Empleado.filaCOmun(codVuelo, nroPuerta, horaVuelo);
            end if;
        end Persona;

        task body Empleado is
            integer i;
        begin
            for i in 1..P loop
                select
                    accept filaMovilidadReducida(cod: IN integer; puerta, hora: OUT integer) do
                        puerta:= obtenerPuerta(cod);
                        hora:= obtenerHora(cod);
                    end filaMovilidadReducida;
                or (when filaMovilidadReducida'count = 0) =>
                    accept filaInmediatos(cod: IN integer; puerta, hora: OUT integer) do
                        puerta:= obtenerPuerta(cod);
                        hora:= obtenerHora(cod);
                    end filaInmediatos;
                or (when filaMovilidadReducida'count = 0 AND filaInmediatos'count = 0) =>
                    accept filaComun(cod: IN integer; puerta, hora: OUT integer) do
                        puerta:= obtenerPuerta(cod);
                        hora:= obtenerHora(cod);
                    end filaComun;
                end select;
            end loop;
        end Empleado;
        begin
            null;
        end Aeropuerto;
```

 2. Existen N personas que van a realizar un pago en la caja de un banco. Las personas son atendidas de acuerdo al orden de llegada, aunque aquellos que sean jubilados tienen prioridad sobre los que no lo sean. Todas las personas esperan a lo sumo 15 minutos para ser atendidas. Si pasado ese tiempo no fueron atendidas, le dejan una nota de reclamo al responsable de informes y se retiran.
```ada
    procedure Banco is
        task type Empleado is
            entry filaJubilado(id: IN integer; respuesta: OUT text);
            entry filaNormal(id: IN integer; respuesta: OUT text);
        end Empleado;
        task type EmpleadoReclamos is
            entry envioReclamos(reclamo: IN text);
        end EmpleadoReclamos;
        task type Persona;
        personas: array(1..N) of Persona;
        
        task body Persona is
            boolean esJubilado;
            integer id;
            text rta, queja;
        begin
            esJubilado:= calcularEdad();
            if esJubilado then
                select
                    Empleado.filaJubilado(id, rta);
                or delay 900;
                    queja:= generarQueja();
                    EmpleadoReclamos.envioReclamos(queja);
                end select;
            else
                select 
                    Empleado.filaNormal(id, rta);
                or delay 900
                    queja:= generarQueja();
                    EmpleadoReclamos.envioReclamos(queja);
                end select;
            end if;
        end Persona;

        task body Empleado is
        begin
            loop
                select
                    accept filaJubilado(id: IN integer; respuesta: OUT text) do
                        respuesta:= atender(id);
                    end filaJubilado;
                or
                    (when filaJubilado'count == 0) =>
                        accept filaNormal(id: IN integer; respuesta: OUT text) do
                            respuesta:= atender(id);
                        end filaNormal;
                end select;
            end loop;
        end Empleado;

        task body EmpleadoReclamos is
            queue reclamos;
        begin
            loop
                accept envioReclamos(reclamo: IN text) do
                    push(reclamos, reclamo);
                end envioReclamos;
            end loop;
        end EmpleadoReclamos;

        begin
            null;
        end Banco;
```

 3. Se quiere modelar un puente colgante de un solo sentido, el cual resiste hasta 100 personas. Suponga que hay una cantidad innumerable de personas que quieren cruzar y que solo lo podrían hacer si no superan la cantidad máxima. Las personas cruzan el puente de acuerdo al orden de llegada, aunque los jubilados tienen prioridad sobre los no jubilados. 

 4. En una competencia de programadores hay P participantes y UN coordinador. El coordinador entrega el problema a resolver a los participantes y recibe las resoluciones para corregir. Cada participante debe conocer el resultado de su trabajo y el orden en que entregó su resolución Nota: Maximizar la concurrencia. 
```ada
    procedure Competencia is

        task type Participante is
            entry entregarEnunciado(e: IN text);
        end Participante;

        task Coordinador is
            entry entregarExamen(examen: IN text; nota, orden: OUT integer);
        end Coordinador;

        participantes: array(1..P) of Participante;

        task body Participante is
            enunciado, examen: text;
            nota, orden: integer;
        begin
            accept entregarEnunciado(e: IN text) do
                enunciado := e;
            end entregarEnunciado;
            examen := resolverEnunciado(enunciado);
            Coordinador.entregarExamen(examen, nota, orden);
        end Participante;

        task body Coordinador is
            enunciado: text;
        begin
            enunciado := generarEnunciado();
            for i in 1..P loop
                participantes(i).entregarEnunciado(enunciado);
            end loop;
            for i in 1..P loop
                accept entregarExamen(examen: IN text; nota, orden: OUT integer) do
                    orden := i;
                    nota := corregirExamen(examen);
                end entregarExamen;
            end loop;
        end Coordinador;

    begin
        null;
    end Competencia;
```
> - El orden de entrega sale del contador `i` del `for`: la cola de la entry `entregarExamen` es FIFO, así que el i-ésimo aceptado es el i-ésimo en entregar.
> - Corregir dentro del `accept` no resta concurrencia: hay un solo coordinador (corrige de a uno igual) y el participante no tiene nada más que hacer que esperar su nota.
> - Ojo con los nombres: el parámetro de la entry no puede llamarse igual que la variable local (`e` vs `enunciado`).

 5. En un negocio de cobros digitales hay P personas que debn pasar por la única caja de cobros para realizar el pago de sus boletas. Las personas son atendidas de acuerdo con el orden de llegada, teniendo prioridad aquellos que deben pagar menos de 5 boletas de los que pagan más. Adicionalmente, las personas embarazadas y los ancianos tienen prioridad sobre los dos casos anteriores. Las personas entregan sus boletas al cajero y el dinero de pago; el cajero les devuelve el vuelto y los recibos de pago. 
```ada
    procedure CobroBoletas is

        task Caja is
            entry filaPrioridad(boletas: IN text; dinero: IN OUT integer; recibos: OUT text);
            entry filaMenos5(boletas: IN text; dinero: IN OUT integer; recibos: OUT text);
            entry filaComun(boletas: IN text; dinero: IN OUT integer; recibos: OUT text);
        end Caja;

        task type Persona;
        personas: array(1..P) of Persona;

        task body Persona is
            boletas, recibos: text;
            dinero, cantBoletas: integer;
            tienePrioridad: boolean;
        begin
            boletas := obtenerBoletas();
            dinero := obtenerDinero();
            tienePrioridad := esEmbarazadaOAnciano();
            cantBoletas := cant(boletas);
            if tienePrioridad then
                Caja.filaPrioridad(boletas, dinero, recibos);
            elsif cantBoletas < 5 then
                Caja.filaMenos5(boletas, dinero, recibos);
            else
                Caja.filaComun(boletas, dinero, recibos);
            end if;
        end Persona;

        task body Caja is
        begin
            loop
                select
                    accept filaPrioridad(boletas: IN text; dinero: IN OUT integer; recibos: OUT text) do
                        dinero := cobrar(boletas, dinero);      -- devuelve el vuelto
                        recibos := generarRecibos(boletas);
                    end filaPrioridad;
                or
                    when (filaPrioridad'count = 0) =>
                        accept filaMenos5(boletas: IN text; dinero: IN OUT integer; recibos: OUT text) do
                            dinero := cobrar(boletas, dinero);
                            recibos := generarRecibos(boletas);
                        end filaMenos5;
                or
                    when (filaPrioridad'count = 0 and filaMenos5'count = 0) =>
                        accept filaComun(boletas: IN text; dinero: IN OUT integer; recibos: OUT text) do
                            dinero := cobrar(boletas, dinero);
                            recibos := generarRecibos(boletas);
                        end filaComun;
                end select;
            end loop;
        end Caja;

    begin
        null;
    end CobroBoletas;
```
> - Una entry por nivel de prioridad: cada cola de entry es FIFO, así que dentro de cada nivel se respeta el orden de llegada.
> - Las guardas con `'count` dan la prioridad: embarazadas/ancianos > menos de 5 boletas > resto.
> - "Menos de 5" es `< 5`, no `<= 5`. En Ada el modo es `IN OUT` (sin barra).

 6.  Se debe modelar la atención en un banco por medio de un único empleado. Los clientes llegan y esperan a lo sumo 10 minutos a ser atendido de acuerdo al orden de llegada. Pasado ese tiempo se retira sin ser atendido.
```ada
    procedure Banco is

        task Empleado is
            entry atencion(respuesta: OUT text);
        end Empleado;

        task type Cliente;
        clientes: array(1..N) of Cliente;

        task body Cliente is
            rta: text;
        begin
            select
                Empleado.atencion(rta);
            or delay 600.0;          -- 10 minutos
                null;                -- se retira sin ser atendido
            end select;
        end Cliente;

        task body Empleado is
        begin
            loop
                accept atencion(respuesta: OUT text) do
                    respuesta := atenderCliente();
                end atencion;
            end loop;
        end Empleado;

    begin
        null;
    end Banco;
```
> - Llamada temporizada (`select ... or delay`): si en 10 minutos el empleado no **empezó** a atenderlo, se cancela la llamada y el cliente se va. Si ya empezó el rendezvous, espera a que termine.
> - La cola de la entry es FIFO → orden de llegada.
> - El tipo de la variable que se pasa tiene que coincidir con el del parámetro (`rta: text` ↔ `respuesta: OUT text`).

 7. Hay E estudiantes que rinden un examen final y un profesor que cuando todos los estudiantes han llegado les entrega el enunciado, luego el profesor va corrigiendo los exámenes y enviando la nota con orden en que se van entregando. Cada alumno realiza el examen, lo entrega y espera a que el profesor le indique la nota. Nota: Maximizar la concurrencia, no realizar busy waiting, todos los procesos deben terminar. 
```ada
    procedure ExamenFinal is

        task type Alumno is
            entry recibirEnunciado(e: IN text);
        end Alumno;

        task Profesor is
            entry llegada;
            entry correccion(examen: IN text; nota: OUT integer);
        end Profesor;

        alumnos: array(1..E) of Alumno;

        task body Alumno is
            enunciado, examen: text;
            nota: integer;
        begin
            Profesor.llegada;
            accept recibirEnunciado(e: IN text) do
                enunciado := e;
            end recibirEnunciado;
            examen := resolver(enunciado);
            Profesor.correccion(examen, nota);
        end Alumno;

        task body Profesor is
            enunciado: text;
        begin
            for i in 1..E loop
                accept llegada;
            end loop;
            enunciado := generarEnunciado();
            for i in 1..E loop
                alumnos(i).recibirEnunciado(enunciado);
            end loop;
            for i in 1..E loop
                accept correccion(examen: IN text; nota: OUT integer) do
                    nota := corregir(examen);
                end correccion;
            end loop;
        end Profesor;

    begin
        null;
    end ExamenFinal;
```
> - El primer `for` es la barrera: el profesor no reparte el enunciado hasta recibir las E llegadas.
> - Entry sin parámetros: se declara `entry llegada;`, se llama `Profesor.llegada;` y se acepta `accept llegada;` (sin paréntesis ni `do null; end`).
> - Las correcciones salen en el orden de entrega porque la cola de `correccion` es FIFO. Todas las tareas terminan: los `for` tienen exactamente E vueltas.

 8. En la farmacia de un hospital hay una única ventanilla para dispensar medicamentos. Los P pacientes internados deben retirar sus medicamentos presentando una receta (un único retiro por persona). Los pacientes son atendidos en el orden de llegada, pero con prioridad de atención: los pacientes de urgencia tienen prioridad sobre los de internación general, y ambos tienen prioridad sobre los pacientes ambulatorios. Cada paciente entrega su receta y recibe la medicación correspondiente. Notas: la función obtenerPrioridad() le permite al paciente conocer su prioridad (retorna 0 si el paciente es de urgencia, 1 si es internado general, o 2 si es ambulatorio). La función obtenerReceta() retorna la receta para el paciente que la invoca mientras que dispensarMedicamento(receta) le retorna al empleado el medicamento para la receta indicada. El empleado atiende de a un paciente por vez, según la prioridad indicada. Todas las tareas deben finalizar. 

 9.  Hay un sitio web para identificación genética que resuelve pedidos de N clientes. Cada cliente trabaja continuamente de la siguiente manera: genera la secuencia de ADN, la envía al sitio web para evaluar y espera el resultado; después de esto puede comenzar a generar la siguiente secuencia de ADN. Para resolver estos pedidos el sitio web cuenta con 5 servidores idénticos que atienden los pedidos de acuerdo al orden de llegada (cada pedido es atendido por un único servicio). Nota: maximizar la concurrencia. Suponga que los servidores tienen una función ResolverAnalisis que recibe la secuencia de ADN y devuelve un entero con el resultado.
```ada
    procedure SitioGenetico is

        task type Cliente is
            entry identificar(id: IN integer);
            entry resultado(res: IN integer);
        end Cliente;

        task Admin is
            entry pedido(idC: IN integer; sec: IN text);
            entry siguiente(idC: OUT integer; sec: OUT text);
        end Admin;

        task type Servidor;

        clientes: array(1..N) of Cliente;
        servidores: array(1..5) of Servidor;

        task body Cliente is
            miId, res: integer;
            sec: text;
        begin
            accept identificar(id: IN integer) do
                miId := id;
            end identificar;
            loop
                sec := generarSecuencia();
                Admin.pedido(miId, sec);
                accept resultado(r: IN integer) do
                    res := r;
                end resultado;
            end loop;
        end Cliente;

        task body Admin is
            cola: queue;
        begin
            loop
                select
                    accept pedido(idC: IN integer; sec: IN text) do
                        push(cola, (idC, sec));
                    end pedido;
                or
                    when not empty(cola) =>
                        accept siguiente(idC: OUT integer; sec: OUT text) do
                            pop(cola, (idC, sec));
                        end siguiente;
                end select;
            end loop;
        end Admin;

        task body Servidor is
            idC, res: integer;
            sec: text;
        begin
            loop
                Admin.siguiente(idC, sec);
                res := ResolverAnalisis(sec);   -- fuera del rendezvous: los 5 trabajan en paralelo
                clientes(idC).resultado(res);
            end loop;
        end Servidor;

    begin
        for i in 1..N loop
            clientes(i).identificar(i);
        end loop;
    end SitioGenetico;
```
> - El Admin con cola garantiza el orden de llegada; sus `accept` solo encolan/desencolan (rendezvous cortos).
> - `ResolverAnalisis` se ejecuta fuera de cualquier rendezvous → los 5 servidores trabajan en paralelo (maximiza la concurrencia).
> - El servidor responde directo al cliente con `clientes(idC).resultado`; el cliente conoce su id porque se lo pasa el programa principal con `identificar`.

 10. En un negocio hay un empleado para atender los pedidos de clientes de N clientes; algunos clientes son ancianos o embarazadas. El empleado atiende los pedidos de acuerdo a la siguiente prioridad: primero las embarazadas, luego los ancianos, y por último el resto. Cada cliente hace sólo un pedido de la siguiente manera: en el caso de las embarazadas, si no son atendidas inmediatamente se retiran; en el caso de los ancianos, esperan a los sumo 5 minutos a ser atendidos, y si no se retira; cualquier otro cliente espera sí o sí hasta ser atendido. El empleado atiende a los clientes de acuerdo al orden de llegada pero manteniendo las siguientes prioridades: primero las embarazadas, luego los ancianos y luego el resto. Nota: suponga que existe una función AtenderPedido() que simula que el empleado está atendiendo a un cliente.

 11. Se quiere modelar el funcionamiento de un banco, al cual llegan clientes que deben realizar un pago y llevarse su comprobante. Los clientes se dividen entre los regulares y los premium, habiendo R clientes regulares y P clientes premium. Existe un único empleado en el banco, el cual atiende de acuerdo al orden de llegada, pero dando prioridad a los premium sobre los regulares. Si a los 30 minutos de llegar un cliente regular no fue atendido, entonces se retira sin realizar el pago. Los clientes premium siempre esperan hasta ser atendidos.

 12. Se debe controlar el acceso a una base de datos. Existen L procesos Lectores y E procesos Escritores que trabajan indefinidamente de la siguiente manera: 
 • Escritor: intenta acceder para escribir, si no lo logra inmediatamente, espera 1 minuto y vuelve a intentarlo de la misma manera.
 • Lector: intenta acceder para leer, si no lo logró en 2 minutos, espera 5 minutos y vuelve a intentarlo de la misma manera. 
 Un proceso Escritor podrá acceder si no hay ningún otro proceso usando la base de datos; al acceder escribe y sale de la BD. Un proceso Lector podrá acceder si no hay procesos Escritores usando la base de datos; al acceder lee y sale de la BD. Siempre se le debe dar prioridad al pedido de acceso para escribir sobre el pedido de acceso para leer.

 1.   la página web del Banco Central exhibe las diferentes cotizaciones del dólar oficial de 20 bancos del país, tanto para la compra como para la venta. Existe una tarea programada que se ocupa de actualizar la página en forma periódica y para ello consulta la cotización de cada uno de los 20 bancos. Cada banco dispone de una API, cuya única función es procesar las solicitudes de aplicaciones externas. La tarea programada consulta de a una API por vez, esperando a lo sumo 5 segundos por su respuesta. Si pasado ese tiempo no respondió, entonces se mostrará vacía la información de ese banco.

 2.   Una empresa de venta de calzado cuenta con S sedes. En la oficina central de la empresa se utiliza un sistema que permite controlar el stock de los diferentes modelos, ya que cada sede tiene una base de datos propia. El sistema de control de stock funciona de la siguiente manera: dado un modelo determinado, lo envía a las sedes para que cada una le devuelva la cantidad disponible en ellas; al final del procesamiento, el sistema informa el total de calzados disponibles de dicho modelo. Una vez que se completó el procesamiento de un modelo, se procede a realizar lo mismo con el siguiente modelo. Nota: suponga que existe una función DevolverStock(modelo,cantidad) que utiliza cada sede donde recibe como parámetro de entrada el modelo de calzado y retorna como parámetro de salida la cantidad de pares disponibles. Maximizar la concurrencia y no generar demora innecesaria.

 3.  Simular el con funcionamiento de un Entrenamiento de Básquet donde hay 20 jugadores y UN entrenador. El entrenador debe distribuir a los jugadores en 2 canchas. Cuando un jugador llega el entrenador le indica la cancha a la cual debe ir para que se dirija a ella y espere hasta que lleguen los 10; en ese momento comienzan a jugar el partido que dura 40 minutos. Cuando ambos partidos han terminado, el entrenador les da a los 20 jugadores una charla de 10 minutos y luego todos se retiran. El entrenador asigna el número de cancha en forma cíclica 1, 2, 1, 2 y así sucesivamente.

 4.   En una empresa de organización de recitales cuentan con un sitio web para la venta de entradas. Existen N personas que quieran comprar una de las E entradas disponibles. Las personas se dividen en pacientes e impacientes. Cuando las personas pacientes intentan hacer la compra en el sitio web, esperan a lo sumo 5 minutos a que el servidor responda. Si pasado ese tiempo no fueran atendidas, lo vuelven a intentar. En el caso de las personas impacientes, si no son atendidas inmediatamente por el servidor, esperan 10 segundos y vuelvan a intentarlo. En todos los casos, el sitio web les dice si pudieron venderle la entrada o no.

 5.  Simular el funcionamiento de un Complejo de Canchas de Paddle que posee 10 canchas y donde hay UN encargado que se encarga de distribuir a las personas en las canchas. Al complejo acuden 40 personas a jugar. Cuando una persona llega el encargado le indique el número de cancha a la cual debe ir, y se dirige a ella; cuando han llegado los 4 jugadores a la cancha, comienzan a jugar el partido que dura 60 minutos; al terminar el partido las 4 personas se retiran. El encargado asigna el número de cancha según el orden de llegada (los 4 primeros a la cancha 1, los siguientes 4 a la 2 y así sucesivamente). Nota: cuando el encargado ya atendió a las 40 personas debe terminar su ejecución.
 6.  La oficina central de una empresa de venta de indumentaria debe calcular cuántas veces fue vendido cada uno de los artículos de su catálogo. La empresa se compone de 100 sucursales y cada una de ellas maneja su propia base de datos de ventas. La oficina central cuenta con una herramienta que funciona de la siguiente manera: ante la consulta realizada para un artículo determinado, la herramienta envía el identificador del artículo a las sucursales, para que cada una calcule cuántas veces fue vendido en ella. Al final del procesamiento, la herramienta debe conocer cuántas veces fue vendido en total, considerando todas las sucursales. Cuando ha terminado de procesar un artículo comienza con el siguiente (suponga que la herramienta tiene una función generarArtículo() que retorna el siguiente ID a consultar). Nota: maximizar la concurrencia. Existe una función ObtenerVentas(ID) que retorna la cantidad de veces que fue vendido el artículo con identificador ID en la base de la sucursal que la llama.