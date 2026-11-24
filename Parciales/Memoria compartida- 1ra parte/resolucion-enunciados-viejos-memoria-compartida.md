# Parcial 1 | Enunciados viejos
# Semáforos
 1. En una planta verificadora de vehículos existen 7 estaciones donde se dirigen 150 vehículos para ser verificados. Cuando un vehículo llega a la planta, el coordinador de la planta le indica a qué estación debe dirigirse. El coordinador selecciona la estación que tenga menos vehículos en ese momento. Una vez qu el vehículo sabe qué estación le fue asignada, se dirige a la misma y espera a que lo llamen para verificar. Luego de la revisión, la estación le entrega un comprobante que indica si pasó la revisión o no. Más allá del resultado, el vehículo se retira de la planta.
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2022---Soluciones-parcial-MC.pdf` (parcial MC 11-10-22, ej. 1, págs. 1-2)
    *Solución a mano (no oficial):* `Parciales/Memoria compartida- 1ra parte/Parcial 1- 1ra fecha 2022.pdf` (ej. 1, pág. 1)
    *Solución propia:*
```
process Vehiculo[i=1..150]{
    int nroCabina;
    text comprobante;

    P(sem_entrada);
    push(colaEspera, i);
    V(sem_entrada);
    V(sem_atencion);
    P(sem_espera[i]);
    nroCabina= estaciones[i];
    P(sem_estacion[nroCabina]);
    push(estacion[nroCabina], i);
    V(sem_estacion[nroCabina]);
    V(sem_estacion_atencion[nroCabina]);
    P(sem_resultado[i]);
    P(sem_cant_estaciones);
    cant_estaciones[nroCabina]--;
    V(sem_cant_estaciones);
    comprobante= dobs[i];
}

process Entrada{
    int id, id_min;

    while(true){
        P(sem_atencion);
        P(sem_entrada);
        pop(colaEspera, id);
        V(sem_entrada);
        P(sem_cant_estaciones);
        id_min= min(cant_estaciones);
        cant_estaciones[id_min]++;
        V(sem_cant_estaciones);
        estaciones[id]= id_min;
        V(sem_espera[id]);
    }
}

process Estacion[i=1..7]{
    int id;
    while(true){
        P(sem_estacion_atencion[i]);
        P(sem_estacion[i]);
        id= pop(estacion[i]);
        V(sem_estacion[i]);
        dobs[id]= Verificar();
        V(sem_resultado[id]);
    }
}
```

 2. Una empresa de turismo posee un micro con capacidad para 50 personas. Hay un único vendedor que atiende a los C clientes (C > 50) que intentan comprar un pasaje de acuerdo al orden de llegada (suponga que la atención de un cliente tarda un par de minutos), si aún hay lugares disponibles, le indica el número de asiento que le tocó. Cada cliente, luego de ser atendidos por el vendedor, se dirige al micro para subir en caso de que le hayan dado un asiento. El micro espera a que los 50 pasajeros hayan subido para realizar el viaje.
    *Solución propia:*
```
process Cliente[i=1..C]{
    int nroTicket;

    P(sem_entrada);
    push(colaEspera, i);
    V(sem_entrada);
    V(sem_atencion);
    P(sem_asignacion[i]);
    nroTicket= tickets[i];
    if (nroTicket != -1){
        P(sem_llegada_micro);
        total++;
        if (total == 50){
            V(sem_micro);
        }
        V(sem_llegada_micro);
        P(sem_fin_viaje[nroTicket]);
    }
}

process Micro{
    P(sem_micro);
    delay(viajeMicro);
    for i=1..50{
        V(sem_fin_viaje[i]);
    }
}
```

 3. En una competencia culinaria se juntan 1 jurado y C concursantes. Una vez que todos los concursantes llegaron, el jurado les asigna la receta que deben realizar. A continuación, los concursantes cocinan el plato pedido (a cada uno le lleva un tiempo variable) y lo exhiben ante el jurado en el orden en que van terminando. El jurado asigna un puntaje a cada concursante, el cual lo guarda para su CV.
    *Solución propia:*
```
sem sem_llegada= 1; int cant= 0; sem sem_aviso_profesor= 0;
sem sem_receta[C]= ([C]0); sem sem_correccion[C]= ([C]0);
sem sem_entregas= 1; sem sem_avisaEntrega= 0; queue colaP;
text recetas[C], puntajes[C];

process Concursante[i=1..C]{
    text receta, plato;
    int puntaje;

    P(sem_llegada);
    cant++;
    if (cant == C){
        V(sem_aviso_profesor);
    }
    V(sem_llegada);
    P(sem_receta[i]);
    receta= recetas[i];
    plato= cocinar(receta);
    P(sem_entregas);
    push(colaP, (i, plato));
    V(sem_entregas);
    V(sem_avisaEntrega);
    P(sem_correccion[i]);
    puntaje= puntajes[i];
}

process Jurado{
    int idC; text plato, puntaje;

    P(sem_aviso_profesor);
    for i=1..C{
        recetas[i]= generarReceta();
        V(sem_receta[i]);
    }

    for i=1..C{
        P(sem_avisaEntrega);
        P(sem_entregas);
        pop(colaP, (idC, plato));
        V(sem_entregas);
        puntaje= corregir(plato);
        puntajes[idC]= puntaje;
        V(sem_correccion[idC]);
    }
}
```

 4. En un examen final hay P alumnos y 3 profesores. Cuando todos los alumnos y los profesores han llegado comienza el examen. Para esto uno de los profesores (el primero en llegar) le da el examen a cada alumno. Cada alumno resuelve su examen, lo entrega y espera que alguno de los profesores lo corrija y le indique la nota. Los profesores corrigen los examenes respetando el orden en que los alumnos van entregando.
    *Solución propia:*
```
int totalAlumnos= 0; sem sem_llegada= 1; sem sem_entregaEnunciados= 0;
sem sem_enunciados[P]= ([P]0); text enunciados[P];
int notas[P]; sem sem_entrega2= 1; sem sem_avisoEntrega= 0; queue colaE;
sem sem_correccion[P]= ([P]0);
int cantP= 0; sem sem_llegadaP= 1; sem sem_llegadaTodosP= 0;
sem sem_inicioCorreccion[3]= ([3]0);

process Alumno[i=1..P]{
    text enunciado, examen;
    int nota;

    P(sem_llegada);
    totalAlumnos++;
    if (totalAlumnos == P){
        V(sem_entregaEnunciados);
    }
    V(sem_llegada);
    P(sem_enunciados[i]);
    enunciado= enunciados[i];
    examen= resolverEnunciado(enunciado);
    P(sem_entrega2);
    push(colaE, (i, examen));
    V(sem_entrega2);
    V(sem_avisoEntrega);
    P(sem_correccion[i]);
    nota= notas[i];
}

process Profesor[i=1..3]{
    boolean entregar; text examen;
    int idA, nota;

    P(sem_llegadaP);
    cantP++;
    if (cantP == 1){
        entregar= true;
    }
    else if (cantP == 3){
        V(sem_llegadaTodosP);
    }
    V(sem_llegadaP);

    if (entregar){
        P(sem_entregaEnunciados);
        P(sem_llegadaTodosP);
        for i=1..P{
            enunciados[i]= generarEnunciado();
            V(sem_enunciados[i]);
        }
        for j=1..3{
            V(sem_inicioCorreccion[j]);
        }
    }
    else {
        P(sem_inicioCorreccion[i]);
    }

    while(true){
        P(sem_avisoEntrega);
        P(sem_entrega2);
        pop(colaE, (idA, examen));
        V(sem_entrega2);
        nota= corregir(examen);
        notas[idA]= nota;
        V(sem_correccion[idA]);
    }
}
```

 5. Simular la atención de una planta verificadora de vehículos, donde se revisan cuestiones de seguridad de vehículos de a uno por vez. Hay N vehículos que deben ser verificados, donde algunos son autos y otros ambulancias. Antes de ser verificados, los autos deben hacer el pago correspondiente en la caja de la planta, donde le entregarán un recibo de pago. Las ambulancias NO pagan. Los vehículos son atendidos de acuerdo al orden de llegada pero dando prioridad a las ambulancias.
    *Solución propia:*
```
sem mutexFila= 1; sem sem_espera[N]= ([N]0);
queue filaE; int pagos[N]; sem dinero_abonado= 0;
sem sem_esperaComprobante[N]= ([N]0); text recibos[N];
sem aviso_atencion= 0;

process Vehiculo[i=1..N]{
    text tipoV, recibo;
    int dinero;

    tipoV= tipoVehiculo();
    P(mutexFila);
    if (tipoV == "ambulancia"){
        insertarOrdenado(filaE, (i, tipoV));
    }
    else{
        push(filaE, (i, tipoV));
    }
    V(mutexFila);
    V(aviso_atencion);
    P(sem_espera[i]);
    if (tipoV == "auto"){
        dinero= tarifa();
        pagos[i]= dinero;
        V(dinero_abonado);
        P(sem_esperaComprobante[i]);
        recibo= recibos[i];
    }
}

