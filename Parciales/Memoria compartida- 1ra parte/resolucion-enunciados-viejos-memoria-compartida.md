# Resoluciones parciales anteriores
# Semáforos
> 1) En una planta verificadora de vehículos existen 7 estaciones donde se dirigen 150 vehículos para ser verificados. Cuando un vehículo llega a la planta, el coordinador de la planta le indica a qué estación debe dirigirse. El coordinador selecciona la estación que tenga menos vehículos en ese momento. Una vez qu el vehículo sabe qué estación le fue asignada, se dirige a la misma y espera a que lo llamen para verificar. Luego de la revisión, la estación le entrega un comprobante que indica si pasó la revisión o no. Más allá del resultado, el vehículo se retira de la planta. 
```
sem esperaV[150]= ([150]0);
sem mutex= 1;
sem esperaCoord = 0;
sem mutexPlanta = 1:
sem plantas[7] = ([7]0);

cola fila, plantas_cant= ([7] 0);
asignadas[150]= ([150] 0);
cola [7] fila_plantas;
boolean finDia= false;
text [150] resultado;

process Vehiculo [id= 1..150]{
    int idPlanta;
    text revision;

    P(mutex);
    fila.push(id); // avisa coordinar
    V(mutex);
    P(esperaV[id]);
    // se dirige a planta
    idPlanta= asignadas[id];
    P(mutexPlanta);
    fila_planta[idPlanta].push(id);
    V(mutexPlanta);
    V(plantas[idPlanta]);
    P(esperaV[id]);
    revision= resultado[id];
    // salida
}

process Coordinador{
    int idV;
    for (int i= 1..150){
        P(esperaCoord);
        P(mutex);
        idV= fila.pop();
        V(mutex);
        P(mutexPlanta);
        idPlanta= obtenerPunteroConMenos();
        plantas_cant[idPlanta]++;
        V(mutexPlanta);
        asignados[idV].idPlanta;
        V(esperaV[id]);   
    }
    P(mutex);
    finDia= true:
    V(mutex);
    for (int i= 1..7) V(planta[id]);
}

process Planta [id= 1..7]{
    P(mutex);
    white (not finDia){
        V(mutex);
        P(plantas[idPlanta]);
        P(mutexPlanta);
        if (fila_planta[id].empty() == false) {
            v= fila_planta[id].pop();
            V(mutexPlanta);
            resultados[v]= verificar();
            V(espera[v]);
            P(mutexPlanta);
            planta_cant[id]--;
            V(mutexPlanta);
        }
        P(mutex);      
    }
    V(mutex);
}
```

> 2) Una empresa de turismo posee un micro con capacidad para 50 personas. Hay un único vendedor que atiende a los C clientes (C > 50) que intentan comprar un pasaje de acuerdo al orden de llegada (suponga que la atención de un cliente tarda un par de minutos), si aún hay lugares disponibles, le indica el número de asiento que le tocó. Cada cliente, luego de ser atendidos por el vendedor, se dirige al micro para subir en caso de que le hayan dado un asiento. El micro espera a que los 50 pasajeros hayan subido para realizar el viaje.
```
int pasaje[C]= [(C) 0];
sem mutex= 1;
sem pedido= 0;
sem subioCliente= 0;
sen atendidos= [(C) 0];
queue fila;

process Pasajero[ id= 1..C]{
    P(mutex);
    push(fila, id);
    V(mutex);
    V(pedido);
    P(asignacion[id]);
    if (pasaje[id] > 0) V(subioCliente);
}

process Vendedor{
    int cantVendida= 0;
    idC= -1;

    for (int i=1 ..C){
        P(pedido);
        P(mutexCola);
        pop(fila, idC);
        V(mutexCola);
        if (cantVendida <= 50){
            cantVendida++;
            pasaje[idC]= darAsiento();
        }
        V(asignacion[idC]);
    }
}

process Micro{
    for (int i=1..50){
        P(subioCliente);
    }
}
```


> 3) En una competencia culinaria se juntan 1 jurado y C concursantes. Una vez que todos los concursantes llegaron, el jurado les asigna la receta que deben realizar. A continuación, los concursantes cocinan el plato pedido (a cada uno le lleva un tiempo variable) y lo exhiben ante el jurado en el orden en que van terminando. El jurado asigna un puntaje a cada concursante, el cual lo guarda para su CV.

```
sem mutexFila= 1;
sem llegaronPacientes= 0;
sem mutexEntrega= 0;
queue fila;
int esperando= 0;
sem esperaReceta[C]= [(C) 0];
text recetasAsignadas[C];
sem avisoJurado= 0;
sem esperaPuntaje[C]= [(C) 0];
int puntaje[C] = [(C) 0];

process Concursante[ id=1..C]{
    text receta;
    int puntaje;

    P(mutexFila);
    push(fila,id);
    esperando++;
    if (esperando == C){
        V(llegaronParticipantes);
    }
    V(mutexFila);
    P(esperaReceta[id]);
    receta= recetasAsignadas[id];
    delay(cocinar);
    P(mutexEntega);
    V(avisoJurado);
    P(esperaPuntaje[id]);
    puntaje= puntajes[id];
}

process Jurado{
    P(llegaronParticipantes);
    for (i= 1..C){
        recetasAsignadas[i]= asignarReceta();
        V(esperaReceta[id]);
    }
    for (i= 1..C){
        P(avisoJurado);
        P(mutexEntega);
        pop(platosEntregados, aux);
        V(mutexEntrega);
        puntaje= calificar(aux);
        puntaje[i]= puntaje;
        V(esperaPuntaje[id]);
    }
}
```

