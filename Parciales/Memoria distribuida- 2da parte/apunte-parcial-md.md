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
