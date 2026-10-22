# Resolución práctica 5 | ADA
1. Se requiere modelar un puente de un único sentido que soporta hasta 5 unidades de peso. El peso de los vehículos depende del tipo: cada auto pesa 1 unidad, cada camioneta pesa 2
unidades y cada camión 3 unidades. Suponga que hay una cantidad innumerable de vehículos (A autos, B camionetas y C camiones). Analice el problema y defina qué tareas,
recursos y sincronizaciones serán necesarios/convenientes para resolverlo.
   - a. Realice la solución suponiendo que todos los vehículos tienen la misma prioridad.
  

    ```ada
    Procedure CrucePuente is

    task Puente is
        entry EntradaAuto;
        entry EntradaCamioneta;
        entry EntradaCamion;
        entry Salida (peso: IN integer);
    end Puente;

    task type Auto;
    task type Camioneta;
    task type Camion;

    autos: array (1..A) of Auto;
    camionetas: array (1..B) of Camioneta;
    camiones: array (1..C) of Camion;

    task body Auto is
    begin
        Puente.EntradaAuto;
        -- cruzar el puente
        Puente.Salida(1);
    end Auto;

    task body Camioneta is
    begin
        Puente.EntradaCamioneta;
        -- cruzar el puente
        Puente.Salida(2);
    end Camioneta;

    task body Camion is
    begin
        Puente.EntradaCamion;
        -- cruzar el puente
        Puente.Salida(3);
    end Camion;

    task body Puente is
        pesoActual: integer := 0;
    begin
        loop
            select
                when (pesoActual + 1 <= 5) =>
                accept EntradaAuto do
                    pesoActual := pesoActual + 1;
                end EntradaAuto;
            or
                when (pesoActual + 2 <= 5) =>
                accept EntradaCamioneta do
                    pesoActual := pesoActual + 2;
                end EntradaCamioneta;
            or
                when (pesoActual + 3 <= 5) =>
                accept EntradaCamion do
                    pesoActual := pesoActual + 3;
                end EntradaCamion;
            or
                accept Salida (peso: IN integer) do
                pesoActual := pesoActual - peso;
                end Salida;
            end select;
        end loop;
    end Puente;

    begin
    null;
    end CrucePuente;
    ```
   - b. Modifique la solución para que tengan mayor prioridad los camiones que el resto de los vehículos.
    ```ada
    task body Puente is
        pesoActual: integer := 0;
    begin
        loop
            select
                when (pesoActual + 1 <= 5) and (EntradaCamion'count = 0) =>
                accept EntradaAuto do
                    pesoActual := pesoActual + 1;
                end EntradaAuto;
            or
                when (pesoActual + 2 <= 5) and (EntradaCamion'count = 0) =>
                accept EntradaCamioneta do
                    pesoActual := pesoActual + 2;
                end EntradaCamioneta;
            or
                when (pesoActual + 3 <= 5) =>
                accept EntradaCamion do
                    pesoActual := pesoActual + 3;
                end EntradaCamion;
            or
                accept Salida (peso: IN integer) do
                pesoActual := pesoActual - peso;
                end Salida;
            end select;
        end loop;
    end Puente;
    ```
