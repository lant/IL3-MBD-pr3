# Guía de resolución de la práctica 2

Este documento es una guía sobre como podría ser la resolución del ejercicio 2 del primer módulo. 

Para empezar, recordemos la descripción de la tarea: 

> ...
> La empresa está empezando a tener un cierto éxito inesperado gracias a una muy buena campaña de marketing y, de un día para otro, el CTO os pide que diseñéis un sistema de bases de datos no relacional NoSQL a partir de 0 con las características siguientes:

> Capacidad de almacenaje big data.
> Debe funcionar en un entorno cloud. El cloud provider que utiliza la empresa es de los baratos y su red falla mucho.
> Dado que los proyectos van a ser temporales, es necesario que la base de datos solo utilice los servidores que sean estrictamente necesarios. Si bastan 5 servidores, no hay que utilizar 6. Dependiendo del número de proyectos que tenga la empresa, este número irá subiendo y bajando.
> Algunos proyectos durarán días, otros, años.
> La base de datos debe ser capaz de insertar muchos datos.
> Los datos provienen de sensores, por lo que no son estrictamente críticos.
> Los datos tendrán que estar disponibles para el equipo de data scientists que los analizarán. En principio pueden realizar cualquier tipo de consulta.
> En caso de error interno en la base de datos, consideramos prioritario que los datos se guarden.
> Existe la posibilidad de que los datos lleguen con retraso (días incluso).
> El CTO os comenta que no podéis utilizar ningún tipo de base de datos NoSQL ya existente, ya que justamente el aspecto distintivo de la empresa, y en lo que se ha basado parte de la campaña de marketing, es que se usará un producto propio. Se pueden utilizar sistemas auxiliares (colas, sistemas de coordinación...).

## Análisis del problema

Dado el enunciado, qué podemos deducir acerca de los requerimientos de la DB? y qué implica si consideramos el teorema CAP. 

Uno de los requisitos es que la base de datos sea distribuida y que escale horizontalmente. Así pues si la base de datos es distribuida habrá que aplicar el teorema CAP. Recordemos que la P significa "Partition Tolerance", que es cómo la base de datos va a actuar en caso de partición entre sus nodos. 

Como muy bien se ha puesto en todas las prácticas tenemos que decidir entre: 

* AP. Potenciar la Availability: Significa que la base de datos facilita el hecho que aunque haya particiones continua operando como si no estuvieran. Más tarde, cuando las particiones de red estén solucionadas ya veremos como solucionamos los conflictos (si los hay) 
* CP. Potenciar la Consistency: Significa que la base de datos prefiere asegurar que los datos son correctos. Así pues, si hay una partición entre dos replicas puede desautorizar o prohibir la actualización de estos valores. Esto puede causar un impacto en los usuarios finales. 

Ninguna es perfecta, ni una es mejor que la otra. En este caso, tocará elegir. 

El enunciado pone explícitamente: 

> En caso de error interno en la base de datos, consideramos prioritario que los datos se guarden.

Así que preferimos que los datos lleguen a la base de datos. En caso de que haya conflictos ya veremos como lo solucionamos más tarde. 

Es importante destacar que si alguien cree que la base de datos debería ser CP no es ningún problema. En este caso no sería exactamente 100% lo que se pide en el enunciado, pero no por esto es un gran problema si el diseño tiene sentido. 

En una base de datos CP, para este caso podría darse el caso que: 

* Haya una partición entre dos nodos. 
* Un sensor (que en principio es un proceso muy simple) envía datos a uno de los nodos que almacena estos datos en concreto. 
* Si la base de datos es CP podría ser que decidiera que como hay una partición en estos nodos podría ser que hubiera un conflicto, así pues el sistema decide no guardar los datos y notificar al cliente. 
* El cliente (el sensor) tendría que almacenar estos datos para volver a enviarlos mas tarde o bien descartarlos. 

Esto no es factible en el mundo de los sensores, ya que normalmente son sistemas muy simples que no tienen una gran cantidad de memoria. Si toman muchas muestras por segundo sin poder enviarlas podría causar un colapso o podría darse la situación en que empieza a descartar datos. Justamente es lo contrario de lo que pide el enunciado. 

Aún así, quisiera insistir en que esto no es motivo para suspender la práctica si el resto del diseño es bueno.

## Casos de uso para los usuarios

Si leemos atentamente el enunciado vamos a ver data scientists tendrían que se capaces de hacer queries y trabajar sobre los datos. 

Podríamos considerar dos escenarios: 

* Vamos a permitir hacer queries complejas: Algo parecido a SQL o queries como las que permite MongoDB y permitir el acceso a sistemas de procesamiento de datos (Spark, Hadoop, ...) 
* Sólo se pueden realizar exploraciones con todos los datos. 

Las dos están bien, pero la primera implica una complejidad adicional. 

