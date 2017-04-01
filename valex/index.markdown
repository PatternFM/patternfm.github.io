---
layout: post
title:  "Valex: Produce predictable and consumable REST error payloads with Valex, a YAML-based validation and exception management library for Java"
date:   2017-02-18
---

# Introduction

An API should provide a useful error responses in a known, consumable format. The representation of an error should be no different than the representation of any other resource, just with its own set of fields. An error response should provide a few things for a developer - a useful error message, a unique error code, and a meaningful HTTP response code. A JSON representation of an error payload would look like:

```
{
  "errors": [
    {
      "code": "ACC-0001",
      "message": "An account username is required."
    }
  ]
}
```

Valex builds on top of the BeanValidation API to give developers the ability to configure an error message, error code, and an exception to throw when a validation check fails, all in one spot.

```yaml
account.username.required:
  code: ACC-0001
  message: An account username is required.
  exception: fm.pattern.valex.UnprocessableEntityException
```

The exception, when thrown, will carry with it the error message and error code as specified in the configuration. If you're using Spring, you can use the @ExceptionHandler to easily convert the exception into the JSON format specified above:

```java
@RestController
public class Endpoint {

	@ResponseBody
	@ResponseStatus(value = HttpStatus.UNPROCESSABLE_ENTITY)
	@ExceptionHandler(UnprocessableEntityException.class)
	public ErrorsRepresentation handleUnprocessableEntity(UnprocessableEntityException exception) {
		return exception.toRepresentation();
	}

	@ResponseBody
	@ResponseStatus(value = HttpStatus.UNAUTHORIZED)
	@ExceptionHandler(AuthenticationException.class)
	public ErrorsRepresentation handleAuthentication(AuthenticationException exception) {
		return exception.toRepresentation();
	}

	@ResponseBody
	@ResponseStatus(value = HttpStatus.FORBIDDEN)
	@ExceptionHandler(AuthorizationException.class)
	public ErrorsRepresentation handleAuthorization(AuthorizationException exception) {
		return exception.toRepresentation();
	}

	@ResponseBody
	@ResponseStatus(value = HttpStatus.NOT_FOUND)
	@ExceptionHandler(EntityNotFoundException.class)
	public ErrorsRepresentation handleEntityNotFound(EntityNotFoundException exception) {
		return exception.toRepresentation();
	}

	@ResponseBody
	@ResponseStatus(value = HttpStatus.INTERNAL_SERVER_ERROR)
	@ExceptionHandler(InternalErrorException.class)
	public ErrorsRepresentation handleInternalError(EntityNotFoundException exception) {
		return exception.toRepresentation();
	}

}
```
Because Valex builds on top of the JSR-303 BeanValidation API, you can annotate your models with standard BeanValidation annotations. A ValidationMessages.properties file can also be used with Valex to minimise the migration path for existing JSR-303 implementations that want to use Valex for more robust API error reporting.


# Getting Started

To get started, add the following dependency to your dependency list:

```xml
<dependency>
    <groupId>fm.pattern</groupId>
    <artifactId>valex</artifactId>
    <version>1.0.6</version>
</dependency>
```

Valex exposes a ValidationService API that can be used to explicitly trigger validation, as well as @Valid annotations that can be used to perform validation declaratively. You can wire these services into your Spring application using the following configuration:

```java
import javax.validation.Validator;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;
import org.springframework.validation.beanvalidation.LocalValidatorFactoryBean;

import fm.pattern.valex.SimpleValidationService;
import fm.pattern.valex.ValidationService;
import fm.pattern.valex.annotations.ValidationAdvisor;

@Configuration
@EnableAspectJAutoProxy
public class ValidationConfiguration {

    @Bean(name = "validator")
    public Validator validator() {
        return new LocalValidatorFactoryBean();
    }

    @Bean
    public ValidationService validationService() {
        return new SimpleValidationService(validator());
    }

    @Bean
    public ValidationAdvisor validationAdvisor() {
        return new ValidationAdvisor(validationService());
    }

}
```
The Valex API relies on an underlying BeanValidation Validator implementation to execute valiation logic on annotated models. In the example above we're bootstrapping the Spring LocalValidatorFactory bean, but you can bootstrap your own validator as well:

```java
@Bean(name = "validator")
public Validator validator() {
    return Validation.buildDefaultValidatorFactory().getValidator();
}
```

If you plan to make use of Valex annotations, make sure you enable AspectJ proxying by including the ```@EnableAspectJAutoProxy``` annotation.