2. Se quiere modelar el funcionamiento de un banco, al cual llegan clientes que deben realizar un pago y retirar un comprobante. Existe un único empleado en el banco, el cual atiende de acuerdo con el orden de llegada.
   - a. Implemente una solución donde los clientes llegan y se retiran sólo después de haber sido atendidos.
    ```ada
    Procedure AtencionCliente is

    task Empleado is
        entry Pago (p: IN texto; comprobante: OUT texto);
    end Empleado;

    task type Cliente;

    clientes: array (1..N) of Cliente;

    task body Cliente is
        comprobante: texto;
    begin
        Empleado.Pago("datos", comprobante);
    end Cliente;

    task body Empleado is
    begin
        loop
            accept Pago (p: IN texto; comprobante: OUT texto) do
                comprobante := resolverOperacion(p);
            end Pago;
        end loop;
    end Empleado;

    begin
    null;
    end AtencionCliente;
    ```
   - b. Implemente una solución donde los clientes se retiran si esperan más de 10 minutos para realizar el pago.
    ```ada
    Procedure AtencionCliente is

    task Empleado is
        entry Pago (p: IN texto; comprobante: OUT texto);
    end Empleado;

    task type Cliente;

    clientes: array (1..N) of Cliente;

    task body Cliente is
        comprobante: texto;
    begin
        select
            Empleado.Pago("datos", comprobante);
        or delay 600.0          -- 10 minutos
            null;                -- se retira sin ser atendido
        end select;
    end Cliente;

    task body Empleado is
    begin
        loop
            accept Pago (p: IN texto; comprobante: OUT texto) do
                comprobante := resolverOperacion(p);
            end Pago;
        end loop;
    end Empleado;

    begin
    null;
    end AtencionCliente;
    ```

   - c. Implemente una solución donde los clientes se retiran si no son atendidos inmediatamente.
    ```ada
    Procedure AtencionCliente is

    task Empleado is
        entry Pago (p: IN texto; comprobante: OUT texto);
    end Empleado;

    task type Cliente;

    clientes: array (1..N) of Cliente;

    task body Cliente is
        comprobante: texto;
    begin
        select
            Empleado.Pago("datos", comprobante);
        else     -- else para inmediatez, delay para demora     
            null;                -- se retira sin ser atendido si inmediatamente no lo atienden
        end select;
    end Cliente;

    task body Empleado is
    begin
        loop
            accept Pago (p: IN texto; comprobante: OUT texto) do
                comprobante := resolverOperacion(p);
            end Pago;
        end loop;
    end Empleado;

    begin
    null;
    end AtencionCliente;
    ```
   - d. Implemente una solución donde los clientes esperan a lo sumo 10 minutos para ser atendidos. Si pasado ese lapso no fueron atendidos, entonces solicitan atención una vez más y se retiran si no son atendidos inmediatamente.
    ```ada
    Procedure AtencionCliente is

    task Empleado is
        entry Pago (p: IN texto; comprobante: OUT texto);
    end Empleado;

    task type Cliente;

    clientes: array (1..N) of Cliente;

    task body Cliente is
        comprobante: texto;
    begin
        select
            Empleado.Pago("datos", comprobante);
        or delay 600.0
            select
                empleado.pago("datos", comprobante);
            else     
                null;
            end select;
        end select;
    end Cliente;

    task body Empleado is
    begin
        loop
            accept Pago (p: IN texto; comprobante: OUT texto) do
                comprobante := resolverOperacion(p);
            end Pago;
        end loop;
    end Empleado;

    begin
    null;
    end AtencionCliente;
    ```


3. Se dispone de un sistema compuesto por **1 central** y **2 procesos periféricos**, que se comunican continuamente. Se requiere modelar su funcionamiento considerando las siguientes condiciones:
   - La central siempre comienza su ejecución tomando una señal del proceso 1; luego toma aleatoriamente señales de cualquiera de los dos indefinidamente. Al recibir una señal de proceso 2, recibe señales del mismo proceso durante 3 minutos.
   - Los procesos periféricos envían señales continuamente a la central. La señal del proceso 1 será considerada vieja (se deshecha) si en 2 minutos no fue recibida. Si la señal del proceso 2 no puede ser recibida inmediatamente, entonces espera 1 minuto y vuelve a mandarla (no se deshecha).
    ```ada
    Procedure Sistema is

   task Central is
      entry Senal1 (s: IN texto);
      entry Senal2 (s: IN texto);
      entry FinTiempo;
   end Central;

   task Timer is
      entry Iniciar;
   end Timer;

   task Periferico1;
   task Periferico2;

   task body Periferico1 is
      senal: texto;
   begin
      loop
         senal := generarSenal();
         select
            Central.Senal1(senal);
         or delay 120.0          -- 2 minutos
            null;                -- la señal se considera vieja: se descarta
         end select;
      end loop;
   end Periferico1;

   task body Periferico2 is
      senal: texto;
      enviada: boolean;
   begin
      loop
         senal := generarSenal();
         enviada := false;
         while (not enviada) loop
            select
               Central.Senal2(senal);
               enviada := true;
            else
               delay 60.0;       -- espera 1 minuto y reintenta con la MISMA señal
            end select;
         end loop;
      end loop;
   end Periferico2;

   task body Timer is
   begin
      loop
         accept Iniciar;
         delay 180.0;            -- cuenta 3 minutos
         Central.FinTiempo;      -- avisa que se terminó el tiempo
      end loop;
   end Timer;

   task body Central is
      senal: texto;
      fin: boolean;
   begin
      -- siempre comienza tomando una señal del proceso 1
      accept Senal1 (s: IN texto) do
         senal := s;
      end Senal1;
      procesar(senal);

      loop
         select
            accept Senal1 (s: IN texto) do
               senal := s;
            end Senal1;
            procesar(senal);
         or
            accept Senal2 (s: IN texto) do
               senal := s;
            end Senal2;
            procesar(senal);

            -- durante 3 minutos recibe solo señales del proceso 2
            Timer.Iniciar;
            fin := false;
            while (not fin) loop
               select
                  accept Senal2 (s: IN texto) do
                     senal := s;
                  end Senal2;
                  procesar(senal);
               or
                  accept FinTiempo;
                  fin := true;
               end select;
            end loop;
         end select;
      end loop;
   end Central;

    begin
        null;
    end Sistema;
    ```
