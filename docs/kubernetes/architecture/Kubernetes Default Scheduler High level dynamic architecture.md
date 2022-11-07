# Kubernetes Default Scheduler High level dynamic architecture
## intro
Normalmente no especificamos en qué nodo queremos que se ejecute un Pod.
Esto es la tarea del Scheduler.

Basicamente la tarea del Scheduler es 
- recibir las notificaciones de la creacion de nuevos Pods mediante el mecanismo de observer del API server
- Modifica el manifiesto para asignarle un worker node donde ejecutarse
- Y Vuelve a escribir el manifiesto en el ETCD a traves del API Server.

El Scheduler **no habla para nada con el Kubelet del nodo** 

- El Kubelet es avisado de nuevo por el mecanismo de notificaciones del API Server de que se ha cambiado un recurso ( el Pod en este caso)
- EL Kubelet lee el nuevo recurso y ve que está asignado a su nodo y por tanto crea el nuevo pod y ejecuta sus containers ( hablando con el runtime que sea Docker o el que haya )


La parte complicada del Scheduler es la parte del algoritmo de decision sobre a que nodo asignar cada nuevo Pod.
El algoritmo podría ser tan tonto como hacerlo aleatoriamente o tan listo como utilizar Machine Learning para anticipar los nodos que se pueden crear ... etc etc 

El algoritmo por defecto esta en medio de estos dos.

## Entendiendo el Algoritmo por defecto del Scheduler.

Se pueden identificar dos partes principales:
- Filtrar todos los nodos que son ACEPTABLES para poder tener el Pod
- Puntuar cada uno de esos nodos segun su prioridad y elegir aquel que tenga una mejor puntuacion para asignarle el pod
  - Si varios tienen la misma puntuacion intenta elegir mediante un Round-Robin para poder balancear la asignacion de nodos.

### Fase 1: Encontrando los nodos aceptables

Para encontrar los nodos aceptables el Scheduler aplica a cada nodo una lista de **predicados** que miran distintas cosas como
- Puede el nodo cumplir los `request` del pod?
- Esta el nodo bajo de recursos? (reporting memory/disk presure condition)
- A pedido el Pod ser asignado a un nodo determinado ( por nombre)
- Tiene el nodo una `label` que corresponda con el `node selector`   del Pod?
- Si el Pod pide ser enlazado con un puerto determinado ... está ya ese puerto en uso en el nodo?
- Si el Pod pide cargar un determinado volumen ...
  - Se puede cargar dicho volumen desde el nodo?
  - Hay algun nodo que ya este usando dicho volumen?
- Tolera el Pod los `taints` del nodo? (`taints` and `tolerations`)
- Especifica el Pod alguna afinidad o antiafinidad por algun nodo o por algun otro pod?
  - Basicamente podemos querer que determinados pods no esten en el mismo nodo
  - o Que determinados pods vayan juntos siempre que sea posible ...

TODAS estos predicados deben de pasar para que el nodo pase a la lista de aceptables.

Una vez pasados todos los nodos por todos los predicados el Scheduler tiene una lista de nodos aceptables.
Cualquiera de estos nodos puede ejecutar el pod porque tiene suficientes recursos y cumple los requisitos del pod.

### Fase 2: Elegir el mejor de los aceptables.

Elegir de entre los aceptables no es sencillo porque basicamente no hay un criterio que sea valido para todas las ocasiones.

````ad-example
Imagina que tienes un cluster con dos nodos.
Ambos nodos son aceptables para ejecutar un pod.
Uno de ellos esta ejecutando 10 pods mientras que el otro no tiene ninguno.

Parece sencillo, el segundo nodo es la elección clara porque tiene menos carga, verdad?

O puede que **NO**. Si estas ejecutando en un entorno cloud puede que quieras ejecutar el nuevo pod en el nodo uno ( que tiene los otros 10) de manera que puedas liberar el nodo que no usas y ahorrar pasta.


Como vemos no hay una sola manera de hacer las cosas.
````

Por defecto los pods que pertenecen al mismo [[Kubernetes Service|Service]] o [[Kubernetes ReplicaSet|ReplicaSet]] se intentan mandar a distintos nodos, aunque esto no está garantizado.

Se pueden definiri reglas de affinity o anti-affinity para configurar todo esto.
## Usando multiples Schedulers.

Se pueden lanzar multiples Schedulers en el cluster.
Luego se elige que Scheduler maneja cada pod con la propiedad del pod: `spec.schedulerName`

Los que no tengan nada o tengan `default-scheduler` como valor, serán tratados por el Scheduler por defecto.