process Verificador{
    int idV, dinero; text tipo, recibo;

    while(true){
        P(aviso_atencion);
        P(mutexFila);
        pop(filaE, (idV, tipo));
        V(mutexFila);
        V(sem_espera[idV]);
        if (tipo == "auto"){
            P(dinero_abonado);
            dinero= pagos[idV];
            cobrar(dinero);
            recibo= generarRecibo();
            recibos[idV]= recibo;
            V(sem_esperaComprobante[idV]);
        }
        Verificar();
    }
}
```

 6. En una clínica hay un médico que debe atender a 20 personas de acuerdo al turno de cada uno de ellos (no puede atender al paciente con turno i+1 si no atendió al que tiene el turno i). Cada paciente ya conoce su turno al comenzar (valor entre 0 y 19 entre 1 y 20). Al llegar espera hasta que le médico lo llame para ser atendido, se dirige al consultorio y luego espera hasta que el médico que lo termine de atender para poder retirarse. Nota: los únicos procesos que se pueden usar los que representan a los pacientes y al médico, se debe asegurar de que nunca haya más de un paciente en el consultorio, no se puede usar el ID del proceso como turno.
    *Solución propia:*
```
sem avisarLlegada[20]= ([20]0); sem ingresarConsultorio[20]= ([20]0);
sem salirConsultorio[20]= ([20]0); sem avisarSalida[20]= ([20]0);

process Persona[i=1..20]{
    int turno;

    turno= obtenerMiTurno();
    V(avisarLlegada[turno]);
    P(ingresarConsultorio[turno]);
    P(salirConsultorio[turno]);
    V(avisarSalida[turno]);
}

process Medico{
    for i=1..20{
        P(avisarLlegada[i]);
        V(ingresarConsultorio[i]);
        delay(atencionMedica);
        V(salirConsultorio[i]);
        P(avisarSalida[i]);
    }
}
```

 7. Se debe simular el uso de UNA máquina expendedora de agua con capacidad para 20 botellas por parte de U usuarios, además existe un repositorio encargado de reponer las botellas de máquina. Cuando un usuario llega a la máquina expendedora, espera su turno (representando el orden de llegada), saca una botella y se retira. Si encuentra la máquina sin botellas, le avisa al repositorio para que cargue nuevamente la máquina con 20 botellas. Espera a que se haga la recarga saca una botella y se retira.
    *Solución propia:*
```
sem mutexFila= 1; sem espera[U]= ([U]0); sem avisarRepositor= 0; sem avisarDisponible= 0;
queue filaE; int botellas= 20; boolean libre= true;

process Usuario[i=1..U]{
    Botella botella;
    int aux;

    P(mutexFila);
    if (libre){
        libre= false;
        V(mutexFila);
    }
    else {
        push(filaE, i);
        V(mutexFila);
        P(espera[i]);
    }
    if (botellas == 0){
        V(avisarRepositor);
        P(avisarDisponible);
    }
    botella= tomarBotella();
    botellas--;
    P(mutexFila);
    if (not empty(filaE)){
        pop(filaE, aux);
        V(espera[aux]);
    }
    else {
        libre= true;
    }
    V(mutexFila);
}

process Repositor{
    while(true){
        P(avisarRepositor);
        delay(reposicion);
        botellas= 20;
        V(avisarDisponible);
    }
}
```

 8. Se tiene un vector A de 1.000.000 de números del cual se debe obtener el promedio de sus valores utilizando 10 procesos Worker. Al terminar todos los procesos deben imprimir el resultado. Maximizar concurrencia, únicamente se pueden usar los 10 procesos worker.
    *Solución propia:*
```
sem mutexContador= 1; const tam= 1000000/10; int nros[1000000]; int terminaron= 0;
int sumatoria= 0; int promedio; int promedios[10]; sem obtenerRtdo[10]= ([10]0);

process Worker[i=1..10]{
    int inicio= (i-1)*tam;
    int fin= i*tam - 1;
    int total= 0;
    int rtdo;

    for j=inicio..fin{
        total= total + nros[j];
    }

    P(mutexContador);
    sumatoria= sumatoria + total;
    terminaron++;
    if (terminaron == 10){
        promedio= sumatoria / 1000000;
        for (k=1..10){
            promedios[k]= promedio;
            V(obtenerRtdo[k]);
        }
    }
    V(mutexContador);
    P(obtenerRtdo[i]);
    rtdo= promedios[i];
    imprimir(rtdo);
}
```

 9. A una acopiadora de cereales llegan 20 camiones que deben descargar su cereal. Los camiones se descargan de a uno por vez, y de acuerdo con orden de llegada. Una vez que el camión llegó, espera a que llegue su turno para comenzar a descargar. Sólo se pueden usar procesos que representen a los camiones; cada camión descarga una sola vez; la descarga del camión se simula por medio de la función DESCARGAR() llamada por el camión.
    *Solución propia:*
```
boolean libre= true; sem mutexFila= 1; queue colaE; sem espera[20]= ([20]0);

process Camion[i=1..20]{
    int aux;

    P(mutexFila);
    if (libre){
        libre= false;
        V(mutexFila);
    }
    else{
        push(colaE, i);
        V(mutexFila);
        P(espera[i]);
    }

    Descargar();
    P(mutexFila);
    if (not empty(colaE)){
        pop(colaE, aux);
        V(espera[aux]);
    }
    else{
        libre= true;
    }
    V(mutexFila);
}
```

 10. Existen B barcos areneros que deben descargar su contenido en una playa. Los barcos se descargan de a uno por vez, y de acuerdo con orden de llegada. Una vez que el barco llegó, espera a que le llegue su turno para comenzar a descargar su contenido, y luego se retira. Sólo se pueden usar los procesos que representen a los barcos, cada barco descarga sólo una vez; suponga que existe la función DESCARGAR() que simula que el barco está descargando contenido en la playa.
    *Solución propia:*
```
boolean libre= true; sem mutexFila= 1; queue colaE; sem espera[20]= ([20]0);

process Barco[i=1..20]{
    int aux;

    P(mutexFila);
    if (libre){
        libre= false;
        V(mutexFila);
    }
    else{
        push(colaE, i);
        V(mutexFila);
        P(espera[i]);
    }

    Descargar();
    P(mutexFila);
    if (not empty(colaE)){
        pop(colaE, aux);
        V(espera[aux]);
    }
    else{
        libre= true;
    }
    V(mutexFila);
}
```

 11. En un laboratorio de genérica trabajan 20 empleados que deben usar un pirosecuenciador de a uno a la vez, de acuerdo con el orden de llegada. Sólo se pueden usar los procesos que representen a los empleados, cada empleado usa sólo una vez el pirosecuenciador, suponga que existe la función USAR() que simula el uso de este instrumento por parte del empleado.
    *Solución propia:*
```
sem mutexFila= 1; queue colaE; sem espera[20]= ([20]0);
boolean libre= true;

process Empleado[i=1..20]{
    int aux;

    P(mutexFila);
    if (not libre){
        push(colaE, i);
        V(mutexFila);
        P(espera[i]);
    }
    else{
        libre= false;
        V(mutexFila);
    }

    Usar();
    P(mutexFila);
    if (not empty(colaE)){
        pop(colaE, aux);
        V(espera[aux]);
    }
    else{
        libre= true;
    }
    V(mutexFila);
}
```

 12. Se debe simular el uso de un sistema virtual de venta de entradas para un evento musical. El sistema cuenta con C cajeros virtuales que atienden indefinidamente. Sin embargo, como la venta de entradas comienza a una hora determinada, sólo atienden a partir del aviso de un timer. Una vez que reciben dicho aviso, los cajeros atienden de acuerdo con el orden de llegada de los compradores. La atención consiste en recibir la solicitud del comprador (datos para el pago) y responderle si puedo comprar junto al comprobante de la operación. Para este evento se cuenta con E entradas y N compradores, donde cada comprador puede solicitar a lo sumo una entrada.
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2024---Soluciones-parcial-MC1-versiona2.pdf` (parcial MC1 7-10-24, ej. 1, págs. 1-2)
```
sem mutexFila= 1;
queue colaC;
sem iniciarVenta= 0;
sem avisarAtencion= 0;
sem esperaRtdo[N]= ([N] 0);
text solicitudes[N];
text comprobantes[N];
int nrosEntrada[N];
sem mutexEntradas= 1;
int entradas= E;

    process Comprador[i=1..N]{
        text solicitud, comprobante;
        int nroEntrada;

        solicitud= generarSolicitud();
        solicitudes[i]= solicitud;
        P(mutexFila);
        push(colaC, i);
        V(mutexFila);
        V(avisarAtencion);
        P(esperaRtdo[i]);
        nroEntrada= nrosEntrada[i];         // -1 si no pudo comprar
        comprobante= comprobantes[i];
    }

    process Caja[i=1..C]{
        int idC, nroE;
        text solicitud, comprobante;

        P(iniciarVenta);
        while (true){
            P(avisarAtencion);
            P(mutexFila);
            pop(colaC, idC);
            V(mutexFila);
            solicitud= solicitudes[idC];
            P(mutexEntradas);
            if (entradas > 0){
                entradas--;
                nroE= entradas;
            } else {
                nroE= -1;
            }
            V(mutexEntradas);
            if (nroE != -1){
                comprobante= generarComprobante(solicitud, nroE);
            } else {
                comprobante= "sin entradas";
            }
            nrosEntrada[idC]= nroE;
            comprobantes[idC]= comprobante;
            V(esperaRtdo[idC]);             // despierta al comprador
        }
    }

    process Timer{
        delay(horaInicio);
        for (i=1..C){
            V(iniciarVenta);
        }
    }
```

 13. El CUIT es una clave que se usa en sistemas tributarios en Argentina para identificar a las personas. Consta de un total de once cifras numéricas siendo la última un dígito verificador (del 0 al 9). Una empresa cuenta con una lista de CUITs que debe procesar, debiendo informar la cantidad de CUITs por dígito verificador. Para ello, dispone de un software que emplea 5 workers, los cuales trabajan colaborativamente procesando de un CUIT por vez cada uno. Al finalizar el procesamiento, el último worker en terminar debe informar los resultados del procesamiento. Notas: la función obtenerDV(CUIT) retorna el dígito verificador para el CUIT recibida como entrada. La lista de CUITs se almacena como una cola global y la solución debe maximizar la concurrencia.