4. En una clínica existe un **médico** de guardia que recibe continuamente peticiones de atención de las **E enfermeras** que trabajan en su piso y de las **P personas** que llegan a la clínica ser atendidos.
    Cuando una *persona* necesita que la atiendan espera a lo sumo 5 minutos a que el médico lo haga, si pasado ese tiempo no lo hace, espera 10 minutos y vuelve a requerir la atención del médico. Si no es atendida tres veces, se enoja y se retira de la clínica.
    Cuando una *enfermera* requiere la atención del médico, si este no lo atiende inmediatamente le hace una nota y se la deja en el consultorio para que esta resuelva su pedido en el momento que pueda (el pedido puede ser que el médico le firme algún papel). Cuando la petición ha sido recibida por el médico o la nota ha sido dejada en el escritorio, continúa trabajando y haciendo más peticiones.
    El *médico* atiende los pedidos dándole prioridad a los enfermos que llegan para ser atendidos. Cuando atiende un pedido, recibe la solicitud y la procesa durante un cierto tiempo. Cuando está libre aprovecha a procesar las notas dejadas por las enfermeras.
    ```ada
    process Clinica is
        task Medico is
            entry atencionP;
            entry atencionE;
            entry notaE(nota: IN text);
        end Medico;

        task type Enfermera;
        task type Paciente;
        enfermeras: array(1..E) of Enfermera;
        pacientes: array(1..P) of Paciente;

        task body Paciente is
            atendido: boolean:= false;
            intentos: integer:= 0;
        begin
            while (not atendido) AND (intentos < 3) loop
                select
                    Medico.atencionP;
                    atentido:= true;
                or delay 300.0
                    intentos:= intentos + 1;
                end select;
                delay 600.0;
            end loop;
        end Paciente;

        task body Enfermero is
            nota: text;
        begin
            loop
                select
                    Medico.atencionE;
                else
                    nota:= generarNota();
                    Medico.notaE;
                end select;
            end loop;
        end Enfermero;

        task body Medico is
            nota: text;
        begin
            loop
                select
                    accept atencionP do
                        delay atencion;
                    end atencionP;
                or when (atencionP'count = 0) =>
                    accept atencionE do
                        delay atencion;
                    end atencionE;
                or when (atencionP'count = 0) AND (atencionE'count = 0) =>
                    accept notaE(nota: IN text) do 
                        procesar(nota);
                    notaE;
                end select;
            end loop;
        end Medico;

        begin
            null;
        end Clinica;

    ```
5. En un sistema para acreditar carreras universitarias, hay UN Servidor que atiende pedidos de U Usuarios de a uno a la vez y de acuerdo con el orden en que se hacen los pedidos. Cada usuario trabaja en el documento a presentar, y luego lo envía al servidor; espera la respuesta de este que le indica si está todo bien o hay algún error. Mientras haya algún error, vuelve a trabajar con el documento y a enviarlo al servidor. Cuando el servidor le responde que está todo bien, el usuario se retira. Cuando un usuario envía un pedido espera a lo sumo 2 minutos a que sea recibido por el servidor, pasado ese tiempo espera un minuto y vuelve a intentarlo (usando el mismo documento).
    ```ada
    process Universidad is
        task Servidor is
            entry recibirDoc(doc: IN text, ok: OUT boolean);
        end Servidor;

        task type Estudiante;
        estudiantes: array (1..U) of Estudiante;

        task body Estudiante is
            dni: text;
            aceptado: boolean:= false;
            recibido: boolean:= false;
        begin
            while (not aceptado) loop
                doc:= prepararDoc();
                while (not recibido) loop
                    select
                        Servidor.recibirDoc(doc, aceptado);
                        recibido:= true;
                    or delay 120.0;
                    end select;
                    delay 60.0;
                end loop;
            end loop;
        end Estudiante;

        task body Servidor is
        begin
            loop
                accept recibirDoc(doc: IN text, ok: OUT boolean) do
                    ok:= procesarDoc(doc);
                end recibirDoc;
            end loop;
        end Servidor;

    begin
        null;
    end Universidad;
    ```