> 4) En un examen final hay P alumnos y 3 profesores. Cuando todos los alumnos y los profesores han llegado comienza el examen. Para esto uno de los profesores (el primero en llegar) le da el examen a cada alumno. Cada alumno resuelve su examen, lo entrega y espera que alguno de los profesores lo corrija y le indique la nota. Los profesores corrigen los examenes respetando el orden en que los alumnos van entregando.
```
sem terminoExamen= 0;
int notas[P]= ([P] 0 );
sem mutexLlegada= 1;
sem avisarProfesor= 0;
mutex examenes=0;
sen esperarEnunciado[P]= [(P) 0];
int esperando= 0;
text enunciados[P];
queue examenes;
sem esperaNota[P]= ([P] 0);

process Alumnos[i- 1..P]{
    P(mutexLlegada);
    esperando++;
    if (esperando == P){
        V(avisarProfesor);
    }
    V(mutexLlegada);
    P(esperarEnunciado[i]);
    enunciado= enunciados[i];
    examen= ResolverEnunciado();
    P(mutexExamenes);
    push(examenes,examen);
    V(mutexExamenes);
    V(terminoExamen);
    P(esperaNota[i]);
    nota= nota[i];
}

process Profesor [j= 1..3]{
    P(mutexProfesor);
    cantProfesores++;
    if (cantProfesores == 1){
        V(mutexProfesor);
        P(avisarProfesor);
        P(todosProfesores);
        for (int i=1..P){
            enunciado= generarEnunciado();
            enunciados[i]= enunciado;
            P(esperarEnunciad[i]);
        }
    } else {
        if (cantProfesores === 3){
            V(mutexProfesor);
            V(todosProfesores);
        } else V(mutexProfesor);
    }
    while (true){
        P(terminoExamen);
        P(mutexExamenes);
        pop(examenes, examen);
        V(mutexExamenes);
        nota= corregirExamen(examen);
        notas[examen.id]=nota;
        V(esperaNota[examen.id]);
    }
}

```
> 5) Simular la atención de una planta verificadora de vehículos, donde se revisan cuestiones de seguridad de vehículos de a uno por vez. Hay N vehículos que deben ser verificados, donde algunos son autos y otros ambulancias. Antes de ser verificados, los autos deben hacer el pago correspondiente en la caja de la planta, donde le entregarán un recibo de pago. Las ambulancias NO pagan. Los vehículos son atendidos de acuerdo al orden de llegada pero dando prioridad a las ambulancias.
```
sem mutexColaPago= 1:
sem esperaRecibo[N]= ([N] 0);
sem mutexColaVerifcacion= 1;
sem avisarSalida= 0;
queue colaPago;
text recibo[N];
queue colaVerificacion;

procedure Vehiculo[i=1..N]{
    string tipoVehiculo;
    text= recibo;
    if (tipoVehiculo== "auto"){
        P(mutexColaPago);
        push(colaPago, i);
        V(mutexColaPago);
        P(esperaRecibo[i]);
        recibo= recibo[i];
    }
    P(mutexColaVerificacion);
    insertarOrdenado(colaVerificacion, tipoVehiculo);
    V(mutexColaVerificacion);
    V(llamarEmpleado);
    P(verificacion[i]);
    serVerificado();
    V(avisarSalida);
}

process Caja{
    int aux;
    while (true){
        P(llamarEmpleado);
        P(mutexColaPago); 
        pop(colaPago, aux);
        recibos[aux]= generarRecibo();
        V(esperaRecibo[aux]);
    }
}

process Verificador{
    int aux;
    while (true){
        P(llamarEmpleado);
        P(mutexColaVerificacion);
        pop(colaVerificacion, aux);
        V(verificacion[aux]);
        P(avisarSalida);
    }
}
```

> 6)  En una clínica hay un médico que debe atender a 20 personas de acuerdo al turno de cada uno de ellos (no puede atender al paciente con turno i+1 si no atendió al que tiene el turno i). Cada paciente ya conoce su turno al comenzar (valor entre 0 y 19 entre 1 y 20). Al llegar espera hasta que le médico lo llame para ser atendido, se dirige al consultorio y luego espera hasta que el médico que lo termine de atender para poder retirarse. Nota: los únicos procesos que se pueden usar los que representan a los pacientes y al médico, se debe asegurar de que nunca haya más de un paciente en el consultorio, no se puede usar el ID del proceso como turno.

