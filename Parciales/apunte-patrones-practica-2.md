# Apunte de patrones — Semáforos (Práctica 2)

Mini-catálogo para repasar rápido antes del parcial. No repite la resolución completa de cada ejercicio (eso está en `apuntes-practica-2.md`) — acá la idea es: mirar un enunciado nuevo, reconocer qué patrón(es) aplican, y recién ahí escribir código.

## Cómo usarla

Ante cualquier ejercicio nuevo: leé el enunciado, identificá las palabras clave de la tabla de abajo, y preguntate cuáles de estos patrones se combinan (casi siempre son 2 o 3 a la vez, no uno solo). Recién después escribís.

## Índice rápido: señal en el enunciado → patrón

- "solo uno a la vez", "sección crítica", "no pueden estar dos al mismo tiempo" → **Mutex simple**
- "hay N recursos/lugares/instancias", "hasta N a la vez" → **Contador de recursos**
- "avisarle a UN proceso puntual", "llamar por id", "despertar al que corresponde" → **Semáforo privado por proceso**
- "todos deben esperar a que lleguen todos", "recién cuando estén todos" → **Barrera**
- "atender en orden de llegada" (sin más vueltas) → **Cliente/servidor** (cola + mutex + contador + privados)
- "atender en orden de llegada" pero varios procesos podrían competir y el orden importa de verdad (un semáforo simple no alcanza) → **Coordinador centralizador**
- "elegir el que tiene menos/más", "el mínimo/máximo entre varios" → **Reducción protegida**
- "hasta K en total, pero máximo X de un subtipo" → **Orden de adquisición: específico antes que general**
- "un recurso con estado propio (libre/ocupado) + gente en fila" → **Pasar la posta** (sin volver a "libre" si hay alguien esperando)
- "todos los procesos deben terminar" → señal de que hay que poner un corte explícito (contador fijo de iteraciones, o barrera de finalización) — no un `while(true)` eterno

## Patrones (esqueleto mínimo)

### 1. Mutex simple
```
sem mutex = 1;
P(mutex);
  -- SC: solo lo estrictamente necesario
V(mutex);
```
Regla: adentro va lo mínimo indispensable; lo que no necesita exclusión mutua se saca afuera (ej. 1).

### 2. Contador de recursos (evita busy waiting)
```
sem recurso = N;   // o el contador que ya existía en el enunciado, convertido en semáforo
P(recurso);
  -- usar el recurso
V(recurso);
```
Si el enunciado ya tiene un contador compartido tipo `cant > 0`, ESE contador se convierte directamente en el semáforo (ej. 4: `pedidos`).

### 3. Semáforo privado por proceso
```
sem privado[N] = ([N] 0);
// quien espera:
P(privado[id]);
// quien avisa a ESE puntual:
V(privado[id]);
```
Nunca uses un semáforo compartido si necesitás despertar a alguien específico — un `V` sobre uno compartido puede despertar a cualquiera (ej. 3, 4, 5, 6, 10, 11, 12).

### 4. Barrera
```
int contador = 0;
sem mutex = 1, barrera = 0;

P(mutex);
  contador++;
  if (contador == N) for (k=1; k<=N; k++) V(barrera);
V(mutex);
P(barrera);
```
Clave: siempre `V(mutex)` **antes** de demorarte en `P(barrera)` — nunca al revés (ej. 3, 7, 8).

### 5. Cliente/servidor (cola de pedidos)
```
sem mutex = 1, pedidos = 0, respuesta[N] = ([N] 0);
cola C;

// cliente
P(mutex); C.push(id); V(mutex);
V(pedidos);
P(respuesta[id]);

// servidor
P(pedidos);
P(mutex); id = C.pop(); V(mutex);
-- procesar
V(respuesta[id]);
```
El esqueleto más reusable de toda la práctica (ej. 4, 6, 12-Recepcionista).