```
sem mutexCola= 1;
queue colaCuits;
sem digitos[10]= ([10] 1);
int cant_digitos[10]= ([10] 0);
int cantTerminaron= 0;
sem mutexAviso= 1;

    process Worker[i=1..5]{
        int cuit, digitoV;
        boolean seguir= true;

        while (seguir){
            P(mutexCola);
            if (empty(colaCuits)){
                seguir= false;
            } else {
                pop(colaCuits, cuit);
            }
            V(mutexCola);
            if (seguir){                    // solo procesa si sacó un CUIT
                digitoV= obtenerDV(cuit);
                P(digitos[digitoV]);
                cant_digitos[digitoV]++;
                V(digitos[digitoV]);
            }
        }
        P(mutexAviso);
        cantTerminaron++;
        if (cantTerminaron == 5){
            for (d=0..9){
                print(d, cant_digitos[d]);
            }
        }
        V(mutexAviso);
    }
```

 14. Existe una sala de cine 3D, a las que asisten N personas a ver una película. Antes de entrar a la sala, los asistentes deben retirar los anteojos 3D en la máquina repartidora que se encuentra en la entrada. Se debe simular el uso de la máquina repartidora de anteojos con capacidad para A anteojos (A < N). Además, existe un repositor encargado de reponer los anteojos en la máquina cuando se agotan. Los usuarios usan la máquina según el orden de llegada. Cuando les toca usarla, sacan un par de anteojos y luego se dirige a la sala. En caso de que la máquina se quede sin anteojos, entonces le debe avisar al repositor para que cargue nuevamente la máquina en forma completa. Luego de la recarga, saca un par de anteojos y se retira. La reposición de anteojos no debe impedir que otros puedan agregarse a la fila.
```
sem mutexFila= 1;
queue cola;
sem esperarTurno[N]= ([N] 0);
boolean libre= true;
int anteojos= A;
sem avisarRepositor= 0;
sem esperaDisponibilidad= 0;

    process Persona[i=1..N]{
        int aux;
        P(mutexFila);
        if (not libre){
            push(cola, i);
            V(mutexFila);
            P(esperarTurno[i]);
        } else {
            libre= false;
            V(mutexFila);
        }
        if (anteojos == 0){
            V(avisarRepositor);
            P(esperaDisponibilidad);
        }
        anteojos--;
        tomarAnteojos();
        P(mutexFila);
        if (not empty(cola)){
            pop(cola, aux);
            V(esperarTurno[aux]);
        } else {
            libre= true;
        }
        V(mutexFila);
    }

    process Repositor{
        while (true){
            P(avisarRepositor);
            delay(reposicion);
            anteojos= A;
            V(esperaDisponibilidad);
        }
    }
```

 15. En una estación de trenes, asisten P personas que deben realizar una carga de su tarjeta SUBE en la terminal disponible. La terminal es utilizada en forma exclusiva por cada personal de acuerdo con el orden de llegada. Implemente una solución usando únicamente procesos Persona. La función usarTerminal() le permite cargar la SUBE en la terminal disponible.
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2023---Soluciones-parcial-MC.pdf` (parcial MC 10-10-23, ej. 1 inciso a, pág. 1)
```
sem mutexFila= 1;
queue fila;
sem espera[P]= ([P] 0);
boolean libre= true;

process Persona[i=1..P]{
    int aux;
    P(mutexFila);
    if (not libre){
        push(fila, i);
        V(mutexFila);
        P(espera[i]);
    } else {
        libre= false;
        V(mutexFila);
    }
    usarTerminal();
    P(mutexFila);
    if (not empty(fila)){
        pop(fila, aux);
        V(espera[aux]);                     // despierta al siguiente
    } else {
        libre= true;
    }
    V(mutexFila);
}
```
 16. En una empresa hay un coordinar y 30 empleados que formarán 3 grupos de 10 empleados cada uno. Cada grupo trabaja en una sección diferente y debe realizar 345 unidades de un producto. Cada empleado al llegar se dirige al coordinador para que le indique el número de grupo al que pertenece y una vez que conoce este dato comienza a trabajar hasta que se han terminado de hacer las 345 unidades correspondientes al grupo (cada unidad es hecha por un único empleado). Al terminar de hacer las 345 unidades los 10 empleados del grupo se deben juntar para retirarse todos juntos. El coordinar debe atender a los empleados de acuerdo al orden de llegada para darle el número de grupo (a los 10 primeros que lleguen se le asigna el grupo 1, a los 10 del medio el 2 y a los 10 últimos el 3). Cuando todos los grupos terminaron de trabajar el coordinador debe informar (imprimir en pantalla) el empleado que más unidades ha realizado (si hubiese más de uno con la misma cantidad máxima debe informarlos a todos ellos. Existe una función generar() que simula la elaboración de una unidad de un producto.
```
sem mutexFila= 1;
queue colaE;
sem avisarAsignacion= 0;
sem esperaTurno[30]= ([30] 0);
int grupos[30]= ([30] 0);
sem mutexCant[3]= ([3] 1);
int cant[3]= ([3] 0);
sem mutexTerminaron[3]= ([3] 1);
int cantTerminaron[3]= ([3] 0);
sem retirarse[3]= ([3] 0);
int cantEmpleado[30]= ([30] 0);
sem mutexAvisarFin= 1;
int empleadosTerminados= 0;
sem avisarCoordinador= 0;

    process Empleado[i=1..30]{
        int nroGrupo;
        boolean seguir= true;
        int cantHechas= 0;

        P(mutexFila);
        push(colaE, i);
        V(mutexFila);
        V(avisarAsignacion);
        P(esperaTurno[i]);
        nroGrupo= grupos[i];
        while (seguir){
            P(mutexCant[nroGrupo]);         // primero reserva la unidad
            if (cant[nroGrupo] < 345){
                cant[nroGrupo]++;
            } else {
                seguir= false;
            }
            V(mutexCant[nroGrupo]);
            if (seguir){                    // y después la fabrica
                generar();
                cantHechas++;
            }
        }
        cantEmpleado[i]= cantHechas;
        P(mutexTerminaron[nroGrupo]);
        cantTerminaron[nroGrupo]++;
        if (cantTerminaron[nroGrupo] == 10){
            for j=1..10 {
                V(retirarse[nroGrupo]);
            }
        }
        V(mutexTerminaron[nroGrupo]);
        P(retirarse[nroGrupo]);
        P(mutexAvisarFin);
        empleadosTerminados++;
        if (empleadosTerminados == 30){
            V(avisarCoordinador);
        }
        V(mutexAvisarFin);
    }

    process Coordinador{
        int idE, max= -1;

        for g=1..3{
            for j=1..10{
                P(avisarAsignacion);
                P(mutexFila);
                pop(colaE, idE);
                V(mutexFila);
                grupos[idE]= g;
                V(esperaTurno[idE]);
            }
        }
        P(avisarCoordinador);
        for e=1..30{
            if (cantEmpleado[e] > max){
                max= cantEmpleado[e];
            }
        }
        for e=1..30{                        // puede haber más de uno con el máximo
            if (cantEmpleado[e] == max){
                print(e, max);
            }
        }
    }