```
sem turnosPacientes[20]= ([20] 0);
sem esperaPaciente = 0;
sem avisaQueSeRetire= 0;
sem esperaQueSeVaya= 0;
process Persona[i=1..20]{
    int miTurno=..;
    P(turnosPacientes[miTurno]);
    V(esperaPaciente);
    P(avisaQueSeRetire);
    V(esperaQueSeVaya);
}

process Medico{
    int proximoTurno;
    for (proximoTurno= 1...20){
        V(turnosPaciente[proximoTurno];
        P(esperaPaciente);
        V(avisaQueSeRetire);
        P(esperaQueSeVaya);
    }
}

```

> 7) Se debe simular el uso de UNA máquina expendedora de agua con capacidad para 20 botellas por parte de U usuarios, además existe un repositorio encargado de reponer las botellas de máquina. Cuando un usuario llega a la máquina expendedora, espera su turno (representando el orden de llegada), saca una botella y se retira. Si encuentra la máquina sin botellas, le avisa al repositorio para que cargue nuevamente la máquina con 20 botellas. Espera a que se haga la recarga saca una botella y se retira. 


> 8) Se tiene un vector A de 1.000.000 de números del cual se debe obtener el promedio de sus valores utilizando 10 procesos Worker. Al terminar todos los procesos deben imprimir el resultado. Maximizar concurrencia, únicamente se pueden usar los 10 procesos worker.

> 9) A una acopiadora de cereales llegan 20 camiones que deben descargar su cereal. Los camiones se descargan de a uno por vez, y de acuerdo con orden de llegada. Una vez que el camión llegó, espera a que llegue su turno para comenzar a descargar. Sólo se pueden usar procesos que representen a los camiones; cada camión descarga una sola vez; la descarga del camión se simula por medio de la función DESCARGAR() llamada por el camión.

> 10) Existen B barcos areneros que deben descargar su contenido en una playa. Los barcos se descargan de a uno por vez, y de acuerdo con orden de llegada. Una vez que el barco llegó, espera a que le llegue su turno para comenzar a descargar su contenido, y luego se retira. Sólo se pueden usar los procesos que representen a los barcos, cada barco descarga sólo una vez; suponga que existe la función DESCARGAR() que simula que el barco está descargando contenido en la playa.

> 11) En un laboratorio de genérica trabajan 20 empleados que deben usar un pirosecuenciador de a uno a la vez, de acuerdo con el orden de llegada. Sólo se pueden usar los procesos que representen a los empleados, cada empleado usa sólo una vez el pirosecuenciador, suponga que existe la función USAR() que simula el uso de este instrumento por parte del empleado.

> 12) Se debe simular el uso de un sistema virtual de venta de entradas para un evento musical. El sistema cuenta con C cajeros virtuales que atienden indefinidamente. Sin embargo, como la venta de entradas comienza a una hora determinada, sólo atienden a partir del aviso de un timer. Una vez que reciben dicho aviso, los cajeros atienden de acuerdo con el orden de llegada de los compradores. La atención consiste en recibir la solicitud del comprador (datos para el pago) y responderle si puedo comprar junto al comprobante de la operación. Para este evento se cuenta con E entradas y N compradores, donde cada comprador puede solicitar a lo sumo una entrada. 

```
sem iniciarVenta[C]= ([C] 0);
sem mutexFila;
sem avisoCompra= (0);
sem esperaCompra[N]= ([N] 0);
queue fila;
compraExitosa boolean[N]= ([N] false);
compra text[N]= ([N] null);
int entradas= E;

procedure Cajero[i=1..C] {
    bool hayClientes= true;
    int cliente;

    V(iniciarVenta);
    while (hayClientes) {
        P(avisoCompra);
        P(mutexFila);
        pop(fila, cliente);
        V(mutexFila);
        P(mutexEntradas);
        if (entrada > 0){
            entrada--;
            quedanEntradas= entrada > 0;
            V(mutexEntradas);
            compra= generarCompra();
            compraExitosa[i]= true;
            compra[i]= compra;
            V(esperaCompra[i]);
        } else {
            quedanEntradas= false;
            V(mutexEntradas);
            compraExitosa[i]= false;
            V(esperaCompra[i]);
        }
        P(mutexFila);
        if (!fila.empty()){
            hayClientes= true;
        } else { hayClientes= false;}
    }
}


procedure Comprador[i=1..N] {
    text comprobante;

    P(mutexFila);
    push(fila, i);
    V(mutexFila);
    V(avisoCompra);
    V(esperaCompra[i]);
    if (compraExitosa[i]){
        comprobante= compra[i];
    }
}

procedure Timer {
    delay(tiempoDeInicio);
    for (i=1..C){
        V(iniciarVenta);
    }
}

```

