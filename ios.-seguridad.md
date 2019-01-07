# Acceso seguro a servidores en iOS

<!--
https://developer.apple.com/documentation/foundation/url_loading_system/handling_an_authentication_challenge/performing_manual_server_trust_authentication
https://developer.apple.com/documentation/foundation/url_loading_system/handling_an_authentication_challenge 
---->

En este capítulo veremos las opciones que tenemos para realizar conexiones seguras con servidores remotos desde iOS.

## Seguridad HTTP

En primer lugar vamos a ver cómo acceder a servicios protegidos con seguridad HTTP estándar. En estos casos, deberemos proporcionar en la llamada al servicio las cabeceras de autenticación al servidor, con las credenciales que nos den acceso a las operaciones solicitadas.

Para quienes no estén muy familiarizados con la seguridad HTTP conviene mencionar el funcionamiento del protocolo a grandes rasgos. Cuando realizamos una petición HTTP a un recurso protegido con seguridad básica, el servidor nos devuelve una respuesta indicándonos que necesitamos autentificarnos para acceder. Es entonces cuando el cliente solicita al usuario las credenciales (usuario y contraseña), y entonces se realiza una nueva petición con dichas credenciales incluidas en una cabecera _Authorization_. Si las credenciales son válidas, el servidor nos dará acceso al contenido solicitado.

<!---
<p>Este es el funcionamiento habitual de la autenticación. En el caso del acceso mediante HttpClient que hemos visto anteriormente, el funcionamiento es el mismo, cuando el servidor nos pida autentificarnos
la librerÌa lanzar· una nueva peticiÛn con las credenciales especificadas en el proveedor de credenciales.</p>
--->

Sin embargo, si sabemos de antemano que un recurso va a necesitar autenticación, podemos también autentificarnos de forma preventiva. La autenticación preventiva consiste en mandar las credenciales en la primera petición, antes de que el servidor nos las solicite. Con esto ahorramos una petición, aunque el inconveniente es que podríamos estar mandando las credenciales en casos en los que no resulten necesarias.


<!---
Con HttpClient podemos activar o desactivar la autenticación preventiva con el siguiente mÈtodo:

<source lang="java">client.getParams().setAuthenticationPreemptive(true);</source>
--->

### Autenticación no preventiva

La autenticación en iOS la puede hacer el delegado de la conexión. Cuando enviamos una petición sin credenciales, el sevidor nos responderá que necesita autenticación invocando al método delegado `didReceive challenge` del protocolo `URLSessionDelegate`. En este método, que puede ser a nivel de sesión o de tarea, podremos especificar los datos de la acreditación.

* A nivel de sesión, se invoca ``URLSession:didReceive challenge:completionHandler`` cuando el servidor requiere autenticación. Este método debe usarse para servidores con SSL/TLS, o cuando todas las peticiones que hagamos para una sesión necesiten la misma acreditación.

* A nivel de petición, podemos usar el método ``URLSession:task:didReceive challenge:completionHandler:`` si el servidor requiere autenticación para una tarea en concreto. Esto es necesario si las peticiones de una misma sesión requieren acreditaciones distintas.

Para usar estos métodos con autenticación Basic debemos crear un objeto `URLCredential` a partir de nuestras credenciales (usuario y contraseña). A continuación vemos un ejemplo típico de implementación de una autenticación a nivel de sesión:


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

Podemos observar que comprobamos los fallos previos de autenticación que hemos tenido. Es decir, si con las credenciales que tenemos en el código ha fallado la autenticación será mejor que cancelemos el acceso, ya que si volvemos a intentar acceder con las mismos credenciales vamos a tener el mismo error. En caso de que sea el primer intento, creamos las credenciales (podemos ver que se puede indicar que se guarden de forma persistente para futuros accesos), y las utilizamos para responder al reto de autenticación (`URLAuthenticationChallenge`).

<!---
completionHandler(.useCredential, URLCredential(trust: challenge.protectionSpace.serverTrust)) // TLS?
-->

A nivel de tarea sería exactamente igual, pero usando el siguiente prototipo, esta vez del protocolo `URLSessionTaskDelegate`:

```swift
func urlSession(
    _ session: URLSession,
    task: URLSessionTask,
    didReceive challenge: URLAuthenticationChallenge,
    completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void)
```

Si es necesaria la autenticación y no implementamos el método a nivel de sesión, entonces iOS llama automáticamente al método a nivel de tarea.

El tipo de persistencia (``URLCredential.Persistence``) puede ser:
* ``none``: Las credenciales no se guardan
* ``forSession``: Las credenciales se guardan para esta sesión
* ``permanent``: Las credenciales se guardan en el _keychain_
* ``synchronizable``: Las credenciales se guardan en el _keychain_, y además se comparten entre dispositivos iOS.

### Autenticación preventiva

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

#### Autenticación TLS

<!---- 
https://stackoverflow.com/questions/34073822/how-to-make-a-https-request-to-a-server-in-swift
----->

Si necesitas conectarte a un servidor con seguridad TLS o hacer peticiones más complejas que las vistas anteriormente, actualmente la mejor opción es usar el framework <a href="https://github.com/Alamofire/Alamofire">AlamoFire</a>, que simplifica mucho las operaciones de conexión en iOS. Por motivos pedagógicos, para los siguientes ejercicios no se permite usar este framework sino que deben usarse los métodos nativos que hemos visto anteriormente.
