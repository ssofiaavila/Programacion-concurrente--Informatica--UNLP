# Explicación práctica pasaje de mensajes sincrónicos (PMS)

## A tener en cuenta
- Los programas se componen SÓLO de procesos (no existen las variables compartidas).
- Los canales son de tipo link (un único emisor y un único receptor) y son sincrónicos. Son estructuras implícitas y no se deben declarar.
- Los procesos interactúan entre ellos únicamente por medio del envío de mensajes (tanto para comunicación como para sincronización por condición).
- No se requere sincronización por exlusión mutua ya que no existen las variables compartidas.

## Sintaxis
### Sentencias de comunicación
- **Sentencias de comunicación:** no existen las estructuras de canales, la comunicación se realiza nombrando al procesos con el cual se realiza la comunicación.
  1. **Envío(!):** la operación es bloqueante y sincrónica, se demora hasta que la recepción haya terminado.
    ```
        destino!port(mensaje);
        destino[i]!port(mensaje);
    ```
  2. **Recepción(?):** la operación es bloqueante y sincrónica.
    ```
        origen?port(variable);
        origen[i]?port(variable);
        origen[*]?port(variable);
    ```
    **port** debe coincidir de etiqueta tanto en el emisor como receptor. Puede no ir. Se usa para diferenciar los tipos de mensajes que se podrían comunicarse entre dos procesos.
    **[*]** es solamente usado en la recepción, indica que el origen puede ser cualquier proceso del arreglo. Si quiere contestarle a ese mismo, deberé usar algún identificador.

### Comunicación guardada
- Uso de comunicación guardada (if y do): permite seleccionar de forma no determinística entre varias alternativas de comunicación (en caso de la práctica de la práctica SOLO RECEPCIONES) en base a las condiciones del proceso y los mensajes que están listos para ser recibidos.
- Guardas: cada alternativa es una guarda con la forma: `B;C -> S`.
    1. `B`: condición booleana que puede no estar (en ese caso se considera true), e indica si el proceso está o no en condiciones de procesar el mensaje recibido en `C`.
    2. `C`: sentencia de comunicación (en la práctica sólo RECEPCIÓN) que seguro debe estar.
    3. `S`: conjunto de sentencias que se ejecutarán en caso de ser elegida la guarda.
- Evaluación de guarda:
    - `Exito`: la condición booleana es Verdadera (o no la tiene) y la comunicación se puede realizar sin producir demora (el emisor está esperando hacer la comunicación).
    - `Fallo`: la condición booleana es Falsa, sin importar lo que ocurra con la sentencia de comunicación.
    - `Bloqueo`: la condición boolean es Verdadera (o no la tiene) pero la comunicación NO se puede realizar sin producir demora (el emisor aún no llegó a la sentencia de envío).
- En el if `guardado` se evalúan todas las guarda en base a eso:
    - Si una o más son EXITOSAS se selecciona una de ellas en forma NO DETERMINISTICA, se ejecuta la sentencia de recepción `C` que forma parte de la guarda, y posteriormente el conjunto de sentencias asociadas a la guarda `S`.
    - Si todas las guardas FALLAN no se selecciona ninguna y se sale del IF sin realizar ninguna acción.
    - Si no hay ninguna guarda EXITOSA pero hay una o más con estado BLOQUEO entonces el proceso se demora en el IF hasta que haya una guarda exitosa. En ese momento se ejecuta igual que el primer caso.
- El do **guardado** funciona de la misma manera que el IF, con la única diferencia que en lugar de hacerlo una vez repite el mecanismo hasta que todas las guardas FALLAN.

### Más info comunicación guardada
- *Guarda:* ```(condición booleana); sentencia de recepción --> sentencia a realizar;```
Si no se especifica la confición booleana se considera verdadera (la condición booleana sólo puede hacer referencia a variables locales del proceso).
Cada guarda tiene tres posibles estados:
1. `Elegible` la condición booleana es verdadera y la sentencia de comunicación se puede resolver inmediatamente.
2. `No elegible` la condición booleana es falsa.
3. `Bloqueada` la condición booleana es verdadera y la sentencia de comunicación no se puede resolver inmediatamente.

