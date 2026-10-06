# Apunte — Teoría 5: Memoria distribuida y PMA

## La idea en una frase

En memoria distribuida los procesos no comparten variables: lo único que comparten son **canales**. En Pasaje de Mensajes Asincrónicos (PMA) el `send` no bloquea y el `receive` sí, y con eso se programan los tres patrones de interacción: filtros, cliente/servidor y pares.

La clase tiene dos partes:

1. Conceptos generales de la programación concurrente en memoria distribuida.
2. PMA: canales, primitivas y ejemplos para cada patrón.

## 1. Memoria distribuida

Una arquitectura de memoria distribuida tiene procesadores con memoria local, una red de comunicaciones y un mecanismo de comunicación y sincronización basado en intercambio de mensajes.

Un **programa distribuido** es un programa concurrente comunicado por mensajes. Supone una arquitectura de memoria distribuida, aunque puede ejecutarse sobre memoria compartida.

Consecuencias de que los procesos solo compartan canales:

- Las variables son locales a un proceso (el proceso es su "cuidador").
- **La exclusión mutua no requiere un mecanismo especial**: nadie más puede tocar esas variables.
- Los procesos interactúan comunicándose, con primitivas de envío y recepción.

### Mecanismos y patrones

| Mecanismos | Patrones de interacción |
| --- | --- |
| Pasaje de Mensajes Asincrónico (PMA) | Productores y consumidores (filtros o pipes) |
| Pasaje de Mensajes Sincrónico (PMS) | Clientes y servidores |
| Remote Procedure Call (RPC) | Pares que interactúan |
| Rendezvous | |

Cada mecanismo es más adecuado para determinados patrones.

### Cómo se relacionan los mecanismos de toda la materia

- **Semáforos**: mejora respecto de busy waiting.
- **Monitores**: combinan exclusión mutua implícita y señalización explícita.
- **Pasaje de mensajes**: extiende los semáforos con datos.
- **RPC y rendezvous**: combinan la interfaz procedural de los monitores con pasaje de mensajes implícito.

## 2. PMA: canales y primitivas

En PMA un canal es una cola de mensajes enviados y todavía no recibidos.

```
chan ch (id1: tipo1, ..., idn: tipon)

chan entrada (char);
chan acceso_disco (INT cilindro, INT bloque, INT cant, CHAR* buffer);
chan resultado[n] (INT);
```

| Primitiva | Sintaxis | Comportamiento |
| --- | --- | --- |
| Send | `send ch(expr1, ..., exprn);` | Agrega un mensaje al final de la cola. **No bloquea** al emisor |
| Receive | `receive ch(var1, ..., varn);` | **Bloquea** al receptor hasta que haya al menos un mensaje; toma el primero y lo guarda en variables locales |
| Empty | `empty(ch)` | Dice si la cola del canal está vacía |

Las variables del `receive` deben tener los mismos tipos que la declaración del canal. Como el `receive` demora al proceso, no hace falta hacer polling.

### Características de los canales

- El acceso es **atómico** y respeta el orden **FIFO**.
- En principio son **ilimitados**.
- Los mensajes no se pierden ni se modifican, y todo mensaje enviado en algún momento puede ser leído.

### Cuidado con `empty`

Sirve cuando el proceso puede hacer trabajo útil mientras espera, pero el resultado puede quedar viejo enseguida:

- Puede dar **false** y que, al seguir ejecutando, ya no haya mensajes, porque otro proceso los recibió. Esto no pasa si el proceso es el único que recibe por ese canal.
- Puede dar **true** y que justo después llegue un mensaje.

### Tipos de canal según su uso

Los canales se declaran globales a los procesos.

| Tipo | Emisores | Receptores |
| --- | --- | --- |
| Mailbox | Cualquiera | Cualquiera |
| Input port | Muchos | Uno |
| Link | Uno | Uno |

### Ejemplo: de caracteres a líneas

