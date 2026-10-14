# Pasaje de mensajes sincrónico (PMS)

1. Suponga que existe un antivirus distribuido que se compone de R procesos robots Examinadores y 1 proceso Analizador. Los procesos Examinadores están buscando
continuamente posibles sitios web infectados; cada vez que encuentran uno avisan la dirección, esperan la respuesta del Analizador, y luego continúan buscando. El proceso
Analizador se encarga de hacer todas las pruebas necesarias con cada uno de los sitios encontrados por los robots para determinar si están o no infectados, para luego responderle
al Examinador. 
   - a. Analice el problema y defina qué procesos, recursos y comunicaciones serán necesarios/convenientes para resolverlo.
       ```
       procesos: Examinados [R], Analizador
       comunicaciones reportar
       ```
   - b. Implemente una solución con PMS donde el Analizador no considere el orden de los pedidos de los Examinadores.
     ```
      process Examinador[i=1..R]{
        text sitioInfectado, respuesta;

        while (true){
          sitioInfectado= analizarWeb();
          Analizador!reportar(sitioInfectado);
        }
      }

      process Analizador {
        texto, reporte, rta;
        
        while (true) {
          Examinador[*]?reportar(reporte);
          Analizar(reporte);
        }
      }

     ```
   - c. Implemente una solución con PMS donde el Analizador sí considere el orden de los pedidos de los Examinadores
     ```
      process Examinador [i=1..R] {
        text sitioInfectado;

        while (true) {
          sitioInfectado= analizarWeb();
          Admin!reportar(sitioInfectado);
        }
      } 

      process Admin {
        queue colaDirecciones;
        text sitio;

        do Examinador?[*]reportar(sitio) -> push(colaDirecciones, sitio);
           not empty(colaDirecciones); Analizador?solicitarDireccion() -> pop(colaDirecciones, sitio);
                                                                          Analizador!entregaDireccion(sitio);
        od
      }

      process Analizador {
        text sitio;

        while (true){
          Admin!solicitarDireccion();
          Admin?entregaDireccion(sitio);
          Analizar(sitio);
        }
      }
     ```

2. En un laboratorio de genética veterinaria hay 3 empleados. El primero de ellos continuamente prepara las muestras de ADN; cada vez que termina, se la envía al segundo
empleado y vuelve a su trabajo. El segundo empleado toma cada muestra de ADN preparada, arma el set de análisis que se deben realizar con ella y espera el resultado para
archivarlo. Por último, el tercer empleado se encarga de realizar el análisis y devolverle el resultado al segundo empleado. 
    ```
      process Empleado1 {
        text muestra;

        while (true) {
          muestra= generarMuestra();
          Admin!enviarMuestra(muestra);
        }
      }

      process Admin {
        text muestra;
        queue recibidos;

        do Empleado1?enviarMuestra(muestra) -> push(recibidos, muestra);
           not empty (recibidos); Empleado2?solicitarMuestra() -> pop(recibidos, muestra);
                                                                  Empleado2!enviar(muestra);
        od
      }

      process Empleado2 {
        text muestra, set, resultado;

        while (true){
          Admin!solicitarMuestra();
          Admin?enviar(muestra);
          set= generarSetAnalisis(muestra);
          Empleado3!enviarSet(set);
          Empleado3?respueta(resultado);
          Archivar(resultado);
        }
      }

      process Empleado3 {
        text set, respuesta;

        while (true) {
          Empleado2?enviarSet(set);
          respuesta= generarRespuesta(set);
          Empleado2!respuesta(set)
        }
      }
    ```