> 13) El CUIT es una clave que se usa en sistemas tributarios en Argentina para identificar a las personas. Consta de un total de once cifras numéricas siendo la última un dígito verificador (del 0 al 9). Una empresa cuenta con una lista de CUITs que debe procesar, debiendo informar la cantidad de CUITs por dígito verificador. Para ello, dispone de un software que emplea 5 workers, los cuales trabajan colaborativamente procesando de un CUIT por vez cada uno. Al finalizar el procesamiento, el último worker en terminar debe informar los resultados del procesamiento. Notas: la función obtenerDV(CUIT) retorna el dígito verificador para el CUIT recibida como entrada. La lista de CUITs se almacena como una cola global y la solución debe maximizar la concurrencia.
```
sem mutexFila=1;
queue fila;
sem mutexNros[5]=([5] 1);
sem cantidad[5]=([5] 0);
int workers= 0;
sem mutexFinalizar;

procedure Workers [i=1..5]{
    int aux, nro;
    boolean sigo= true;
    
    while (sigo){
        P(mutexFila);
        if (not vacia(fila)){
            pop(fila, aux);
        } else {
            sigo= false;
        }
        if (aux != null){
            nro= obtenerDV(aux);
            P(mutexNros[nro]);
            cantidad[nro]++;
            V(mutexNros[nro]);
        }
    }
    P(mutexFinalizar);
    workers++;l
    if (workers==5)[
        V(mutexFinalizar);
        for (i=1..5){
            System.println(cantidad[n]);
        }
    ]
}
```

> 14) Existe una sala de cine 3D, a las que asisten N personas a ver una película. Antes de entrar a la sala, los asistentes deben retirar los anteojos 3D en la máquina repartidora que se encuentra en la entrada. Se debe simular el uso de la máquina repartidora de anteojos con capacidad para A anteojos (A < N). Además, existe un repositor encargado de reponer los anteojos en la máquina cuando se agotan. Los usuarios usan la máquina según el orden de llegada. Cuando les toca usarla, sacan un par de anteojos y luego se dirige a la sala. En caso de que la máquina se quede sin anteojos, entonces le debe avisar al repositor para que cargue nuevamente la máquina en forma completa. Luego de la recarga, saca un par de anteojos y se retira. La reposición de anteojos no debe impedir que otros puedan agregarse a la fila.
```
sem mutexFila=1;
boolean libre=true;
sem esperaTurno[N]= ([50] 0);
const BOTELLAS_TOTAL=...l
sem llamarRepositor, hayBotellas= 0;

process Persona [i= 1..N]{
    P(mutexFila);
    if (not libre){
        push(fila, i);
        V(mutexFila);
        P(esperaTurno[i]);
    } else {
        libre= true;
        V(mutexFila);
    }
    if (botellasDisponibles > 0){
        botellasDisponibles--;
        V(mutexMaquina);
    } else {
        V(llamarARepositor);
        P(hayBotellas);
    }
    P(mutexFila);
    if (!fila.empty()){
        pop(fila, aux);
        V(esperaTurno[aux]);
    } else {
        libre= true;
        V(mutexFila);
    }
}
```

> 15) En una estación de trenes, asisten P personas que deben realizar una carga de su tarjeta SUBE en la terminal disponible. La terminal es utilizada en forma exclusiva por cada personal de acuerdo con el orden de llegada. Implemente una solución usando únicamente procesos Persona. La función usarTerminal() le permite cargar la SUBE en la terminal disponible.
```
sem mutexFila= 1;
queue fila;
sem esperando[P]= ([P] 0);
int aux;

process Persona[i= 1...P]{
    P(mutexFila);
    if (not libre){
        push(fila, i);
        V(mutexFila);
        P(esperando[i]);
    } else {
        libre= false;
        V(mutexFila);
    }
    UsarTerminal();
    P(mutexFila);
    if (!fila.empty()){
        push(fila, aux);
        V(esperando[aux]);
    } else {
        libre= true;
        V(mutexFila);
    }
}
```