# Configure Error Codes, Messages and Exceptions
You can configure your application's error messages, error codes and exceptions using a YAML configuration file or a properties configuration file. It's simply a matter of syntax preference in terms of which method you choose. The properties file is available to support an easier migration path if you have an existing BeanValidation implementation with a ValidationMessages.properties file already defined.

### YAML Configuration
If you choose to go down the YAML configuration path, place a file named **ValidationMessages.yml** on the root of your classath. An example YAML configuration file will look like this:


```yaml
default:
  exception: fm.pattern.valex.UnprocessableEntityException

account.id.required:
  message: "An account id is required."
  code: ACC-0006

account.username.required:
  message: "An account username is required."
  code: ACC-0001

account.username.size:
  message: "An account username must be between {min} and {max} characters."
  code: ACC-0002

account.username.conflict:
  message: "The username {validatedValue} is already in use."
  code: ACC-0003
  exception: fm.pattern.valex.ResourceConflictException

account.password.required:
  message: "An account password is required."
  code: ACC-0004

account.password.size:
  message: "An account password must be between {min} and {max} characters."
  code: ACC-0005

account.username.not_found:
  message: "No such username: %s"
  code: ACC-0008
  exception: fm.pattern.valex.EntityNotFoundException
```


### Properties Configuration

If you choose to go down the Java properties file configuration path, place a file named **ValidationMessages.properties** on the root of your classath. An example properties configuration file will look like this:

```
default.exception=fm.pattern.valex.UnprocessableEntityException

account.id.required=An account id is required.
account.id.required.code=ACC-0006

account.username.required=An account username is required.
account.username.required.code=ACC-0001

account.username.size=An account username must be between {min} and {max} characters.
account.username.size.code=ACC-0002

account.username.conflict=The username {validatedValue} is already in use.
account.username.conflict.code=ACC-0003
account.username.conflict.exception=fm.pattern.valex.ResourceConflictException

account.password.required=An account password is required.
account.password.required.code=ACC-0004

account.password.size=An account password must be between {min} and {max} characters.
account.password.size.code=ACC-0005

account.username.not_found=No such username: %s
account.username.not_found.code=ACC-0008
account.username.not_found.exception=fm.pattern.valex.EntityNotFoundException

```

### Configuration Detail

Let's look at a specific configuration element in detail:

```yaml
account.id.required:
  message: "An account id is required."
  code: ACC-0006
  exception: fm.pattern.valex.UnprocessableEntityException
```

**account.id.required**  
The *key* used to resolve the error message, code and exception for an error. The dot notatation in the key name is purely conventional; Valex treats the key as an opaque value.

**message**   
The error message that should be returned when validation fails for this key. The YAML messages should generally be surrounded by double quotes (double quotes are required when you use *interpolated messaging* - more on this shortly).

**code**   
The error code that should be returned when validation fails for this key. The error code presented above is conventional; choose an error code scheme to suit your needs.

**exception**    
The fully qualified classname of the _ReportableException_ that should be returned when validation fails for this key. Valex requires all configured exceptions to be _ReportableExceptions_, and a number of _ReportableException_ implementations are provided out of the box:
* UnprocessableEntityException
* AuthenticationException
* AuthorizationException
* BadRequestException
* ResourceConflictException
* EntityNotFoundException


### Default Exceptions
You can configure a default exception class to return when a validation element does not contain an explicit *exception* property.

**YAML Configuration**

```yaml
default:
  exception: fm.pattern.valex.UnprocessableEntityException
```  

**Properties Configuration**

```
default.exception=fm.pattern.valex.UnprocessableEntityException
```

In both cases, an UnprocessableEntityException will be returned when a validation element does not explicitly define an exception property.


# Validation using the ValidationService