```
chan entrada(char), salida(char [CantMax]);

Process Proceso_1
{ char a;
  WHILE (true)
    { leer_carácter_por_teclado(a);
      send entrada(a);
    }
}

Process Carac_a_Linea
{ char linea[CantMax], int i = 0;
  WHILE (true)
    { receive entrada(linea[i]);
      WHILE (linea[i] ≠ CR and i < CantMax)
        { i := i + 1;
          receive entrada(linea[i]);
        }
      linea[i] := EOL;
      send salida(linea);
      i := 0;
    }
}

Process Proceso_2
{ char res[CantMax];
  WHILE (true)
    { receive salida(res);
      imprimir_en_pantalla(res);
    }
}
```

## 3. Filtros: red de ordenación

Un **filtro** es un proceso que recibe mensajes de uno o más canales de entrada y envía mensajes a uno o más canales de salida. Su salida es función de su estado inicial y de los valores recibidos.

El problema es ordenar N números. Un único filtro `Sort` recibiría todo, ordenaría y enviaría. Para saber que recibió todo tiene tres opciones:

- Conoce N de antemano.
- N llega como primer elemento.
- La lista termina con un valor especial o **centinela**.

### Merge network

Es más eficiente una red de procesos chicos que ejecutan en paralelo. La idea es mezclar repetidamente dos listas ordenadas en una lista ordenada del doble de tamaño.

Cada filtro `Merge` recibe dos streams ordenados (`in1`, `in2`) y produce uno ordenado (`out`). Los streams terminan con el centinela `EOS`.

```
chan in1(int), in2(int), out(int);

Process Merge
{ int v1, v2;
  receive in1(v1);
  receive in2(v2);
  while (v1 ≠ EOS) and (v2 ≠ EOS)
    { if (v1 ≤ v2) { send out(v1); receive in1(v1); }
      else { send out(v2); receive in2(v2); }
    }
  if (v1 == EOS) while (v2 ≠ EOS) { send out(v2); receive in2(v2); }
  else while (v1 ≠ EOS) { send out(v1); receive in1(v1); }
  send out(EOS);
}
```

Compara los dos valores que tiene, envía el menor y repone del canal que usó. Cuando un stream se termina, vacía el otro. Al final agrega su propio `EOS`.

Para ordenar n valores:

- Hacen falta **n-1 procesos** Merge.
- El ancho de la red es **log₂ n**.

Ejemplo con 8 valores: 4 Merge producen listas de 2, otros 2 producen listas de 4, y 1 produce la lista de 8. En total, 7.

### Cómo conectar los filtros

| Opción | Cómo funciona |
| --- | --- |
| Static naming | Arreglo global de canales. Cada Merge recibe de dos elementos del arreglo y envía a otro. Hay que embeber el árbol en el arreglo |
| Dynamic naming | Canales globales y procesos parametrizados: a cada uno se le dan 3 canales al crearlo. Todos los Merge son idénticos, pero hace falta un coordinador |

Los filtros se pueden conectar de distintas maneras y reemplazar entre sí. Solo hace falta que la salida de uno cumpla las suposiciones de entrada del siguiente.

## 4. Clientes y servidores

Un **servidor** es un proceso que maneja pedidos (*requests*) de procesos clientes.

### Monitores activos

Hay una **dualidad entre monitores y pasaje de mensajes**: cada uno puede simular al otro. Un monitor es un manejador de recurso pasivo; se lo simula con un proceso servidor activo.

El esquema básico:

- Un canal **general** de requerimientos, por el que envían todos los clientes.
- Un canal de respuesta **propio** de cada cliente.

La pregunta de la diapositiva es por qué el canal de respuesta es propio. Si fuera compartido, cualquier cliente podría recibir la respuesta destinada a otro. Por eso el cliente manda su id en el pedido.

**Una operación:**

```
chan requerimiento (int idCliente, tipos de los valores de entrada);
chan respuesta[n] (tipos de los resultados);

Process Servidor
{ int idCliente;
  declaración de variables permanentes;
  código de inicialización;
  while (true)
    { receive requerimiento(idCliente, valores de entrada);
      cuerpo de la operación op;
      send respuesta[idCliente](resultados);
    }
}

Process Cliente [i = 1 to n]
{ send requerimiento(i, argumentos);
  receive respuesta[i](resultados);
}
```