### 6. Coordinador centralizador (orden estricto)
Cuando hace falta preservar el orden de llegada de verdad (un semáforo no garantiza FIFO entre los que esperan), un único proceso Coordinador es quien pide los recursos, uno por uno, en el orden de la cola:
```
process Coordinador
{ for (k=1; k<=TOTAL; k++)
  { P(pedidos);
    P(mutex); id = cola.pop(); V(mutex);
    P(recursoEspecífico); P(recursoGeneral);
    V(privado[id]);   // recién ahí le da el pase a ESE proceso
  }
}
```
El orden queda garantizado gratis porque el Coordinador es secuencial: no puede pasar al siguiente sin resolver al anterior (ej. 10-a).

### 7. Reducción protegida (mínimo/máximo)
```
sem mutex = 1;
int valores[K];

P(mutex);
  mejor = 0;
  for (k=1; k<K; k++) if (valores[k] MEJOR_QUE valores[mejor]) mejor = k;
  -- actualizar valores[mejor] ACÁ MISMO, sin soltar el mutex
V(mutex);
```
Punto crítico: elegir + reservar tienen que ser un solo bloque atómico. Si soltás el mutex entre "elegir" y "reservar", dos procesos pueden elegir lo mismo (ej. 8, 12-b).

### 8. Orden de adquisición: específico antes que general
```
sem tipoA = X, tipoB = Y, total = X+Y;

P(tipoA);   // o P(tipoB) según corresponda — SIEMPRE primero
P(total);
-- usar
V(total);
V(tipoA);
```
Si pedís primero el general, un proceso puede quedarse bloqueado sosteniendo un permiso general que otro tipo podría estar necesitando (ej. 4, 10-b).

### 9. Pasar la posta (recurso con dueño)
```
boolean libre = true;
cola C;
sem mutex = 1, espera[N] = ([N] 0);

// tomar
P(mutex);
if (libre) { libre = false; V(mutex); }
else { C.push(id); V(mutex); P(espera[id]); }

// soltar
P(mutex);
if (C.empty()) libre = true;
else { aux = C.pop(); V(espera[aux]); }   // OJO: no vuelve a poner libre=true
V(mutex);
```
El recurso sigue "ocupado", solo cambia de dueño directamente — nunca generás un reintento (ej. 6).

## Checklist de las 4 preguntas (repaso rápido)

1. ¿Qué es realmente compartido, y qué operaciones sobre eso necesitan exclusión mutua (ni de más ni de menos)?
2. ¿Hay una espera que se pueda resolver con semáforo en vez de busy waiting?
3. ¿Hay que despertar a uno en particular, a cualquiera, o a todos?
4. ¿Toda salida del código deja los semáforos consistentes? (todo `P` tiene su `V` en algún camino; nunca esperás en un semáforo con un mutex todavía tomado)

## Errores que ya cometiste y vale la pena repasar antes del parcial

Estos son los bugs reales que fueron apareciendo en tus intentos — son los más típicos de encontrar en un parcial:

- Confundir `P` con `V` al liberar recursos al final de un proceso (ej. 9, 10).
- Proteger el `push` de una cola compartida pero olvidarse de proteger el `pop` del otro lado (ej. 11).
- Usar `if (contador == K)` para detectar "cada K", cuando el contador nunca se resetea — se dispara una sola vez en toda la ejecución. Mejor `% K == 0`, o si reseteás, restar K (no fijar en 0), para no perder a los que llegaron de más mientras tanto (ej. 11).
- Reutilizar el mismo nombre de variable para el índice de un `for` y para un valor que se lee dentro del propio loop (ej. 11).
- Colas o semáforos privados que se pisan entre dos tipos de proceso porque ambos usan el mismo rango de índices sin distinguir el tipo (ej. 10).
- Nombres de semáforo/variable duplicados por descuido (`sem total` y otra variable también llamada `total`) (ej. 10).
- Llamar a la función de "acción" (`VacunarPersona()`, `Hisopar()`, etc.) desde el proceso equivocado, cuando el enunciado dice explícitamente de quién es esa función (ej. 11, 12).