Sólo se puede usar dentro de un `if` o un `do` guardado: 
- El `if` funciona de la siguiente manera: de todas las guardas **elegibles** se selecciona una en forma no determinística, se realiza la sentencia de comunicación correspondiente, y luego las acciones aosicadas a esa guarda. Si todas las guardas tienen el estado de **no elegibles**, se sale sin hacer nada. Si no hay ninguna guarda elegible, pero algunas están en estado **bloqueado**, se queda esperando en el if hasta que alguna se vuelva elegible.
- El `do` funciona de la siguiente manera: sigue iterando de la misma manera que el `if` hasta que todas las guardas sean `no elegibles`.


# Explicación simple de las guardas

## ¿Qué es una guarda?
"Guarda" viene de **guardia**: es lo que vigila la entrada a un bloque de código y decide si se puede pasar o no. Pensalo como un guardia en una puerta:
```
B ; C   -->   S
└──┬──┘       └┬┘
 guardia     lo que hay
(¿pasás?)    detrás de la puerta
```
- Lo que está **antes de la flecha** (`B ; C`) es el guardia: revisa si se cumplen las condiciones para entrar.
- Lo que está **después de la flecha** (`S`) es lo que se hace si el guardia te deja pasar.

Por ejemplo, en:
```
not empty(Buffer); Mantenimiento?pedido() --> Mantenimiento!reporte(pop(Buffer));
```
el guardia dice: *"solo te dejo pasar a hacer el `pop` y mandar el reporte si hay algo en el buffer **y** Mantenimiento vino a pedir"*.

## B filtra, C sincroniza y S ejecuta
Cada parte de la guarda tiene un rol distinto. **B no ejecuta nada ni se sincroniza con nadie**: es solo una pregunta de sí o no que el proceso se hace a sí mismo.

| Parte | Qué es | ¿Se comunica con otro proceso? | Qué hace |
|---|---|---|---|
| `B` | Condición booleana | **No.** Solo mira variables locales | Decide si **me conviene** atender esto ahora |
| `C` | Recepción (`?`) | **Sí.** Acá está la sincronización con el emisor | Recibe el mensaje, si hay alguien esperando para mandarlo |
| `S` | Sentencias | Puede ser (por ejemplo, un `!` de respuesta) | Hace el trabajo una vez que se eligió la guarda |

```
not empty(Buffer); Mantenimiento?pedido() --> Mantenimiento!reporte(pop(Buffer));
└──────┬───────┘   └─────────┬──────────┘     └──────────────┬──────────────────┘
       B                     C                               S
 "¿tengo algo        "¿Mantenimiento vino     "le saco un reporte a la
  en la cola?"        a pedir? lo recibo"      cola y se lo mando"
  (solo pregunta)     (se sincroniza)          (ejecuta)
```

Lo que sí hace B es **habilitar o deshabilitar** la sincronización. Si B es falsa, la guarda falla y ni siquiera se intenta la recepción. El emisor sigue esperando en su `!` hasta que, en otra vuelta del `do`, B sea verdadera.

**Sobre la palabra:** se usa "guarda" de dos maneras, y las dos son válidas:
1. **Sentido estricto:** la guarda es solo la parte de la izquierda (`B ; C`), la condición.
2. **Sentido amplio:** se le dice "guarda" a toda la línea (`B ; C --> S`), o sea, a cada alternativa del `if`/`do`.

Cuando se dice "el `do` tiene dos guardas", se habla de **dos alternativas**. Cuando se dice "la guarda falla", se habla de que **la condición no se cumple** y el guardia no deja pasar. Un `if`/`do` es "guardado" porque cada alternativa tiene su propio guardia en la entrada.

## El problema que resuelven: un `?` suelto te deja esperando a una sola persona
Si el Admin del Ejemplo 2 se escribiera sin guardas:
```
process Admin {
    do true ->
        Testeo?reporte(R);  push(Buffer, R);
        Mantenimiento?pedido();  Mantenimiento!reporte(pop(Buffer));
    od
}
```
Como la recepción es **bloqueante**, el Admin queda atado a un orden fijo:
- Si Testeo manda 5 reportes seguidos, el Admin recibe el primero y se queda trabado en `Mantenimiento?pedido()`. Testeo no puede dejar el segundo.
- Si Mantenimiento pide trabajo y la cola está vacía, igual intenta hacer `pop`.

Lo que se necesita es que el Admin pueda decir: **"atiendo al que llegue primero, entre los que puedo atender ahora"**. Eso es la comunicación guardada.

## Cada guarda son dos preguntas
Para evaluar `B ; C --> S`, el proceso se hace dos preguntas, siempre en este orden:
1. **¿Me conviene atender esto ahora?** Es `B`, y mira solo variables locales (por ejemplo: ¿tengo algo en la cola?).
2. **¿Hay alguien parado esperando para mandarme esto?** Es `C`, o sea, ¿el emisor ya llegó a su `!`?

