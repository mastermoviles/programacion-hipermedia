# iOS. Proyecto

El proyecto consistirá en hacer una aplicación para iOS que gestione el videoclub que se ha implementado en sesiones anteriores. Partiremos del código de la aplicación `Pelis` que hicimos en la sesión anterior.

## Listado

Vamos a cambiar en el `MasterViewController` la carga de películas. Esta vez haremos una petición GET al servidor _gbrain.dlsi.ua.es_, que gestiona los servicios del videoclub.

Para empezar, añade al proyecto la clase `Connection` que hemos implementado en el ejercicio anterior \(si no lo habías hecho anteriormente en el ejercicio de _pelis_\).

Prepara `MasterViewController` para usar esta clase, añadiendo el protocolo `ConnectionDelegate` e implementando los siguientes métodos:

```swift
func connectionSucceed(_ connection: Connection, with data: Data)
func connectionFailed(_ connection: Connection, with error: String)
```

Implementa el código para hacer la petición del listado de películas desde el método `createPelis`. Recuerda que es un método `GET`, indícalo en el `HTTPMethod`. Si tras recibir los datos no se muestra nada en la tabla, probablemente debas añadir la siguiente instrucción:

```swift
self.tableView.reloadData()
```

Los datos JSON de las películas están almacenados en un array. Por cada elemento del array crea una nueva película mediante un constructor de la clase `Pelicula` que reciba un diccionario con el contenido de la posición correspondiente de dicho array. Añade el campo `identif` \(de tipo _Int?_\) a la clase `Peli` y guárdalo cuando leas el valor de `id` del JSON. Esto te hará falta para implementar el resto de opciones.

## Alquilar / devolver

En la vista `DetailViewController` implementa la opción de alquiler mostrando un botón \(en cualquier parte visible\) que indique el estado actual con el texto _Alquilada_ o _Libre_, y que pueda pulsarse para cambiar su estado.

Si la conexión ha sido válida y se ha alquilado o devuelto la película, se debe actualizar su campo `rented` y el título del botón en el interfaz.

Esta petición requiere autenticación Basic. Puedes hacerla de tipo preventivo y a nivel de tarea, añadiendo un método adicional a la clase `Connection` que acepte usuario y contraseña para hacer la petición:

```swift
func startConnection(_ session:URLSession, with request:URLRequest, login:String, password:String)
```

Recuerda que en _gbrain_ esto sólo funcionará para las películas del final del listado. En las del principio \(las añadidas inicialmente por los profesores\) siempre fallará la autenticación por falta de permisos.

## Añadir

Vamos a implementar la opción para añadir una película. Para esto, puedes descomentar el código de `addButton` en `viewDidLoad`.

En el método `insertNewObject` tendremos que mostrar una vista para pedir los datos de la película a añadir. Para ello se proporciona en los materiales la clase `PelisTableViewController`. El método quedaría de la siguiente forma:

```swift
func insertNewObject(_ sender: Any) {
    let pelisAdd = PelisTableViewController()
    pelisAdd.delegate = self
    self.navigationController?.pushViewController(pelisAdd, animated: true)
}
```

Hace falta crear un nuevo método `init` \(sin parámetros\) en la clase `Pelicula` para inicializar los datos de una película vacía. Las nuevas películas estarán libres por defecto.

En `PelisTableViewController` debes implementar el método `save` que se llama cuando el usuario pulsa el botón para guardar la película. También debes modificar `connectionSucceed` para actualizar el identificador de la película que se ha recibido, antes de devolverla al delegado mediante el método `saved`.

Este método `saved` hay que implementarlo en `MasterViewController` para actualizar el array local de películas sin tener que volver a hacer una llamada al servidor.

## Eliminar

Descomenta el código de edición de las tablas en `MasterViewController`, tanto en el método `viewDidLoad` como `commit editingStyle`, y devolviendo `true` en `canEditRowAtIndexPath`.

Haz los cambios correspondientes para que cuando se elimine una película en la tabla también se haga en el servidor, de modo que si falla en el servidor no se llegue a borrar en la tabla.

Al tener dos tipos de conexiones en el mismo controlador, en el método `connectionSucceed` necesitarás identificar cuál de ellas es la que ha hecho la petición. Para ello, lo más sencillo es crear una nueva variable en la clase `Connection` que almacene el nombre de la conexión:

```swift
var name : String?
```

Añadimos un nuevo método `init` en `Connection`:

```swift
init(name:String, delegate:ConnectionDelegate) {
    self.delegate = delegate
    self.name = name
}
```

Ahora, podemos crear una conexión para borrar, por ejemplo:

```swift
let connectionDelete = Connection(name : "delete", delegate : self)
```

Y podremos saber desde qué conexión se ha recibido la respuesta en el método `connectionSucceed`:

```swift
func connectionSucceed(_ connection: Connection, with data: Data) {
       if connection.name == "delete" {
          // Gestionar borrado
       }
       else {
         // Gestionar otra conexión
       }
}
```

## Editar

Añade un botón `Edit` en `DetailViewController` a la derecha de la barra de navegación para editar una película. Cuando el usuario lo pulse, deberá poder modificar los datos de la película usando el mismo controlador que para añadirla \(`PelisTableViewController`\). Ojo, tendrás que hacer una copia de la película a modificar por si el usuario cancela los cambios volviendo atrás sin guardarla.

En esta opción, en lugar de mostrar en la barra de navegación _Add movie_, deberá indicarse el nombre de la película, y cuando se guarde deberá actualizarse tanto en el listado local como en el servidor.

Para actualizar la tabla necesitarás acceder al `MasterViewController` desde `DetailViewController`. Puedes hacerlo del siguiente modo:

```swift
  let parentController =  self.splitViewController!.viewControllers[0] as! UINavigationController
  let masterController = parentController.viewControllers[0] as! MasterViewController
  masterController.tableView.reloadData()
```

## Apartado opcional

* Añade un formulario para pedir usuario y contraseña la primera vez que se necesite. Se puede usar un `UIAlertController`, y guardar el usuario y la contraseña en `NSUserDefaults.standardUserDefaults()`. En realidad, la información sensible como las contraseñas debería guardarse en el _KeyChain_, pero esto se verá más adelante en la asignatura de persistencia.

