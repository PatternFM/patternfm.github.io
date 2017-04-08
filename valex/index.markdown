---
layout: valex
title:  "Produce predictable and consumable REST error payloads with Valex, a YAML-based validation and exception management library for Java."
---

# Introduction

An API should provide useful error responses in a predictable and consumable format. An error response should provide a few things for a developer - a useful error message, a unique error code, and a meaningful HTTP response code.

Valex can help you produce meaningful responses that include an appropriate HTTP response code as well as a JSON payload that looks like this:

```
{
  "errors": [
    {
      "code": "ACC-1000",
      "message": "An account username is required."
    },
    {
      "code": "ADD-1001",
      "message": "An account password is required."
    }
  ]
}
```

The JSON payload is entirely configuration driven, so you can produce a JSON payload that looks like this just as easily:

```
{
  "errors": [
    {
      "code": "ACC-1000",
      "message": "An account username is required.",
      "field": "account.username",
      "support_url": "https://support.yoursite.com/kb/articles/acc-1000.html"
    },
    {
      "code": "ADD-1001",
      "message": "An account password is required.",
      "field": "account.password",
      "support_url": "https://support.yoursite.com/kb/articles/add-1001.html"
    }
  ]
}
```

Valex does not return HTTP responses itself, but gives application developers enough information about an error to _construct_ an appropriate HTTP response, while Valex provides the error payload. 

Valex has native support for [JSR-303 BeanValidation](http://beanvalidation.org/1.0/) annotated models, which you can validate explicitly using the ValidationService API, or declaratively with @Valid annotations. 

The [Quick Start](#quickstart) guide will walk through all of the steps required to configure Valex, trigger validation on a BeanValidation annotated model, and report on errors using the Spring Framework @ExceptionResolver annotation.


<a name="quickstart"></a>

# Quick Start

#### 1. Add Valex to your project dependency list


**Maven**   

```xml
<dependency>
    <groupId>fm.pattern</groupId>
    <artifactId>valex</artifactId>
    <version>1.0.7</version>
</dependency>
```

**Gradle**

```
compile group: 'fm.pattern', name: 'valex', version: '1.0.7'
```

#### 2. Place a file named ValidationMessages.yml on the root of your classpath

```yaml
account.username.required:
  code: ACC-1000
  message: An account username is required.
  exception: fm.pattern.valex.UnprocessableEntityException

account.password.required:
  code: ACC-1001
  message: An account password is required.
  exception: fm.pattern.valex.UnprocessableEntityException
```

#### 3. (Optional) Annotate a model with JSR-303 Bean Validation annotations 
You can skip this step if you're not validating JSR-303 annotated models. 

```java
public class Account {

  @NotBlank(message = "{account.username.required}")
  private String username;

  @NotBlank(message = "{account.password.required}")
  private String password;

}
```

#### 4. (Optional) Configure the Valex validation APIs
You can skip this step if you’re not validating JSR-303 annotated models.

Valex exposes a ValidationService that can be used to explicitly trigger validation, as well as @Valid annotations (executed by the ValidationAdvisor) that can be used to perform validation declaratively. You can wire these services into your Spring application using the following configuration:

```java
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
The ValidationAdvisor bean and @EnableAspectJAutoProxy annotation are only required when you want to use the Valex @Valid annotations. 

The ValidationService relies on an underlying [BeanValidation](http://beanvalidation.org/1.0/) Validator implementation to execute valiation logic on annotated models. In the example above we're using the Spring LocalValidatorFactory bean, but any BeanValidation Validator will do:

```java
@Bean(name = "validator")
public Validator validator() {
    return Validation.buildDefaultValidatorFactory().getValidator();
}
```

#### 5. Setup your Spring @ExceptionHandler
The Spring @ExceptionHandler annotation allows you to intercept specific exceptions thrown by your application. This technique allows you to assign an appropriate HTTP response code to an exception class configured in the ValidationMessages.yml file. 

The toRepresentation() method of the Valex UnprocessableEntityException returns an ```ErrorsRepresentation``` object, which produces the appropriate JSON payload described in the introduction.

```java
@RestController
public class Endpoint {

	@ResponseStatus(value = HttpStatus.UNPROCESSABLE_ENTITY)
	@ExceptionHandler(UnprocessableEntityException.class)
	public ErrorsRepresentation handleUnprocessableEntity(UnprocessableEntityException exception) {
		return exception.toRepresentation();
	}

}
```


#### 6. Trigger validation 
Valex provides a few ways to trigger validation, but we'll start with the Valex @Valid annotation. In this case, we instruct the annotation to throw an exception if validation fails by passing a ```throwException = true``` argument to the annotation.

```java
public interface AccountService {

    public Account create(@Valid(throwException = true) Account account);

}
```

```java
public class AccountService {

    public Result<Account> create(Account account) {
    	if(StringUtils.isBlank(account.getUsername()) {
    		return Result.reject("account.username.required");
    	}
    }

}
```

```java
public class AccountService {

    public Result<Account> create(Account account) {
    	if(StringUtils.isBlank(account.getUsername()) {
    		Result.raise("account.username.required");
    	}
    }

}
```

<a name="configuration"></a>

# Configure error payloads and exceptions
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

# Custom Configuration Filename

```
  java -jar myfile.jar -Dvalex.config=MyFile.yml
```
