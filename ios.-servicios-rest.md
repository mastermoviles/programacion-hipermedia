# Parsing y servicios REST en iOS

## Parsing en iOS

En la primera parte de esta sesión veremos cómo parsear la información recibida de un servidor, tanto en JSON como en XML.

### Parsing de JSON

El parsing de JSON no se incorporó al SDK de iOS hasta la versión 5.0. Anteriormente contábamos con diferentes librerías que podíamos incluir para realizar esta tarea, como  (<a href="https://github.com/johnezang/JSONKit">*JSONKit*</a>) o (<a href="https://github.com/stig/json-framework/">*JSON-framework* </a>). Sin embargo, actualmente podemos trabajar con JSON directamente con las clases de Cocoa Touch sin necesidad de incluir ninguna librería adicional.

Para esto simplemente necesitaremos la clase `JSONSerialization`. A partir de ella obtendremos el contenido del JSON en una jerarquía de objetos. El método `jsonObject` de la clase `JSONSerialization` nos devolverá un diccionario o un array según si el elemento principal del JSON es un objeto o una lista, respectivamente.

<!--- https://developer.apple.com/swift/blog/?id=37 -->


<!--let response = try JSONSerialization.jsonObject(with: data, options:JSONSerialization.ReadingOptions(rawValue:0)) as? [String: Any]-->

```swift
let data: Data // Contenido JSON obtenido de la red, por ejemplo
do {
    let json = try JSONSerialization.jsonObject(with: data, options: [])
   // Hacer algo con la variable json
} catch {
    print (error.localizedDescription)
}
```

A veces la información JSON está en forma de diccionario (el elemento principal del JSON es un objeto), y otras organizada como un array (el elemento principal es una lista).  Vamos a ver un ejemplo cuando el elemento principal es un objeto:

```json
{
  "someKey": 42.0,
  "anotherKey": {
    "someNestedKey": true
  }
}
```

En este caso, nuestro código podría ser el siguiente:

```swift
if let dictionary = json as? [String: Any] {
  if let number = dictionary["someKey"] as? Double {
    // Procesamos el valor number
  }
  if let nestedDictionary = dictionary["anotherKey"] as? [String: Any] {
    // Accedemos al diccionario anotherKey para hacer algo con sus valores
  }
  for (key, value) in dictionary {
		// Si quisiéramos acceder a todos los pares clave/valor del diccionario raíz
  }
}
```

Si en su lugar el elemento principal del JSON fuera un array:

```json
[
  "hello", 3, true
]
```

Podríamos procesarlo del siguiente modo:

```swift
if let array = json as? [Any] {
	for object in array {
		// Acceder a todos los objetos del array
	}
}
```

El objeto `JSONSerialization` también nos permite realizar la transformación en el sentido inverso, permitiendo transformar una jerarquía de objetos `Array` y `Dictionary` en una representación JSON. Para eso contaremos con el método `jsonObject`:

```swift
var dict = ["someKey": 42.0, "anotherKey": "prueba"] as [String :Any]

do {
        let data = try JSONSerialization.data(withJSONObject: dict, options: .prettyPrinted)
        if let str = String(data: data, encoding: .utf8) { // Para imprimirlo por pantalla
            print (str)
        }

} catch {
        print (error.localizedDescription)
}
```


#### Parsing JSON con MVC

Hemos visto la forma básica de serializar o deserializar datos en JSON. Sin embargo, nuestras apps suelen seguir el patrón de diseño MVC, por lo que normalmente es más limpio y conveniente convertir directamente los objetos JSON al formato de nuestro modelo. Vamos a ver un ejemplo de cómo se haría la deserialización. Dado el siguiente modelo:

```swift
class Restaurant
{
  let name: String
  let location: (latitude: Double, longitude: Double)
  let meals: [String]
}
```

Y el siguiente documento JSON:

```json
{
	"name": "Caffè Macs",
	"coordinates": {
		"lat": 37.330576,
		"lng": -122.029739
	},
	"meals": ["breakfast", "lunch", "dinner"]
}
```

