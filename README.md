[![Build Status](https://travis-ci.org/coveo/feign-error-decoder.svg?branch=master)](https://travis-ci.org/coveo/feign-error-decoder)
[![MIT license](http://img.shields.io/badge/license-MIT-brightgreen.svg)](https://github.com/coveo/feign-error-decoder/blob/master/LICENSE)
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/com.coveo/feign-error-decoder/badge.svg)](https://maven-badges.herokuapp.com/maven-central/com.coveo/feign-error-decoder)

# [Feign](https://github.com/OpenFeign/feign) Reflection `ErrorDecoder`

This small library implements `ErrorDecoder` to provide a simple way to map a key returned on an API to a specific exception declared thrown on the [client interface](https://github.com/OpenFeign/feign#basics). 

It's very useful in an application with many microservices calling each others when you want to handle specific checked or runtime exceptions on the client side.

[Blog post](http://source.coveo.com/2016/02/19/microservices-and-exception-handling/) explaining more about the rationale of this library.

# Maven
Now available on Maven Central : 

```xml
    <dependency>
      <groupId>com.coveo</groupId>
      <artifactId>feign-error-decoder</artifactId>
      <version>1.0.0</version>
    </dependency>
```

# Usage
A complete exemple is shown in this repo as the base setup for the unit tests.
## Requirements
### Base Exception class
In order to use this library, you need a base exception which all the exceptions declared thrown on the client interface will inherit from. This exception needs to provide a way to access a unique `String` key per subclass.
```java
public abstract class ServiceException extends Exception {
  private String errorCode;
  //Constructors omitted
  
  public String getErrorCode() {
    return errorCode;
  }
}
```
### Class for the error response
A Class representing the body of the response sent on the API in case of error is also needed. This is the Class that will be instantiated by Feign `Decoder`. It needs a method to get the key that will be used to get the correct exception. It can also optionally have a message that will be injected in the `detailMessage` field of `Throwable`. 
```java
public class ErrorCodeAndMessage {
  private String message;
  private String errorCode;
  //getters and setters omitted
```

## How to use
With these requirements met, all left to do is to extend `ReflectionErrorDecoder` and implement the abstract methods required like this : 
```java
import feign.Response;
import feign.codec.ErrorDecoder;

public class ServiceExceptionErrorDecoder
    extends ReflectionErrorDecoder<ErrorCodeAndMessage, ServiceException> {

  public ServiceExceptionErrorDecoder(Class<?> apiClass) {
    super(apiClass, ErrorCodeAndMessage.class, ServiceException.class);
  }

  @Override
  protected String getKeyFromException(ServiceException exception) {
    return exception.getErrorCode();
  }

  @Override
  protected String getKeyFromResponse(ErrorCodeAndMessage apiResponse) {
    return apiResponse.getErrorCode();
  }

  @Override
  protected String getMessageFromResponse(ErrorCodeAndMessage apiResponse) {
    return apiResponse.getMessage();
  }
}
```
Use the [Feign builder](https://github.com/OpenFeign/feign#customization) to inject the `ErrorDecoder` in your client.
# Optional customization
## Abstract exception support with classpath scanning
An optional dependency on [Spring Context](https://github.com/spring-projects/spring-framework/tree/master/spring-context) is included in the library to allow classpath scanning to fetch the children of abstract exception classes. This allows you to declare a specific base exception as thrown on the client interface and let the library scan all the possible exceptions that can be thrown.

To enable this, all you need to do is have Spring framework available in your project. By default, it will scan the exception children in all packages. To restrict the base package to be scanned, simply use the constructor with the `basePackage` field.

## Custom `Decoder`
By default, this project uses the `JacksonDecoder` implementation of Feign `Decoder` interface. A protected setter is available to use your own `Decoder`.

## Custom fallback `ErrorDecoder`
`ErrorDecoder.Default` is used by default when no exception is found in the scanned exceptions. A protected setter is available to use your own fallback `ErrorDecoder`.

## Supported constructor arguments
The library has a default list of supported argument types for the exception constructors. It supports empty and constructors with any number of `String` or `Throwable` in any order. To extend supported exception types, just override the method `protected List<Object> getSupportedConstructorArgumentInstances()`. Just make sure to return the default types of `String` and `Throwable` if you still want them to be supported.

# Contributing
PR are always welcome and please open an issue if you find any bugs or wish to request an additional feature. 