| ¿B es verdadera? | ¿El emisor ya está esperando? | Estado | Traducción |
|---|---|---|---|
| No | (no importa) | **Fallo / No elegible** | "Esto ahora no me interesa" |
| Sí | Sí | **Éxito / Elegible** | "Me interesa y está listo, lo puedo hacer ya" |
| Sí | No | **Bloqueo** | "Me interesa, pero todavía no vino nadie" |

Cuando `B` es falsa, ni se mira la comunicación. Por eso un Mantenimiento que pide trabajo con la cola vacía **se queda esperando en su `!`**: es un envío sincrónico y el Admin no lo acepta. No se pierde nada; va a ser atendido cuando la cola tenga algo.

## Qué hace el `if` / `do` con esos estados
Primero mira las guardas en estado **Éxito**:
- **Hay al menos una con Éxito:** elige una cualquiera (no determinístico), hace su recepción `C` y después ejecuta `S`.
- **No hay ninguna con Éxito, pero hay alguna en Bloqueo:** el proceso se duerme hasta que alguna pase a Éxito.
- **Todas en Fallo:** sale sin hacer nada. El `if` termina; el `do` también termina, porque es su única condición de salida.

La única diferencia entre los dos es que el `do` vuelve a empezar después de ejecutar una guarda.

## Seguimiento con el Admin del Ejemplo 2
```
do Testeo?reporte(R) --> push(Buffer,R);
   not empty(Buffer); Mantenimiento?pedido() --> Mantenimiento!reporte(pop(Buffer));
od
```

| Momento | Situación | Guarda 1 (reporte) | Guarda 2 (pedido) | Qué pasa |
|---|---|---|---|---|
| t1 | Buffer vacío, nadie esperando | Bloqueo | **Fallo** (cola vacía) | El Admin se duerme |
| t2 | Mantenimiento hace `Admin!pedido()` | Bloqueo | Fallo | Sigue igual. Mantenimiento queda esperando en su `!` |
| t3 | Testeo hace `Admin!reporte(R1)` | **Éxito** | Fallo | Recibe R1 y lo encola. Buffer = [R1] |
| t4 | Siguiente vuelta del `do` | Bloqueo | **Éxito** (cola no vacía y Mantenimiento seguía esperando desde t2) | Atiende el pedido y le manda R1 |
| t5 | Buffer no vacío; Testeo manda un reporte y Mantenimiento pide a la vez | Éxito | Éxito | **Elige cualquiera**. La otra se atiende en la próxima vuelta |

En t2, la condición `not empty(Buffer)` no rechazó a Mantenimiento: solo **postergó** su atención hasta que tuviera sentido atenderlo.

## Dudas típicas
- **¿Por qué `B` no puede usar el contenido del mensaje?** Porque `B` se evalúa **antes** de recibir. Todavía no se sabe qué van a mandar; solo se puede usar lo que el proceso ya tiene (contadores, colas, flags).
- **¿Por qué en las guardas solo hay `?` y nunca `!`?** Es una restricción de la práctica. Por eso aparece el patrón de que el trabajador avisa con `Admin!pedido(id)`, el Admin lo recibe en una guarda y después le responde con `Mantenimiento[id]!...` dentro de `S`. Ese `!` dentro de `S` sí está permitido.
- **¿Cuándo termina un `do`?** Solo cuando **todas** las guardas fallan, es decir, todas las `B` son falsas. Si alguna guarda no tiene `B`, cuenta como `true` y el `do` no termina nunca. Eso es justo lo que se busca para un administrador o un buffer.
- **¿Para qué sirve que pueda terminar?** Para casos como "atender a N clientes y terminar": `cant < N; Cliente[*]?llegada(id) --> cant++`. Cuando `cant` llega a N, todas fallan y el `do` sale.

## Resumen en una frase
> La guarda es *"si me conviene (B) y alguien ya está listo para mandarme esto (C), entonces lo hago (S)"*, y el `if`/`do` elige entre todas las guardas que cumplen las dos condiciones.


# Ejemplo 1
En una empresa de software hay un empleado *Testeo* que prueba un nuevo producto para encontrar errores, cuando encuentra uno generan un reporte para que otro empleado *Mantenimiento* corrija el error y le responda. El empleado *Mantenimiento* toma los reportes para evaluarlos, hacer las correcciones necesarias y responderle al empleado *Testeo*.