**Múltiples operaciones:** el pedido incluye qué operación se quiere, y el servidor decide con un `if` o `case`.

```
type clase_op = enum(op1, ..., opn);
type tipo_arg = union(arg1: tipoAr1, ..., argn: tipoArn);
type tipo_result = union(res1: tipoRe1, ..., resn: tipoRen);
chan request(int idCliente, clase_op, tipo_arg);
chan respuesta[n](tipo_result);

Process Servidor
{ int IdCliente; clase_op oper; tipo_arg args;
  tipo_result resultados;
  código de inicialización;
  while (true)
    { receive request(IdCliente, oper, args);
      if (oper == op1) { cuerpo de op1; }
      ...
      elsif (oper == opn) { cuerpo de opn; }
      send respuesta[IdCliente](resultados);
    }
}

Process Cliente [i = 1 to n]
{ tipo_arg mis_args;
  tipo_result mis_resultados;
  send request(i, opk, mis_args);
  receive respuesta[i](mis_resultados);
}
```

### Con sincronización por condición

Hasta acá el servidor nunca necesitaba demorar un pedido. El caso general es un monitor con variables condición. El ejemplo es un administrador de un recurso de múltiples unidades (bloques de memoria, impresoras). Para los clientes el cambio es transparente; cambia el servidor.

La clave: el servidor **no puede quedarse esperando**. Si no hay unidades, tiene que **guardar el pedido y diferir la respuesta**.

```
type clase_op = enum(adquirir, liberar);
chan request(int idCliente, claseOp oper, int idUnidad);
chan respuesta[n] (int id_unidad);

Process Administrador_Recurso
{ int disponible = MAXUNIDADES;
  set unidades = valor inicial disponible;
  queue pendientes;
  while (true)
    { receive request(IdCliente, oper, id_unidad);
      if (oper == adquirir)
        { if (disponible > 0)
            { disponible = disponible - 1;
              remove(unidades, id_unidad);
              send respuesta[IdCliente](id_unidad);
            }
          else push(pendientes, IdCliente);
        }
      else
        { if empty(pendientes)
            { disponible = disponible + 1;
              insert(unidades, id_unidad);
            }
          else
            { pop(pendientes, IdCliente);
              send respuesta[IdCliente](id_unidad);
            }
        }
    }
}

Process Cliente [i = 1 to n]
{ int id_unidad;
  send request(i, adquirir, 0);
  receive respuesta[i](id_unidad);
  # Usa la unidad
  send request(i, liberar, id_unidad);
}
```

Al liberar, si hay pedidos pendientes, la unidad se le pasa directamente al primero. El cliente que libera no espera respuesta.

### Dualidad entre monitores y pasaje de mensajes

| Programas con monitores | Programas basados en PM |
| --- | --- |
| Variables permanentes | Variables locales del servidor |
| Identificadores de procedures | Canal request y tipos de operación |
| Llamado a procedure | `send request(); receive respuesta` |
| Entry del monitor | `receive request()` |
| Retorno del procedure | `send respuesta()` |
| Sentencia `wait` | Salvar pedido pendiente |
| Sentencia `signal` | Recuperar y procesar pedido pendiente |
| Cuerpos de los procedures | Sentencias del "case" según la clase de operación |

La eficiencia depende de la arquitectura: con memoria compartida conviene la invocación a procedimientos y las variables condición; en arquitecturas físicamente distribuidas tienden a ser más eficientes los mecanismos de pasaje de mensajes.

### Sentencia de alternativa múltiple

El mismo administrador, con un canal por operación y guardas:

```
chan pedido (int idCliente);
chan liberar (int idUnidad);
chan respuesta[n] (int idUnidad);

Process Administrador_Recurso
{ int disponible = MAXUNIDADES;
  set unidades = valor inicial disponible;
  int id_unidad, idCliente;
  while (true)
    { if ((not empty(pedido)) and (disponible > 0)) →
            receive pedido(idCliente);
            disponible = disponible - 1;
            remove(unidades, id_unidad);
            send respuesta[idCliente](id_unidad);
      □ (not empty(liberar)) →
            receive liberar(id_unidad);
            disponible = disponible + 1;
            insert(unidades, id_unidad);
      fi
    }
}

Process Cliente [i = 1 to n]
{ int id_unidad;
  send pedido(i);
  receive respuesta[i](id_unidad);
  # Usa la unidad
  send liberar(id_unidad);
}
```

La diapositiva deja como pregunta las ventajas y desventajas. Lo que veo comparando las dos versiones:

- **Ventaja**: no hace falta la cola `pendientes`. Los pedidos que no se pueden atender quedan en el canal `pedido` hasta que haya unidades.
- **Desventaja**: si ninguna guarda es verdadera, el `if` no hace nada y el `while (true)` vuelve a evaluar. Es busy waiting.

### Continuidad conversacional

A alumnos hacen consultas a D docentes. Un alumno espera a que un docente lo atienda y después le hace todas sus consultas hasta quedarse sin dudas. Los docentes son servidores idénticos: cualquiera que esté libre puede atender.

```
chan atención (int);
chan consulta[D] (string);
chan rta_atención[A] (int);
chan respuesta[A] (string);

Process Alumno [a = 1 to A]
{ int idDocente;
  string preg, res;
  send atención(a);
  receive rta_atención[a](idDocente);
  while (tenga consultas para hacer)
    { send consulta[idDocente](preg);
      receive respuesta[a](res);
    }
  send consulta[idDocente]('FIN');
}

Process Docente [d = 1 to D]
{ string preg, res;
  int idAlumno;
  bool seguir = false;
  while (true)
    { receive atención(idAlumno);
      send rta_atención[idAlumno](d);
      seguir = true;
      while (seguir)
        { receive consulta[d](preg);
          if (preg == 'FIN') seguir = false
          else
            { res = resolver la pregunta (preg)
              send respuesta[idAlumno](res);
            }
        }
    }
}
```

| Canal | Tipo | Para qué |
| --- | --- | --- |
| `atención` | Global, compartido por todos | El alumno pide atención; lo toma cualquier docente libre |
| `rta_atención[a]` | Propio del alumno | El docente le dice quién lo atiende |
| `consulta[d]` | Propio del docente | El alumno le habla siempre al mismo docente |
| `respuesta[a]` | Propio del alumno | El docente le contesta |

Se llama **continuidad conversacional** porque, desde que pide atención hasta la última consulta, el alumno conversa con el mismo docente.

`atención` es un canal del que reciben todos los docentes. Si cada canal pudiera tener un solo receptor, haría falta un proceso intermedio que reciba los pedidos y los reparta entre los docentes libres.

## 5. Pares interactuantes: intercambio de valores

Cada uno de N procesos tiene un valor local V. Al final todos tienen que conocer el mínimo y el máximo. Hay tres soluciones según la arquitectura.

### Centralizada

Todos le envían su valor a un proceso central, que calcula y reenvía el resultado.

```
chan valores(int), resultados[n-1] (int minimo, int maximo);

Process P[0]
{ int v; int nuevo, minimo = v, máximo = v;
  for [i = 1 to n-1]
    { receive valores(nuevo);
      if (nuevo < minimo) minimo = nuevo;
      if (nuevo > maximo) maximo = nuevo;
    }
  for [i = 1 to n-1]
    send resultados[i-1](minimo, maximo);
}

Process P[i = 1 to n-1]
{ int v; int minimo, máximo;
  send valores(v);
  receive resultados[i-1](minimo, maximo);
}
```

La diapositiva pregunta si se puede usar un único canal de resultados. Creo que sí: todos los mensajes de resultado son iguales, así que no importa cuál reciba cada proceso.

### Simétrica

Hay un canal entre cada par de procesos y todos ejecutan el mismo algoritmo. Es un ejemplo de **SPMD**: el mismo programa sobre datos distintos.