```

 17. Se debe simular el uso de una máquina expendedora de gaseosas con capacidad para 100 latas por parte de U usuarios. Además existe un repositor encargado de reponer las latas de la máquina. Los usuarios usan la máquina según el orden de llegada. Cuando les toca usarla, sacan una lata y luego se retiran. En el caso de que la máquina se queda sin latas, entonces le debe avisar al repositor para que cargue nuevamente la máquina en forma completa. Luego de la recarga, saca una lata y se retira. Mientras se reponen las latas se deben permitir que otros usuarios puedan agregarse a la fila.
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2023---Soluciones-parcial-3.pdf` (parcial 18-12-23, ej. 1, págs. 1-2)
```
sem mutexFila= 1;
boolean libre= true;
queue colaU;
sem espera[U]= ([U] 0);
sem avisarRepositor= 0;
sem avisarDisponible= 0;
int cant= 100;

    process Usuario[i=1..U]{
        int aux;
        P(mutexFila);
        if (libre){
            libre= false;
            V(mutexFila);
        } else{
            push(colaU, i);
            V(mutexFila);
            P(espera[i]);
        }
        if (cant == 0){
            V(avisarRepositor);
            P(avisarDisponible);
        }
        cant--;
        sacarLata();
        P(mutexFila);
        if (not empty(colaU)){
            pop(colaU, aux);
            V(espera[aux]);
        } else {
            libre= true;
        }
        V(mutexFila);
    }

    process Repositor{
        while (true){
            P(avisarRepositor);
            delay(reposicion);
            cant= 100;
            V(avisarDisponible);
        }
    }
```

 18. En una fábrica de muebles trabajan 50 empleados. Al llegar, los empleados forman 10 grupos de 5 personas cada uno, de acuerdo al orden de llegada (los 5 primeros en llegar forman el primer grupo, los 5 siguientes al segundo grupo, y así sucesivamente). Cuando un grupo se ha terminado de formar, todos sus integrantes se ponen a trabajar. Cada grupo debe armar M muebles (cada mueble es armado por un solo empleado) mientras haya muebles por armar en el grupo los empleados los irán resolviendo (cada mueble es armado por un solo empleado). Cada empleado puede tardar distinto tiempo en armar un mueble. Sólo se pueden usar los procesos Empleado y todos deben terminar su ejecución.
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2022---Soluciones-parcial-fecha-3.pdf` (parcial 13-12-22, ej. 1, pág. 1)
```
sem mutexLlegada= 1;
int llegados= 0;
int cant_llegados[10]= ([10] 0);
sem mutexGrupo[10]= ([10] 1);
sem iniciarGrupo[10]= ([10] 0);
sem mutexMuebles[10]= ([10] 1);
int cantMuebles[10]= ([10] 0);

    process Empleado[i=1..50]{
        int nroGrupo;
        boolean seguir= true;

        P(mutexLlegada);
        nroGrupo= (llegados / 5) + 1;       // 0..4 -> grupo 1, 5..9 -> grupo 2, ...
        llegados++;
        V(mutexLlegada);
        P(mutexGrupo[nroGrupo]);
        cant_llegados[nroGrupo]++;
        if (cant_llegados[nroGrupo] == 5){
            for j=1..5{
                V(iniciarGrupo[nroGrupo]);
            }
        }
        V(mutexGrupo[nroGrupo]);
        P(iniciarGrupo[nroGrupo]);
        while (seguir){
            P(mutexMuebles[nroGrupo]);      // primero reserva el mueble
            if (cantMuebles[nroGrupo] < M){
                cantMuebles[nroGrupo]++;
            } else {
                seguir= false;
            }
            V(mutexMuebles[nroGrupo]);
            if (seguir){                    // y después lo arma
                armarMueble();
            }
        }
    }
```

 19. Simular un examen técnico para concursos NoDocentes de la facultad. En el mismo participan 100 personas distribuidas en 4 concursos con un coordinador en cada una de ellos. Cada persona ya conoce en qué concurso participa. El coordinador de cada concurso espera hasta que lleguen las 25 personas correspondientes al mismo, les entrega el examen a resolver y luego corrige los exámenes de esas 25 personas de acuerdo al orden en que van entregando. Cada persona al llegar debe esperar a que su coordinador le de el examen, lo resuelve, lo entrega para que su coordinador lo evalúe y espera hasta que le deje la nota para luego retirarse. Sólo usar los procesos que representen a las personas y a los coordinadores, todos deben terminar.
```
sem mutexFila[4]=([4] 1);
cantLlegados[4]=([4] 0);
queue fila[4];
sem avisarLlegada[4]=([4] 0);
sem esperaEnunciado=[100]= ([100] 0);
enunciados[100];
sem mutexExamenes[4]=([4] 1);
queue examenes[4];
sem avisarCorreccion[4]=([4] 0);
sem esperaCorreccion[100]=([100] 0);
notas[100];

    process NoDocente[i=1..100]{
        int nroConcurso, nota;
        text enunciado, examen;

        nroConcurso= obtenerNroConcurso();
        P(mutexFila[nroConcurso]);
        cantLlegados[nroConcurso]++;
        push(fila[nroConcurso], i);
        if (cantLlegados[nroConcurso] == 25){
            V(avisarLlegada[nroConsurso]);
        }
        V(mutexFila[nroConcurso]);
        P(esperaEnunciado[i]);
        enunciado= enunciados[i];
        examen= resolverExamen(enunciado);
        P(mutexExamenes[nroConcurso]);
        push(examenes[nroConcurso], (i, examen));
        V(mutexExamenes[nroConcurso]);
        V(avisarCorreccion[nroConcurso]);
        P(esperaCorreccion[i]);
        nota= notas[i];
    }

    process Coordinador[i=1..4]{
        int id, nota;
        text examen;

        P(avisarLlegada[i]);
        for j=1..25{
            pop(fila[i], id);
            enunciados[i]= generarEnunciado();
            V(esperaEnunciado[id]);
        }        
        for j=1..25{
            P(avisarCorreccion[i]);
            P(mutexExamenes[i]);
            pop(examenes[i], (id, examen));
            V(mutexExamenes[i]);
            nota= corregir(examen);
            notas[id]= nota;
            V(esperarCorreccion[id]);
        }
    }
```
 20. Se debe simular el funcionamiento de una mesa de votación en una elección donde hay 2 listas candidatas (A y B). A la misma acuden 300 personas para votar en el cuarto oscuro (se dispone de una función obtenervoto() que retorna la lista a votar A o B). Además está el presidente de mesa que es quien habilita a las personas a pasar a pasar al cuarto oscuro de a una a la vez de acuerdo al orden de llegada y cuando todas las personas han votado determina cual es la lista ganadora.
```
sem mutexFila= 1;
queue fila;
sem avisarLlegada=0;
sem esperaIngreso[300]= ([300] 0);
int cant[2]= ([2] 0);
sem avisarSalida= 0;

    process Persona[i=1..300]{
        char voto;

        P(mutexFila);
        push(fila, i);
        V(mutexFila);
        V(avisarLlegada);
        P(esperaIngreso[i]);
        voto= obtenerVoto();
        if (voto == 'A'){               // no hace falta mutex: hay una sola persona en el cuarto oscuro
            cant[1]++;
        } else{
            cant[2]++;
        }
        V(avisarSalida);
    }

    process Presidente{
        int id;
        for (j=1..300){
            P(avisarLlegada);
            P(mutexFila);
            pop(fila, id);
            V(mutexFila);
            V(esperaIngreso[id]);
            P(avisarSalida);            // espera a que salga antes de dejar pasar a otro
        }
        if (cant[1] > cant[2]) print("Gana A");
        else if (cant[2] > cant[1]) print("Gana B");
        else print("Empate");
    }
