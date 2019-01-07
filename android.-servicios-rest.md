# Android. Servicios Rest

## Parsing y servicios REST en Android

### Servicios Rest

Como ya hemos visto antes, en Android para descargar el contenido desde una URL podemos utilizar la clase `HttpURLConnection`. Por defecto al realizar una conexión, si no indicamos nada más, se realizará por `GET`. Si llamamos al método `setDoOutput(true)` la petición se realizará por `POST`. Y para realizar una petición de otro tipo, como `PUT` o `DELETE`, tenemos que indicarlo mediante el método `setRequestMethod`.

A continuación se repasa la gestión de los códigos de estado, del uso de cabeceras y por ultimo se incluyen ejemplos de todos los tipos de peticiones.

#### Códigos de estado

Como ya vimos en la sesión anterior, a partir del método `getResponseCode` de la clase `HttpURLConnection` podemos obtener el **códigos de estado** que se envía en la cabecera de la respuesta. Si el codigo es igual a 200 indicará que la petición es correcta y que se puede descargar el contendio. En caso de que haya algún error devolverá un valor distinto a 200 correspondiente al código del error \(Podéis consultar la lista completa de códigos en [https://en.wikipedia.org/wiki/List\_of\_HTTP\_status\_codes](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes)\). Los códigos de error usados dependerán de la API, no es obligatorio usarlos todos, y algunas usarán solo algunos de ellos para indicar un error genérico. Lo que sí que es estándar es devolver el código 200 si la petición es correcta. Para comprobar esto podemos hacer:

```java
if( http.getResponseCode() == HttpURLConnection.HTTP_OK ) {
    // Correcto! Ya podemos descargar el cotenido!
}
```

#### Cabeceras

Para añadir cabeceras a una petición usamos el método `setRequestProperty`.

Por ejemplo para indicar el _user-agent_, el formato de la codificación esperada y el formato en texto plano de los datos enviados usaríamos:

```java
http.setRequestProperty("User-Agent", "...");
http.setRequestProperty("Accept-Charset", "UTF-8");
http.setRequestProperty("Content-Type", "text/plain; charset=utf-8");
```

Para indicar que el formato de los datos enviados y de la respuesta esperada es JSON tenemos que añadir:

```java
http.setRequestProperty("Content-Type", "application/json");
http.setRequestProperty("Accept", "application/json");
```

Para indicar que vamos a usar XML pondríamos:

```java
http.setRequestProperty("Content-type", "application/xml");
http.setRequestProperty("Accept", "application/xml");
```

O para desactivar la compresión Gzip de la comunicación:

```java
http.setRequestProperty("Accept-Encoding", "identity");
```

Y para consultar las cabeceras de la respuesta podemos usar el método `getHeaderFields` que devolverá toda la lista de cabeceras:

```java
for (Map.Entry<String, List<String>> k : http.getHeaderFields().entrySet()) {
    for (String v : k.getValue()){
        Log.d("Headers", k.getKey() + ":" + v);
    }
}
```

O para consultar una cabecera específica podemos usar:

```java
String type = http.getHeaderField("Content-type");
```

#### Petición GET

Por ejemplo para realizar una petición tipo GET a una URI y obtener su contenido tendríamos que hacer:

```java
public String peticionGET( String strUrl )
{
    HttpURLConnection http = null;
    String content = null;
    try {
        URL url = new URL( strUrl );
        http = (HttpURLConnection)url.openConnection();
        http.setRequestProperty("Content-Type", "application/json");
        http.setRequestProperty("Accept", "application/json");

        if( http.getResponseCode() == HttpURLConnection.HTTP_OK ) {
            StringBuilder sb = new StringBuilder();
            BufferedReader reader = new BufferedReader(
                    new InputStreamReader( http.getInputStream() ));
            String line;
            while ((line = reader.readLine()) != null) {
                sb.append(line);
            }
            content = sb.toString();
            reader.close();
        }
    }
    catch(Exception e) {
        e.printStackTrace();
    }
    finally {
        if( http != null ) http.disconnect();
    }
    return content;
}
```

Este método recibe como entrada una cadena con la dirección URL a la cual se quiere realizar la peticion GET y devuelve como respuesta el contenido de la misma. En caso de que hubiera algún error devolvería _null_.

En esta función también hemos especificado el tipo MIME de los datos de la peticion \(`Content-Type`\) y el tipo de representación que queremos obtener como respuesta \(mediante la cabecera `Accept`\).

#### Petición POST

Para realizar una petición por `POST` tenemos que llamar al método `setDoOutput(true)` y además indicar el tipo de petición POST mediante el metodo `setRequestMethod`:

```java
public int peticionPOST( String strUrl, String data )
{
    HttpURLConnection http = null;
    int responseCode = -1;

    try {
        URL url = new URL( strUrl );
        http = (HttpURLConnection) url.openConnection();
        http.setRequestMethod("POST");
        http.setRequestProperty("Content-Type", "application/json");
        http.setRequestProperty("Accept", "application/json");
        http.setDoOutput(true);

        PrintWriter writer = new PrintWriter(http.getOutputStream());
        writer.print(data);
        writer.flush();

        responseCode = http.getResponseCode();
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        if (http != null) http.disconnect();
    }
    return responseCode;
}
```

En este caso la función, además de la URL, recibe un segundo parámetro con los datos a enviar. En este ejemplo se reciben los datos como una cadena \(que ya tendrá los datos almacenados\) y se envían a servidor usando un _output stream_. A la clase `PrinterWriter` se le puede pasar todo tipo de datos que queramos enviar, como por ejemplo un objecto JSON, o incluso podríamos establecer distintos parámetros a enviar de la forma:

```java
Uri.Builder builder = new Uri.Builder()
        .appendQueryParameter("firstParam", paramValue1)
        .appendQueryParameter("secondParam", paramValue2)
        .appendQueryParameter("thirdParam", paramValue3);
String query = builder.build().getEncodedQuery();

PrintWriter writer = new PrintWriter(http.getOutputStream());
writer.print( query );
writer.flush();
```

En la función de ejemplo simplemente se lee el código de la respuesta y con eso ya podemos saber si el servidor ha procesado bien o no la petición. En caso de que queramos leer el cuerpo de la respuesta lo podemos hacer de la misma forma que en la petición tipo `GET`. A veces, según la API usada, también nos interesará leer la cabecera `Location` de respuesta, ya que en ella se suele devolver la URI del nuevo recurso generado.

#### Petición PUT

Las peticiones PUT son muy similares a las POST, también tenemos que usar los métodos `setDoOutput(true)` e indicar el tipo de petición PUT mediante el metodo `setRequestMethod`:

```java
public int peticionPUT( String strUrl, String data )
{
    HttpURLConnection http = null;
    int responseCode = -1;

    try {
        URL url = new URL( strUrl );
        http = (HttpURLConnection) url.openConnection();
        http.setRequestMethod("PUT");
        http.setRequestProperty("Content-Type", "application/json");
        http.setRequestProperty("Accept", "application/json");
        http.setDoOutput(true);

        PrintWriter writer = new PrintWriter(http.getOutputStream());
        writer.print(data);
        writer.flush();

        responseCode = http.getResponseCode();
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        if (http != null) http.disconnect();
    }
    return responseCode;
}
```

Como se puede ver recibe como parámetros la URL y los datos a enviar, y devuelve el código de respuesta para que podamos comprobar si todo ha funcionado correctamente. Al igual que en las peticiones tipo POST también podemos enviar otro tipo de datos y leer el cuerpo de la respuesta si fuese necesario.

#### Petición DELETE

Y por último para realizar una petición tipo DELETE simplemente tendríamos que indicarlo con el método `setRequestMethod` y por último llamar a `getResponseCode` para lanzar la petición y comprobar el código de respuesta.

```java
public int peticionDELETE( String strUrl )
{
    HttpURLConnection http = null;
    int responseCode = -1;
    try {
        URL url = new URL( strUrl );
        http = (HttpURLConnection) url.openConnection();
        http.setRequestMethod("DELETE");
        http.setRequestProperty("Content-Type", "application/json");
        http.setRequestProperty("Accept", "application/json");

        // Conectar y obtener el codigo de respuesta
        responseCode = http.getResponseCode();
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        if (http != null) http.disconnect();
    }
    return responseCode;
}
```

Como respuesta de esta función se devolverá el código de la petición, de esta forma podremos comprobar desde fuera si se ha eliminado correctamente o si ha habido algún error.

#### Autentificación en servicios remotos

Los servicios REST estan fuertemente vinculados al protocolo HTTP, por lo que los mecanismos de seguridad utilizados también deberían ser los que define dicho protocolo. Pueden utilizar los diferentes tipos de autentificación definidos en HTTP: _Basic_, _Digest_ y _X.509_. Sin embargo, cuando se trata de métodos que se dejan disponibles para que servicios externos puedan acceder a ellos, utilizar directamente estos mecanismos básicos de seguridad puede resultar peligroso. En estos casos la autentificación suele realizarse mediante el protocolo _OAuth_. Este último se sale de los contenidos del curso, en la siguiente sección nos centraremos en la seguridad HTTP básica.

**Seguridad HTTP básica**

Para acceder a servicios protegidos con seguridad HTTP estándar deberemos proporcionar en la llamada al servicio las cabeceras de autentificación con los credenciales que nos den acceso a las operaciones solicitadas.

Por ejemplo, desde un cliente Android en el que utilicemos la API de red estándar de Java SE deberemos definir un `Authenticator` que proporcione estos datos:

```java
Authenticator.setDefault(new Authenticator() {
    protected PasswordAuthentication getPasswordAuthentication() {
        return new PasswordAuthentication (
              "usuario", "password".toCharArray());
    }
});
```

Para quienes no estén muy familiarizados con la seguridad en HTTP, conviene mencionar el funcionamiento del protocolo a grandes rasgos. Cuando realizamos una petición HTTP a un recurso protegido con seguridad básica, HTTP nos devuelve una respuesta indicándonos que necesitamos autentificarnos para acceder. Es entonces cuando el cliente solicita al usuario las credenciales \(usuario y password\), y entonces se realiza una nueva petición con dichas credenciales incluidas en una cabecera _Authorization_. Si las credenciales son válidas, el servidor nos dará acceso al contenido solicitado.

Este es el funcionamiento habitual de la autentificación. En el caso del acceso mediante `HttpURLConnection` que hemos visto anteriormente, el funcionamiento es el mismo, cuando el servidor nos pida autentificarnos la librería lanzará una nueva petición con las credenciales especificadas en el proveedor de credenciales.

Sin embargo, si sabemos de antemano que un recurso va a necesitar autentificación, podemos también autentificarnos de forma **preventiva**. La autentificación preventiva consiste en mandar las credenciales en la primera petición, antes de que el servidor nos las solicite. Con esto ahorramos una petición, pero podríamos estar mandando las credenciales en casos en los que no resulta necesario.

Con `HttpURLConnection` podemos activar la autentificación preventiva añadiendo nosotros mismos la cabecera:

```java
URL url = new URL(strUrl);
http = (HttpURLConnection) url.openConnection();

String encoded = Base64.encodeToString((user + ":" + pass).getBytes(), 0);
http.setRequestProperty("Authorization", "Basic "+encoded);
```

Es importante destacar que para que todos estos métodos sean seguros las conexiones se tendrían que realizar con HTTPS ya que en otro caso la conexion no sería segura.

### Parsing de estructuras XML

En las comunicaciones por red es muy común transmitir información en formato XML, el ejemplo más conocido depués del HTML, son las noticias RSS. En este último caso, al delimitar cada campo de la noticia por tags de XML se permite a los diferentes clientes lectores de RSS obtener sólo aquellos campos que les interese mostrar.

Android nos ofrece dos maneras de trocear o "parsear" XML. El `SAXParser` y el `XmlPullParser`:

* El parser SAX requiere la implementación de manejadores que reaccionan a eventos tales como encontrar la apertura o cierre de una etiqueta, o encontrar atributos.
* Por el contrario `XmlPullParser` necesita menos implementación ya que consiste en iterar sobre el árbol de XML \(sin tenerlo completo en memoria\) conforme el código lo va requiriendo, indicándole al parser que tome la siguiente etiqueta \(método `next()`\) o texto \(método `nextText()`\).

A continuación mostramos un ejemplo sencillo de uso del `XmlPullParser`. Préstese atención a las sentencias y constantes resaltadas, para observar cómo se identifican los distintos tipos de etiqueta, y si son de apertura o cierre. También se puede ver cómo encontrar atributos y cómo obtener su valor.

```java
try {
  URL text = new URL("http://www.ua.es");

  XmlPullParserFactory parserCreator = XmlPullParserFactory.newInstance();
  XmlPullParser parser = parserCreator.newPullParser();
  parser.setInput(text.openStream(), null);
  int parserEvent = parser.getEventType();

  while (parserEvent != XmlPullParser.END_DOCUMENT)
  {
    switch (parserEvent) {
        case XmlPullParser.START_DOCUMENT:
          break;
        case XmlPullParser.END_DOCUMENT:
          break;
        case XmlPullParser.START_TAG:
          String tag = parser.getName();
          if (tag.equalsIgnoreCase("title"))
          {
            Log.i("XML","El titulo es: "+ parser.nextText());
          }
          else if(tag.equalsIgnoreCase("meta"))
          {
            String name = parser.getAttributeValue(null, "name");
            if(name.equalsIgnoreCase("description")) {
              Log.i("XML","La descripción es:" +
                          parser.getAttributeValue(null,"content"));
            }
          }
          break;
        case XmlPullParser.END_TAG:
          break;
    }

    parserEvent = parser.next();
  }
} catch (Exception e) {
  Log.e("Net", "Error en la conexion de red", e);
}
```

> Nota: el ejemplo anterior, igual que todas las operaciones que requieran acceso a Internet, tendrá que estar en un hilo.

El ejemplo anterior serviría para imprimir en el LogCat el título del siguiente fragmento de página web, que en este caso sería "Universidad de Alicante", y para encontrar el `meta` cuyo atributo `name` sea "description" y mostrar el valor de su atributo `content`:

```markup
<html xmlns="http://www.w3.org/1999/xhtml" lang="es" xml:lang="es">
<head>
<title>Universidad de Alicante*</title>
<meta name="Description" content="Informacion Universidad Alicante.
  Estudios, masteres, diplomaturas, ingenierias, facultades, escuelas"/>
<meta http-equiv="pragma" content="no-cache" />
<meta name="Author" content="Universidad de Alicante" />
<meta name="Copyright" content="&copy; Universidad de Alicante" />
<meta name="robots" content="index, follow" />
```

### Parsing de estructuras JSON

JSON es una representación muy utilizada para formatear los recursos solicitados a un servicio web RESTful. El formato sigue la misma notación de objetos de JavaScript, por lo tanto ocupa muy poco, es texto plano y puede ser manipulado muy fácilmente utilizando JavaScript, PHP u otros lenguajes.

La gramática de los objetos JSON es simple y requiere la agrupación de la definición de los datos y valores de los mismos. En primer lugar, los elementos están contenidos dentro de llaves `{` y `}` y separados por comas; los valores de los diferentes elementos se organizan en pares con la estructura `"nombre":"valor"`; y finalmente, las secuencias de elementos \(o arrays\) están contenidas entre corchetes `[` y `]`. Y esto es todo! Para una descripción detallada de la gramática, podéis consultar  [http://www.json.org/fatfree.html](http://www.json.org/fatfree.html)

Con la definición de agrupaciones anterior, podemos combinar múltiples conjuntos para crear cualquier tipo de estructura requerido. El siguiente ejemplo muestra una descipción JSON de un objeto con información sobre una lista de mensajes:

```javascript
[
    {"texto":"Hola, ¿qué tal?", "usuario":"Pepe" },
    {"texto":"Fetén", "usuario":"Ana" }
]
```

Antes de visualizar cualquiera de los valores de una respuesta JSON, necesitamos convertirla en otro tipo de estructura para poder procesarla. Dentro de la API de Android encontramos una serie de clases que nos permiten analizar y componer mensajes JSON. Las dos clases fundamentales son `JSONArray` y `JSONObject`. La primera de ellas representa una lista de elementos, mientras que la segunda representa un objeto con una serie de propiedades. Podemos combinar estos dos tipos de objetos para crear cualquier estructura JSON. Cuando en el JSON encontremos una lista de elementos \(`[ ... ]`\) se representará mediante `JSONArray`, mientras que cuando encontremos un conjunto de propiedades _clave-valor_ encerrado entre llaves \(`{ ... }`\) se representará con `JSONObject`. Por ejemplo, para procesar el JSON del ejemplo anterior podríamos utilizar el siguiente código:

```java
JSONArray mensajes = new JSONArray(contenido);

for(int i=0;i<mensajes.length();i++) {
    JSONObject mensaje = mensajes.getJSONObject(i);

    String texto = mensaje.getString("texto");
    String usuario = mensaje.getString("usuario");

    //...
}
```

El objeto `JSONArray` nos permite conocer el número de elementos que contiene \(`length`\), y obtenerlos a partir de su índice, con una serie de métodos `get-`. Los elementos pueden ser de tipos básicos \(`boolean`, `double`, `int`, `long`, `String`\), o bien ser objetos o listas de objetos \(`JSONObject`, `JSONArray`\).

Los objetos \(`JSONObject`\) tienen una serie de campos, a los que también se accede mediante una serie de métodos `get-` que pueden ser de los mismos tipos que en el caso de las listas. En el ejemplo anterior son cadenas \(_texto_, y _usuario_\), pero podrían ser listas u otros objetos anidados.

Esta librería no sólo nos permite analizar JSON, sino que también podemos componer mensajes con este formato. Los objetos `JSONObject` y `JSONArray` tienen para cada método `get-`, un método `put-` asociado que nos permite añadir campos o elementos. Una vez añadida la información necesaria, podemos obtener el texto JSON mediante el método `toString()` de los objetos anteriores. A continuación se incluye un ejemplo:

```java
JSONObject jobj = new JSONObject();
jobj.put("texto", "Texto de prueba");
jobj.put("usuario", "Juan" );

String json = jobj.toString();
```

Además es importante que capturemos las excepciones al procesar cadenas en JSON ya que en caso de intentar obtener un tipo de elemento que no estuviera presente se lanzaría una excepción del tipo `JSONException`.

Esta librería es sencilla y fácil de utilizar, pero puede generar demasiado código para parsear estructuras de complejidad media. Existen otras librerías que podemos utilizar como _GSON_ \([`http://sites.google.com/site/gson/gson-user-guide`](http://sites.google.com/site/gson/gson-user-guide)\) o _Jackson_ \([`http://wiki.fasterxml.com/JacksonInFiveMinutes`](http://wiki.fasterxml.com/JacksonInFiveMinutes)\) que nos facilitarán notablemente el trabajo, ya que nos permiten mapear el JSON directamente con nuestros objetos Java, con lo que podremos acceder al contenido JSON de forma similar a como se hace en Javascript.

## Ejercicios

### Ejercicio 1 - Cliente Android mediante servicios REST del videoclub \(3 puntos\)

En este ejercicio se pide realizar una aplicación Android que actue de cliente de los servicios públicos tipo REST del videoclub. Para esto tenéis que crear una nueva aplicación, llamada `Videoclub`, que va a tener solamente dos actividades: un listado con las películas y la vista detalle de una película \(la apariencia de estas dos pantallas se deja a vuestra elección\). En las plantillas se facilita la clase `Movie.java` que podéis utilizar para almacenar los datos de las películas.

En la primera actividad, el listado de películas, se tendrá que:

* Solicitar en primer lugar el listado al servidor realizando una petición tipo GET a la URL "[http://gbrain.dlsi.ua.es/videoclub/api/v1/catalog](http://gbrain.dlsi.ua.es/videoclub/api/v1/catalog)" \(recordad que tiene que ser asíncrona\).
* El servidor podrá devolver:
  * El array de películas en JSON y el código de respuesta 200.
  * Los códigos de error 404 o 500.
* Una vez completada la descarga tendréis que procesar el JSON \(creando una lista de objetos tipo _Movie_\).
* A partir de la lista obtenida se creará un adaptador que irá asociado a la lista. Este adaptador realizará la descarga de las portadas usando la técnica _lazy loading_ \(el código de esta parte es muy parecido al del último ejercicio de la sección anterior\).

Al pulsar sobre un elemento de esta lista se tendrá que abrir la vista detalle de la misma pasándole \(en el _intent_ de llamada\) los datos de la película.

En la vista detalle primero tendréis que recoger el _intent_ de llamada con todos los datos de la película, mostrar la información y crear un hilo para descargar la portada de la película.