Lo que debe estar claro es que si hay que hacer un procesamiento de datos debemos permitir que los sistemas de procesamiento de datos tengan acceso a los datos. Para ello se puede diseñar una funcionalidad de exportar los datos a S3, HDFS, o incluso permitir que la base de datos tenga compatibilidad con estos sistemas, o sea, que Spark pueda leer directamente de la base de datos para hacer el proceso. 

Por ejemplo, Cassandra ofrece un `Connector` para que Spark use directamente las tablas de Cassandra para crear los RDDs: [Datastax Spark Connector](https://github.com/datastax/spark-cassandra-connector)


## Diseño del sistema. 

Lo que se pretende en esta práctica que vayan saliendo dudas cuando se hace la toma de decisiones y ver como las decisiones tomadas van influyendo en el diseño del sistema. Como se ve en el temario, el uso de algunas técnicas para solucionar un problema concreto puede llevar a nuevos problemas que también hay que solucionar. 

Un ejemplo claro es usar la replicación (solución) para solucionar la Availability (problema). Cuando más copias haya mejor, pero esto significa que cuando haya una actualización de los datos estos, tengan que ser actualizados en todas las copias, para solucionar esto ... He aquí la dificultad de la práctica. Hay que hacer un buen razonamiento de los pros/cons y ver como se desarrollan los problemas y soluciones. 

Dado el enunciado una posibilidad de diseño sería tener, en lineas generales: 

* Una arquitectura P2P. El código va a ser más complicado, pero también más tolerante a fallos. 
* Replicación de los datos configurable. Dado que algunos proyectos pueden ser muy grandes puede ser interesante poder modificar la replicación a valores pequeños (o incluso no usar replicación para algunos proyectos) 
* Guardar los datos en un formato columnar. El tipo de datos que se guarda (tráfico que viene de sensores) se conoce normalmente como `time series`. Los formatos columnares son muy populares para almacenar este tipo de datos. Este punto lo pongo como ampliación, no era necesario para la práctica, ya que no está en la teoría. Lo vais a tratar en otros módulos del máster. 

Dicho esto, la base de datos que proponemos es del tipo BASE (recordemos, Basically Available, Soft State & Eventual Consistency). Más aún, teniendo en cuenta que algunos de los datos pueden llegar fuera de orden en el sistema. 

### Particionado de los datos

Va a ser básico poder particionar los datos, vamos pues, a utilizar Sharding. Cada nodo, o grupo de nodos se va a encargar de almacenar unos datos concretos. Para ello, hay que seleccionar una buena llave de partición, más aún cuando estamos hablando de datos que provienen de sensores. 

Os invito a echar una ojeada a los montones de información que hay disponible en Internet acerca del tema. Concretamente [este artículo de documentación de Google BigQuery](https://cloud.google.com/bigtable/docs/schema-design-time-series) lo explica muy bien. Se trataría pues de decidir la llave de particionado que: 

* No tenga acumulación de datos por tiempo. 
* Reparta la carga entre los nodos de forma ordenada. 

Un factor a tener en cuenta es que el enunciado especifica que hay que ser capaz de añadir o quitar nodos al sistema según convenga. Si el sharding es manual o bien no se puede cambiar esto es un problema muy grande, ya que al añadir (o quitar) nodos habrá que reparticionar todos los datos, causando así un paro en el sistema. Cosa que sería inaceptable. Así pues, una solución sería adoptar un algoritmo de Consistent Hashing para evitar que haya grandes reparticionados al modificar el número de nodos que forman parte del sistema. 

### Replicación

Como hemos dicho los datos van a estar replicados. Esto tiene unas claras ventajas: 

* Mejor tolerancia a fallos
* Mejor availability
* Mejoramos la velocidad de acceso a los datos por los clientes, ya que los nodos pueden estar repartidos por distintos datacenters. 

Y también tiene sus problemas. Concretamente en el sistema que estamos diseñando se habla de que no vamos a parar si hay particiones de red. Así pues, si hay una partición los nodos van a seguir aceptando escrituras. Una vez la red esté restaurada entre los nodos tendremos que ver como juntamos los datos. 

Dado que son datos de sensores, que habrá muchos y que probablemente no sean críticos vamos a optar por una solución con mecanismos de reconciliación, no vamos a entrar en algoritmos de consenso. Una propuesta que podría tener sentido es tener un proceso de background en la base de datos que implemente 'Asynchronous Repair', esto significa que va buscando diferencias entre las replicas de los nodos y los va solucionando. 

Habría también la posibilidad de ejecutar las técnicas de reparación cuando se efectúan las lecturas, pero si el sistema tiene pocas lecturas; como en este caso, ya que en principio las únicas lecturas las hacen los data scientists cuando quieren ejecutar un experimento, las resoluciones podrían acumularse y hacer el proceso de lectura mucho más lento.