```

 21. En un examen final hay P alumnos y 3 profesores. Cuando todos los alumnos y profesores han llegado comienza el examen. Para esto uno de los profesores (el primero en llegar) le da el examen a cada alumno. Cada alumno resuelve su examen, lo entrega y espera a que alguno de los profesores lo corrija y le indique la nota. Los profesores corrigen los exámenes respetando el orden en que los alumnos van entregando.
```
sem mutexLlegada= 1;
int cantLlegados= 0;
sem avisarDisponible= 0;
sem esperaEnunciado[P]= ([P] 0);
enunciados[P];
sem mutexExamenes= 1;
queue colaE;
sem avisarEntrega= 0;
sem esperaNota[P]= ([P] 0);
notas[P];
sem mutexP= 1;
int llegada= 0;
sem iniciarEntrega= 0;
sem iniciarCorreccion[3]= ([3] 0);

    process Alumno[i=1..P]{
        int nota;
        text enunciado, examen;

        P(mutexLlegada);
        cantLlegados++;
        if (cantLlegados == P){
            V(avisarDisponible);
        }
        V(mutexLlegada);
        P(esperaEnunciado[i]);
        enunciado= enunciados[i];
        examen= resolver(enunciado);
        P(mutexExamenes);
        push(colaE, (i, examen));
        V(mutexExamenes);
        V(avisarEntrega);
        P(esperaNota[i]);
        nota= notas[i];
    }

    process Profesor[i=1..3]{
        int id, nota;
        text examen;

        P(mutexP);
        llegada++;
        if (llegada == 1){
            V(mutexP);
            P(iniciarEntrega);              // espera a los otros 2 profesores
            P(avisarDisponible);            // espera a todos los alumnos
            for (j=1..P){
                enunciados[j]= generarEnunciado();
                V(esperaEnunciado[j]);
            }
            for (j=1..3){
                V(iniciarCorreccion[j]);
            }
        } else {
            if (llegada == 3){
                V(iniciarEntrega);
            }
            V(mutexP);
        }
        P(iniciarCorreccion[i]);
        while (true){
            P(avisarEntrega);
            P(mutexExamenes);
            pop(colaE, (id, examen));
            V(mutexExamenes);
            nota= corregir(examen);
            notas[id]= nota;
            V(esperaNota[id]);
        }
    }
```


 22. En una clínica hay un médico que debe atender a 20 pacientes de acuerdo al turno de cada uno de ellos (no puede atender al paciente con turno i+1 si aún no se atendió al turno i). Cada paciente ya conoce su turno al comenzar, entre 1 a 20). Al llegar espera hasta que el médico lo llame para ser atendido, se dirige al consultorio y luego espera hasta que el médico lo llame para ser atendido, se dirige al consultorio y luego espera hasta que el médico lo termine de atender para retirarse. Los únicos procesos que se pueden usar son los que representen a los pacientes y al médico. Se debe asegurar que nunca haya más de un paciente en el consultorio, no se puede usar el ID del proceso como turno.

 23. Simular la atención de una planta verificadora de vehículos, donde se revisan cuestiones de seguridad de vehículos de a uno por vez. Hay N vehículos que deben ser verificados, donde algunos son autos y otros ambulancias. Antes de ser verificados, los autos deben hacer el pago correspondiente en la caja de la planta, donde le entregarán el recibo de pago. Las ambulancias no pagan. Los vehículos son atendidos de acuerdo al orden de llegada pero siempre dando prioridad a las ambulancias.

 24. Simular un examen escrito que deben rendir 60 alumnos repartidos en 3 aulas (20 alumnos en cada una) con un docente en cada una de ellas. Cada alumno ya tiene asignado el aula en la que debe rendir. El docente de cada aula espera hasta que sus 20 alumnos hayan llegado para darles el enunciado del examen (el mismo para todos) y luego les corrige el examen de acuerdo al orden en que van entregando. Cada alumno cuando llega debe esperar a que su docente le dé el enunciado del examen, lo resuelve, lo entrega para que el mismo lo corrija y le deje su nota. Cuando el alumno ya tiene su nota se retira.

 25. Simular la atención de una salita médica para vacunar contra el covid. Hay una enfermera encargada de vacunar 30 pacientes, cada paciente tiene un turno asignado. La enfermera atiende a los pacientes de acuerdo al turno que cada uno tiene asignado. Cada paciente al llegar espera a que sea su turno y se dirige al consultorio para que la enfermera lo vacune y luego se retira.

 26. Suponga un juego donde hay 30 competidores. Cuando los jugadores llegan avisan al encargado, una vez que están los 30, el encargado del jucgo les entrega un número aleatorio del 1 al 15 de tal manera que dos competidores tendrán el mismo número (Suponga que existe una función DarNumero() que devuelve en forma aleatoria un número del 1 al 15, el encargado no se guarda el numero que les asigna a los competidores). Una vez que ya se entregaron los 30 números, los competidores buscarán concurrentemente su compañero que tenga el mismo número (tenga en cuenta que pueden empezar a buscar cuando todos los competidores tengan un número ; Además la búsqueda de un jugador no interfiere con la búsqueda de otros que tengan distinto número). Cuando los competidores SE encuentran permanecen en una sala durante 15 minutos y dejan de jugar, Luego cada uno de los competidores avisa al encargado que terminó de jugar y espera que su compañero (el que tenía el mismo número) también avisa que finalizó para luego irse ambos, el encordado cuando llega el segundo competidor les devuelve a ambos el resultado que obtuvieron que es el orden en el que se van. (Los primeros en irse, tendrán como resultado 1, los últimos 15). Para modelizar el tiempo utilice la función Delay(x) que produce un retardo de x minutos.

 27. En una estación de trenes: existen P personas que deben realizar una carga de su tarjeta SUBE en la terminal disponible. La terminal es utilizada en forma exhaustiva por cada persona de acuerdo con el orden de llegada. Implemente una solución utilizando únicamente procesos Persona. La función usarTerminal() le permite cargar la SUBE en la terminal disponible.
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2023---Soluciones-parcial-MC.pdf` (parcial MC 10-10-23, ej. 1 inciso a, pág. 1)

 28. Para un experimento se tiene una red con 15 controladores de temperatura y dos módulos centrales. Los controladores cada cierto tiempo toman la temperatura mediante la función medir() y la envía para que alguna de las centrales le indique qué debe hacer (número de 1 a 10) y luego realiza esa acción mediante la función actuar(). Las centrales atienden los pedidos de los controladores de acuerdo al orden de llegada, usando la función determinar() para determinar la acción que deberá hacer ese controlador (número 1 a 10). El tiempo que espera cada controlador para tomar nuevamente la temperatura empieza a contar después de haber ejecutado la función actuar().

 29. En un restorán trabajan C cocineros y M mozos. De forma repetida, los cocineros preparan un plato y lo dejan listo en la bandeja de platos terminados, mientras que los mozos toman los platos de esta bandeja para repartirlos entre los comensales. Tanto los cocineros como los mozos trabajan de a un plato por vez. Modele el funcionamiento del restorán considerando que la bandeja de platos listos puede almacenar hasta P platos. No es necesario modelar a los comensales ni que los procesos terminen. NOTA: es el problema de productores/consumidores con tamaño de buffer limitado visto en la página 14 de la teoría 4.
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2022---Soluciones-parcial-fecha-2.pdf` (recuperatorio 2022, ej. 1, pág. 1)

 30. En una estación de trenes, asisten P personas que deben realizar una carga de su tarjeta SUBE. Hay T terminales disponibles. Las personas realizan una única fila (de acuerdo con el orden de llegada) y la carga la realizan en la primera terminal que se libera. Sólo deben emplearse procesos Persona. Nota: la función UsarTerminal(t) le permite cargar la SUBE en la terminal t. (Es el inciso b del ejercicio de la SUBE: el inciso a es el 15/27 con una sola terminal).
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2023---Soluciones-parcial-MC.pdf` (parcial MC 10-10-23, ej. 1 inciso b, págs. 1-2)

 31. Un sistema debe validar un conjunto de 10000 transacciones que se encuentran disponibles en una estructura de datos. Para ello, el sistema dispone de 7 workers, los cuales trabajan colaborativamente validando de a 1 transacción por vez cada uno. Cada validación puede tomar un tiempo diferente y para realizarla los workers disponen de la función Validar(t), la cual retorna como resultado un número entero entre 0 al 9. Al finalizar el procesamiento, el último worker en terminar debe informar la cantidad de transacciones por cada resultado de la función de validación. Nota: maximizar la concurrencia.
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2023---Soluciones-parcial-2.pdf` (parcial 4-12-23, ej. 1, pág. 1)

 32. Se debe simular el procesamiento de un sistema clasificador de 365 datasets meteorológicos. Todos los datasets se encuentran ya cargados en una estructura de datos. Para procesarla, el sistema dispone de 5 workers que trabajan colaborativamente procesando los datasets de a uno por vez. Cada procesamiento puede tomar un tiempo diferente y para realizarla los workers disponen de la función Procesar(d), la cual retorna como resultado un booleano que indica si los datos corresponden a clima tropical (true) o no (false). De acuerdo con el resultado, cada worker inserta el dataset en una cola de "tropicales" o de "no-tropicales". Nota: maximizar concurrencia.
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2025---Soluciones-parcial-MC1.pdf` (parcial MC1 6-10-25 (tema 1), ej. 1, pág. 1)

 33. En un taller de impresión 3D hay N clientes que solicitan piezas para ser impresas y esperan el resultado para poder retirarlas. El taller cuenta con una impresora 3D que fabrica las piezas en el orden de llegada de los pedidos. Una vez que le entregan la pieza, el cliente debe informar en pantalla si cumplió lo solicitado. Nota: la función ImprimirPieza(p) imprime la pieza en cuestión; la función ObtenerPedido(id) retorna el pedido de pieza para el cliente id; VerificarPieza(p) retorna true si el pedido p cumple lo solicitado o false en caso contrario.
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2025---Soluciones-parcial-MC1.pdf` (parcial MC1 6-10-25 (tema 1), ej. 2, pág. 2)

 34. Se debe simular el procesamiento de un sistema clasificador de 300 imágenes médicas. Todas las imágenes se encuentran ya cargadas en una estructura de datos. Para procesarla, el sistema dispone de 10 workers que trabajan colaborativamente procesando las imágenes de a una por vez. Cada procesamiento puede tomar un tiempo diferente y para realizarla los workers disponen de la función Procesar(img), la cual retorna como resultado un booleano que indica si los datos corresponden a un tejido anómalo (true) o no (false). De acuerdo con el resultado, cada worker inserta el dataset en una cola de "anomalas" o de "no-anomalas". Nota: maximizar concurrencia.
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2025---Soluciones-parcial-MC1.pdf` (parcial MC1 6-10-25 (tema 2), ej. 1, pág. 4)

 35. En un servicio técnico de celulares hay N clientes que llevan sus dispositivos a reparar y esperan la devolución. El local cuenta con un técnico que atiende las reparaciones de acuerdo con el orden de llegada de los clientes. Una vez que le entregan el dispositivo, el cliente debe informar en pantalla si fue reparado. Nota: la función RepararDispositivo(d) repara el dispositivo d; la función ObtenerPedido(id) retorna el pedido de dispositivo para el cliente id; VerificarDispositivo(d) retorna true si el dispositivo d fue reparado o false en caso contrario.
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2025---Soluciones-parcial-MC1.pdf` (parcial MC1 6-10-25 (tema 2), ej. 2, pág. 5)

 36. Una empresa de software cuenta con un sistema de compilación distribuida para acelerar el procesamiento de proyectos grandes. El sistema dispone de N workers, donde cada worker procesa exactamente un módulo. La compilación de cada módulo es independiente de las demás y la puede realizar cada worker en forma local. Una vez compilado, el worker debe subir su archivo objeto a un servidor de almacenamiento compartido (esta acción tarda un tiempo considerable). Este servidor solo puede ser usado por un worker a la vez, respetando el orden de llegada. La última etapa del proceso (enlace final) solo puede ejecutarse cuando todos los workers ya subieron su archivo al servidor. Una vez que todos los archivos fueron subidos, uno de los workers se ocupa de enlazar el proyecto para generar el ejecutable final (puede ser cualquiera de ellos, pero sólo uno). Implemente una solución al problema utilizando únicamente los procesos worker. Notas: maximizar la concurrencia; existe la función compilarModulo() que compila el módulo del worker que la invoca retornando un archivo objeto como resultado; existe la función subirArchivoObjeto(ao) sube el archivo objeto recibido como argumento al servidor compartido; la función enlazarProyecto() retorna el ejecutable final que debe guardarse en una variable compartida llamada eje.
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2025---Soluciones-parcial-MC2.pdf` (parcial MC2 1-12-25, ej. 1, pág. 1)

