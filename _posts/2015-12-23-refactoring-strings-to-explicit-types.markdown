---
layout: post
title:  "Refactoring Strings to explicit types for Spring RestController"
date:   2015-12-23
---
> Effective Java Item 50: Avoid strings where other types are more appropriate

While implementing a server API I started using Strings as parameter types. Problem with having more than one parameter with type String
is that the method signatures in the call stack getting cumbersome, less flexible and more error prone, as Item 50 concludes. 
 
```java
String someMethod(String, String, String)
```

So, let's start with a @RestController with String types and consider following request mapping

```java
@RequestMapping(path = "/{schema}/{table}", method = RequestMethod.GET)
public ResponseEntity<List<ResultDTO>> getPrimaryKeyInfo(@PathVariable String schema, @PathVariable String table, @RequestParam(value = "database") String database) {
    List<ResultDTO> result = databaseService.getResult(schema, table, database);

    return new ResponseEntity<>(result, HttpStatus.OK);
}
```

where `ResultDTO` is

```java
public class ResultDTO {
    private final String schema;
    private final String table;
       
    public ResultDTO(String schema, String table) {
        this.schema = schema;
        this.table = table;
    }
    
    // other stuff
}
```

While `schema` and `table` kind of not predictable content the variable `database` is fix.

## First, refactoring database to enum

Create an appropriate enum

```java
public enum Database {
    DEV, TEST, PROD
}
```

Now, the type `Database` can be introduced into `DatabaseService`

```java
@RequestMapping(path = "/{schema}/{table}", method = RequestMethod.GET)
public ResponseEntity<List<ResultDTO>> getPrimaryKeyInfo(@PathVariable String schema, @PathVariable String table, @RequestParam(value = "database") String databaseId) {
    Database database = Database.valueOf(databaseId);
    List<ResultDTO> result = databaseService.getResult(schema, table, database);

    return new ResponseEntity<>(result, HttpStatus.OK);
}
```

A problem arises if database is used at more than one place. It's not DRY to make the conversion every time. Also, it's better to have the explicit type
in the request mapping as well. For that Spring comes with `org.springframework.format.Formatter`. So let's write an `DatabaseFormatter`
 
```java
public class DatabaseFormatter implements Formatter<Database> {
    @Override
    public Database parse(String databaseId, Locale locale) throws ParseException {
        try {
            return Database.valueOf(databaseId);
        } catch (IllegalArgumentException e) {
            // do some error handling
        }
    }

    @Override
    public String print(Database database, Locale locale) {
        return database.toString();
    }
}
```

The new formatter can be registered directly in the @RestController

```java
@RestController
@RequestMapping("/api/schemas")
public class DatabaseInfoController {

    // ...

    @InitBinder
    public void initBinder(WebDataBinder binder) {
        binder.addCustomFormatter(new DatabaseFormatter());
    }
}
```

Or, to be even more DRY, in an [ControllerAdvice](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/bind/annotation/ControllerAdvice.html). 

```java
@ControllerAdvice(annotations = RestController.class)
public class MyControllerAdvice {

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ResponseEntity<String> sqlException(Exception e) {
        return new ResponseEntity<>(e.toString(), HttpStatus.INTERNAL_SERVER_ERROR);
    }

    @InitBinder
    public void initBinder(WebDataBinder binder) {
        binder.addCustomFormatter(new DatabaseFormatter());
    }
    
}
```

As parameter the annotation takes the configuration mechanism which Controllers shall be adviced. In this example all @RestController are adviced.
But it's also possible to advice just Controllers in a base package for example. As bonus, ControllerAdvices can also take @ExceptionHandler that apply to
all considered Controller.

Now, we have a strongly typed database parameter in the request mapping

```java
@RequestMapping(path = "/{schema}/{table}", method = RequestMethod.GET)
public ResponseEntity<List<ResultDTO>> getPrimaryKeyInfo(@PathVariable String schema, @PathVariable String table, @RequestParam(value = "database") Database database) {
    List<ResultDTO> result = databaseService.getResult(schema, table, database);

    return new ResponseEntity<>(result, HttpStatus.OK);
}
```

## Second, introduce types for schema and table