6. En una playa hay 5 equipos de 4 personas cada uno (en total son 20 personas donde cada una conoce previamente a que equipo pertenece). Cuando las personas van llegando esperan con los de su equipo hasta que el mismo esté completo (hayan llegado los 4 integrantes), a partir de ese momento el equipo comienza a jugar. El juego consiste en que cada integrante del grupo junta 15 monedas de a una en una playa (las monedas pueden ser de 1, 2 o 5 pesos) y se suman los montos de las 60 monedas conseguidas en el grupo. Al finalizar cada persona debe conocer el grupo que más dinero junto. **Nota:** maximizar la concurrencia. Suponga que para simular la búsqueda de una moneda por parte de una persona existe una función *Moneda()* que retorna el valor de la moneda encontrada.
    ```
    process JuntarMonedas is
        task Persona is
            entry resultadoJuego(ganador: IN integer);
        end Persona;

        task Equipo is
            entry avisarLlegada;
            entry iniciarJuego;
        end Equipo;

        task Contador is
            entry monadasJuntadas(nroEquipo: IN integer, monto: IN integer);
        end Contador;

        task Contador;
        personas: array (1..20) of Persona;
        equipos: array(1..5) of Equipo;

        task body of Persona is
            nroGrupo: integer;
            monto: integer:= 0;
        begin
            nroGrupo:= obtenerGrupo;
            equipo(nroGrupo).avisarLlegada;
            equipo(nroGrupo).iniciarJuego;
            for (1..15) loop
                monto:= monto + Moneda();
            end loop
            Contador.monedasJuntadas(nroGrupo, monto);
            accept resultadoJuego(ganador: IN integer);
        end Persona;

        task body of  Equipo is
        begin
            for(i:=1..4) loop
                accept avisarLlegada;
            end loop;
            for (i:=1..4) loop
                accept iniciarJuego;
            end loop;
        end Equipo;

        task body of Contador is
            equipos array(1..5) of integer:= (others => 0);
            ganador: integer;
        begin
            for (i:=1..20) loop
                accept mondasJuntadas(nroEquipo: IN integer, monto: IN integer) do
                    equipos(nroEquipo):= equipos(nroEquipo)+ monto;
                end monedasJuntadas;
            end loop;
            ganador:= calcularGanador(equipos);
            for (i:=1..20) loop
                personas(i).resultadoJuego(ganador);
            end loop;
        end Contador;

        begin
            null;
        end JuntarMonedas;         
                                                                                                                                                                                       
    ```
7. Se debe calcular el valor promedio de un vector de 1 millón de números enteros que se encuentra distribuido entre 10 procesos Worker (es decir, cada Worker tiene un vector de 100 mil números). Para ello, existe un Coordinador que determina el momento en que se debe realizar el cálculo de este promedio y que, además, se queda con el resultado. **Nota:** maximizar la concurrencia; este cálculo se hace una sola vez.
    ```ada
        process Contador is
            task Worker is
                entry iniciar;
            end Worker;

            task Coordinador is
                entry sumatoria(suma: IN integer);
            end Coordinador;

            task Coordinador;
            task type Worker;
            workers: array(1..10) of worker;

            task body of Worker is
                nros: array(1..100000) of integer;
                total: integer:= 0;
            begin
                accept iniciar;
                for(i:=1..100000) loop
                    total:= nros(i) + total;
                end loop;
                Coordinador.sumatoria(total);
            end Worker;

            task body of Coordinador is
                total: integer:= 0;
                promedio: integer;
            begin
                for (i:=1..10) loop
                    accept sumatoria(suma: IN integer) do
                        total:= total + suma;
                    end sumatoria;
                end loop
                promedio:= total/10;
            end Coordinador;

            begin
                null;
            end Contador;
    ```
