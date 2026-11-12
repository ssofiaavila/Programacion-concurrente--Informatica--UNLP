# Parcial 2 - 1ra fecha - Memoria distribuida - Tema 1 (2026)

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