> 16) En una empresa hay un coordinar y 30 empleados que formarán 3 grupos de 10 empleados cada uno. Cada grupo trabaja en una sección diferente y debe realizar 345 unidades de un producto. Cada empleado al llegar se dirige al coordinador para que le indique el número de grupo al que pertenece y una vez que conoce este dato comienza a trabajar hasta que se han terminado de hacer las 345 unidades correspondientes al grupo (cada unidad es hecha por un único empleado). Al terminar de hacer las 345 unidades los 10 empleados del grupo se deben juntar para retirarse todos juntos. El coordinar debe atender a los empleados de acuerdo al orden de llegada para darle el número de grupo (a los 10 primeros que lleguen se le asigna el grupo 1, a los 10 del medio el 2 y a los 10 últimos el 3). Cuando todos los grupos terminaron de trabajar el coordinador debe informar (imprimir en pantalla) el empleado que más unidades ha realizado (si hubiese más de uno con la misma cantidad máxima debe informarlos a todos ellos. Existe una función generar() que simula la elaboración de una unidad de un producto.
```
queue fila;
sem mutexFila= 0;
sem avisarCoordinador= 0;
sem asignarEquipos= 0;
sem avisarFinalizacion[3]= ([3] 0);
int llegados[3]= ([3] 0);
sem esperaAsignacion[30]= ([30] 0);
grupoAsignado[30]= ([30] 0);
int cantGenerada[3]= ([3] 0);
sem llegada[3]= ([3] 0);

process Empleado[i=1..30]{
    int nroGrupo;
    boolean seguir= true;
    
    P(mutexFila);
    push(fila, i);
    V(mutexFila);
    V(asignarEquipos);
    P(esperaAsignacion[i]);
    nroGrupo= grupoAsignado[i];

    while (seguir){
        Generar();
        P(mutexCantidad[nroGrupo]);
        if (cantidadGenerados[nroGrupo] < 345){
            cantidadGenerada[nroGrupo++]l
            cantEmpelados[i]++;
            V(mutexCantidad[nroGrupo]);
        } else {
            seguir= false;
            P(mutexCantidad[nroGrupo]);
        }
    }
    P(llegada[nroGrupo]);
    llegados[nroGrupo]++;
    if (llegados[nroGrupo==10]){
        for(i=1..10){
            V(avisarFinalizacion[nroGrupo]);
        }
    } else {
        P(avisarFinalizacion[nroGrupo]);
    }
    P(finalizado);
    finalizaron ++;
    if (finalizacion == 30){
        V(avisarCoordinador);
    } else {
        V(finalizado);
    }
}

procedure Coordinador {
    int aux:
    for (i=1..30){
        P(asignarEquipos);
        P(mutexFila);
        pop(fila, aux);
        V(mutexFila);
        grupoAsignado[aux].asignarGrupo();
        V(esperaAsignacion[aux]);
    }
    P(avisarCoordinador);
    ganador= obtenerGanador();
    System.out.println(ganador);
}
```

> 17) Se debe simular el uso de una máquina expendedora de gaseosas con capacidad para 100 latas por parte de U usuarios. Además existe un repositor encargado de reponer las latas de la máquina. Los usuarios usan la máquina según el orden de llegada. Cuando les toca usarla, sacan una lata y luego se retiran. En el caso de que la máquina se queda sin latas, entonces le debe avisar al repositor para que cargue nuevamente la máquina en forma completa. Luego de la recarga, saca una lata y se retira. Mientras se reponen las latas se deben permitir que otros usuarios puedan agregarse a la fila.

> 18) En una fábrica de muebles trabajan 50 empleados. Al llegar, los empleados forman 10 grupos de 5 personas cada uno, de acuerdo al orden de llegada (los 5 primeros en llegar forman el primer grupo, los 5 siguientes al segundo grupo, y así sucesivamente). Cuando un grupo se ha terminado de formar, todos sus integrantes se ponen a trabajar. Cada grupo debe armar M muebles (cada mueble es armado por un solo empleado) mientras haya muebles por armar en el grupo los empleados los irán resolviendo (cada mueble es armado por un solo empleado). Cada empleado puede tardar distinto tiempo en armar un mueble. Sólo se pueden usar los procesos Empleado y todos deben terminar su ejecución.

> 19) Simular un examen técnico para concursos NoDocentes de la facultad. En el mismo participan 100 personas distribuidas en 4 concursos con un coordinador en cada una de ellos. Cada persona ya conoce en qué concurso participa. El coordinador de cada concurso espera hasta que lleguen las 25 personas correspondientes al mismo, les entrega el examen a resolver y luego corrige los exámenes de esas 25 personas de acuerdo al orden en que van entregando. Cada persona al llegar debe esperar a que su coordinador le de el examen, lo resuelve, lo entrega para que su coordinador lo evalúe y espera hasta que le deje la nota para luego retirarse. Sólo usar los procesos que representen a las personas y a los coordinadores, todos deben terminar.

```
sem mutexFila[4]= ([4] 0);
int llegadosGrupo[4]= ([4] 0);
sem avisoCoordinador[4]= ([4] 0);
sem esperaEnunciado[4]= ([4] 0);
sem mutexExamenesACorregir[4]= ([4] 0);
sem avisoExamenListo[4]= ([4] 0);
int resultados[4][25];
int enunciados [4][25]; //matriz para guardar los enunciados para los 100 alumnos
queue examenesACorregir[4]= ([4] 0);
sem esperarResultado[4][25]= ([4][25] 0);

procedure Persona [i= 1..100]{
    text enunciado, examen;
    int concurso= ...;

    P(mutexFila[concurso]); // me sumo a la cantidad por grupo
    llegadosGrupo[concurso]++;
    if (llegadosGrupo[concurso] == 25){ // si estamos todos le avisamos al coordinador
        V(avisoCoordinador[concurso]);
    }
    V(mutexFila[concurso]);
    P(esperaEnunciado[grupo]); //espero a mi enunciado
    enunciado= enunciados[concurso].[i];
    examen= realizarExamen(enunciado); // hago el examen
    P(mutexExamenesACorregir[concurso]); //sumo mi examen a la lista de los que hay que corregir
    push(examenesACorregir[concurso], examen); //vector de colas
    V(mutexExamenesACorregir[concurso]);
    V(avisoExamenListo[concurso][i]);
    P(esperarResultado[concurso][i]);
    resultado= resultados[i];
}

procedure Coordinador [i= 1..4]{
    text examen;
    P(avisoCoordinador[i]);
    for(m=1..25){
        enunciados[i][m]= generarEnunciado();
        V(esperaEnunciado[i]);
    }
    for (m= 1..25){
        P(avisoExamenListo[i]);
        P(mutexExamenesACorregir[i]);
        pop(examenesACorregir[i], examen);
        V(mutexExamenesACorregir[i]);
        resultado= corregirExamen(examen);
        resultados[i][examen.id]= resultado;
        P(esperarResultado[i][examen.id]);
    }
}
```

