# Preguntas examenes promoción 
1. Defina programa concurrente, programa paralelo y programa distribuido.
2. ¿A qué se denomina propiedad del programa? ¿Qué son las propiedades de seguridad y vida?
3. Defina fairness. Relacionar dicho concepto con las políticas de scheduling.
4. Describa los distintos tipos de fairness.
5. ¿Cuáles son las propiedades que debe cumplir un protocolo de E/S a una sección crítica?
6. ¿Cuáles son los defectos que presenta la sincronización por busy waiting? Diferencie esta situación respecto de los semáforos.
7. Explique la semántica de la instrucción de grano grueso AWAIT y su relación con las instrucciones Test & Set o Fetch & Add.
8. Definir el problema general de alocación de recursos y su resolución mediante una política SJN. ¿Minimiza el tiempo promedio de espera? ¿Es fair? Si no lo es, plantee una alternativa que lo sea.
9. ¿En qué consiste la técnica de passing the baton? Aplicar este concepto a la resolución del problema de lectores y escritores.
10. Explique el concepto de broadcast y sus dificultades de implementación en un ambiente distribuido, con pasaje de mensajes sincrónico y asincrónico.
11. Analice que tipos de mecanismos de pasaje de mensajes son más adecuados para resolver problemas del tipo Cliente-Servidor, Pares que interactúan, filtro y productores- consumidores. Justificar.
12. Describir la solución usando la criba de Eratóstenes al problema de hallar los números primos entre 2 y n. ¿Cómo termina el algoritmo? ¿Qué modificaría para que no termine de esa manera?
13. Sea el problema en el cual N procesos poseen inicialmente cada uno un valor V, y el objetivo es que todos conozcan cual es el máximo y cuál es el mínimo de todos los valores.
14. Defina el concepto de granularidad. ¿Qué relación existe entre la granularidad de programas y de procesadores?
15. ¿En qué consiste la comunicación guardada (introducida por CSP) y cuál es su utilidad?
16. ¿Cuál es la utilidad de la técnica de passing the baton? ¿Qué relación encuentra con la técnica de passing the condition?
17. Explique sintéticamente los 7 paradigmas de interacción entre procesos en programación distribuida. Ejemplifique.
18. ¿Cuál es el objetivo de la programación paralela?
19. Defina las métricas de speedup y eficiencia. ¿Cuál es el significado de cada una de ellas (qué miden)? ¿Cuál es el rango de valores para cada una?
20. ¿En qué consiste la ley de Amdahl?
21. Analice conceptualmente la resolución de problemas con memoria compartida y memoria distribuida. Compare con respecto a facilidad de programación.
22. ¿Qué se entiende por arquitecturas de grano fino? ¿Son más adecuadas para programas con mucha o poca comunicación?
23. Explique sintéticamente los paradigmas de resolución de problemas concurrente. Ejemplifique.
24. Analice conceptualmente los modelos de mensajes sincrónicos y asincrónicos. Compárelos en términos de concurrencia y facilidad de programación.
25. ¿En qué consisten las arquitecturas SIMD y MIMD? ¿Para qué tipo de aplicaciones es más adecuada cada una?
26. Defina sincronización entre procesos y mecanismos de sincronización.
27. Defina el concepto de sincronización barrier. ¿Cuál es su utilidad?
28. ¿Qué es una barrera simétrica?
29. Describa combining tree barrier y butterfly barrier. Marque ventajas y desventajas de cada una.
30. ¿En qué consiste la propiedad de “A lo sumo una vez” y qué efecto tiene sobre las sentencias de un programa concurrente? De ejemplos de sentencias que cumplan y de sentencias que no cumplan con ASV.
31. Defina el concepto de “sincronización barrier”. ¿Cuál es su utilidad? Mencione alguna de las soluciones posibles usando variables compartidas.
32. Indique por qué puede considerarse que existe una dualidad entre los mecanismos de monitores y pasaje de mensajes. Ejemplifique.
33. Defina el concepto de no determinismo. Ejemplifique.
34. ¿Por qué las propiedades de vida dependen de la política de scheduling? ¿Cuándo una política de scheduling es fuertemente fair?
35. Defina el problema de sección crítica. Compare los algoritmos para resolver este problema (spin locks, Tie Breaker, Ticket, Bakery). Marque ventajas y desventajas de cada uno.
36. Defina scheduling, deadlock, fairness, inanición y overloading. Ejemplifique cada uno.
37. Analice qué tipo de mecanismo de pasaje de mensajes son más adecuados para resolver problemas de tipo cliente-servidor, pares que interactúan, filtros y productores-consumidores. Justifique.
38. Describa el paradigma “bag of tasks”. ¿Cuál es la principal ventaja del mismo?
39. Defina monitores. Diferencie monitores y semáforos.
40. ¿Qué relación encuentra entre el paralelismo recursivo y la estrategia de dividir y conquistar? ¿Cómo aplicaría este concepto a un problema de ordenación de un arreglo?
41. ¿Qué condiciones deben darse para que ocurra deadlock?
42. Definir lo que es la sincronización entre procesos y explicar cuales son los dos tipos de sincronización en que se clasifica. Dar un ejemplo (explicarlo con palabras, no código) de un problema donde se usen ambos tipos de sincronización.
43. Definir comunicación entre procesos. Explicar cuáles son los dos tipos de comunicación que existen.
44. Monitores: indicar como se realiza en monitores la comunicación y la sincronización (ambos tipos de sincronización) entre procesos. Explicar la diferencia entre los protocolos de sincronización en monitores: signal and wait (S&W) y signal and continue (S&C).
45. PMA y PMS: Explicar el funcionamiento general de PMA y PMS. Indicar y explicar las sentencias que posee para realizar las comunicaciones básicas.
46. RPC y Rendezvous: Explicar las características comunes y las diferencias entre Rendczvous y RPC. Indicar que cosas de la comunicación guardada de rendezvous "conceptual" no se tienen en el rendezvous provisto, por ADA.
47. Librería Pthreads: explicar cómo se maneja la sincronización (ambos tipos de sincronización) en la librería Pthreads, como se relacionan entre ellas y por qué para realizar una de las sincronizaciones necesita de la otra?.
48. Librerías para Pasaje de Mensajes: Explicar las 4 diferentes tipos de SEND en que se clasifican las comunicaciones punto a punto en las lihrerias de Pasaje de Mensajes en general (no se refiere particularmente a MPI). Indicar cuales se corresponden con las herramientas PMA y PMS de la práctica 4.
49. Propiedad a lo sumo una vez (ASV).
50. ¿Alocación de recursos. ¿Es SJF fair? Cómo se resuelve?
51. ¿Qué son RPC y Rendevouz y para qué se usan.
52. ¿Qué se entiende por *no determinismo*? ¿Cómo se aplica este concepto a la ejecución concurrente?
53. Defina comunicación. Explique los mecanismos de comunicación que conozca.
54. Defina sincronización. Explique los mecanismos de sincronización que conozca. 
55. ¿Qué significa el problema de "interferencia" en programación concurrente? ¿Cómo puede evitarse?
56. Defina acciones atómicas condicionales e incondicionales. Ejemplifique.
57. Define propiedad de seguridad y de vida.
58. ¿Qué es una política de scheduling? Relacione con fairness. ¿Qué tipos de fairness conoce? ¿Por qué las propiedades de vida dependen de la política de scheduling?
59. Describa la técnica de passing the baton. ¿Cuál es su utlidad en la resolución de problemas mediante semáforos?
60. Describa en funcionamiento de los monitores como herramienta de sincronización. ¿Cómo se realiza la comunicación y la sincronización por condición entre procesos en esta herramienta.
61. ¿Qué diferencias existen entre las disciplinas de señalización *signal and wait* y *signal and continue*.
62. 
