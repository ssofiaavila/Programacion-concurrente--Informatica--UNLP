# Parcial 2 - 1ra fecha - Memoria distribuida (12/11/2026)

# Tema 1

> Nota: enunciados reconstruidos de memoria luego de rendir el parcial, pueden no ser exactos. Las soluciones siguen lo que se resolvió en el parcial.

## 1) PMA

Hay N clientes que hacen envíos en el correo. Los envíos pueden ser de 3 tipos: certificado, encomienda o carta simple. Cada cliente hace su envío y recibe el comprobante. Hay un empleado que recibe los envíos dando prioridad primero a los certificados, luego a las encomiendas y por último a las cartas simples.

### Solución

```
chan certificados(int, text);
chan encomiendas(int, text);
chan simples(int, text);
chan atencion();
chan comprobantes[1..N](text);

process Cliente[i=1..N]{
    text envio, tipo, comprobante;

    envio = prepararEnvio();
    tipo = obtenerTipo();
    if (tipo == 'certificado') send certificados(i, envio);
    else if (tipo == 'encomienda') send encomiendas(i, envio);
    else send simples(i, envio);
    send atencion();                      // primero el dato, DESPUÉS el aviso
    receive comprobantes[i](comprobante);
}

process Empleado{
    int idC;
    text envio, comprobante;

    while (true){
        receive atencion();               // se bloquea hasta que haya algún envío: sin busy waiting
        if (not empty(certificados)) receive certificados(idC, envio);
        else if (not empty(encomiendas)) receive encomiendas(idC, envio);
        else receive simples(idC, envio);
        comprobante = procesarEnvio(envio);
        send comprobantes[idC](comprobante);
    }
}
```

> - **Prioridad sin busy waiting:** un canal por tipo y un canal de aviso común (patrón 4 de PMA). El empleado se bloquea en `receive atencion()` y recién ahí revisa los canales en orden de prioridad.
> - El orden **dato → aviso** garantiza que, al recibir el aviso, el envío ya está en algún canal, así que el `receive` elegido nunca se bloquea.
> - El cliente manda su id para recibir el comprobante por su canal privado `comprobantes[i]`.
> - Dentro de cada tipo se respeta el orden de llegada porque cada canal es FIFO.

## 2) ADA

Hay N clientes que desean grabar los cristales de su vehículo: entregan la documentación y reciben un comprobante. Cada cliente espera a lo sumo 10 minutos; si no es atendido, se retira sin el grabado. Hay un único operario que atiende indefinidamente; si pasados 3 minutos no recibe ningún pedido, limpia el local y retoma la atención. Los clientes deben terminar.

### Solución

```ada
    procedure Grabado is

        task Operario is
            entry grabar(doc: IN text; comprobante: OUT text);
        end Operario;

        task type Cliente;
        clientes: array(1..N) of Cliente;

        task body Cliente is
            doc, comprobante: text;
        begin
            doc := obtenerDocumentacion();
            select                                  -- llamada temporizada
                Operario.grabar(doc, comprobante);
            or delay 600.0;                         -- 10 minutos
                null;                               -- se retira sin el grabado
            end select;
        end Cliente;

        task body Operario is
        begin
            loop                                    -- atiende indefinidamente
                select
                    accept grabar(doc: IN text; comprobante: OUT text) do
                        comprobante := grabarCristales(doc);
                    end grabar;
                or delay 180.0;                     -- 3 minutos sin pedidos
                    limpiarLocal();
                end select;
            end loop;
        end Operario;

    begin
        null;
    end Grabado;
```

> - **Cliente:** entry call temporizada (`select ... or delay 600.0`). Sin loop, así que termina como pide el enunciado.
> - **Operario:** `select` de accept con `or delay 180.0` (patrón L de ADA): el `delay` del lado del que **acepta** significa "espero a lo sumo 3 minutos a que me llamen".
> - El grabado va dentro del `accept` porque el cliente tiene que esperar su comprobante.

---

# Tema 2

> Enunciados transcriptos de la hoja del parcial. Las soluciones son una propuesta de resolución.

## 1) PMS

En una escuela hay una única cabina de votación que deben usar P votantes, y un Presidente de mesa que está a cargo. Los votantes deben votar por orden de llegada. Cuando le toca su turno, el Presidente le avisa al votante que puede pasar a votar. Mientras el votante vota, el Presidente lo anota en el padrón y prepara su comprobante (dispone de funciones `AnotarPadrón(id)` y `GenerarComprobante(id)`). Cuando el votante termina de votar, se lo informa al Presidente, quien le entrega el comprobante. Recién entonces el Presidente pasa al siguiente. Cada votante vota una sola vez. **Nota:** maximizar la concurrencia; no generar demora innecesaria; todos los procesos deben terminar su ejecución.