*Habrá dos procesos: Testeo y Mantenimiento. Testeo le debe enviar el reporte a Mantenimiento y esperar a que le responda para continuar trabajando. Por lo tanto no se requiere modelar un buffer donde se vayan almacenando los reportes, ya que nunca habrá más de uno a la vez.*
```
process Testeo{
    texto R, Res;
    while (true){
        R= generarReporteConProblema;
        Mantenimiento!reporte(R);
        Mantenimiento?respuesta(Res);
    }
}

process Mantenimiento{
    texto Rep, Res;
    while(true){
        Testeo?reporte(Rep);
        Res= resolver(Rep);
        Testeo!respuesta(Res);
    }
}
```
Ambos procesos se comunican directamente entre ellos 2 veces por cada reporte generado, una con el *reporte* y otra con el *resultado*.

# Ejemplo 2
En una empresa de software hay un empleado *Testeo* que prueba un nuevo producto para encontrar errores, cuando encuentra uno generan un reporte para que otro empleado *Mantenimiento* corrija el error **(no requiere una respuesta para seguir trabajando) y continua.** El empleado *Mantenimiento* toma los reportes para evaluarlos y hacer las correcciones necesarias.

**Partimos de la solución del ejercicio 1, sin cambios se limitará la concurrencia porque el proceso Testeo esperará a que Mantenimiento termine en generar el reporte.** Por esa razón, habrá que usar un **Buffer**.

```
process Testeo {
    texto R, Res;
    while (true){
        R= generarReporteConProblema();
        Admin!reporte(R);
    }
}

process Mantenimiento {
    texto Rep, Res;
    while (true){
        Admin!pedido();
        Admin?reporte(Rep);
        Res= resolver(Rep);
    }
}

process Admin{
    cola Buffer;
    texto R;
    do Testeo?reporte(R) --> push(Buffer,R);
        not empty(Buffer); Mantenimiento?pedido() --> Mantenimiento!reporte(pop(Buffer));
}
```
# Ejemplo 3
En una empresa de software hay *N empleados Testeo* que prueban un nuevo producto para encontrar errores, cuando encuentran uno generan un reporte para que otro empleado *Mantenimiento* corrija el error y le responda. El empleado *Mantenimiento* toma los reportes para evaluarlos de acuerdo al orden de llegada, hace las correcciones necesarias y le responde al empleado *Testeo* correspondiente.

*Partimos de la solución del ejemplo 1 pero con N procesos Testeo.*
```
process Admin{
    cola Fila; texto R; int idT;
    do Testeo[*]?reporte(R, idT) --> push(fila, (R, idT));
        not empty(Fila); Mantenimiento?pedido() --> pop(Fila, (R, idT));
                                                    Mantenimiento!reporte(R, idT);
}

process Testeo[id: 1..N] {
    texto R, Res;
    while (true){
        R= generarReporteConProblema();
        Admin!reporte(R, id);
        Mantenimiento?respuesta(Res);
    }
}

process Mantenimiento {
    texto Rep, Res;
    int idT;
    while (true){
        Admin!pedido();
        Admin?reporte(Rep, idT);
        Res= resolver(Rep);
        Testeo[idT]!respuesta(Res);
    }
}
```

# Ejemplo 4
En una empresa de software hay N Empleados *Testeo* que prueban un nuevo producto para encontrar errores, cuando encuentra uno generan un reporte para que uno de los *3 empleados Mantenimiento* corrija el error y le responda. Los empleados *Mantenimiento* toma los reportes para evaluarlos de acuerdo al orden de llegada, hace las correcciones necesarias y le responde al empleado *Testeo* correspondiente.

```
process Admin {
    cola Fila; 
    texto R;
    int idT, idM;
    do Testeo[*]?reporte(R, idT) --> push( Fila, (R, idT));
        not empty(Fila); Mantenimiento[*]?pedido(idM) --> pop( Fila, (R, idT));
                                                            Mantenimiento[idM]!reporte(R,idT);
    od
}

process Testeo[id:1..N]{
    texto R, Res;
    while (true){
        R= generarReporteConProblema();
        Admin!reporte(R,id);
        Mantenimiento[*]?respuesta(Res);
    }
}

process Mantenimiento[id:1..3]{
    texto Rep, Res;
    int idT;
    while (true){
        Admin!pedido(id);
        Admin?reporte(Rep,idT);
        Res= resolver(Rep);
        Testeo[idT]!respuesta(Res);
    }
}
```