For to get rid of type String for `schema` and `table` an identifier type is introduced. A simple wrapper of a String.

```java
public class SchemaId {

    private final String value;

    private SchemaId(String value) {
        assert value != null : "value must not be null";
        this.value = value;
    }

    public static SchemaId create(String value) {
        return new SchemaId(value);
    }

    @Override
    public String toString() {
        return value;
    }
    
    // equals and hashcode
}
```

Great stuff. Again, create Formatter and configure them in @InitBinder in the advice to have type safety right from the beginning of the call stack.
 
```java
@ControllerAdvice(annotations = RestController.class)
public class MyControllerAdvice {
     // ...
 
     @InitBinder
     public void initBinder(WebDataBinder binder) {
         binder.addCustomFormatter(new DatabaseFormatter());
         binder.addCustomFormatter(new SchemaIdFormatter());
         binder.addCustomFormatter(new TableIdFormatter());
     }

}
```
 
Now, our request mapping looks exactly as wanted: No parameters with type String any more

```java
@RequestMapping(path = "/{schema}/{table}", method = RequestMethod.GET)
public ResponseEntity<List<ResultDTO>> getPrimaryKeyInfo(@PathVariable SchemaId schema, @PathVariable TableId table, @RequestParam(value = "database") Database database) {
    List<ResultDTO> result = databaseService.getResult(schema, table, database);

    return new ResponseEntity<>(result, HttpStatus.OK);
}
```

But wait, if we also refactor the `ResultDTO`

```java
public class ResultDTO {
    private final SchemaId schema;
    private final TableId table;
       
    public ResultDTO(SchemaId schema, TableId table) {
        this.schema = schema;
        this.table = table;
    }
    
    // ...
}
```

we run in trouble and get an exception

```
org.springframework.http.converter.HttpMessageNotWritableException: Could not write content: No serializer found for class xyz.SchemaId
nested exception is com.fasterxml.jackson.databind.JsonMappingException: No serializer found for class xyz.SchemaId
```

Ah, ok. While spring can already convert the input from String to the new strong types, the output cannot be converted. Jackson is not amused.
 
This can be solved with registering `JsonSerializer` for jackson. For that the `Jackson2ObjectMapperBuilder` must be configured manually and
added as converter to `HttpMessageConverter`. One possibility is to let the application class extend of `WebMvcConfigurerAdapter` and
overwrite `configureMessageConverters(List<HttpMessageConverter<?>>)`.

```java
@ComponentScan
@EnableAutoConfiguration
public class MyApplication extends WebMvcConfigurerAdapter {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.json();

        builder.serializerByType(SchemaId.class, new ToStringSerializer());
        builder.serializerByType(TableId.class, new ToStringSerializer());

        builder.serializationInclusion(JsonInclude.Include.NON_NULL);
        builder.propertyNamingStrategy(PropertyNamingStrategy.CAMEL_CASE_TO_LOWER_CASE_WITH_UNDERSCORES);
        builder.serializationInclusion(JsonInclude.Include.NON_EMPTY);
        converters.add(new MappingJackson2HttpMessageConverter(builder.build()));
    }

}
```

As a bonus, the `Jackson2ObjectMapperBuilder` can be configured with serialization inclusions etc. Here, as serializer the standard 
`ToStringSerializer` can be used, because the value of the SchemaId and TableId can be retrieved with `toString()`, see above.

## Conclusion

I'm convinced that it's a good pattern to use strongly typed parameters to reduce cumbersome and error-prone APIs. I've shown that with spring `@RestController`
it's easy to use type safe parameters from request mapping down the call stack. This by using `WebDataBinder` to register [Formatter](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/format/Formatter.html)
and to use an [@ControllerAdvice](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/bind/annotation/ControllerAdvice.html) 
to configrure all `@RestController` at one time. Also consider to register 
[@ExceptionHandler](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/bind/annotation/ExceptionHandler.html) in the `ControllerAdvice`s.
Then, with configuring `Jackson2ObjectMapperBuilder` an [JsonSerializer](https://fasterxml.github.io/jackson-databind/javadoc/2.2.0/com/fasterxml/jackson/databind/JsonSerializer.html) 
can be added, that the strong types are converted seamlessly back to Strings. 
