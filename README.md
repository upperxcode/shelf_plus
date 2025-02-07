# Shelf Plus

**Shelf Plus** is a **quality of life** addon for your server-side development within
the Shelf platform. It's a great base to **start off** your apps fast, while
**maintaining full compatibility** with the **Shelf** ecosystem.

```dart
import 'package:shelf_plus/shelf_plus.dart';

void main() => shelfRun(init);

Handler init() {
  var app = Router().plus;

  app.get('/', () => 'Hello World!');

  return app;
}
```

It comes with a lot of awesome features, like **zero-configuration** initializer, build-in **hot-reload**
and a **super powerful** and **intuitive router upgrade**. Continue reading and get to know why
you can't ever code without **Shelf Plus**.



&nbsp;

## Router Plus

Router Plus is a **high-level abstraction layer** sitting directly on [shelf_router](https://pub.dev/packages/shelf_router). 
It **shares the same [routing logic](https://pub.dev/documentation/shelf_router/latest/shelf_router/Router-class.html)**
but allows you to handle responses in a very simple way.

```dart
var app = Router().plus;

app.use(middleware());

app.get('/text', () => 'I am a text');

app.get(
    '/html/<name>', (Request request, String name) => '<h1>Hello $name</h1>',
    use: typeByExtension('html'));

app.get('/file', () => File('path/to/file.zip'));

app.get('/person', () => Person(name: 'John', age: 42));
```

The core mechanic is called **ResponseHandler** which continuously refines a data structure,
until it resolves in a [Shelf Response](https://pub.dev/documentation/shelf/latest/shelf/Response-class.html).
This extensible system comes with support for text, json, binaries, files, json serialization and Shelf [Handler](https://pub.dev/documentation/shelf/latest/shelf/Handler.html). 

You can access the **Router Plus** by calling the **`.plus`** getter on a regular Shelf Router.

```dart
var app = Router().plus;
```


&nbsp;

### Routes API

The API mimics the [Shelf Router](https://pub.dev/documentation/shelf_router/latest/shelf_router/Router-class.html)
methods. You basically use an HTTP verb, define a route to match and specify a handler,
that generates the response.

```dart
app.get('/path/to/match', () => 'a response');
```

You can return any type, as long the **ResponseHandler** mechanism has a capable
resolver to handle that type.

If you need the [Shelf Request](https://pub.dev/documentation/shelf/latest/shelf/Request-class.html)
object, specify it as the first parameter. Any other parameter will match the
route parameters, if defined.

```dart
app.get('/minimalistic', () => 'response');

app.get('/with/request', (Request request) => 'response');

app.get('/clients/<id>', (Request request, String id) => 'response: $id');

app.get('/customer/<id>', (Request request) {
  // alternative access to route parameters
  return 'response: ${request.routeParameter('id')}';
});
```


&nbsp;

### Middleware

Router Plus provides several options to place your middleware ([Shelf Middleware](https://pub.dev/documentation/shelf/latest/shelf/Middleware.html)).

```dart
var app = Router().plus;

app.use(middlewareA); // apply to all routes

// apply to a single route
app.get('/request1', () => 'response', use: middlewareB);

// combine middleware with + operator
app.get('/request2', () => 'response', use: middlewareB + middlewareC);
```

You can also apply middleware dynamically inside a route handler, using the `>>` operator.

```dart
app.get('/request/<value>', (Request request, String value) {
  return middleware(value) >> 'response';
});
```


&nbsp;

### ResponseHandler

ResponseHandler process the **return value** of a route handler, until it matches a
[Shelf Response](https://pub.dev/documentation/shelf/latest/shelf/Response-class.html).

#### Build-in ResponseHandler

| Source                                   | Result                                                | Use case                                                                                 |
| ---------------------------------------- | ----------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `String`                                 | Shelf `Response`                                      | Respond with a text (text/plain)                                                         |
| `Uint8List`, `Stream<List<int>>`         | Shelf `Response`                                      | Respond with binary (application/octet-stream)                                           |
| `Map<String, dynamic>`, `List<dynamic>>` | Shelf `Response`                                      | Respond with JSON (application/json)                                                     |
| Any Type having a `toJson()` method      | `Map<String, dynamic>`, `List<dynamic>>` *(expected)* | Provide serialization support for classes                                                |
| Shelf `Handler`                          | Shelf `Response`                                      | Processing Shelf-based Middleware or Handler                                             |
| `File` (dart:io)                         | Shelf `Response`                                      | Respond with file contents (using [shelf_static](https://pub.dev/packages/shelf_static)) |

*Example:*

```dart
import 'dart:io';

import 'package:shelf_plus/shelf_plus.dart';

void main() => shelfRun(init);

Handler init() {
  var app = Router().plus;

  app.get('/text', () => 'a text');

  app.get('/binary', () => File('data.zip').openRead());

  app.get('/json', () => {'name': 'John', 'age': 42});

  app.get('/class', () => Person('Theo'));

  app.get('/handler', () => typeByExtension('html') >> '<h1>Hello</h1>');

  app.get('/file', () => File('thesis.pdf'));

  return app;
}

class Person {
  final String name;

  Person(this.name);

  // can be generated by tools (i.e. json_serializable package)
  Map<String, dynamic> toJson() => {'name': name};
}
```

#### Custom ResponseHandler

You can add your own ResponseHandler by using a [Shelf Middleware](https://pub.dev/documentation/shelf/latest/shelf/Middleware.html)
created with the `.middleware` getter on a ResponseHandler function. 

```dart
// define custom ResponseHandler
ResponseHandler catResponseHandler = (Request request, dynamic maybeCat) =>
    maybeCat is Cat ? maybeCat.interact() : null;

// register custom ResponseHandler as middleware
app.use(catResponseHandler.middleware);

app.get('/cat', () => Cat());
```
```dart
class Cat {
  String interact() => 'Purrrrr!';
}
```


&nbsp;

### Cascading multiple routers

Router Plus is compatible to a [Shelf Handler](https://pub.dev/documentation/shelf/latest/shelf/Handler.html).
So, you can also use it in a [Shelf Cascade](https://pub.dev/documentation/shelf/latest/shelf/Pipeline-class.html).
This package provides a `cascade()` function, to quickly set up a cascade.

```dart
import 'package:shelf_plus/shelf_plus.dart';

void main() => shelfRun(init);

Handler init() {
  var app1 = Router().plus;
  var app2 = Router().plus;

  app1.get('/maybe', () => Response.notFound('no idea'));

  app2.get('/maybe', () => 'got it!');

  return cascade([app1, app2]);
}
```




&nbsp;


&nbsp;

## Middleware collection

This package comes with additional [Shelf Middleware](https://pub.dev/documentation/shelf/latest/shelf/Middleware.html)
to simplify common tasks.


&nbsp;

### setContentType

Sets the `content-type` header of a `Response` to the specified **mime-type**.

```dart
app.get('/one', () => setContentType('application/json') >> '1');

app.get('/two', () => '2', use: setContentType('application/json'));
```


&nbsp;

### typeByExtension

Sets the `content-type` header of a `Response` to the **mime-type** of the
specified **file extension**.

```dart
app.get('/', () => '<h1>Hi!</h1>', use: typeByExtension('html'));
```


&nbsp;

### download

Sets the `content-disposition` header of a `Response`, so browsers will download the
server response instead of displaying it. Optionally you can define a specific **file name**.

```dart
app.get('/wallpaper/download', () => File('image.jpg'), use: download());

app.get('/invoice/<id>', (Request request, String id) {
  File document = pdfService.generateInvoice(id);
  return download(filename: 'invoid_$id') >> document;
});
```



&nbsp;


&nbsp;

## Shelf Run

Shelf Run is **zero-configuration** web-server initializer with **hot-reload** support.

```dart
import 'package:shelf_plus/shelf_plus.dart';

void main() => shelfRun(init);

Handler init() {
  return (Request request) => Response.ok('Hello!');
}
```

It's important to use a dedicated `init` function, returning a [Shelf Handler](https://pub.dev/documentation/shelf/latest/shelf/Handler.html),
for hot-reload to work properly.

Shelf Run uses a default configuration, that can be modified via **environment variables**:

| Environment variable | Default value | Description                              |
| -------------------- | ------------- | ---------------------------------------- |
| SHELF_PORT           | 8080          | Port to bind the shelf application to    |
| SHELF_ADDRESS        | localhost     | Address to bind the shelf application to |
| SHELF_HOTRELOAD      | true          | Enable hot-reload                        |