> 20) Se debe simular el funcionamiento de una mesa de votación en una elección donde hay 2 listas candidatas (A y B). A la misma acuden 300 personas para votar en el cuarto oscuro (se dispone de una función obtenervoto() que retorna la lista a votar A o B). Además está el presidente de mesa que es quien habilita a las personas a pasar a pasar al cuarto oscuro de a una a la vez de acuerdo al orden de llegada y cuando todas las personas han votado determina cual es la lista ganadora.

> 21) En un examen final hay P alumnos y 3 profesores. Cuando todos los alumnos y profesores han llegado comienza el examen. Para esto uno de los profesores (el primero en llegar) le da el examen a cada alumno. Cada alumno resuelve su examen, lo entrega y espera a que alguno de los profesores lo corrija y le indique la nota. Los profesores corrigen los exámenes respetando el orden en que los alumnos van entregando.


> 22) En una clínica hay un médico que debe atender a 20 pacientes de acuerdo al turno de cada uno de ellos (no puede atender al paciente con turno i+1 si aún no se atendió al turno i). Cada paciente ya conoce su turno al comenzar, entre 1 a 20). Al llegar espera hasta que el médico lo llame para ser atendido, se dirige al consultorio y luego espera hasta que el médico lo llame para ser atendido, se dirige al consultorio y luego espera hasta que el médico lo termine de atender para retirarse. Los únicos procesos que se pueden usar son los que representen a los pacientes y al médico. Se debe asegurar que nunca haya más de un paciente en el consultorio, no se puede usar el ID del proceso como turno.

> 23) Simular la atención de una planta verificadora de vehículos, donde se revisan cuestiones de seguridad de vehículos de a uno por vez. Hay N vehículos que deben ser verificados, donde algunos son autos y otros ambulancias. Antes de ser verificados, los autos deben hacer el pago correspondiente en la caja de la planta, donde le entregarán el recibo de pago. Las ambulancias no pagan. Los vehículos son atendidos de acuerdo al orden de llegada pero siempre dando prioridad a las ambulancias.

> 24) Simular un examen escrito que deben rendir 60 alumnos repartidos en 3 aulas (20 alumnos en cada una) con un docente en cada una de ellas. Cada alumno ya tiene asignado el aula en la que debe rendir. El docente de cada aula espera hasta que sus 20 alumnos hayan llegado para darles el enunciado del examen (el mismo para todos) y luego les corrige el examen de acuerdo al orden en que van entregando. Cada alumno cuando llega debe esperar a que su docente le dé el enunciado del examen, lo resuelve, lo entrega para que el mismo lo corrija y le deje su nota. Cuando el alumno ya tiene su nota se retira.

> 25) Simular la atención de una salita médica para vacunar contra el covid. Hay una enfermera encargada de vacunar 30 pacientes, cada paciente tiene un turno asignado. La enfermera atiende a los pacientes de acuerdo al turno que cada uno tiene asignado. Cada paciente al llegar espera a que sea su turno y se dirige al consultorio para que la enfermera lo vacune y luego se retira.


> 26) Suponga un juego donde hay 30 competidores. Cuando los jugadores llegan avisan al encargado, una vez que están los 30, el encargado del jucgo les entrega un número aleatorio del 1 al 15 de tal manera que dos competidores tendrán el mismo número (Suponga que existe una función DarNumero() que devuelve en forma aleatoria un número del 1 al 15, el encargado no se guarda el numero que les asigna a los competidores). Una vez que ya se entregaron los 30 números, los competidores buscarán concurrentemente su compañero que tenga el mismo número (tenga en cuenta que pueden empezar a buscar cuando todos los competidores tengan un número ; Además la búsqueda de un jugador no interfiere con la búsqueda de otros que tengan distinto número). Cuando los competidores SE encuentran permanecen en una sala durante 15 minutos y dejan de jugar, Luego cada uno de los competidores avisa al encargado que terminó de jugar y espera que su compañero (el que tenía el mismo número) también avisa que finalizó para luego irse ambos, el encordado cuando llega el segundo competidor les devuelve a ambos el resultado que obtuvieron que es el orden en el que se van. (Los primeros en irse, tendrán como resultado 1, los últimos 15). Para modelizar el tiempo utilice la función Delay(x) que produce un retardo de x minutos.

> 27) En una estación de trenes: existen P personas que deben realizar una carga de su tarjeta SUBE en la terminal disponible. La terminal es utilizada en forma exhaustiva por cada persona de acuerdo con el orden de llegada. Implemente una solución utilizando únicamente procesos Persona. La función usarTerminal() le permite cargar la SUBE en la terminal disponible.