```
chan valores[n] (int);

Process P[i = 0 to n-1]
{ int v = ..., nuevo, minimo = v, máximo = v;
  for [k = 0 to n-1 st k <> i]
    send valores[k](v);
  for [k = 0 to n-1 st k <> i]
    { receive valores[i](nuevo);
      if (nuevo < minimo) minimo = nuevo;
      if (nuevo > maximo) maximo = nuevo;
    }
}
```

La diapositiva pregunta si se puede usar un único canal. Creo que no: un proceso podría recibir sus propios mensajes o varias copias del mismo valor, y perderse otros.

### Anillo

P[i] recibe de P[i-1] y envía a P[i+1]. Tiene dos etapas: en la primera vuelta cada proceso compara con su valor y pasa el mínimo y el máximo parciales; en la segunda circulan los valores globales.

```
chan valores[n] (int minimo, int maximo);

Process P[0]
{ int v = ..., minimo = v, máximo = v;
  send valores[1](minimo, maximo);
  receive valores[0](minimo, maximo);
  send valores[1](minimo, maximo);
}

Process P[i = 1 to n-1]
{ int v = ..., minimo, máximo;
  receive valores[i](minimo, maximo);
  if (v < minimo) minimo = v;
  if (v > maximo) maximo = v;
  send valores[(i+1) mod n](minimo, maximo);
  receive valores[i](minimo, maximo);
  if (i < n-1) send valores[i+1](minimo, maximo);
}
```

P[0] es distinto porque arranca el procesamiento.

### Comparación

| Solución | Mensajes | Observaciones |
| --- | --- | --- |
| Centralizada | 2(N-1) | Los mensajes al coordinador se envían casi al mismo tiempo; solo el primer `receive` demora mucho |
| Simétrica | N(N-1) | La más corta y sencilla de programar. Con una primitiva de broadcast serían N mensajes |
| Anillo | 2N-1 | Inherentemente lineal y lenta: los mensajes dan dos vueltas completas y cada proceso espera al anterior |

Centralizada y anillo usan una cantidad lineal de mensajes, pero el rendimiento es muy distinto. En la centralizada los envíos ocurren en paralelo. En el anillo son secuenciales: el último proceso tiene que esperar a que todos los anteriores, uno por vez, reciban y reenvíen.

En la simétrica los mensajes pueden transmitirse en paralelo si la red lo soporta, pero el overhead de comunicación acota el speedup.

## 6. Trampas y autoevaluación

### Trampas típicas

- **Compartir variables entre procesos.** En pasaje de mensajes solo se comparten canales.
- **Usar un canal de respuesta compartido** cuando las respuestas son distintas para cada cliente.
- **Olvidar mandar el id del cliente** en el pedido. El servidor no sabe a quién responder.
- **Hacer que el servidor se bloquee esperando una condición.** Tiene que guardar el pedido y seguir atendiendo.
- **Confiar en `empty`.** El resultado puede dejar de ser cierto de inmediato si hay otros receptores.
- **Decir que anillo y centralizada rinden igual** porque las dos son lineales en mensajes.

### Preguntas para chequear que se entendió

1. ¿Por qué en memoria distribuida la exclusión mutua no requiere un mecanismo especial?
2. ¿Cuáles son los cuatro mecanismos de procesamiento distribuido y los tres patrones de interacción?
3. ¿Qué diferencia hay entre `send` y `receive` en PMA?
4. ¿Qué diferencia hay entre mailbox, input port y link?
5. ¿Por qué hay que tener cuidado con `empty`?
6. ¿Cómo sabe un filtro que recibió todos los datos?
7. ¿Cuántos procesos Merge hacen falta para ordenar n valores?
8. ¿Por qué cada cliente tiene un canal de respuesta propio?
9. ¿Cómo se traducen `wait` y `signal` de un monitor a un servidor con PMA?
10. ¿Qué es la continuidad conversacional?
11. Comparar las soluciones centralizada, simétrica y en anillo en cantidad de mensajes y en rendimiento.

## Material

- Diapositivas: `Clases teóricas/teoria-5.pdf`. La diapositiva 2 tiene los links a los dos videos con audio.