# Monitores
 1. Resolver con monitores el siguiente problema. En un sistema operativo se ejecutan 20 procesos que periódicamente realizan cierto cómputo mediante la función Procesar(). Los resultados de dicha función son persistidos en un archivo, para lo que se requiere de acceso al subsistema de E/S. Sólo un proceso a la vez puede hacer uso del subsistema de E/S, y el acceso al mismo se define por la prioridad del proceso (menos valor indica mayor prioridad).
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2022---Soluciones-parcial-MC.pdf` (parcial MC 11-10-22, ej. 2, pág. 3)
    *Solución a mano (no oficial):* `Parciales/Memoria compartida- 1ra parte/Parcial 1- 1ra fecha 2022.pdf` (ej. 2, pág. 2)
```
    process Proceso[i=1..20]{
        int prioridad= obtenerPrioridad();
        while (true){
            resultados= Procesar();
            SubsistemaES.pedir(i, prioridad);
            Persistir(resultados);
            SubsistemasES.liberar();
        }
    }

    Monitor SubsistemaES{
        int esperando= 0;
        int usando= 0;
        cond colas[N];
        queue(int, int) en_espera;

        procedure pedir(int id, int prioridad){
            if (usando > 0){
                esperando++;
                push(en_espera, (prioridad, id));
                wait[colas[id]];
            } else {
                usando++;
            }
        }

        procedure liberar(){
            if (esperando > 0){
                int id= pop(en_espera);
                signal(colas[id]);
                esperando--
            } else {
                usando--;
            }
        }
    }
```

 2. Simular una atención en una Salita Medica para vacunar contra el coronavirus. Hay UNA enfermera encargada de vacunar a 30 pacientes. Cada paciente tiene un turno asignado (valor entre 1 y 30 ya conocido por el paciente). La enfermera atiende a los pacientes de acuerdo al turno que cada uno tiene asignado. Cada paciente al llegar espera a que sea su turno y se dirige al consultorio para que la enfermera lo vacune y luego se retira. Nota: suponer que existe una función Vacunar() que simula la atención del paciente por parte de la enfermera. Todos los procesos DEBEN terminar.
```
    Monitor Sala{
        boolean llego[30]= ([30] false);
        cond esperaEnfermera;
        cond esperaFin[30];

        procedure llegar(int turno){
            llego[turno]= true;
            signal(esperaEnfermera);        
            wait(esperaFin[turno]);         // espera hasta que lo vacunen
        }

        procedure esperarPaciente(int turno){
            while (not llego[turno]) wait(esperaEnfermera);     
        }

        procedure terminarAtencion(int turno){
            signal(esperaFin[turno]);
        }
    }

    process Paciente[i=1..30]{
        int turno;

        turno= obtenerNroTurno();
        Sala.llegar(turno);
    }

    process Enfermera{
        for t=1..30{                        // atiende por turno: 1, 2, ..., 30
            Sala.esperarPaciente(t);
            Vacunar();                      // fuera del monitor
            Sala.terminarAtencion(t);
        }
    }
```

 3. Simular el uso de un puente de un solo carril y sentido por donde puede pasar un vehículo a la vez. Hay N vehículos donde algunos son autos y otros ambulancias. Los vehículos deben pasar de acuerdo al orden de llegada, pero siempre dando prioridad a las ambulancias. Suponga que cada vehículo tarda 5 minutos en pasar por el puente.
```
    process Vehiculo[i=1..N]{
        text tipoVehiculo;

        tipoVehiculo= obtenerTipo();
        Puente.ingreso(i, tipoVehiculo);
        CruzarPuente();
        Puente.salida();
    }

    Monitor Puente{
        boolean ocupado= false;
        queue filaE;
        cond esperaIngreso[N];
        process ingreso(int id, text tipoVehiculo){
            if (ocupado){
                insertarPorPrioridad(filaE, id, tipoVehiculo);
                wait(esperaIngreso[id]);
            } else {
                ocupado= true;
            }
        }

        process salida(){
            int aux;

            if (not empty filaE){
                pop(filaE, aux);
                signal(esperaIngreso[aux]);
            } else {
                ocupado= false;
            }
        }
    }
```

 4. Se debe simular una carrera a campo traviesa con C corredores, donde en la mitad del recorrido hay un puente colgante que puede ser usado por una única persona a la vez. Cuando los C corredores han llegado al punto de partida comienza la carrera.Cuando un corredor llega al puente espera su turno (respetando el orden de llegada al mismo) y lo cruza (suponga que tarda un par de minutos en cruzarlo) luego continua su carrera hasta llegar a la meta. Nota: Cada corredor pasa solo una vez por el puente. Solo se pueden usar procesos que representen a los corredores.
```
    process Corredor[i=1..C]{
        Puente.llegar();
        Correr();
        Puente.ingresar(i);
        CruzarPuente();
        Puente.salir();
    }

    Monitor Puente{
        boolean libre= true;
        queue fila;
        cond espera[C];

        procedure llegar(){
            llegados++;
            if (llegados < C){
                wait(esperaInicio);
            } else {
                signal_all(esperaInicio);
            }
        }

        procedure ingresar(int id){
            if (not libre){
                push(fila, id);
                wait(espera[id]);
            } else{
                libre= false;
            }
        }

        process salir(){
            int id;

            if not empty fila{
                pop(fila, id);
                signal(espera[id]);
            } else {
                libre= true;
            }
        }
    }
