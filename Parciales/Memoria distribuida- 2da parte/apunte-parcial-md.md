# Apunte parcial - Memoria distribuida

Dudas que van surgiendo mientras estudio.

---

## ¿Cuál es la diferencia principal entre PMA y PMS?

La diferencia principal está en **si el emisor se bloquea o no al enviar**.

### PMA: Pasaje de Mensajes Asincrónico
- **`send` no bloquea.** El emisor deja el mensaje en el canal y sigue ejecutando.
- Los canales funcionan como **colas (buffers) de capacidad ilimitada**, en teoría.
- Solo **`receive` bloquea**, cuando el canal está vacío.
- Los canales son **globales**, se declaran con nombre (`chan pedidos(int)`) y cualquier proceso puede enviar o recibir por ellos.
- Existe `empty(canal)` para preguntar si hay mensajes.
- Sintaxis: `send canal(x)` / `receive canal(x)`.

### PMS: Pasaje de Mensajes Sincrónico (estilo CSP)
- **`!` (send) bloquea** hasta que el receptor ejecute el `?` correspondiente. Hay un **rendezvous**: los dos procesos se sincronizan en ese punto.
- **No hay buffer.** El mensaje pasa directo de un proceso al otro.
- Los canales son **enlaces directos entre procesos**: `Proceso!dato` / `Proceso?dato`. Hay que nombrar al otro proceso.
- Como no hay `empty`, para atender a varios emisores se usa **comunicación guardada**: `if` / `do` con guardas `condición; Proceso?msg -> ...`.

### Consecuencias prácticas (las que suelen evaluarse)

| | PMA | PMS |
|---|---|---|
| Bloqueo del emisor | No | Sí |
| Buffer | Sí (cola) | No |
| Concurrencia | Mayor, el emisor sigue trabajando | Menor, ambos esperan |
| Riesgo de deadlock | Menor | Mayor (p. ej. dos procesos que hacen `!` entre sí al mismo tiempo) |
| Buffer de N lugares | Viene "gratis" con el canal | Hay que programar un proceso intermedio (administrador/buffer) |
| Elegir entre varios mensajes | `if not empty(c1) -> ...` | Guardas con `?` en `do`/`if` |

**En una frase:** en PMA enviar es "dejar una carta en el buzón y seguir", y en PMS es "darle algo en mano a alguien, esperando a que lo agarre".

---

## ¿Cómo le mando un mensaje a un proceso en particular?

### PMS
No hay canales: **cada lado nombra directamente al otro proceso**.
- **Emisor:** indica a quién le manda. Si es de un arreglo, pone el índice: `Persona[idP]!pasar()`. Si es un proceso único: `Admin!pedido(...)`.
- **Receptor:** indica de quién espera recibir. Uno específico: `Terminal?pasar()`, `Persona[3]?liberar()`. Cualquiera de un arreglo: `Persona[*]?avisar(idP)`.
- Con el `[idP]` del emisor ya alcanza. El receptor **no necesita un `[i]` propio** para que le llegue.

### PMA
Los procesos no se nombran, se comunican por canales. Para que un mensaje le llegue a un proceso en particular hace falta un **arreglo de canales**: `chan respuesta[1..N](...)`, con `send respuesta[id](...)` y `receive respuesta[miId](...)`.

---

## ¿Y si un receptor tiene varios emisores?

- **Varios procesos del mismo arreglo:** `Persona[*]?avisar(idP)`.
  - No sabés quién mandó, así que si hay que responderle, **el emisor tiene que mandar su id** como parámetro.
  - El `[*]` existe solo para arreglos.
- **Procesos distintos** (por ejemplo Admin y Terminal): se usa una sentencia de guardas, con una rama por emisor.
  ```
  do Persona[*]?avisar(idP) -> push(cola, idP);
  □ Terminal?listo() -> ...
  od
  ```

---

## Orden de llegada: ¿PMS respeta el orden como PMA?

**No.**
- **PMA:** el canal es una **cola FIFO**. `send` no bloquea, el mensaje se encola y `receive` saca el más viejo. **El orden de llegada sale gratis.**
- **PMS:** no hay cola. Si varios emisores están bloqueados en su `!` hacia el mismo receptor, `Persona[*]?` elige uno **no determinísticamente**, no el que llegó primero. Lo mismo pasa con varias guardas habilitadas en un `if`/`do`.

### Regla práctica
- **PMS + "orden de llegada" → hace falta un proceso con cola** que siempre esté disponible para recibir y encolar enseguida. Atiende sacando de la cola.
  - **Puede ser el mismo servidor**, si no queda ocupado haciendo trabajo largo (por ejemplo el Empleado que solo da paso a una máquina que usa otro).
  - **Tiene que ser un Admin aparte** si el servidor trabaja (una impresora imprimiendo, por ejemplo). Mientras trabaja no puede recibir y se pierde el orden.
- **PMA:** en general alcanza con el canal compartido. El Admin aparece por otras razones: prioridades con varias colas, o lógica extra para asignar servidores.

### Error típico: atender por id en vez de por llegada
```
for i = 1..P {
  Deportista[i]?llegada();      // MAL: atiende 1, 2, 3... sin importar quién llegó primero
  ...
}
```
Tampoco alcanza con cambiarlo a `Deportista[*]?llegada(id)` si después el servidor se queda bloqueado esperando el `salida()`. Los que llegan mientras tanto quedan trabados en su `!` y después `[*]` elige al azar.

### Patrón correcto (el servidor hace de admin)
```
process Deportista[id: 1..P] {
  Empleado!llegada(id);
  Empleado?usar();
  delay(tiempoUso);
  Empleado!salida();
}

process Empleado {
  queue cola; int idD;
  boolean libre = true;
  do Deportista[*]?llegada(idD) -> push(cola, idD);
  □ libre and not empty(cola) ->
        pop(cola, idD);
        libre = false;
        Deportista[idD]!usar();
  □ not libre; Deportista[*]?salida() -> libre = true;
  od
}
```
El Empleado **nunca se queda esperando a uno en particular**: siempre puede encolar a los que llegan, aunque la máquina esté ocupada.
