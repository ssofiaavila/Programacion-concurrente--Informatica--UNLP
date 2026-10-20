# Resolución práctica 5 | ADA
1. Se requiere modelar un puente de un único sentido que soporta hasta 5 unidades de peso. El peso de los vehículos depende del tipo: cada auto pesa 1 unidad, cada camioneta pesa 2
unidades y cada camión 3 unidades. Suponga que hay una cantidad innumerable de vehículos (A autos, B camionetas y C camiones). Analice el problema y defina qué tareas,
recursos y sincronizaciones serán necesarios/convenientes para resolverlo.
- a. Realice la solución suponiendo que todos los vehículos tienen la misma prioridad.
- b. Modifique la solución para que tengan mayor prioridad los camiones que el resto de los
vehículos.

**Análisis**
- Tareas: `Auto`, `Camioneta` y `Camion` (task types, arreglos de A, B y C) + una tarea `Puente`.
- Recurso a proteger: el peso actual sobre el puente → variable **local** de `Puente` (no hay variables compartidas).
- Sincronización: entrada al puente (esperar si no entra por peso) y salida (liberar peso).
- Un entry de entrada **por tipo de vehículo**: en el `when` no se pueden usar parámetros del entry, así que el tipo lo indica el entry mismo. La salida es una sola con parámetro porque no lleva `when` (el parámetro se usa dentro del cuerpo).
- Los vehículos cruzan **fuera** del accept para no reducir la concurrencia.

**a) Misma prioridad**
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

**b) Prioridad a los camiones**

Mismas tareas y mismos bodies de los vehículos que en a). Solo cambia el `select` del Puente: autos y camionetas entran solo si además **no hay camiones esperando** (`EntradaCamion'count = 0`). El camión solo chequea el peso y la salida sigue sin `when` (si se frenara, el peso nunca bajaría).
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

**Notas / errores a evitar**
- Entre las alternativas del `select` va **`or`**.
- El `'count` se consulta sobre el **entry** (`EntradaCamion'count`), no sobre el task type.
- Si hay un camión esperando que todavía no cabe, autos y camionetas también esperan: eso es darle prioridad.
- Un `select` elige una alternativa solo si su `when` es verdadero **y** hay entry calls pendientes; si ninguna cumple, espera en el `select` completo (no se queda trabado en un accept).
- Sintaxis: `pesoActual: integer := 0;` (`:` para el tipo, `:=` para asignar); los `task type` se separan con `;`; `task body Nombre is`.

2. Se quiere modelar el funcionamiento de un banco, al cual llegan clientes que deben realizar un pago y retirar un comprobante. Existe un único empleado en el banco, el cual atiende de acuerdo con el orden de llegada.
- a. Implemente una solución donde los clientes llegan y se retiran sólo después de haber sido atendidos.
- b. Implemente una solución donde los clientes se retiran si esperan más de 10 minutos para realizar el pago.
- c. Implemente una solución donde los clientes se retiran si no son atendidos inmediatamente.
- d. Implemente una solución donde los clientes esperan a lo sumo 10 minutos para ser atendidos. Si pasado ese lapso no fueron atendidos, entonces solicitan atención una vez más y se retiran si no son atendidos inmediatamente.

**Análisis**
- Tareas: una tarea `Empleado` y un arreglo de N tareas `Cliente`.
- El cliente paga y recibe el comprobante en **un solo encuentro**: el pago entra como `IN` y el comprobante vuelve como `OUT`, y se resuelve **dentro del accept** (aprovecha la comunicación bidireccional).
- El orden de llegada lo garantiza la cola implícita del entry `Pago` (FIFO).

**a) Los clientes se retiran solo después de ser atendidos**
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

**b) Los clientes se retiran si esperan más de 10 minutos**

Igual que a), pero el cliente usa un **entry call temporal** (`select … or delay`). El `or delay` cuenta el tiempo hasta que el empleado **empieza** el accept: si lo empiezan a atender antes de los 10 minutos, el cliente se queda aunque el pago tarde más.
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

**Notas / errores a evitar**
- Asignación con `:=` (`comprobante := resolverOperacion(p);`), no con `:`.
- Cada `end Nombre` termina con `;`.
- El programa es `procedure`, no `process` (eso es notación de PMA/PMS).
- Si el `or delay` no tiene nada que hacer, igual va `null;`.