```

 5. Simular la atención en un centro de vacunación con 8 puestos para vacunación contra el coronavirus. Al centro acuden 200 paciente donde cada uno de ellos ya conoce el puesto al que se debe dirigir. En cada puestro hay un empleado para vacunar a los pacientes asignados a dicho puesto y lo hace de acuerdo al orden dado por la edad de paciente (cuando está libre atiende al de mayor edad que esté esperando en ese momento en ese puesto). Cada paciente al llegar al puesto que tenía asignado espera a que lo llamen y se dirige a la silla para que el empleado lo vacune y luego se retira. Suponer que existe una función Vacunar() que simula la atención del paciente. Suponer que cada puesto tiene asignado 25 pacientes. Todos los procesos deben terminar.
```
    Monitor Puesto[p=1..8]{
        queue fila;                         // ordenada por edad, de mayor a menor
        cond esperarLlamado[200];
        cond avisarDisponible;
        cond esperarIngreso;
        cond avisoRetiro;
        boolean llegoPaciente= false;

        procedure avisarLlegada(int id, int edad){
            insertarOrdenado(fila, id, edad);
            signal(avisarDisponible);
            wait(esperarLlamado[id]);
        }

        procedure llamarPaciente(){
            int id;
            while (empty(fila)) wait(avisarDisponible);
            pop(fila, id);                  // el de mayor edad en este momento
            signal(esperarLlamado[id]);
        }

        procedure sentarse(){               // avisa y espera el fin en la misma llamada
            llegoPaciente= true;
            signal(esperarIngreso);
            wait(avisoRetiro);
        }

        procedure esperarPaciente(){
            while (not llegoPaciente) wait(esperarIngreso);
            llegoPaciente= false;           // queda listo para el próximo
        }

        procedure avisarFin(){
            signal(avisoRetiro);
        }
    }

    process Paciente[i=1..200]{
        int nroPuesto, edad;

        edad= obtenerEdad();
        nroPuesto= obtenerPuesto();
        Puesto[nroPuesto].avisarLlegada(i, edad);
        Puesto[nroPuesto].sentarse();
    }

    process Empleado[e=1..8]{
        for j=1..25{
            Puesto[e].llamarPaciente();
            Puesto[e].esperarPaciente();
            Vacunar();
            Puesto[e].avisarFin();
        }
    }
```

 6. En una sala se juntan 20 conferencistas y un coordinador para una conferencia internacional. Cuando todos han llegado a la sala (los 20 conferenciados y el coordinador) el coordinador abre la sesión con una presentación de 30 minutos y luego cada conferencista realiza su presentación de 10 minutos de a uno a la vez y de acuerdo con el orden que lleguen a la sala. Cuando todas las presentaciones terminaron, las personas (conferencistas y coordinador) se retiran.
```
    Monitor Conferencia{
        int cant= 0;
        boolean llegaronTodos= false;
        boolean terminaron= false;
        queue fila;
        cond llegaronTotal;
        cond darConferencia[20];
        cond retirarseConferencia;

        procedure avisarLlegada(int id){
            cant++;
            push(fila, id);
            if (cant == 20){
                llegaronTodos= true;
                signal(llegaronTotal);
            }
            wait(darConferencia[id]);       // espera su turno para presentar
        }

        procedure finalizar(){
            int aux;
            if (not empty(fila)){
                pop(fila, aux);
                signal(darConferencia[aux]);    // le pasa la palabra al siguiente
                wait(retirarseConferencia);
            } else {                        // fue el último en presentar
                terminaron= true;
                signal_all(retirarseConferencia);
            }
        }

        procedure iniciarPresentacion(){
            if (not llegaronTodos){
                wait(llegaronTotal);
            }
        }

        procedure inicioConferencistas(){   // da la palabra y espera el fin en la misma llamada
            int aux;
            pop(fila, aux);
            signal(darConferencia[aux]);
            while (not terminaron) wait(retirarseConferencia);
        }
    }

    process Conferencista[i=1..20]{
        Conferencia.avisarLlegada(i);
        delay(10 min);                      // su presentación
        Conferencia.finalizar();
    }

    process Coordinador{
        Conferencia.iniciarPresentacion();
        delay(30 min);                      // presentación de apertura
        Conferencia.inicioConferencistas();
    }
```
 7. En una sala hay un profesor auxiliar y 100 alumnos. Cada alumno continuamente hace consultas que pueden ser de dos tipos: TEÓRICAS o PRÁCTICAS y cada vez que tiene una consulta para hacer se la envía por mail al docente correspondiente y espera a que este le envíe la respuesta. El profesor solo atiende consultas TEÓRICAS y el auxiliar sólo consultas PRÁCTICAS cada uno resuelve sus consultas de acuerdo con el órden de llegada. Maximizar concurrencia, el alumno sabe de qué tipo es cada consulta, los procesos no deben terminar.
```
    Monitor Mesa[m=1..2]{                   // 1 = profesor (teóricas), 2 = auxiliar (prácticas)
        queue consultas;                    // de (id, consulta), por orden de llegada
        cond hayConsulta;
        cond esperaRta[100];
        text respuestas[100];
        boolean listo[100]= ([100] false);

        procedure enviarConsulta(int id, text consulta, out text respuesta){
            push(consultas, (id, consulta));
            signal(hayConsulta);
            while (not listo[id]) wait(esperaRta[id]);     // espera la respuesta en la misma llamada
            respuesta= respuestas[id];
            listo[id]= false;               // queda listo para la próxima consulta
        }

        procedure obtenerConsulta(out int id, out text consulta){
            while (empty(consultas)) wait(hayConsulta);
            pop(consultas, (id, consulta));
        }

        procedure responder(int id, text respuesta){
            respuestas[id]= respuesta;
            listo[id]= true;
            signal(esperaRta[id]);
        }
    }

    process Alumno[i=1..100]{
        text consulta, tipo, respuesta;

        while (true){
            consulta= generarConsulta();
            tipo= obtenerTipo(consulta);
            if (tipo == "TEORICA"){
                Mesa[1].enviarConsulta(i, consulta, respuesta);
            } else {
                Mesa[2].enviarConsulta(i, consulta, respuesta);
            }
        }
    }

    process Docente[d=1..2]{                // 1 = profesor, 2 = auxiliar
        int id;
        text consulta, respuesta;

        while (true){
            Mesa[d].obtenerConsulta(id, consulta);
            respuesta= resolver(consulta);  // fuera del monitor: máxima concurrencia
            Mesa[d].responder(id, respuesta);
        }
    }
```

 8. En una acopiadora de cereales hay 2 empleados, uno para atender a los camiones de maíz y otro para los de girasol. Hay 30 camiones que llegan para descargar su carga (15 de maíz y 15 de girasol), cuando el camión llega espera hasta que el empleado correspondiente le avise que puede descargar el cereal. Cada empleado hace descargar los camiones que le corresponden de a uno a la vez y de acuerdo con el orden de llegada. El camión sabe qué tipo de cereal lleva; todos los procesos deben terminar.
```
    Monitor Acopiadora[a=1..2]{             // 1 = maíz, 2 = girasol
        queue fila;
        cond esperaIngreso;
        cond esperaTurno[30];
        cond avisoSalida;
        boolean salioCamion= false;

        procedure avisarLlegada(int id){
            push(fila, id);
            signal(esperaIngreso);
            wait(esperaTurno[id]);          // espera a que el empleado le avise que descargue
        }

        procedure avisarSalida(){
            salioCamion= true;
            signal(avisoSalida);
        }

        procedure siguienteCamion(){
            int aux;
            while (empty(fila)) wait(esperaIngreso);
            pop(fila, aux);
            signal(esperaTurno[aux]);
        }

        procedure esperarSalida(){
            while (not salioCamion) wait(avisoSalida);
            salioCamion= false;             // queda listo para el próximo camión
        }
    }

    process Camion[i=1..30]{
        text tipo;
        int nroAcopiadora;

        tipo= obtenerTipo();
        if (tipo == "maiz"){
            nroAcopiadora= 1;
        } else {
            nroAcopiadora= 2;
        }
        Acopiadora[nroAcopiadora].avisarLlegada(i);
        descargar();
        Acopiadora[nroAcopiadora].avisarSalida();
    }

    process Empleado[e=1..2]{
        for (j=1..15){
            Acopiadora[e].siguienteCamion();
            Acopiadora[e].esperarSalida();
        }
    }
