# ts-ioc-di
Typescript IoC container and DI
[![CircleCI](https://circleci.com/gh/glebivanov816/ts-ioc-di.svg?style=svg)](https://circleci.com/gh/glebivanov816/ts-ioc-di)

# About
This container can be used to create full graphs of objects using its features.

# Features
- Constructor injection
- Property injection
- Method injection
- Method/constructor argument injection
- Autowiring

# Dependencies
- Typescript (DI is implemented with typescript types metadata as a dependency)
- Reflect-metadata

# Examples
## Initialization and bindings
```
import { Container } from 'ts-ioc-di';

const container = new Container();

// binds Concrete to container
container.bind(Concrete);

// binds Abstract class to its implementation
container.bind(Abstract, Concrete);

// binds Abstract class to its implementation with extra constructor arguments
container.bind(Abstract, Concrete, [1, 10]);

// binds factory for resolving Abstract
container.bindFactory(Abstract, (container: Container) => container.resolve(Concrete));

// binds Concrete as singleton
container.singleton(Concrete);

// binds Concrete as singleton implementation of Abstract
container.singleton(Abstract, Concrete);

// binds Concrete as singleton implementation of Abstract with extra constructor arguments
container.singleton(Abstract, Concrete, [1, 2]);

// binds factory of Concrete as singleton implementation of Abstract
container.singletonFactory(Abstract, (container: Container) => container.resolve(Concrete));

// binds instance as implementation of Abstract
container.instance(Abstract, container.resolve(Concrete));
```

## Dependency injection
DI is implemented with decorators
```
import { Injectable, Inject, InjectArg, InjectArgs } from 'ts-ioc-di';
```
All classes that should be resolved with DI must be decorated with `@Injectable`
```
@Injectable
class UserRepository { }
```
### Constructor injection
```
// users property will be auto-injected

@Injectable
class Authenticator {
  public constructor(
    private users: UserRepository
  ) { }
}
```
#### Constructor injection with argument injection
```
interface Repository { }

// ...

@Injectable
class Authenticator {
  public constructor(
    @InjectArg(UserRepository) private users: Repository
  ) { }
}
```

### Property injection
```
@Injectable class UserRepository { }

// ...

class Authenticator {
  @Inject()
  private users?: UserRepository;
}
```
#### Property injection with injected class specification

```
interface Repository { }

// ...

@Injectable class UserRepository implements Repository { }

// ...

class Authenticator {
  @Inject(UserRepository)
  private users?: Repository;
}
```

### Method injection
```
class ViewModel {
  @InjectArgs()
  public created(handler: CreatedHandler): void {
    handler.onEvent(this);
  }
}
```
#### Method injection with argument injection
```
interface EventHandler { }

// ...

@Injectable class CreatedHandler implements EventHandler { }

// ...

class ViewModel {
  @InjectArgs()
  public created(@InjectArg(CreatedHandler) handler: EventHandler): void {
    handler.onEvent(this);
  }
}
```

### Autowiring

If you want, you can create classes which dependencies are resolved automatically after regular instantiation

```
@Autowired()
class Authenticator {
  @Inject()
  private users: UserService;
}

// UserService is automatically injected
const authenticator = new Authenticator();
```

But there are some tricks behind this behavior

#### Autowiring containers setting

By default, autowiring uses last instantiated container to resolve dependencies, but you can ovverride it

```
import { autowiredBuilder, Container } from 'ts-ioc-di';

// This container instance is used for Autowired to resolve deps
autowiredBuilder.setDefaultContainer(new Container());
```

Also, you can use specific containers for corresponding classes

```
import { autowiredBuilder, Container } from 'ts-ioc-di';

class A { }
class B { }

// This overrides usage of default container
autowiredBuilder.setContainer(A, new Container());
autowiredBuilder.setContainer(B, new Container());
```

#### Autowiring and constructor injection

You can use any type of DI that is described above, but constructor injection is a bit tricky with autowiring.
A following mechanism to control it exists.

```
import { Autowired } from 'ts-ioc-di';

@Autowired(let useConstructorInjection = true)
class Authenticator {
  public constructor(
    private users: UserService,
    private ...rest: Array<any>
  ) { }
}
```
In this case constructor injection is enabled and other arguments passed to constructor are in the rest.

```
import { Autowired } from 'ts-ioc-di';

@Autowired(let useConstructorInjection = false)
class Authenticator {
  public constructor(
    private ...rest: Array<any>
  ) { }
}
```
In that case arguments passed to constructor are directly forwarded to `Authenticator`.

Default behavior is to not use constructor injection with autowiring, because it may be quite misleading. 

## Low-level API
Low-level api is represented by `InstanceBuilder` and `InstanceBuilderFactory` which will return you an instance of `InstanceBuilder`.

```
import { Container } from 'ts-ioc-di';

const container = new Container();

class UserService { }

// ...

import { InstanceBuilderFactory } from 'ts-ioc-di';

const instanceBuilder = InstanceBuilderFactory.create(UserService, container);
const extraConstructorArguments = [1, 2, 3];

const userService = instanceBuilder
  .createInstance(extraConstructorArguments) // .setProduct(instance)
  .injectProperties()
  .injectMethods()
  .getProduct()

```
As you can see, you can build classes with that API without directly resolving them from container.
This may be useful for integration with libraries or writing your own decorators to extend DI possibilities.

## Extra
### Aliases
If you want to use one class as alias for another - you are welcome.
```
import { Container, Injectable } from 'ts-ioc-di';

@Injectable class A { }
@Injectable class B { }
@Injectable class C { }

const container = new Container();

// This is how aliases are registered
container.instance(A, new A());
container.bind(B, A);
container.bind(C, B);

// Instance of A which was registered before is resolved
container.resolve(C);
```
### Primitives
If you really want to bind a primitive value to container, you have the following option
```
import { Container, Injectable } from 'ts-ioc-di';

@Injectable class SomeImportantToken extends String { }

const container = new Container();

container.instance(SomeImportantToken, 'VALUE');
```
You can use the same trick with, e.g. `Number`
```
@Injectable class VeryImportantNumber extends Number { }

container.instance(VeryImportantNumber, Math.random());
```
And then you can use these classes as dependencies for DI
```
@Injectable
class Test {
  public constructor(
    private token: SomeImportantToken,
    private number: SomeImportantNumber
  ) { }
}
```

### Interfaces
If you want to bind an interface, you can't - there are no interfaces at runtime.
```
interface Service { }
class UserService implements Service {}

// Error: Service only refers to a type but is being used as a value here.
container.bind(Service, UserService);
```
You should use abstract classes instead of them.
```
abstract class Service { }
class UserService extends Service {}

// Works!
container.bind(Service, UserService);
```
### Argument injection
Methods which are decorated with `@InjectArgs` can accept regular arguments too.
```
class ViewModel {
  // otherArg and otherArgs are listed after injected arguments
  @InjectArgs()
  public created(handler?: CreatedHandler, otherArg: any, ...otherArgs: Array<any>): void {
    handler.onEvent(this);
  }
}
```

### Memento
If you need to manipulate with particular states of container, you can use memento for it.
```
const container = new Container();

container.bind(Abstract, Concrete);

// Internal state is saved into memento
const memento = container.save();

container.unbind(Abstract);
container.restore(memento);

// Internal state is restored, so Abstract can be resolved 
container.resolve(Abstract);
```