3. Se dispone de un sistema compuesto por **1 central** y **2 procesos periféricos**, que se comunican continuamente. Se requiere modelar su funcionamiento considerando las siguientes condiciones:
- La central siempre comienza su ejecución tomando una señal del proceso 1; luego toma aleatoriamente señales de cualquiera de los dos indefinidamente. Al recibir una señal de proceso 2, recibe señales del mismo proceso durante 3 minutos.
- Los procesos periféricos envían señales continuamente a la central. La señal del proceso 1 será considerada vieja (se deshecha) si en 2 minutos no fue recibida. Si la señal del proceso 2 no puede ser recibida inmediatamente, entonces espera 1 minuto y vuelve a mandarla (no se deshecha).
```
```
4. En una clínica existe un **médico** de guardia que recibe continuamente peticiones de atención de las **E enfermeras** que trabajan en su piso y de las **P personas** que llegan a la clínica ser atendidos.
Cuando una *persona* necesita que la atiendan espera a lo sumo 5 minutos a que el médico lo haga, si pasado ese tiempo no lo hace, espera 10 minutos y vuelve a requerir la atención del médico. Si no es atendida tres veces, se enoja y se retira de la clínica.
Cuando una *enfermera* requiere la atención del médico, si este no lo atiende inmediatamente le hace una nota y se la deja en el consultorio para que esta resuelva su pedido en el momento que pueda (el pedido puede ser que el médico le firme algún papel). Cuando la petición ha sido recibida por el médico o la nota ha sido dejada en el escritorio, continúa trabajando y haciendo más peticiones.
El *médico* atiende los pedidos dándole prioridad a los enfermos que llegan para ser atendidos. Cuando atiende un pedido, recibe la solicitud y la procesa durante un cierto tiempo. Cuando está libre aprovecha a procesar las notas dejadas por las enfermeras.
```
```
5. En un sistema para acreditar carreras universitarias, hay UN Servidor que atiende pedidos de U Usuarios de a uno a la vez y de acuerdo con el orden en que se hacen los pedidos. Cada usuario trabaja en el documento a presentar, y luego lo envía al servidor; espera la respuesta de este que le indica si está todo bien o hay algún error. Mientras haya algún error, vuelve a trabajar con el documento y a enviarlo al servidor. Cuando el servidor le responde que está todo bien, el usuario se retira. Cuando un usuario envía un pedido espera a lo sumo 2 minutos a que sea recibido por el servidor, pasado ese tiempo espera un minuto y vuelve a intentarlo (usando el mismo documento).
```
```
6. En una playa hay 5 equipos de 4 personas cada uno (en total son 20 personas donde cada una conoce previamente a que equipo pertenece). Cuando las personas van llegando esperan con los de su equipo hasta que el mismo esté completo (hayan llegado los 4 integrantes), a partir de ese momento el equipo comienza a jugar. El juego consiste en que cada integrante del grupo junta 15 monedas de a una en una playa (las monedas pueden ser de 1, 2 o 5 pesos) y se suman los montos de las 60 monedas conseguidas en el grupo. Al finalizar cada persona debe conocer el grupo que más dinero junto. **Nota:** maximizar la concurrencia. Suponga que para simular la búsqueda de una moneda por parte de una persona existe una función *Moneda()* que retorna el valor de la moneda encontrada.
```
```
7. Se debe calcular el valor promedio de un vector de 1 millón de números enteros que se encuentra distribuido entre 10 procesos Worker (es decir, cada Worker tiene un vector de 100 mil números). Para ello, existe un Coordinador que determina el momento en que se debe realizar el cálculo de este promedio y que, además, se queda con el resultado. **Nota:** maximizar la concurrencia; este cálculo se hace una sola vez.
```
```
8. Hay un sistema de reconocimiento de huellas dactilares de la policía que tiene 8 Servidores para realizar el reconocimiento, cada uno de ellos trabajando con una Base de Datos propia; a su vez hay un Especialista que utiliza indefinidamente. El sistema funciona de la siguiente manera: el Especialista toma una imagen de una huella (TEST) y se la envía a los servidores para que cada uno de ellos le devuelva el código y el valor de similitud de la huella que más se asemeja a TEST en su BD; al final del procesamiento, el especialista debe conocer el código de la huella con mayor valor de similitud entre las devueltas por los 8 servidores. Cuando ha terminado de procesar una huella comienza nuevamente todo el ciclo. **Nota:** suponga que existe una función *Buscar(test, código, valor)* que utiliza cada Servidor donde recibe como parámetro de entrada la huella test, y devuelve como parámetros de salida el código y el valor de similitud de la huella más parecida a test en la BD correspondiente. Maximizar la concurrencia y no generar demora innecesaria.
```
```
9. Una empresa de limpieza se encarga de recolectar residuos en una ciudad por medio de 3 camiones. Hay P personas que hacen reclamos continuamente hasta que uno de los camiones pase por su casa. Cada persona hace un reclamo y espera a lo sumo 15 minutos a que llegue un camión; si no pasa, vuelve a hacer el reclamo y a esperar a lo sumo 15 minutos a que llegue un camión; y así sucesivamente hasta que el camión llegue y recolecte los residuos. Sólo cuando un camión llega, es cuando deja de hacer reclamos y se retira. Cuando un camión está libre la empresa lo envía a la casa de la persona que más reclamos ha hecho sin ser atendido. **Nota:** maximizar la concurrencia.
```
```
