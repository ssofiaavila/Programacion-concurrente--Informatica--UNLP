# Pasaje de mensajes sincrónico (PMS)

1. Suponga que existe un antivirus distribuido que se compone de R procesos robots Examinadores y 1 proceso Analizador. Los procesos Examinadores están buscando
continuamente posibles sitios web infectados; cada vez que encuentran uno avisan la dirección, esperan la respuesta del Analizador, y luego continúan buscando. El proceso
Analizador se encarga de hacer todas las pruebas necesarias con cada uno de los sitios encontrados por los robots para determinar si están o no infectados, para luego responderle
al Examinador. 
   - a. Analice el problema y defina qué procesos, recursos y comunicaciones serán necesarios/convenientes para resolverlo.
       ```
       ```
   - b. Implemente una solución con PMS donde el Analizador no considere el orden de los pedidos de los Examinadores.
     ```
     ```
   - c. Implemente una solución con PMS donde el Analizador sí considere el orden de los pedidos de los Examinadores
     ```
     ```

2. En un laboratorio de genética veterinaria hay 3 empleados. El primero de ellos continuamente prepara las muestras de ADN; cada vez que termina, se la envía al segundo
empleado y vuelve a su trabajo. El segundo empleado toma cada muestra de ADN preparada, arma el set de análisis que se deben realizar con ella y espera el resultado para
archivarlo. Por último, el tercer empleado se encarga de realizar el análisis y devolverle el resultado al segundo empleado. 
    ```
    ```
3. En un examen final hay N alumnos y P profesores. Cada alumno resuelve su examen, lo entrega y espera a que alguno de los profesores lo corrija y le indique la nota. Los
profesores corrigen los exámenes respetando el orden en que los alumnos van entregando.  
   - a. Considerando que P=1.
     ```
     ```
   - b. Considerando que P>1.
     ```
     ```
   - c. Ídem b) pero considerando que los alumnos no comienzan a realizar su examen hasta que todos hayan llegado al aula.
     ```
     ```
Nota: maximizar la concurrencia; no generar demora innecesaria; todos los procesos deben terminar su ejecución.

4. En una exposición aeronáutica hay un simulador de vuelo (que debe ser usado con exclusión mutua) y un empleado encargado de administrar su uso. Hay P personas que
esperan a que el empleado las deje acceder al simulador de a una por vez, la usan por un rato y luego se retiran.
   - a. Implemente una solución donde el empleado sólo se ocupa de garantizar la exclusión mutua (sin importar el orden). 
     ```
     ```
   - b. Modifique la solución anterior para que el empleado los deje acceder según el orden de su identificador (hasta que la persona i no lo haya usado, la persona i+1 debe esperar).
      ```
      ```
   - c. Modifique la solución a) para que el empleado considere el orden de llegada para dar acceso al simulador
      ```
      ```
Nota: cada persona usa sólo una vez el simulador. 
5. En un estadio de fútbol hay una máquina expendedora de gaseosas que debe ser usada por E Espectadores de acuerdo con el orden de llegada. Cuando el espectador accede a la máquina en su turno, usa la máquina y luego se retira para dejar al siguiente. Nota: cada espectador usa la máquina sólo una vez.