# Explicación simple del `do` del Admin (Ejemplo 4)
```
do Testeo[*]?reporte(R, idT) --> push(Fila, (R, idT));
   not empty(Fila); Mantenimiento[*]?pedido(idM) --> pop(Fila, (R, idT));
                                                     Mantenimiento[idM]!reporte(R, idT);
od
```

## La idea con un ejemplo de la vida real
Pensar al Admin como una **secretaria con una bandeja de papeles** (la `Fila`). La secretaria atiende dos tipos de personas:
- **Testeo:** viene a **dejar** un reporte.
- **Mantenimiento:** viene a **pedir** un reporte para trabajar.

La secretaria se queda todo el día en su escritorio repitiendo lo mismo: *"¿quién viene? lo atiendo y vuelvo a esperar"*. Ese "repetir siempre" es el `do ... od`.

## Línea 1: alguien de Testeo viene a dejar un reporte
```
Testeo[*]?reporte(R, idT) --> push(Fila, (R, idT));
```
- "Si algún Testeo (cualquiera, por eso el `*`) viene a dejar un reporte, lo recibo y lo pongo en la bandeja."
- No tiene condición antes del `;` porque **siempre** se aceptan reportes nuevos.
- Como con `*` no se sabe quién mandó el mensaje, el Testeo manda también su `idT`. Así, más adelante, Mantenimiento sabe a quién responderle.

## Línea 2: alguien de Mantenimiento viene a pedir trabajo
```
not empty(Fila); Mantenimiento[*]?pedido(idM) --> pop(Fila, (R, idT));
                                                  Mantenimiento[idM]!reporte(R, idT);
```
- "Si **hay papeles en la bandeja** y **algún Mantenimiento vino a pedir**, saco el primer reporte y se lo doy a ese Mantenimiento."
- `not empty(Fila)` evita atender a Mantenimiento cuando no hay nada para darle. Si la bandeja está vacía, esta línea ni se considera.
- Mantenimiento manda su `idM` para que el Admin sepa a cuál de los 3 contestarle.

## ¿Y si pasan las dos cosas a la vez?
Si al mismo tiempo hay un Testeo esperando para dejar un reporte y un Mantenimiento esperando para pedir uno (con la bandeja no vacía), el Admin **elige uno cualquiera** (no determinismo). Al otro lo atiende en la próxima vuelta del `do`.

## ¿Y si no viene nadie?
El Admin **se queda esperando** en el `do` hasta que llegue alguien que pueda atender. No gasta tiempo preguntando.

## ¿Cuándo termina?
Nunca. Un `do` termina solo cuando **todas** las condiciones son falsas, y la línea 1 no tiene condición (siempre es verdadera). Es lo que se busca para un administrador.

## ¿Por qué Mantenimiento tiene que "pedir" en vez de que el Admin le mande directamente?
Porque en la práctica, **en las guardas sólo se puede recibir (`?`), nunca enviar (`!`)**. Entonces el Admin no puede tener una guarda que diga "le mando a Mantenimiento". La solución es que Mantenimiento avise que está libre (`Admin!pedido(id)`) y el Admin, al recibir ese aviso, le responde con el reporte.

Ventaja: el Admin nunca se queda trabado esperando a un proceso en particular. Si los Mantenimiento están ocupados, sigue juntando reportes en la bandeja.

## Resumen en una frase
> El Admin repite siempre: *"si llega un reporte, lo guardo; si tengo reportes guardados y un Mantenimiento me pide, le doy uno"*.


# Patrones y situaciones de la práctica 4 (PMS)

Resumen de lo que se repitió en los ejercicios 1 al 5. Los nombres son genéricos (Cliente, Admin, Trabajador): en cada ejercicio se reemplazan por los del enunciado.

## 1. ¿Qué patrón uso? (decisión rápida)

| El enunciado dice... | Patrón | Apareció en |
|---|---|---|
| "no importa el orden" | **A.** Servidor que recibe con `[*]` | 1b, 4a |
| "orden de llegada" | **B.** Admin con cola | 1c, 2, 3a, 3b, 4c, 5 |
| "de a uno por vez" / exclusión mutua | **C.** Coordinador de recurso | 4a, 4c, 5 |
| "según el orden de su identificador" | **D.** `for` + recepción de un proceso específico | 4b |
| una etapa produce y la siguiente consume (cadena) | **E.** Pipeline | 2 |
| "no comienzan hasta que todos hayan llegado" | **F.** Barrera | 3c |
| "todos los procesos deben terminar" | **G.** Situaciones de corte | 3a, 3b, 3c, 5 |