3. En un examen final hay N alumnos y P profesores. Cada alumno resuelve su examen, lo entrega y espera a que alguno de los profesores lo corrija y le indique la nota. Los
profesores corrigen los exámenes respetando el orden en que los alumnos van entregando.  
   - a. Considerando que P=1.
     ```
      process Alumno [i=1..N] {
        text examen;
        int nota;

        examen= resolverExamen();
        Admin!entregarExamen(examen, i);
        Profesor?respuesta(nota);
      }

      process Admin {
        text examen;
        queue examenes;
        int idA;
        do Alumno[*]?entregarExamen(examen) -> push(examenes, (examen, idA));
           not empty (examenes); Profesor?espera() --> pop(examenes, examen, idA);
                                                       Profesor!enviar(examen, idA);
        od
      }

      process Profesor {
        text examen;
        int nota, id;

        while (true){
          Admin!espera();
          Admin?enviar(examen, id);
          nota= corregir(examen);
          Alumno[id]!respuesta(nota);
        }
      }
     ```
   - b. Considerando que P>1.
     ```
      process Alumno [i=1..N] {
        text examen;
        int nota;
        
        examen= resolverExamen();
        Admin!entregarExamen(examen,i);
        Profesor?respuesta(nota);
      }

      process Profesor [i=1..P] {
        text examen,
        int nota, id;

        while (true){
          Admin!espera(i);
          Admin?enviar(examen,id);
          nota= corregir(examen);
          Alumno[id]!respuesta(nota);
        }
      }

      process Admin {
        text examen;
        queue examenes;
        int idA, idP;

        do Alumno[*]?entregarExamen(examen) -> push(examenes, (examen, idA));
           not empty(examenes); Profesor?espera(idP); --> pop(examenes, (examen, idA));
                                                          Profesor[idP]!enviar(examen,idA);
        od
      }
     ```
   - c. Ídem b) pero considerando que los alumnos no comienzan a realizar su examen hasta que todos hayan llegado al aula.
     ```
      process Alumno [i=1..N] {
        text examen;
        int nota;
        
        examen= resolverExamen();
        Admin!entregarExamen(examen,i);
        Profesor?respuesta(nota);
      }

      process Profesor [i=1..P] {
        text examen,
        int nota, id;

        while (true){
          Admin!espera(i);
          Admin?enviar(examen,id);
          nota= corregir(examen);
          Alumno[id]!respuesta(nota);
        }
      }

      process Admin {
        text examen;
        queue examenes;
        int idA, idP, cant=0;


        for (i= 1..N) {
          Alumno[*]?avisarLlegada();
          cant++;
        }

        for (i-1..N){
          Alumno[i]!iniciarExamen();
        }

        do Alumno[*]?entregarExamen(examen) -> push(examenes, (examen, idA));
           not empty(examenes); Profesor?espera(idP); --> pop(examenes, (examen, idA));
                                                          Profesor[idP]!enviar(examen,idA);
        od
      }
     ```
Nota: maximizar la concurrencia; no generar demora innecesaria; todos los procesos deben terminar su ejecución.

4. En una exposición aeronáutica hay un simulador de vuelo (que debe ser usado con exclusión mutua) y un empleado encargado de administrar su uso. Hay P personas que
esperan a que el empleado las deje acceder al simulador de a una por vez, la usan por un rato y luego se retiran.
   - a. Implemente una solución donde el empleado sólo se ocupa de garantizar la exclusión mutua (sin importar el orden). 
     ```
      process Persona [i=1..P]{
        Empleado!avisar(i);
        Empleado?pasar();
        UsarSimulador();
        Empleado!liberar();
      }

      process Empleado {
        int idP;
        for (i-1..P){
          Persona[*]?avisar(idP);
          Persona[idP]!pasar();
          Persona[idP]?liberar();
        }
      }
     ```
   - b. Modifique la solución anterior para que el empleado los deje acceder según el orden de su identificador (hasta que la persona i no lo haya usado, la persona i+1 debe esperar).
      ```
        process Persona[i=1..P]{
          Empleado!avisar(i);
          Empleado?pasar();
          UsarSimulador();
          Empleado!liberar();
        }

        process Empleado {
          for (i=1.P){
            Persona[i]?avisar();
            Persona[i]!pasar();
            Persona[i]?liberar();
          }
        }
      ```
   - c. Modifique la solución a) para que el empleado considere el orden de llegada para dar acceso al simulador
      ```
        process Persona[i=1..P]{
          Admin!avisar(i);
          Empleado?pasar();
          UsarSimulador();
          Empleado!liberar();
        }

        process Admin(){
          queue esperando;
          int idP;

          do Persona[*]?avisar(idP) -> push(esperando, idP);
             not empty(esperando); Empleado?libre() -> pop(examenes, idP);
                                                       Empleado!enviar(idP);
          od
        }

        process Empleado {
          int idP;

          while (true){
            Admin!libre();
            Admin?enviar(idP);
            Persona[idP]!pasar();
            Persona[idP]?liberar();
          }
        }
      ```
Nota: cada persona usa sólo una vez el simulador. 
5. En un estadio de fútbol hay una máquina expendedora de gaseosas que debe ser usada por E Espectadores de acuerdo con el orden de llegada. Cuando el espectador accede a la máquina en su turno, usa la máquina y luego se retira para dejar al siguiente. Nota: cada espectador usa la máquina sólo una vez.
```
        process Espectador [i=1..E] {
          Admin!llegada(i);
          Admin?turno();
          UsarMaquina();
          Admin!liberarMaquina();
        }

        process Admin {
          boolean libre= true;
          int idE, espectadores= 0; eActual;
          queue colaE;

          do espectadores < E; Espectador[*]?llegada(idE) -> if (empty(colaE)){
                                                              libre= false;
                                                              Espectaodor[idE]?turno();
                                                              eActual= idE; 
                                                            } else { push(colaE, idE)}
            espectadores < E; Espectador[eActual]?liberarMaquina() -> if (empty(colaE)){
                                                                        libre= true;
                                                                      } else {
                                                                        pop(colaE, idE);
                                                                        Espectador[idE]!turno();
                                                                        eActual= idE;
                                                                      }
          od            
        }
```