Use the *ValidationService* validate() method to perform validation against your [BeanValidation](http://beanvalidation.org/1.0/) annotated models. The validate() method returns a typed *Result* object, which contain the final (and immutable) state of a validation event.

```java
import fm.pattern.valex.Result;
import fm.pattern.valex.ValidationService;

@Service
class AccountServiceImpl implements AccountService {

    private final ValidationService validationService;

    public AccountServiceImpl(ValidationService validationService) {
        this.validationService = validationService;
    }

    // Validates the account, and returns the validation Result containing all
    // validation errors if the account is invalid.
    public Result<Account> create(Account account) {
        Result<Account> result = validationService.validate(account);
        if(result.rejected()) {
            return result;
        }
        ...
        return Result.accept(account);
    }

    // Validates the account, and uses the Result doThrow() method to throw the
    // underlying exception bound to the result if the account is invalid.
    public Account create(Account account) {
        Result<Account> result = validationService.validate(account);
        if(result.rejected()) {
            result.doThrow();
        }
        ...
        return account;
    }    

    // Validates the account using the fluent orThrow() method, which returns the
    // account if the account is valid, otherwise throws the underlying exception
    // bound to the result.
    public Account create(Account account) {
        Account valid = validationService.validate(account).orThrow();
        ...
        return account;
    }      

}
```

Since all ValidationService methods delegate validation execution to the configured BeanValidation [Validator](http://docs.jboss.org/hibernate/beanvalidation/spec/1.0/api/javax/validation/Validator.html), you can use [JSR-303 BeanValidation](http://beanvalidation.org/1.0/) annotations to annotate your model.


```java
@UniqueValue(property = "username", message = "{account.username.conflict}")
public class Account {

  @NotBlank(message = "{account.id.required}")
  @Size(min = 3, max = 128, message = "{account.username.size}")
  private String id;

  @NotBlank(message = "{account.username.required}")
  @Size(min = 3, max = 128, message = "{account.username.size}")
  private String username;

  @NotBlank(message = "{account.password.required}")
  @Size(min = 8, max = 255, message = "{account.password.size}")
  private String password;

}
```

### Validation Groups
Valex provides first class support for BeanValidation groups. The validate() method takes a varargs argument of classes that will be used to perform targeted validation.

```java
@Service
class AccountServiceImpl implements AccountService {

    public Result<Account> create(Account account) {
        Result<Account> result = validationService.validate(account, First.class, Second.class);
        ...
        return Result.accept(account);
    }     

}
```

# Declarative Validation using Annotations

Valex provides a parameter-scoped @Valid annotation that will automatically trigger validation and return a typed Result (by default) if validation fails.

Implementation note: When you have an interface and an underlying implementation class, the @Valid annotation must be declared on the *interface* method, and not the *concrete* method.

```java
public interface AccountService {

    // The @Valid annotation is declared on the interface method.
    public Result<Account> create(@Valid Account account);

}

@Service
class AccountServiceImpl implements AccountService {

    public Result<Account> create(Account account) {
        ... // if we get here the account passed validation
    }

}      
```

You can configure the @Valid annotation to throw an exception on validation failure by providing the  ```throwException = true``` argument to the @Valid annotation.

```java
import fm.pattern.valex.Result;
import fm.pattern.valex.annotations.Valid;

public interface AccountService {

    // Throw an exception if the account is invalid.
    public Account create(@Valid(throwException = true) Account account);

}
```

### Annotation Validation Groups

You can specify one or more validation groups to apply when the @Valid annotation is kicked off.

```java
import fm.pattern.valex.Result;
import fm.pattern.valex.annotations.Valid;

public interface AccountService {

    public Account create(@Valid(First.class) Account account);

}
```


# Message Interpolation
Valex supports two types of message interpolation that can be used to produce dynamic error messages.

### BeanValidation Interpolation
BeanValidation message interpolation allows you to inject *annotation attribute values* into error messages. As an example, take the following @Size annotated field with **min** and **max** attributes:

```java
@Size(min = 8, max = 255, message = "{account.password.size}")
private String password;
```

To inject the min and max attribute values into an error message, address the attributes in {braces}:
```
account.password.size:
  message: "Your password must be between {min} and {max} characters."
  code: ACC-0005
```

The {validatedValue} expression can be used to inject the value of the field being validated into your error message.

```java
@Size(min = 8, max = 255, message = "{account.username.size}")
private String username;
```

```
account.username.size:
  message: "The username {validatedValue} must be between {min} and {max} characters."
```

### Result Interpolation
Result objects can be created directly by using the *Result* accept() and reject() methods. The reject() method accepts a key to a configured message or a literal message along with a varargs array for message parameters. The reject() method message format is compatible with the [Java Formatter API](https://docs.oracle.com/javase/8/docs/api/java/util/Formatter.html).

```java
public Result<Account> findById(String id) {
    Account account = repository.findById(id);
    if(account != null) {
    	return Result.accept(account);
    }

    // Return a result by looking up the 'account.not_found' error, and inject the 'id' value into the error message.
    return Result.reject("account.not_found", id);

    // Return a Result using the given literal error message.
    return Result.reject("No such account id: %s", id);
}
```


# Group Sequences

Valex provides three GroupSequence implementations for creating (Create.class), updating (Update.class) and deleting (Delete.class) entities. Each GroupSequence has the following definition:

```java
@GroupSequence({CreateLevel1.class, CreateLevel2.class, CreateLevel3.class, CreateLevel4.class, CreateLevel5.class})
public interface Create {

}
```

Each level represents and ordered configuration step. Valiation will begin by inspecting the level 1 class, and if validation fails at this level no further levels will be inspected. Validation will only continue up the level sequence if validation succeeds at pevious levels.

Using the three Valex groups sequences reduces the boilerplate implementation required to support these common use cases yourself, but you are free to implement your own group sequences as well.

Here's an example of the Valex Create group sequence applied to an account object. In this example, all of the @NotBlank checks (level 1) will be run first, and if successful the @Size checks (level 2) will be run next, and if successfull the @UniqueValue check (level 3) will be run last.

```java
@UniqueValue(property = "username", message = "{account.username.conflict}", groups = {CreateLevel3.class})
public class Account {

  @NotBlank(message = "{account.id.required}", groups = {CreateLevel1.class})
  @Size(min = 3, max = 128, message = "{account.username.size}", groups = {CreateLevel1.class})
  private String id;

  @NotBlank(message = "{account.username.required}", groups = {CreateLevel1.class})
  @Size(min = 3, max = 128, message = "{account.username.size}", groups = {CreateLevel2.class, UpdateLevel2.class})
  private String username;

  @NotBlank(message = "{account.password.required}", groups = {CreateLevel1.class})
  @Size(min = 8, max = 255, message = "{account.password.size}", groups = { CreateLevel2.class})
  private String password;

}
```

### Group Sequence Annotations
If you plan to use the Valex group sequences, you can also take advantage of the Valex @Create, @Update and @Delete annotations as a shorthand notation to apply group sequence based validation.

```java
import fm.pattern.valex.Result;

import fm.pattern.valex.annotations.Create;
import fm.pattern.valex.annotations.Update;
import fm.pattern.valex.annotations.Delete;

public interface AccountService {

    // Applies the Create group sequence during validation.
    public Result<Account> create(@Create Account account);

    // Applies the Update group sequence during validation.
    public Result<Account> update(@Update Account account);

    // Applies the Delete group sequence during validation.
    public Result<Account> delete(@Delete Account account);

}
```

The annotations can also be configured to throw the underlying validation result exception if you do not want the annotations to return typed *Result* objects.

```java
import fm.pattern.valex.annotations.Create;
import fm.pattern.valex.annotations.Update;
import fm.pattern.valex.annotations.Delete;

public interface AccountService {

    public Account create(@Create(throwException = true) Account account);

    public Account update(@Update(throwException = true) Account account);

    public Account delete(@Delete(throwException = true) Account account);
}
```



# Validation Strategy

We'll run through a strategy where validation is performed in an application's business layer, and service interfaces within the business layer return typed *Results*. This approach has a number of benefits, but is not required to use Valex.  

Let's take the following AccountService interface as an example:

```java
public interface AccountService {

   Account create(Account account);

   Account update(Account account);

   Account delete(Account account);

   Account findById(String id);

   Account findByUsername(String username);
}
```

The contract isn't expressive enough to report on both success and failure conditions in a consistent fashion. If an error occurs within the create(), update() or delete() methods, communication about the error would have to be propagated via RuntimeException.

The findById() and findByUsername() methods return an account if found, but could otherwise throw a RuntimeException or return null by convention.

A more expressive contract can provide a simple, consistent approach to handling success and error conditions:

```java
public interface AccountService {

   Result<Account> create(Account account);

   Result<Account> update(Account account);

   Result<Account> delete(Account account);

   Result<Account> findById(String id);

   Result<Account> findByUsername(String username);

}
```

When methods return a typed *Result*, clients can call the fluent orThrow() method on the Result to return the typed object if the method call was successful, or throw the Result's underlying exception (as configured in the ValidationMesages configuration file) if the method call failed:

```java
Account pending = new Account();
...
Account account = accountService.create(pending).orThrow();
```

This approach doesn't force API clients to implement try/catch logic around method calls since the *Result* object is effectively acting as a catch block (instead of catching an exception explicitly, the method is conditionally *returning* one to you). A *Result* object carries the same intent as a try/catch block, but does so in a less obtrusive way.

Callers may optionally inspect the Result object:

```java
Account pending = new Account();
...
Result<Account> result = accountService.create(pending);
if(result.rejected()) {
  List<Reportable> errors = result.getErrors();
  log.error("Failed to create account: " + errors);
}
```

Applying this pattern consistently across a program can help reduce congitive load, since callers can expect a consistent and well-defined response from API calls.