## 2. Reglas que se repiten en todos los ejercicios
1. **En las guardas sólo van recepciones (`?`)**. Los envíos (`!`) van en el cuerpo `S`, después de la flecha.
2. **Si recibo con `[*]`, el emisor tiene que mandarme su id** (como dato del mensaje). Si no, no sé a quién contestarle.
3. **Para respetar el orden de llegada hace falta un Admin con una cola.** El trabajador no recibe directo de los clientes: le *pide* trabajo al Admin (`Admin!pedirTrabajo()`) y el Admin le responde dentro de `S`.
4. **El que hace el trabajo le responde directo al cliente** (`Alumno[id]!respuesta(nota)`), sin pasar por el Admin. Así el Admin queda libre para seguir atendiendo.
5. **El Admin sólo reparte; nunca hace el trabajo pesado** (corregir, imprimir, analizar). Si lo hiciera, dejaría de recibir y se perdería concurrencia.
6. **No se puede usar `empty` sobre canales**, pero sí sobre una cola propia del proceso (`not empty(cola)` es una condición booleana sobre una variable local).
7. **El `!` es sincrónico**: aunque el cliente no espere una respuesta, queda bloqueado en el `!` hasta que el otro reciba. Si el servidor tarda y el cliente no debe frenarse, hace falta un Admin con cola en el medio (patrón B).
8. Cuando se sabe cuántas veces se repite algo, usar `for (k=1..N)` en vez de `while (true)`.

## 3. Patrones

### A. Servidor sin orden (Ej. 1b, 4a)
**Idea:** el servidor recibe de cualquiera con `[*]` y atiende en el orden que le toque.
```
process Cliente[i=1..C] {
  Servidor!pedido(datos, i);        // mando mi id para que me puedan responder
  Servidor?respuesta(res);          // sólo si debe esperar la respuesta
}

process Servidor {
  while (true) {
    Cliente[*]?pedido(datos, idC);
    res= atender(datos);
    Cliente[idC]!respuesta(res);
  }
}
```
**Ojo:** si el servidor tarda y los clientes no deben esperar a que termine, esta versión los frena (ver regla 7): pasar al patrón B.

### B. Admin con cola: orden de llegada (Ej. 1c, 2, 3a, 3b, 4c, 5)
**Idea:** el Admin tiene dos alternativas en un `do`: *recibir y encolar* siempre, y *entregar trabajo* sólo si hay algo en la cola y un trabajador lo pidió.
```
process Cliente[i=1..N] {
  Admin!pedido(dato, i);
  Trabajador[*]?respuesta(res);     // lo atiende cualquier trabajador, le responde directo
}

process Admin {
  queue cola;  T dato;  int idC, idT;
  do Cliente[*]?pedido(dato, idC) -> push(cola, (dato, idC));
     not empty(cola); Trabajador[*]?pedirTrabajo(idT) -> pop(cola, (dato, idC));
                                                         Trabajador[idT]!trabajo(dato, idC);
  od
}

process Trabajador[t=1..T] {
  T dato;  int idC;
  while (true) {
    Admin!pedirTrabajo(t);
    Admin?trabajo(dato, idC);
    res= procesar(dato);
    Cliente[idC]!respuesta(res);
  }
}
```
**Variantes que aparecieron:**
- **Un solo trabajador (P=1, Ej. 3a):** se saca el id y el `[*]`: `Profesor?espera()` en la guarda y `Profesor!enviar(examen, id)` en el cuerpo. Con un solo emisor no hace falta distinguir quién pidió.
- **Varios trabajadores (P>1, Ej. 3b):** cada uno manda su id (`Admin!espero(i)`), y el Admin responde con `Profesor[idP]!enviar(...)`. Además el Alumno recibe con `Profesor[*]?respuesta(nota)`, porque no sabe cuál de los profesores lo va a corregir.
- **El que produce no espera (Ej. 2, Ej. 5):** es el mismo Admin, funciona como buffer entre el productor y el consumidor.

