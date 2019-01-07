# iOS. Servicios Rest

## Parsing y servicios REST en iOS

### Parsing en iOS

En la primera parte de esta sesión veremos cómo parsear la información recibida de un servidor, tanto en JSON como en XML.

#### Parsing de JSON

El parsing de JSON no se incorporó al SDK de iOS hasta la versión 5.0. Anteriormente contábamos con diferentes librerías que podíamos incluir para realizar esta tarea, como \([_JSONKit_](https://github.com/johnezang/JSONKit)\) o \([_JSON-framework_](https://github.com/stig/json-framework/)\). Sin embargo, actualmente podemos trabajar con JSON directamente con las clases de Cocoa Touch sin necesidad de incluir ninguna librería adicional.

Para esto simplemente necesitaremos la clase `JSONSerialization`. A partir de ella obtendremos el contenido del JSON en una jerarquía de objetos. El método `jsonObject` de la clase `JSONSerialization` nos devolverá un diccionario o un array según si el elemento principal del JSON es un objeto o una lista, respectivamente.

```swift
let data: Data // Contenido JSON obtenido de la red, por ejemplo
do {
    let json = try JSONSerialization.jsonObject(with: data, options: [])
   // Hacer algo con la variable json
} catch {
    print (error.localizedDescription)
}
```

A veces la información JSON está en forma de diccionario \(el elemento principal del JSON es un objeto\), y otras organizada como un array \(el elemento principal es una lista\). Vamos a ver un ejemplo cuando el elemento principal es un objeto:

```javascript
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

```javascript
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

**Parsing JSON con MVC**

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

```javascript
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

Puedes encontrar más ejemplos de cómo trabajar con JSON en [este enlace de Apple](https://developer.apple.com/swift/blog/?id=37).

#### Parsing de XML

En el SDK de iOS contamos con la clase `XMLParser` para analizar XML. Con esta librería el análisis se realiza de forma parecida a los parsers SAX de Java. Este es el parser principal incluido en el SDK, aunque también contamos dentro del SDK con `libxml2`, escrito en C, que incluye tanto un parser SAX como DOM. Además encontramos otras librerías que podemos incluir en nuestro proyecto como parsers DOM de XML:

| Parser | URL |  |  |
| :--- | :--- | :--- | :--- |
| TBXML | [http://www.tbxml.co.uk/](http://www.tbxml.co.uk/) |  |  |
| TouchXML | [https://github.com/TouchCode/TouchXML](https://github.com/TouchCode/TouchXML) |  |  |
| KissXML | [http://code.google.com/p/kissxml](http://code.google.com/p/kissxml) |  |  |
| TinyXML | [http://www.grinninglizard.com/tinyxml/](http://www.grinninglizard.com/tinyxml/) |  |  |
| GDataXML | [http://code.google.com/p/gdata-objectivec-client](http://code.google.com/p/gdata-objectivec-client) |  |  |

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

```markup
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

Podemos observar que cada vez que encuentra una etiqueta de apertura obtenemos tanto la etiqueta como sus atributos. Cada vez que se abre un nuevo mensaje se van introduciendo en el objeto de tipo `UAMensaje` los datos que se encuentran en el XML, hasta encontrar la etiqueta de cierre \(en nuestro caso el texto, aunque podríamos tener etiquetas anidadas\).

Para que se ejecute el _parser_ que hemos implementado mediante el delegado deberemos crear un objeto `XMLParser` y proporcionarle dicho delegado \(en el siguiente ejemplo suponemos que nuestro objeto `self` hace de delegado\). El parser se debe inicializar proporcionando el contenido XML a analizar \(encapsulado en un objeto `Data`\):

```swift
let parser = XMLParser(data:self.content)
parser.delegate = self
let result = parser.parse()
```

Tras inicializar el _parser_, lo ejecutamos llamando al método `parse`, que realizará el análisis de forma síncrona, y nos devolverá `true` si todo ha ido bien, o `false` si ha habido algún error al procesar la información. También devolverá `false` si durante el _parsing_ llamamos al método `parser.abortParsing()`.

### Acceso a servicios REST desde iOS

En iOS podemos acceder a servicios REST utilizando las clases para conectar con URLs vistas en anteriores sesiones. Por ejemplo, para hacer una consulta al servidor de OpenWeatherMap podríamos utilizar el siguiente código para iniciar la conexión \(recordemos que este método de conexión es asíncrono\):

```swift
  let url = URL(string: "http://api.openweathermap.org/data/2.5/weather?q=Alicante")!
  let request = URLRequest(url:url)     
  let session = URLSession(configuration:URLSessionConfiguration.default)

  session.dataTask(with: request, completionHandler: { data, response, error in
           // Se recibe la respuesta como se ha visto en el capítulo de red
       }).resume() // En esta línea lanzamos la petición asíncrona
```

Podemos modificar los datos de la petición y de esta forma establecer todos los datos necesarios para la petición al servicio: método HTTP, mensaje a enviar \(como XML o JSON\), y cabeceras \(para indicar el tipo de contenido enviado, o los tipos de representaciones que aceptamos\). Por ejemplo:

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

### Seguridad HTTP

En primer lugar vamos a ver cómo acceder a servicios protegidos con seguridad HTTP estándar. En estos casos, deberemos proporcionar en la llamada al servicio las cabeceras de autenticación al servidor, con las credenciales que nos den acceso a las operaciones solicitadas.

Para quienes no estén muy familiarizados con la seguridad HTTP conviene mencionar el funcionamiento del protocolo a grandes rasgos. Cuando realizamos una petición HTTP a un recurso protegido con seguridad básica, el servidor nos devuelve una respuesta indicándonos que necesitamos autentificarnos para acceder. Es entonces cuando el cliente solicita al usuario las credenciales \(usuario y contraseña\), y entonces se realiza una nueva petición con dichas credenciales incluidas en una cabecera _Authorization_. Si las credenciales son válidas, el servidor nos dará acceso al contenido solicitado.

Sin embargo, si sabemos de antemano que un recurso va a necesitar autenticación, podemos también autentificarnos de forma preventiva. La autenticación preventiva consiste en mandar las credenciales en la primera petición, antes de que el servidor nos las solicite. Con esto ahorramos una petición, aunque el inconveniente es que podríamos estar mandando las credenciales en casos en los que no resulten necesarias.

#### Autenticación no preventiva

La autenticación en iOS la puede hacer el delegado de la conexión. Cuando enviamos una petición sin credenciales, el sevidor nos responderá que necesita autenticación invocando al método delegado `didReceive challenge` del protocolo `URLSessionDelegate`. En este método, que puede ser a nivel de sesión o de tarea, podremos especificar los datos de la acreditación.

* A nivel de sesión, se invoca `URLSession:didReceive challenge:completionHandler` cuando el servidor requiere autenticación. Este método debe usarse para servidores con SSL/TLS, o cuando todas las peticiones que hagamos para una sesión necesiten la misma acreditación.
* A nivel de petición, podemos usar el método `URLSession:task:didReceive challenge:completionHandler:` si el servidor requiere autenticación para una tarea en concreto. Esto es necesario si las peticiones de una misma sesión requieren acreditaciones distintas.

Para usar estos métodos con autenticación Basic debemos crear un objeto `URLCredential` a partir de nuestras credenciales \(usuario y contraseña\). A continuación vemos un ejemplo típico de implementación de una autenticación a nivel de sesión:

```swift
func urlSession(
    _ session: URLSession,
    didReceive challenge: URLAuthenticationChallenge,
    completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void)
{
    print("task-didReceiveChallenge")

    if challenge.previousFailureCount > 0 {
        print("Error: Please check the credential")
        completionHandler(.cancelAuthenticationChallenge, nil)
    }
    else {
        let credential = URLCredential(user:"mi_login", password:"mi_password", persistence: .forSession)
        completionHandler(.useCredential, credential)
    }
}
```

Podemos observar que comprobamos los fallos previos de autenticación que hemos tenido. Es decir, si con las credenciales que tenemos en el código ha fallado la autenticación será mejor que cancelemos el acceso, ya que si volvemos a intentar acceder con las mismos credenciales vamos a tener el mismo error. En caso de que sea el primer intento, creamos las credenciales \(podemos ver que se puede indicar que se guarden de forma persistente para futuros accesos\), y las utilizamos para responder al reto de autenticación \(`URLAuthenticationChallenge`\).

A nivel de tarea sería exactamente igual, pero usando el siguiente prototipo, esta vez del protocolo `URLSessionTaskDelegate`:

```swift
func urlSession(
    _ session: URLSession,
    task: URLSessionTask,
    didReceive challenge: URLAuthenticationChallenge,
    completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void)
```

Si es necesaria la autenticación y no implementamos el método a nivel de sesión, entonces iOS llama automáticamente al método a nivel de tarea.

El tipo de persistencia \(`URLCredential.Persistence`\) puede ser:

* `none`: Las credenciales no se guardan
* `forSession`: Las credenciales se guardan para esta sesión
* `permanent`: Las credenciales se guardan en el _keychain_
* `synchronizable`: Las credenciales se guardan en el _keychain_, y además se comparten entre dispositivos iOS.

#### Autenticación preventiva

Este tipo de autenticación se suele usar para servicios REST, y consiste en enviar las credenciales antes de que las pida el servidor. De este modo nos ahorramos un mensaje de petición y de respuesta, por lo que es más eficiente y ahorramos ancho de banda. Este esquema es recomendable cuando sabemos de antemano que nuestra petición va a necesitar autenticación.

Al igual que en el caso anterior, también podemos hacer la autenticación a nivel de sesión o de tarea. En cualquier caso, deberemos codificar en Base64 la cadena para autenticación. En el caso de autenticación Basic, dados un usuario y una contraseña podemos generar la cadena del siguiente modo:

```swift
let login = "my_login"
let password = "mi_password"
let authStr = login + ":" + password
if let authData = authStr.data(using: .utf8) {
    let authValue = "Basic \(authData.base64EncodedString(options: []))"
    // Ya podemos usar authValue
}
```

En el código podemos ver que primero se genera la cadena de credenciales, concatenando `login:password`. A continuación esta cadena se codifica en base64, y con esto ya podemos añadir una cabecera `Authorization` cuyo valor es la cadena `Basic <credenciales_base64>`. Una vez tengamos la cadena de las credenciales, si queremos añadir la autenticación preventiva a la sesión podemos especificarla en la configuración:

```swift
let config = URLSessionConfiguration.default
config.httpAdditionalHeaders = ["Accept" : "application/json",
                                "Authorization" : authValue]
let session = URLSession(configuration: config)
```

Si en lugar de hacerlo a nivel de sesión lo que queremos es añadir esta autenticación a una petición, tenemos que modificar su request:

```swift
var request = URLRequest(url: URL(string:"http://www.ua.es")!)
request.addValue(authValue, forHTTPHeaderField:"Authorization")
```

**Autenticación TLS**

Si necesitas conectarte a un servidor con seguridad TLS o hacer peticiones más complejas que las vistas anteriormente, actualmente la mejor opción es usar el framework [AlamoFire](https://github.com/Alamofire/Alamofire), que simplifica mucho las operaciones de conexión en iOS. Para los siguientes ejercicios no se permite usar este framework, por motivos pedagógicos deben usarse los métodos nativos que hemos visto anteriormente.

## Ejercicios de servicios REST en iOS

### Weather app \(2 puntos\)

En este primer ejercicio vamos a practicar el parsing de XML y JSON. Para ello haremos una aplicación que nos permita visualizar el tiempo de una ciudad accediendo a la API de [Openweathermap](http://openweathermap.org/api).

Se proporciona una plantilla `Weather` que ya realiza la llamada asíncrona a la API. Según el usuario elija XML o JSON, la respuesta del servidor se recibirá en el formato correspondiente. Crea una nueva conexión y lánzala en el método `search`.

Se pide parsear la respuesta en ambos formatos para poder mostrar la información en pantalla. Sólo se solicitan unos pocos datos, que son la temperatura actual, la humedad, velocidad del viento, el país, y la descripción \(que es un mensaje, como por ejemplo _clear skies_\).

Hay que completar los métodos `parseXML` y `parseJSON` para mostrar la información correspondiente en los outlets del interfaz. Es recomendable comenzar con `parseJSON`, completando el método `init` de la clase `Weather`. Para parsear el XML hace falta añadir al final del `ViewController` los métodos `didStartElement`, `didEndElement` y `foundCharacters`.

Cuando hayas terminado de implementar estos métodos, descarga la imagen del icono que se encuentra en el campo `icon` de la respuesta para mostrarlo en el `UIImageView`, dentro del método `updateView`. Ejemplo de la URL correspondiente al icono `10d`: [http://openweathermap.org/img/w/10d.png](http://openweathermap.org/img/w/10d.png).