### Solución

```
process Votante[i=1..P]{
    text comprobante;

    Admin!llegada(i);
    Presidente?pasar();
    votar();
    Presidente!termine();
    Presidente?entregar(comprobante);
}

process Admin{
    queue cola;
    int idV, cant = 0;

    do cant < P; Votante[*]?llegada(idV) -> push(cola, idV);
                                            cant++;
       not empty(cola); Presidente?siguiente() -> pop(cola, idV);
                                                  Presidente!turno(idV);
    od
}

process Presidente{
    int idV;
    text comprobante;

    for (k=1..P){
        Admin!siguiente();
        Admin?turno(idV);
        Votante[idV]!pasar();
        AnotarPadron(idV);                    // mientras el votante vota
        comprobante = GenerarComprobante(idV);
        Votante[idV]?termine();
        Votante[idV]!entregar(comprobante);
    }
}
```

> - **Orden de llegada en PMS ⇒ proceso con cola.** El Presidente queda ocupado durante cada votación (anota, prepara el comprobante y espera el `termine`), así que no puede recibir llegadas: si los votantes le hicieran `!` directo, quedarían bloqueados y después `[*]` elegiría cualquiera. Por eso va un **Admin** que siempre está listo para encolar.
> - **Maximizar la concurrencia:** el Presidente le da paso al votante **antes** de anotarlo y generar el comprobante, así esas tareas se hacen **mientras** el votante vota (no antes ni después).
> - **Exclusión mutua de la cabina:** el Presidente no pide el siguiente hasta recibir el `termine` de ese votante (`Votante[idV]?`, no `[*]`).
> - **Terminación:** el Presidente hace un `for` de P vueltas; el `do` del Admin termina cuando ya llegaron los P (`cant == P`) y la cola quedó vacía.

## 2) ADA

Un castillo inflable soporta hasta 500 kg de peso a la vez. Hay A niños, B jóvenes y C adultos que quieren subir al castillo inflable. Se asume que el peso de las personas es el mismo para todos los de su rango etario, siendo 30 kg para los niños, 70 kg para los jóvenes y 90 kg para los adultos. Cada persona juega una única vez durante un tiempo variable y luego se retira. Los niños tienen mayor prioridad que el resto. **Nota:** todas las tareas deben terminar.

### Solución

```ada
    procedure CastilloInflable is

        task Castillo is
            entry entrarNino;
            entry entrarJoven;
            entry entrarAdulto;
            entry salir(p: IN integer);
        end Castillo;

        task type Nino;
        task type Joven;
        task type Adulto;
        ninos: array(1..A) of Nino;
        jovenes: array(1..B) of Joven;
        adultos: array(1..C) of Adulto;

        task body Nino is
        begin
            Castillo.entrarNino;
            jugar();                        -- fuera del rendezvous: juegan varios a la vez
            Castillo.salir(30);
        end Nino;

        task body Joven is
        begin
            Castillo.entrarJoven;
            jugar();
            Castillo.salir(70);
        end Joven;

        task body Adulto is
        begin
            Castillo.entrarAdulto;
            jugar();
            Castillo.salir(90);
        end Adulto;

        task body Castillo is
            peso: integer := 0;
        begin
            for i in 1..2*(A+B+C) loop      -- cada persona entra una vez y sale una vez
                select
                    when (peso + 30 <= 500) =>
                        accept entrarNino;
                        peso := peso + 30;
                or
                    when (peso + 70 <= 500 and entrarNino'count = 0) =>
                        accept entrarJoven;
                        peso := peso + 70;
                or
                    when (peso + 90 <= 500 and entrarNino'count = 0) =>
                        accept entrarAdulto;
                        peso := peso + 90;
                or
                    accept salir(p: IN integer) do
                        peso := peso - p;
                    end salir;
                end select;
            end loop;
        end Castillo;

    begin
        null;
    end CastilloInflable;
```

> - **El peso en la guarda:** en el `when` no se pueden usar los parámetros del entry, por eso hay **un entry por rango etario** y el peso de cada uno va fijo en su guarda (30, 70, 90).
> - **Prioridad a los niños:** jóvenes y adultos solo entran si `entrarNino'count = 0`. Entre jóvenes y adultos no hay prioridad.
> - **Juegan varios a la vez:** el juego va entre `entrar` y `salir`, fuera del rendezvous (mismo patrón que el puente / lectores). `salir` nunca lleva guarda.
> - **Todas las tareas terminan:** el castillo sabe que va a haber exactamente `2*(A+B+C)` encuentros (una entrada y una salida por persona) y usa un `for` en vez de un `loop` infinito.
