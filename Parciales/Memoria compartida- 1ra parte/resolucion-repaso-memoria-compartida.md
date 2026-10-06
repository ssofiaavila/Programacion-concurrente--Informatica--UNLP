# Práctica de repaso | Memoria compartida
# Semáforos
> 1. Resolver los problemas siguientes
>  * En una estación de trenes, asisten P personas que deben realizar una carga de su tarjeta SUBE en la terminal disponible. La terminal es utilizada en forma exclusiva por cada persona de acuerdo con el orden de llegada. Implemente una solución utilizando únicamente procesos Persona. Nota: la función UsarTerminal() le permite cargar la SUBE en la terminal disponible. 
```
sem mutexFila= 1;
boolean libre= true;
queue fila;
sem espera[P]= ([P] 0);

process Persona[ i=1..P]{
    int aux;
    P(mutexFila);
    if (not libre){
        push(fila,i);
        V(mutexFila);
        P(espea[i]);
    } else {
        libre= false;
        V(mutexFila);
    }
    UsarTerminal();
    P(mutexFila);
    if (!fila.empty()){
        pop(fila, aux);
        V(espera[aux]);
        V(mutexFila);
    } else {
        libre= true;
        V(mutexFila);
    }
}
```
>  * Resuelva el mismo problema anterior pero ahora considerando que hay T terminales disponibles. Las personas realizan una única fila y la carga la realizan en la primera terminal que se libera. Recuerde que sólo debe emplear procesos Persona. Nota: la función UsarTerminal(t) le permite cargar la SUBE en la terminal t. 
```
queue terminales;
sem mutexFila= 0;
int libre= T;
sem mutexTerminal= 1;
sem espera[P]= ([P] 0);

procedure Persona[i=1..P]{
    int terminal;
    int aux;

    P(mutexFila);
    if (libre == 0){
        push(fila, i);
        V(mutexFila);
        P(espera[i]);
    } else {
        libre= false;
        V(mutexFila);
    }
    P(mutexTerminal);
    pop(terminales, terminal);
    V(mutexTerminal);
    UsarTerminal(terminal);
    P(mutexTerminal);
    push(terminales, terminal);
    V(mutexTerminal);
    P(mutexFila);
    if (!fila.empty()){
        pop(fila, aux);
        V(espera[aux]);
    } else {
        libre++;
        V(mutexFila);
    }
}
```

> 2. Implemente una solución para el siguiente problema. Un sistema debe validar un conjunto de 10000 transacciones que se encuentran disponibles en una estructura de datos. Para ello, el sistema dispone de 7 workers, los cuales trabajan colaborativamente validando de a 1 transacción por vez cada uno. Cada validación puede tomar un tiempo diferente y para realizarla los workers disponen de la función Validar(t), la cual retorna como resultado un número entero entre 0 al 9. Al finalizar el procesamiento, el último worker en terminar debe informar la cantidad de transacciones por cada resultado de la función de validación. Nota: maximizar la concurrencia. 
```

```

> 3. Implemente una solución para el siguiente problema. Se debe simular el uso de una máquina expendedora de gaseosas con capacidad para 100 latas por parte de U usuarios. Además, existe un repositor encargado de reponer las latas de la máquina. Los usuarios usan la máquina según el orden de llegada. Cuando les toca usarla, sacan una lata y luego se retiran. En el caso de que la máquina se quede sin latas, entonces le debe avisar al repositor para que cargue nuevamente la máquina en forma completa. Luego de la recarga, saca una botella y se retira. Nota: maximizar la concurrencia; mientras se reponen las latas se debe permitir que otros usuarios puedan agregarse a la fila.

# Monitores
> 1. Resolver el siguiente problema. En una elección estudiantil, se utiliza una máquina para voto electrónico. Existen N Personas que votan y una Autoridad de Mesa que les da acceso a la máquina de acuerdo con el orden de llegada, aunque ancianos y embarazadas tienen prioridad sobre el resto. La máquina de voto sólo puede ser usada por una persona a la vez. Nota: la función Votar() permite usar la máquina.

> 2. Resolver el siguiente problema. En una empresa trabajan 20 vendedores ambulantes que forman 5 equipos de 4 personas cada uno (cada vendedor conoce previamente a qué equipo pertenece). Cada equipo se encarga de vender un producto diferente. Las personas de un equipo se deben juntar antes de comenzar a trabajar. Luego cada integrante del equipo trabaja independientemente del resto vendiendo ejemplares del producto correspondiente. Al terminar cada integrante del grupo debe conocer la cantidad de ejemplares vendidos por el grupo. Nota: maximizar la concurrencia.

> 3. Resolver el siguiente problema. En una montaña hay 30 escaladores que en una parte de la subida deben utilizar un único paso de a uno a la vez y de acuerdo con el orden de llegada al mismo. Nota: sólo se pueden utilizar procesos que representen a los escaladores; cada escalador usa sólo una vez el paso.