> 28) Para un experimento se tiene una red con 15 controladores de temperatura y dos módulos centrales. Los controladores cada cierto tiempo toman la temperatura mediante la función medir() y la envía para que alguna de las centrales le indique qué debe hacer (número de 1 a 10) y luego realiza esa acción mediante la función actuar(). Las centrales atienden los pedidos de los controladores de acuerdo al orden de llegada, usando la función determinar() para determinar la acción que deberá hacer ese controlador (número 1 a 10). El tiempo que espera cada controlador para tomar nuevamente la temperatura empieza a contar después de haber ejecutado la función actuar().


# Monitores

> 5. Simular la atención en un centro de vacunación con 8 puestros para vacunación contra el coronavirus. Al centro acuden 200 paciente donde cada uno de ellos ya conoce el puesto al que se debe dirigir. En cada puestro hay un empleado para vacunar a los pacientes asignados a dicho puestro y lo hace de acuerdo al orden dado por la edad de paciente (cuando está libre atiende al de mayor edad que esté esperando en ese momento en ese puesto). Cada paciente al llegar al puesto que tenía asignado espera a que lo llamen y se dirige a la silla para que el empleado lo vacune y luego se retira. Suponer que existe una función Vacunar() que simula la atención del paciente. Suponer que cada puesto tiene asignado 25 pacientes. Todos los procesos deben terminar.

```
Monitor Puesto[i=1..8]{

queue fila;
cond avisoLlegada;
cond esperaLlamado[25];
cond esperaSalida;

    procedure vacunarse(i: in int, edad: in int){
        insertarsePorEdad(fila, i, edad);
        signal(avisoLlegada);
        wait(esperaLlamado[i]);
        signal(esperaSalida);
    }

    procedure vacunar(paciente: out paciente){
        if (! fila.empty()){
            wait(avisoLlegada);
        }
        pop(fila, paciente);
    }

    procedure esperarSalida(i: in int){
        signal(esperaLlamado[i]);
        wait(esperaSalida);
    }

}

procedure Paciente [i= 1..200]{
    int nroPuesto=...;
    int edad=...;
    Puesto[nroPuesto].vacunarse(i, edad);
}

procedure Empleado [i=1..8]{
    for (i=1..25){
        Puesto[i].vacunar(paciente);
        vacunar(paciente);
        Puesto[i].esperarSalida(paciente.id); //podría haber modelado mejor la interacción entre Paciente/ Empleado pero me cansé
    }
}
```

> 6. En una sala se juntan 20 conferencistas y un coordinador para una conferencia internacional. Cuando todos han llegado a la sala (los 20 conferenciados y el coordinador) el coordinador abre la sesión con una presentación de 30 minutos y luego cada conferencista realiza su presentación de 10 minutos de a uno a la vez y de acuerdo con el orden que lleguen a la sala. Cuando todas las presentaciones terminaron, las personas (conferencistas y coordinador) se retiran.

```
Monitor Admin {
int llegaron= 0, dieronCharla= 0;
queue filaConferencistas;
cond inicioCoordinador, espera[20];

    procedure ingresarEvento(i: in int){
        llegaron++;
        push(filaConferencistas, i);
        if (llegaron == 20){
            signal(inicioCoordinador);
        }
        wait(espera[i]);
    }

    procedure iniciarCharla(){
        if (llegaron < 20){ wait(inicioCoordinador)};
    }

    procedure llamarPrimerConferencista(){
        int proximo;
        push(filaConferencistas, proximo);
        signal(espera[i]);
        wait(finalizacion);
    }

    procedure finalizarCharla(){
        int siguiente;
        dieronCharla++;

        if (dieronCharla < 20 ){
            pop(filaConferencistas, siguiente);
            signal(siguiente);
            wait(finalizacion);
        } else {
            signal_all(finalizacion);
        }
    }
}

process Conferencista[i=1..20]{
    Admin.ingresarEvento(i);
    delay(10"); // doy mi conferencia
    Admin.finalizarCharla();
}

process Coordinador{
    Admin.iniciarCharla();
    delay(30"); // charla del coordinador
    Admin.llamarPrimerConferencista();
}
```

> 7. En una sala hay un profesor auxiliar y 100 alumnos. Cada alumno continuamente hace consultas que pueden ser de dos tipos: TEÓRICAS o PRÁCTICAS y cada vez que tiene una consulta para hacer se la envía por mail al docente correspondiente y espera a que este le envíe la respuesta. El profesor solo atiende consultas TEÓRICAS y el auxiliar sólo consultas PRÁCTICAS cada uno resuelve sus consultas de acuerdo con el órden de llegada. Maximizar concurrencia, el alumno sabe de qué tipo es cada consulta, los procesos no deben terminar.