podemos crear un constructor para nuestro modelo a partir de un diccionario:

```swift
init?(json: [String: Any]) {

    guard let name = json["name"] as? String,
        let coordinatesJSON = json["coordinates"] as? [String: Double],
        let latitude = coordinatesJSON["lat"],
        let longitude = coordinatesJSON["lng"],
        let mealsJSON = json["meals"] as? [String]
    else {
            return nil
    }

    self.name = name
    self.location = (latitude, longitude)
    self.meals = mealsJSON
}
```

Y la llamada a nuestro constructor sería:

```swift
let data: Data // Contenido JSON obtenido de la red, por ejemplo
if let jsonData = try? JSONSerialization.jsonObject(with: data, options: []) as! [String:Any] {
    let r = Restaurant(json:jsonData)
}
```

Puedes encontrar más ejemplos de cómo trabajar con JSON en <a href="https://developer.apple.com/swift/blog/?id=37">este enlace de Apple</a>.

#### Parsing JSON con métodos Codable

<!--- https://hackernoon.com/everything-about-codable-in-swift-4-97d0e18a2999 --->

Swift4 permite serializar clases, registros (_struct_) o tipos enumerados (_enum_) para leer o escribir en JSON. Para codificar o decodificar un tipo personalizado podemos usar la opción `Encodable`, `Decodable`o `Codable`, que permiten tanto codificación como decodificación JSON como puede verse en el siguiente ejemplo:

```swift
struct Restaurant: Codable
{
    let name: String
    let latitude: Double
    let longitude: Double
    let meals: [String]
}

// Ejemplo codificación

let restaurant = Restaurant(name: "Hibiscus", latitude: 10, longitude: 10, meals: ["Mediterranean", "Arroces"])
let encodedData = try? JSONEncoder().encode(restaurant)

// Ejemplo decodificación recibiendo un restaurante en JSON desde servidor en la variable data

if let jsonData = jsonString.data(using: .utf8) {
    let restaurant = try? JSONDecoder().decode(Restaurant.self, from: jsonData)
}
```

Aunque esto suele ser suficiente para la mayoría de casos, a veces podemos querer omitir algunas variables en el proceso de serialización, o poner nombres a nuestras variables que no coinciden exactamente con los del JSON. Para resolver estas dos cuestiones, swift introdujo las `CodingKeys` que podemos ver en el siguiente ejemplo:

```swift
struct Photo: Codable
{
    
    // Esta propiedad no se incluye en CodingKeys, por lo que no se codificará o decodificará
    var format: String = "png"
    
    // Propiedades a codificar/decodificar junto con sus nombres alternativos (en el caso de title y url, que en JSON vendría como name y link)
    enum CodingKeys: String, CodingKey
    {
        case title = "name"
        case url = "link"
        case isSample
        case metaData
        case type
        case size
    }
}
```