### C. Coordinador de recurso: exclusión mutua (Ej. 4a)
**Idea:** el recurso (simulador, máquina, impresora) se usa de a uno porque el coordinador hace los **tres pasos completos antes de volver a recibir**: avisar, dar paso y esperar a que liberen.
```
process Persona[i=1..P] {
  Empleado!avisar(i);
  Empleado?pasar();
  usarRecurso();
  Empleado!liberar();
}

process Empleado {
  int idP;
  for (k=1..P) {
    Persona[*]?avisar(idP);
    Persona[idP]!pasar();
    Persona[idP]?liberar();         // hasta que no libere, no atiende a nadie más
  }
}
```
**Clave:** la exclusión mutua no la da ningún `lock`: la da que el Empleado se queda esperando `liberar` de *esa* persona (`Persona[idP]?`, no `[*]`).

**C2. Orden de llegada con el recurso exclusivo (Ej. 4c):** se separan los roles. El **Admin** guarda la cola de espera y el **Empleado** pide la siguiente persona cuando queda libre.
```
process Persona[i=1..P] {
  Admin!avisar(i);
  Empleado?pasar();
  usarRecurso();
  Empleado!liberar();
}

process Admin {
  queue esperando;  int idP, llegaron= 0;
  do llegaron < P; Persona[*]?avisar(idP) -> push(esperando, idP);
                                              llegaron++;
     not empty(esperando); Empleado?libre() -> pop(esperando, idP);
                                               Empleado!enviar(idP);
  od
}

process Empleado {
  int idP;
  for (k=1..P) {
    Admin!libre();
    Admin?enviar(idP);
    Persona[idP]!pasar();
    Persona[idP]?liberar();
  }
}
```

**C3. Todo en un solo Admin con estado `libre` (Ej. 5):** el Admin lleva una variable `libre`, la persona que está usando el recurso (`eActual`) y una cola.
```
process Espectador[i=1..E] {
  Admin!llegada(i);
  Admin?turno();
  usarMaquina();
  Admin!liberarMaquina();
}

process Admin {
  boolean libre= true;
  int idE, eActual, llegaron= 0;
  queue cola;
  do llegaron < E; Espectador[*]?llegada(idE) -> llegaron++;
                                                 if (libre) {
                                                   libre= false;
                                                   eActual= idE;
                                                   Espectador[idE]!turno();
                                                 } else push(cola, idE);
     not libre; Espectador[eActual]?liberarMaquina() -> if (empty(cola)) libre= true;
                                                        else {
                                                          pop(cola, idE);
                                                          eActual= idE;
                                                          Espectador[idE]!turno();
                                                        }
  od
}
```
**Ojo:** la condición del `if` de la llegada es `libre` (no `empty(cola)`: la cola puede estar vacía y la máquina ocupada). Y la segunda guarda lleva `not libre` para no esperar a `eActual` cuando todavía no hay nadie usando el recurso.

### D. Orden por identificador: `for` + recepción específica (Ej. 4b)
**Idea:** no hace falta cola. El orden lo impone el `for` y el hecho de recibir de `Persona[i]` (un proceso puntual) en lugar de `Persona[*]`.
```
process Persona[i=1..P] {
  Empleado!avisar();
  Empleado?pasar();
  usarRecurso();
  Empleado!liberar();
}

process Empleado {
  for (i=1..P) {
    Persona[i]?avisar();
    Persona[i]!pasar();
    Persona[i]?liberar();
  }
}
```
**Ojo:** la persona i+1 queda bloqueada en su `!avisar` hasta que el Empleado llegue a su turno del `for`. No se pierde nada.

### E. Pipeline: cadena de procesos (Ej. 2)
**Idea:** cada etapa le pasa su resultado a la siguiente. Entre dos etapas se decide según si el que envía debe esperar o no.
- **El que envía NO espera (etapa 1 → etapa 2):** va un Admin con cola en el medio (patrón B con un solo consumidor).
- **El que envía SÍ espera el resultado (etapa 2 ↔ etapa 3):** comunicación directa pedido-respuesta, sin buffer, porque nunca habrá más de un pedido pendiente.
```
process Etapa1 {
  while (true) { m= producir();  Admin!enviar(m); }
}

process Admin {                      // buffer entre etapa 1 y 2
  queue cola;
  do Etapa1?enviar(m) -> push(cola, m);
     not empty(cola); Etapa2?pedir() -> pop(cola, m);
                                        Etapa2!muestra(m);
  od
}

process Etapa2 {
  while (true) {
    Admin!pedir();
    Admin?muestra(m);
    set= armarSet(m);
    Etapa3!enviarSet(set);           // etapa 3 es pedido-respuesta
    Etapa3?resultado(res);
    archivar(res);
  }
}

process Etapa3 {
  while (true) {
    Etapa2?enviarSet(set);
    res= analizar(set);
    Etapa2!resultado(res);
  }
}
```