```

// NOTA: 1= PROFESOR, 2= AUXILIAR, podría hacer un monitor + proceso nombrado a cada entidad pero repito código

Monitor Mail[i= 1..2]{

    procedure enviarConsulta(consulta: in Consulta, respuesta: in text){
        push(consultas, consulta);
        signal(hayConsulta);
        wait(hayRespuesta);
        pop(respuestas, respuesta);
    }

    procedure obtenerConsulta(consulta: in Consulta){
        if (!consultas.empty()){
            wait(hayConsulta);
        }
        pop(consultas, consulta);
    }

    procedure enviarRespuesta(respuesta: in Respuesta){
        pop(respuestas, respuesta);
        signal(hayRespuesta);
    }
}
procedure Alumno{
    Consulta consulta;
    queue respondidas;

    while (true){
        consulta= generarConsulta();
        Mail[consulta.tipo].enviarConsulta(respuesta);
        push(respondidas, respuesta);
    }
}

procedure Docente [i= 1..2]{
    Consulta consulta;
    text respuesta;

    while (true){
        Mail[i].obtenerConsulta(consulta);
        respuesta= generarRespuesta(consulta);
        Mail[i].enviarRespuesta(respuesta);
    }
}
```

> 8. En una acopiadora de cereales hay 2 empleados, uno para atender a los camiones de maíz y otro para los de girasol. Hay 30 camiones que llegan para descargar su carga (15 de maíz y 15 de girasol), cuando el camión llega espera hasta que el empleado correspondiente le avise que puede descargar el cereal. Cada empleado hace descargar los camiones que le corresponden de a uno a la vez y de acuerdo con el orden de llegada. El camión sabe qué tipo de cereal lleva; todos los procesos deben terminar.

```

//NOTA: 1= maíz, 2= girasol
Monitor Admin [i=1..2]{
queue esperando;

    procedure avisarLlegada(i: in int){
        push(esperando, i);
        signal(hayCamion);
        wait(realizarDescarga);
    }

    procedure llamarCamion(){
        if (!esperando.empty()){
            wait(hayCamion);
        }
        pop(esperando, aux);
        signal(realizarDescarga);
        wait(descargaFinalizada);
    }

    procedure avisarFinalizacion(){
        signal(descargaFinalizada);
    }
}

procedure Camion [i= 1..30]{
    int tipo=...;
    Admin[tipo].avisarLlegada(i);
    delay(descargar); //tiempo que demora la descarga
    Admin[tipo].avisarFinalizacion();
}

procedure Empleado [i=1..2]{

    for (i=1..15){
        Admin[i].llamarCamion();
    }

}
```

> 9. En una oficina hay un supervisor, un empleado y 50 personas que solicitan por mail una operación. El empleado SOLO atiende CONSULTAS, mientras que el supervisor sólo atiende TRÁMITES; cada uno atiende sus pedidos de acuerdo con el orden de llegada. Cada persona envía UNA solicitud que puede ser un TRÁMITE o para una CONSULTA, y luego espera a que le envíe el resultado. La persona sabe de qué tipo es la consulta, el empleado y el supervisor NO deben terminar su ejecución.

```
NOTA: 1= SUPERVISOR, 2= EMPLEADO, podría hacer un monitor + proceso nombrado a cada entidad pero repito código


Monitor Oficina[i=1..2]{

    procedure enviarConsulta(operacion: in Operacion, respuesta: out respuesta){
        push(operaciones, operacion);
        signal(hayConsulta);
        wait(esperaRespuesta);
        pop(respuestas, respuesta);
    }

    procedure responderConsulta(operacion: out Operacion){
        if (!operaciones.empty()){
            wait(hayConsulta);
        }
        pop(operaciones, operacion);
    }

    procedure enviarRespuesta(respuesta: in Respuesta){
        push(respuestas, respuesta);
        signal(esperaRespuesta);
    }
}

procedure Persona [i=1..50]{
    Operacion operacion;
    Oficina[operacion.tipo].enviarConsulta(operacion);
}

procedure Personal [i=1..2]{
    Operacion operacion;
    text respuesta;

    while (true){
        Admin[i].responderConsulta(operacion);
        respuesta= generarRespuesta(operacion);
        Admin[i].enviarRespuesta(respuesta);
    }
}

```

> 10. Existen N personas que desean acceder a un mirador al borde del lago Nahuel Huapi en Bariloche. Como el
    mirador es angosto, sólo puede ser usado por una persona a la vez.
    a. El accedo al mirador es por orden de llegada.
    b. El acceso al mirador es por orden de llegada, pero dando prioridad a los mayores de 60 años.

```
procedure Persona [i=1..N]{
    Mirador.ingresar(i);
    delay(paseoMirador);
    Mirador.salir();
}

Monitor Mirador {

    bool libre= true;
    cond pasarAMirador;
    queue fila;

    procedure ingresar(i: in int){
        if (not libre){
            push(fila, i);
            wait(pasarAMirador);
        } else{
            libre= false;
        }
    }

    procedure salir(){
        if (!cola.empty()){
            pop(fila, siguiente);
            signal(pasarAMirador);
        }
        else {
            libre= true;
        }
    }
}
```