```

 9.  En una oficina hay un supervisor, un empleado y 50 personas que solicitan por mail una operación. El empleado SOLO atiende CONSULTAS, mientras que el supervisor sólo atiende TRÁMITES; cada uno atiende sus pedidos de acuerdo con el orden de llegada. Cada persona envía UNA solicitud que puede ser un TRÁMITE o para una CONSULTA, y luego espera a que le envíe el resultado. La persona sabe de qué tipo es la consulta, el empleado y el supervisor NO deben terminar su ejecución.
```
    Monitor Mesa[m=1..2]{                   // 1 = supervisor (trámites), 2 = empleado (consultas)
        queue fila;                         // de (id, mensaje), por orden de llegada
        cond mensajePendiente;
        cond esperaRta[50];
        text rtas[50];

        procedure enviarMensaje(int id, text mensaje, out text rtdo){
            push(fila, (id, mensaje));
            signal(mensajePendiente);
            wait(esperaRta[id]);            // espera en la misma llamada: el aviso no se pierde
            rtdo= rtas[id];
        }

        procedure atender(out int id, out text mensaje){
            while (empty(fila)) wait(mensajePendiente);
            pop(fila, (id, mensaje));
        }

        procedure enviarRespuesta(int id, text rta){
            rtas[id]= rta;
            signal(esperaRta[id]);
        }
    }

    process Persona[i=1..50]{
        text tipo, mensaje, rtdo;
        int nro;

        mensaje= generarMensaje();
        tipo= obtenerTipo(mensaje);
        if (tipo == "tramite"){
            nro= 1;
        } else {
            nro= 2;
        }
        Mesa[nro].enviarMensaje(i, mensaje, rtdo);
    }

    process Empleado[e=1..2]{               // 1 = supervisor, 2 = empleado
        int id;
        text mensaje, respuesta;

        while (true){
            Mesa[e].atender(id, mensaje);
            respuesta= resolverMensaje(mensaje);    // fuera del monitor
            Mesa[e].enviarRespuesta(id, respuesta);
        }
    }
```

 10. Existen N personas que desean acceder a un mirador al borde del lago Nahuel Huapi en Bariloche. Como el
    mirador es angosto, sólo puede ser usado por una persona a la vez.
    a. El acceso al mirador es por orden de llegada.
    b. El acceso al mirador es por orden de llegada, pero dando prioridad a los mayores de 60 años.
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2024---Soluciones-parcial-MC1-versiona2.pdf` (parcial MC1 7-10-24, ej. 2 (incisos a y b), págs. 2-5)
```
a. Por orden de llegada

    Monitor Mirador{
        queue fila;
        cond espera[N];
        boolean libre= true;

        procedure ingresar(int id){
            if (not libre){
                push(fila, id);
                wait(espera[id]);
            } else {
                libre= false;
            }
        }

        procedure salir(){
            int aux;
            if (not empty(fila)){
                pop(fila, aux);
                signal(espera[aux]);        // le pasa el mirador directo, libre sigue en false
            } else {
                libre= true;
            }
        }
    }

    process Persona[i=1..N]{
        Mirador.ingresar(i);
        usarMirador();
        Mirador.salir();
    }

b. Por orden de llegada, con prioridad a los mayores de 60

    Monitor Mirador{
        queue filaMayores, filaResto;       // cada una respeta el orden de llegada
        cond espera[N];
        boolean libre= true;

        procedure ingresar(int id, int edad){
            if (not libre){
                if (edad > 60){
                    push(filaMayores, id);
                } else {
                    push(filaResto, id);
                }
                wait(espera[id]);
            } else {
                libre= false;
            }
        }

        procedure salir(){
            int aux;
            if (not empty(filaMayores)){
                pop(filaMayores, aux);
                signal(espera[aux]);
            } else if (not empty(filaResto)){
                pop(filaResto, aux);
                signal(espera[aux]);
            } else {
                libre= true;
            }
        }
    }

    process Persona[i=1..N]{
        int edad;

        edad= obtenerEdad();
        Mirador.ingresar(i, edad);
        usarMirador();
        Mirador.salir();
    }
```

 11. En una planta verificadora de vehículos existen 5 estaciones de verificación. Hay 75 vehículos que van para ser verificados, cada uno conoce el número de estación a la cual debe ir. Cada vehículo se dirige a la estación correspondiente y espera a que lo atiendan. Una vez que le entregan el comprobante de verificación, el vehículo se retira. Considere que en cada estación se atienden a los vehículos de acuerdo con el orden de llegada. Nota: maximizar la concurrencia.
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2022---Soluciones-parcial-fecha-2.pdf` (recuperatorio 2022, ej. 2, pág. 2)
```
    Monitor Estacion[s=1..5]{
        queue fila;
        cond hayVehiculo;
        cond espera[75];
        text comprobantes[75];
        boolean listo[75]= ([75] false);

        procedure llegar(int id, out text comp){
            push(fila, id);
            signal(hayVehiculo);
            while (not listo[id]) wait(espera[id]);
            comp= comprobantes[id];
        }

        procedure siguiente(out int id){
            while (empty(fila)) wait(hayVehiculo);
            pop(fila, id);
        }

        procedure entregar(int id, text comp){
            comprobantes[id]= comp;
            listo[id]= true;
            signal(espera[id]);
        }
    }

    process Vehiculo[i=1..75]{
        int nroEstacion;
        text comp;

        nroEstacion= obtenerEstacion();
        Estacion[nroEstacion].llegar(i, comp);
    }

    process Verificador[s=1..5]{
        int id;
        text comp;

        while (true){                       // no se sabe cuántos vehículos van a cada estación
            Estacion[s].siguiente(id);
            comp= verificar(id);            // fuera del monitor: maximiza la concurrencia
            Estacion[s].entregar(id, comp);
        }
    }
```

 12. En un comedor estudiantil hay un horno microondas que debe ser usado por E estudiantes de acuerdo con el orden de llegada. Cuando el estudiante accede al horno, lo usa y luego se retira para dejar al siguiente. Nota: cada Estudiante usa sólo una vez el horno; los únicos procesos que se pueden usar son los "estudiantes".
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2022---Soluciones-parcial-fecha-3.pdf` (parcial 13-12-22, ej. 2, pág. 2)

 13. En una elección estudiantil, se utiliza una máquina para voto electrónico. Existen N Personas que votan y una Autoridad de Mesa que les da acceso a la máquina de acuerdo con el orden de llegada, aunque ancianos y embarazadas tienen prioridad sobre el resto. La máquina de voto sólo puede ser usada por una persona a la vez. Nota: la función Votar() permite usar la máquina.
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2023---Soluciones-parcial-MC.pdf` (parcial MC 10-10-23, ej. 2, pág. 3)

 14. En una empresa trabajan 20 vendedores ambulantes que forman 5 equipos de 4 personas cada uno (cada vendedor conoce previamente a que equipo pertenece). Cada equipo se encarga de vender un producto diferente. Las personas de un equipo se deben juntar antes de comenzar a trabajar. Luego cada integrante del equipo trabaja independientemente del resto vendiendo ejemplares del producto correspondiente. Al terminar cada integrante del grupo debe conocer la cantidad de ejemplares vendidos por el grupo. Nota: maximizar la concurrencia.
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2023---Soluciones-parcial-2.pdf` (parcial 4-12-23, ej. 2, pág. 2)

 15. En una montaña hay 30 escaladores que en una parte de la subida deben utilizar un único paso de a uno a la vez y de acuerdo al orden de llegada al mismo. Nota: sólo se pueden utilizar procesos que representen a los escaladores; cada escalador usa sólo una vez el paso.
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2023---Soluciones-parcial-3.pdf` (parcial 18-12-23, ej. 2, pág. 2)

 16. En un evento hay 50 participantes que quieren tomarse fotos en una cabina fotográfica. La cabina puede ser usada por una persona a la vez, siguiendo el orden de llegada, pero dando prioridad a las personas de mayor edad (cuando la cabina está libre la debe usar la persona de mayor edad entre las que estén esperando para usarla). Cada participante se toma una única foto y luego se retira. Nota: solo se pueden utilizar procesos que representen a los participantes. La función obtenerEdad() retorna la edad de la persona; la función TomarFoto() simula el uso de la cabina.
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2025---Soluciones-parcial-MC1.pdf` (parcial MC1 6-10-25 (tema 1), ej. 3, pág. 3)

 17. En un parque natural hay 80 excursionistas que, en una parte del recorrido, deben cruzar un puente angosto que solo permite el paso de una persona a la vez. Los excursionistas cruzan siguiendo el orden de llegada pero dando prioridad a las personas de mayor edad (cuando el puente está libre, lo debe usar la persona de mayor edad entre las que estén esperando para usarlo). Cada excursionista cruza el puente una sola vez durante la excursión. Nota: solo se pueden utilizar procesos que representen a los excursionistas; la función obtenerEdad() retorna la edad de la persona; la función CruzarPuente() simula el cruce.
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2025---Soluciones-parcial-MC1.pdf` (parcial MC1 6-10-25 (tema 2), ej. 3, pág. 6)

 18. En un banco se utiliza un cajero automático para realizar operaciones bancarias. Existen N Clientes que deben usarlo y un Empleado de seguridad que administra el acceso de acuerdo con el orden dado por la edad (cuando el cajero está libre, deja pasar al de mayor edad de entre los que están esperando por entrar). El cajero automático solo puede ser usado por una persona a la vez. Implemente una solución al problema utilizando procesos para representar a los Clientes y al Empleado. Notas: existe la función UsarCajero() que representa el uso del cajero; cada cliente conoce su edad mediante la función obtenerEdad().
    *Solución:* `Parciales/Soluciones años anteriores/PConcu-2025---Soluciones-parcial-MC2.pdf` (parcial MC2 1-12-25, ej. 2, pág. 2)