8. Hay un sistema de reconocimiento de huellas dactilares de la policía que tiene 8 Servidores para realizar el reconocimiento, cada uno de ellos trabajando con una Base de Datos propia; a su vez hay un Especialista que utiliza indefinidamente. El sistema funciona de la siguiente manera: el Especialista toma una imagen de una huella (TEST) y se la envía a los servidores para que cada uno de ellos le devuelva el código y el valor de similitud de la huella que más se asemeja a TEST en su BD; al final del procesamiento, el especialista debe conocer el código de la huella con mayor valor de similitud entre las devueltas por los 8 servidores. Cuando ha terminado de procesar una huella comienza nuevamente todo el ciclo. **Nota:** suponga que existe una función *Buscar(test, código, valor)* que utiliza cada Servidor donde recibe como parámetro de entrada la huella test, y devuelve como parámetros de salida el código y el valor de similitud de la huella más parecida a test en la BD correspondiente. Maximizar la concurrencia y no generar demora innecesaria.
    ```ada
    process HuellasDactilares is
        task Especialista is
            resultado(valor: IN integer, codigo: IN integer);
        end Especialista;

        task Servidor is
            analizarHuella(test: IN text);
        end Servidor;
        
        task Especialista;
        task type Servidor;
        servidores: array(1..8) of Servidor;

        task body of Especialista is
            codigo_max, valor_max: integer:= 0;
            text: text;
        begin
            loop
                test:= tomarImagen();
                for (i:= 1..8) loop
                    servidores(i).analizarHuella(test);
                end loop;
                for (i:=1..8) loop
                    accept resultado(valor: IN integer, codigo: IN integer) do
                        if (valor > valor_max) then
                            valor_max:= valor;
                            codigo_max:= codigo;
                        end if;
                    end resultado;
                end loop;
            end loop;
        end Especialista;

        task body of Servidor do
            test: text;
            codigo,valor: interger;
        begin
            loop
                accept analizarHuella (test: IN text) do
                    Buscar(test, codigo, valor);
                    Especialista.resultado(valor,codigo);
                end analizarHuella;
            end loop;
        end Servidor;

        begin
            null;
        end HuellasDactilares;
    ```
9. Una empresa de limpieza se encarga de recolectar residuos en una ciudad por medio de 3 camiones. Hay P personas que hacen reclamos continuamente hasta que uno de los camiones pase por su casa. Cada persona hace un reclamo y espera a lo sumo 15 minutos a que llegue un camión; si no pasa, vuelve a hacer el reclamo y a esperar a lo sumo 15 minutos a que llegue un camión; y así sucesivamente hasta que el camión llegue y recolecte los residuos. Sólo cuando un camión llega, es cuando deja de hacer reclamos y se retira. Cuando un camión está libre la empresa lo envía a la casa de la persona que más reclamos ha hecho sin ser atendido. **Nota:** maximizar la concurrencia.
    ```ada
    procedure Recoleccion is

        task Empresa is
            entry Reclamo (id: IN integer);
            entry Siguiente (idP: OUT integer);
        end Empresa;

        task type Persona is
            entry Ident (pos: IN integer);
            entry Llegada;
        end Persona;

        task type Camion;

        personas: array (1..P) of Persona;
        camiones: array (1..3) of Camion;

        task body Persona is
            miId: integer;
            atendida: boolean := false;
        begin
            accept Ident (pos: IN integer) do
                miId := pos;
            end Ident;
            while (not atendida) loop
                Empresa.Reclamo(miId);
                select
                    accept Llegada;              -- llegó un camión
                    atendida := true;
                or delay 900.0                  -- 15 minutos sin camión
                    null;                        -- vuelve a reclamar
                end select;
            end loop;
        end Persona;

        task body Camion is
            idP: integer;
        begin
            loop
                Empresa.Siguiente(idP);         -- a qué casa ir
                personas(idP).Llegada;          -- llega a la casa
                -- recolectar los residuos
            end loop;
        end Camion;

        task body Empresa is
            reclamos: array (1..P) of integer := (others => 0);
            asignada: array (1..P) of boolean := (others => false);
            cantPend: integer := 0;            -- personas con reclamos sin camión asignado
            max: integer;
        begin
            loop
                select
                    accept Reclamo (id: IN integer) do
                    if (not asignada(id)) then
                        if (reclamos(id) = 0) then
                            cantPend := cantPend + 1;
                        end if;
                        reclamos(id) := reclamos(id) + 1;
                    end if;
                    end Reclamo;
                or
                    when (cantPend > 0) =>
                    accept Siguiente (idP: OUT integer) do
                        max := 0;
                        for i in 1..P loop
                            if (not asignada(i)) and (reclamos(i) > max) then
                                max := reclamos(i);
                                idP := i;
                            end if;
                        end loop;
                        asignada(idP) := true;
                        cantPend := cantPend - 1;
                    end Siguiente;
                end select;
            end loop;
        end Empresa;

        begin
        for i in 1..P loop
            personas(i).Ident(i);
        end loop;
        end Recoleccion;
    ```