### F. Barrera: esperar a que lleguen todos (Ej. 3c)
**Idea:** el Admin cuenta las llegadas con un `for` y, cuando llegaron todos, los libera con otro `for`. Los procesos hacen `!avisarLlegada` y se quedan esperando en `?comenzar`.
```
process Alumno[i=1..N] {
  Admin!avisarLlegada();
  Admin?comenzar();                 // no empieza hasta que lleguen todos
  examen= resolverExamen();
  ...
}

process Admin {
  for (k=1..N) Alumno[*]?avisarLlegada();     // junto a los N
  for (k=1..N) Alumno[k]!comenzar();          // los libero a todos
  ...                                         // sigue con el patrón B
}
```
El segundo `for` no genera demora: todos los alumnos ya están bloqueados esperando el `comenzar`.

### G. Situaciones de corte: que todos los procesos terminen (Ej. 3a, 3b, 3c, 5)
Un `do` termina cuando **todas** sus guardas son falsas, y un proceso bloqueado esperando a alguien que ya no va a mandar nada **no termina**. Hay que pensar siempre quién corta y cómo:

| Situación | Cómo se corta | Ejemplo |
|---|---|---|
| El trabajador sabe cuántas veces trabaja | `for (k=1..N)` en lugar de `while (true)` | Profesor con P=1 (3a), Empleado (4a, 4b, 4c) |
| El Admin sabe cuántos pedidos van a llegar | Contador en la condición de la guarda de llegada: `cantE < N;` | Admin de 3a, 3b; `llegaron < E;` en 5 |
| Hay varios trabajadores y no se sabe cuántos pedidos atiende cada uno | **Centinela:** el Admin responde `ok= false` a cada trabajador que pide cuando ya no queda nada, y el trabajador hace `while (ok)` | Ej. 3b (abajo) |

**Patrón del centinela (Ej. 3b):**
```
process Trabajador[t=1..T] {
  T dato;  int idC;  boolean ok;
  Admin!espero(t);
  Admin?enviar(dato, idC, ok);
  while (ok) {
    res= procesar(dato);
    Cliente[idC]!respuesta(res);
    Admin!espero(t);
    Admin?enviar(dato, idC, ok);
  }
}

process Admin {
  queue cola;  T dato;  int idC, idT;
  int cantE= 0, cantT= 0;           // cantE: pedidos recibidos; cantT: trabajadores ya despedidos
  do cantE < N; Cliente[*]?pedido(dato, idC) -> push(cola, (dato, idC));
                                                 cantE++;
     not empty(cola); Trabajador[*]?espero(idT) -> pop(cola, (dato, idC));
                                                    Trabajador[idT]!enviar(dato, idC, true);
     empty(cola) && cantE == N && cantT < T; Trabajador[*]?espero(idT) -> cantT++;
                                                                          Trabajador[idT]!enviar(-1, -1, false);
  od
}
```
**Cómo leerlo:** la tercera guarda sólo está habilitada cuando ya llegaron los N pedidos y la cola está vacía; ahí cada trabajador que pida trabajo recibe `false` y termina. Cuando se despidieron los T trabajadores (`cantT == T`) y no queda nada, todas las guardas fallan y el Admin también termina.

## 4. Checklist antes de dar por terminado un ejercicio
- [ ] ¿Hay algún `!` dentro de una guarda? Tiene que ir después de la flecha.
- [ ] Si recibo con `[*]`, ¿el emisor manda su id?
- [ ] Si hay varios trabajadores, ¿el cliente recibe la respuesta con `Trabajador[*]?`?
- [ ] ¿Los mensajes coinciden entre emisor y receptor (mismo port y misma cantidad de parámetros)?
- [ ] ¿Se puede producir un deadlock? Si A hace `B!x; B?y;` entonces B tiene que hacer `A?x; A!y;` (si los dos empiezan enviando, se bloquean).
- [ ] ¿El Admin hace trabajo pesado? Pasárselo a un trabajador.
- [ ] ¿Todos los procesos terminan? Revisar `while (true)`, `do` sin contador y procesos que quedan esperando un mensaje que nunca llega (patrón G).
- [ ] ¿La guarda usa el contenido del mensaje que todavía no recibió? No se puede: la condición `B` sólo mira variables locales.