Puedes encontrar más información sobre `Codable` en [este enlace](https://hackernoon.com/everything-about-codable-in-swift-4-97d0e18a2999).

### Parsing de XML

En el SDK de iOS contamos con la clase `XMLParser` para analizar XML. Con esta librería el análisis se realiza de forma parecida a los parsers SAX de Java. Este es el parser principal incluido en el SDK, aunque también contamos dentro del SDK con `libxml2`, escrito en C, que incluye tanto un parser SAX como DOM. Además encontramos otras librerías que podemos incluir en nuestro proyecto como parsers DOM de XML:

<table>
		<tr><th>Parser</th><th>URL</th><th></th><th></th></tr>
		<tr><td>TBXML</td><td colspan="3"><a href="http://www.tbxml.co.uk/">http://www.tbxml.co.uk/</a></td></tr>
		<tr><td>TouchXML</td><td colspan="3"><a href="https://github.com/TouchCode/TouchXML">https://github.com/TouchCode/TouchXML</a></td></tr>
		<tr><td>KissXML</td><td colspan="3"><a href="http://code.google.com/p/kissxml">http://code.google.com/p/kissxml</a></td></tr>
		<tr><td>TinyXML</td><td colspan="3"><a href="http://www.grinninglizard.com/tinyxml/">http://www.grinninglizard.com/tinyxml/</a></td></tr>
		<tr><td>GDataXML</td><td colspan="3"><a href="http://code.google.com/p/gdata-objectivec-client">http://code.google.com/p/gdata-objectivec-client</a></td></tr>
</table>

Nos vamos a centrar en el estudio de `XMLParser` por ser el parser principal incluido en la API de Cocoa Touch.

Para implementar un parser con esta librería deberemos crear una clase que adopte el protocolo `XMLParserDelegate`. Este define, entre otros, los siguientes métodos:

```swift
func parser(_ parser: XMLParser,
    didStartElement: String,
       namespaceURI: String?,
      qualifiedName: String?,
         attributes: [String : String] = [:])

func parser(_ parser: XMLParser,
     didEndElement: String,
      namespaceURI: String?,
     qualifiedName: String?)

func parser(_ parser: XMLParser,
   foundCharacters: String)
```

Podemos observar que nos informa de tres tipos de eventos: `didStartElement`, `didEndElement` y `foundCharacters`. El análisis del XML será secuencial, es decir, el parser irá leyendo el documento y nos irá notificando los elementos que encuentre. Cuando se abra una etiqueta, llamará al método `didStartElement` de nuestro parser, cuando encuentre texto llamará a `foundCharacters`, y cuando se cierra la etiqueta llamará a `didEndElement`. Será responsabilidad nuestra implementar de forma correcta estos tres eventos, y guardar la información de estado que necesitemos durante el análisis.

Por ejemplo, imaginemos un documento XML sencillo como el siguiente:

```xml
<![CDATA[<mensajes>
    <mensaje usuario="pepe">Hola, ¿qué tal?</mensaje>
    <mensaje usuario="ana">Fetén</mensaje>
</mensajes>]]>
```

Podemos analizarlo mediante un parser `XMLParser` como el siguiente:

```swift

// Modelo:
class UAMensaje
{
    var usuario : String?
    var texto: String?
}

// Código en nuestro controlador:

    var listaMensajes = [Any]()
    var currentMessage : UAMensaje?

    func parserDidStartDocument(_ parser: XMLParser) { // Se invoca al comenzar el parsing
    }

    func parserDidEndDocument(_ parser: XMLParser) { // Se invoca cuando hemos terminado el parsing
    }

    func parser(_ parser: XMLParser,
                didStartElement elementName: String,
                namespaceURI: String?,
                qualifiedName: String?,
                attributes attributeDict: [String : String] = [:])
    {
        if elementName.lowercased() == "mensajes" {
            // Ok, no hacer nada
        }
        else if elementName.lowercased() == "mensaje" {
            self.currentMessage = UAMensaje()
            self.currentMessage!.usuario = attributeDict["usuario"]
        }
        else { // Si no puede haber etiquetas distintas a mensaje o mensajes
            parser.abortParsing()
        }
    }

    func parser(_ parser: XMLParser,
                didEndElement elementName: String,
                namespaceURI: String?,
                qualifiedName: String?)
    {
        if elementName.lowercased() == "mensaje" {
            if let message = self.currentMessage {
                self.listaMensajes.append(message)
            }
        }
    }

    func parser(_ parser: XMLParser,
                foundCharacters characters: String)
    {
        // Quitamos espacios en blanco
        let trimmedString = characters.trimmingCharacters(in: .whitespacesAndNewlines)

        if trimmedString != "" {
            self.currentMessage?.texto = trimmedString
        }
    }
```

Podemos observar que cada vez que encuentra una etiqueta de apertura obtenemos tanto la etiqueta como sus atributos. Cada vez que se abre un nuevo mensaje se van introduciendo en el objeto de tipo `UAMensaje` los datos que se encuentran en el XML, hasta encontrar la etiqueta de cierre (en nuestro caso el texto, aunque podríamos tener etiquetas anidadas).

Para que se ejecute el _parser_ que hemos implementado mediante el delegado deberemos crear un objeto `XMLParser` y proporcionarle dicho delegado (en el siguiente ejemplo suponemos que nuestro objeto `self` hace de delegado). El parser se debe inicializar proporcionando el contenido XML a analizar (encapsulado en un objeto `Data`):

```swift
let parser = XMLParser(data:self.content)
parser.delegate = self
let result = parser.parse()
```

Tras inicializar el _parser_, lo ejecutamos llamando al método `parse`, que realizará el análisis de forma síncrona, y nos devolverá `true` si todo ha ido bien, o `false` si ha habido algún error al procesar la información. También devolverá `false` si durante el _parsing_ llamamos al método `parser.abortParsing()`.


## Acceso a servicios REST desde iOS

En iOS podemos acceder a servicios REST utilizando las clases para conectar con URLs vistas en anteriores sesiones. Por ejemplo, para hacer una consulta al servidor de OpenWeatherMap podríamos utilizar el siguiente código para iniciar la conexión (recordemos que este método de conexión es asíncrono):

```swift
  let url = URL(string: "http://api.openweathermap.org/data/2.5/weather?q=Alicante,ES")!
  let request = URLRequest(url:url)     
  let session = URLSession(configuration:URLSessionConfiguration.default)

  session.dataTask(with: request, completionHandler: { data, response, error in
           // Se recibe la respuesta como se ha visto en el capítulo de red
       }).resume() // En esta línea lanzamos la petición asíncrona
```


Podemos modificar los datos de la petición y de esta forma establecer todos los datos necesarios para la petición al servicio: método HTTP, mensaje a enviar (como XML o JSON), y cabeceras (para indicar el tipo de contenido enviado, o los tipos de representaciones que aceptamos). Por ejemplo:

```swift
  let url = URL(string: "http://localhost/videoclub/api/v1/catalog")!
  let datosPelicula = ... // Componer mensaje JSON con datos de la peli a crear
  var request = URLRequest(url:url)

  request.httpMethod = "POST"
  request.httpBody = datosPelicula
  request.setValue("application/json", forHTTPHeaderField: "Accept")
  request.setValue("application/json", forHTTPHeaderField: "Content-Type")  
```

Podemos ver que en la petición POST hemos establecido todos los datos necesarios. Por un lado su bloque de contenido, con los datos del recurso que queremos añadir en la representación que consideremos adecuada. En este caso suponemos que utilizamos XML como representación. En tal caso hay que avisar de que el contenido lo enviamos con este formato, mediante la cabecera `Content-Type`, y de que la respuesta también queremos obtenerla en XML, mediante la cabecera `Accept`.

# Ejercicios de servicios REST en iOS

## Weather app (2 puntos)

En este ejercicio vamos a practicar el parsing de XML y JSON. Para ello haremos una aplicación que nos permita visualizar el tiempo de una ciudad accediendo a la API de <a href="http://openweathermap.org/api">Openweathermap</a>.

Se proporciona una plantilla `Weather` que ya realiza la llamada asíncrona a la API. Según el usuario elija XML o JSON, la respuesta del servidor se recibirá en el formato correspondiente. Crea una nueva conexión y lánzala en el método `search`.

Se pide parsear la respuesta en ambos formatos para poder mostrar la información en pantalla. Sólo se solicitan unos pocos datos, que son la temperatura actual, la humedad, velocidad del viento, el país, y la descripción (que es un mensaje, como por ejemplo _clear skies_).

Hay que completar los métodos `parseXML` y `parseJSON` para mostrar la información correspondiente en los outlets del interfaz. Es recomendable comenzar con `parseJSON`, completando el método `init` de la clase `Weather`. Para parsear el XML hace falta añadir al final del `ViewController` los métodos `didStartElement`, `didEndElement` y `foundCharacters`.

Cuando hayas terminado de implementar estos métodos, descarga la imagen del icono que se encuentra en el campo `icon` de la respuesta para mostrarlo en el `UIImageView`, dentro del método `updateView`. Ejemplo de la URL correspondiente al icono `10d`: http://openweathermap.org/img/w/10d.png.
