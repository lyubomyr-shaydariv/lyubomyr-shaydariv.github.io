.. title: Sequential Mockito verifications refactoring with fluent interfaces
.. date: 2016-09-23 18:41:00 +0300
.. tags: java, mockito, java-8
.. category: java
.. type: text

> This article was originally written in Russian and published on September 12, 2016 at [Habrahabr](https://habrahabr.ru/post/309752/)

Static methods have one powerful and not very desirable feature: they can be invoked from any place of the code, and they also can't really dictate the order of such invocations.
Usually such control is really important, but sometimes the real order does not make much sense.
For example, assertions and verifications in unit tests often do not require to be executed in strict order.
To ensure that all verifications are done, [Mockito](http://mockito.org/) provides a static method named `verifyNoMoreInteractions(...)`.
Sometimes, accidentally, this method can be invoked before the last `verify(...)` is invoked so a bitter "red" test will be the result.
But what if delegate the verification order check to the compiler?

<!-- TEASER_END -->

Let's consider that a unit has the following test:

```java
public abstract class AbstractStructuredLoggingTest<T> {

	private final IStructuredLogger mockStructuredLogger = mock(IStructuredLogger.class);

	private T unit;

	@Nonnull
	protected abstract T createUnit(@Nonnull IStructuredLogger logger);

	protected final IStructuredLogger getMockStructuredLogger() {
		return mockStructuredLogger;
	}

	protected final T getUnit() {
		return unit;
	}

	@Before
	public void initializeMockStructuredLogger() {
		// Setting up the mock, `selfAnswer()` -- is a Mockito answer returning the mock itself
		when(mockStructuredLogger.begin()).thenAnswer(selfAnswer());
		when(mockStructuredLogger.put(any(LogEntryKey.class), any(Object.class))).thenAnswer(selfAnswer());
		when(mockStructuredLogger.log(any(Scope.class), any(Severity.class), any(String.class))).thenAnswer(selfAnswer());
		when(mockStructuredLogger.end()).thenAnswer(selfAnswer());
		// Injecting the logger to the unit (at least, formally). It doesn't really matter how it is under the hood
		unit = createUnit(mockStructuredLogger);
	}

	@After
	public void resetMockStructuredLogger() {
		try {
			// Verify that are no more unverified mock interactions
			verifyNoMoreInteractions(mockStructuredLogger);
		} finally {
			// Just in case, reset the mock since the errors would affect as a cascade...
			reset(mockStructuredLogger);
		}
	}

}
```

```java
public final class AdministratorServiceStructuredLoggingTest
		extends AbstractStructuredLoggingTest<IAdministratorService> {

	private static final String USERNAME = "john.doe";
	private static final String PASSWORD = "opZK2lkXa";
	private static final String FIRST_NAME = "john";
	private static final String LAST_NAME = "doe";
	private static final String EMAIL = "john.doe@acme.com";

	@Nonnull
	protected IAdministratorService createUnit(@Nonnull final IStructuredLogger logger) {
		return createAdministratorService(logger);
	}
	
	@Test
	public void testCreate() {
		final T unit = getUnit();
		unit.create(USERNAME, PASSWORD, FIRST_NAME, LAST_NAME, EMAIL);
		final IStructuredLogger mockStructuredLogger = getMockStructuredLogger();
		verify(mockStructuredLogger).put(eq(OPERATION_CALLER_CLASS), any(IAdministratorService.class));
		verify(mockStructuredLogger).put(eq(OPERATION_CALLER_METHOD), any(Method.class));
		verify(mockStructuredLogger).put(eq(OPERATION_TYPE), eq(CREATE));
		verify(mockStructuredLogger).put(eq(OPERATION_OBJECT_TYPE), eq(ADMINISTRATOR));
		verify(mockStructuredLogger).put(eq(VALUE_ADMINISTRATOR_NAME), eq(USERNAME));
		verify(mockStructuredLogger).put(eq(VALUE_FIRST_NAME), eq(FIRST_NAME));
		verify(mockStructuredLogger).put(eq(VALUE_LAST_NAME), eq(LAST_NAME));
		verify(mockStructuredLogger).put(eq(VALUE_EMAIL), eq(EMAIL));
		verify(mockStructuredLogger).log(eq(APP_DEV), eq(INFO), any(String.class));
		verifyNoMoreInteractions(mockStructuredLogger);
	}

}
```

It's not hard to get what is tested in the test: it just tests if the given unit has invoked all important method from the structured logger.
Since the test ends with `verifyNoMockInteractions(...)`, it's ensured that the mock does not have unverified methods.
By the way, the structured logger interface is extremely straight-forware, but I'll put it somewhat cut because it was taken from a real project.

```java
public interface IStructuredLogger {

	// This method does not participate the testing, but it makes sense because it can mark the very beginning and the very end of a logged message.
	@Nonnull
	IStructuredLogger begin()
			throws IllegalStateException;

	// Fills up a logged message that should be logged.
	// key -- an enumeration (enum) of all possible logged message keys (OPERATION_CALLER_CLASS, VALUE_FIRST_NAME и т.д.)
	// value -- arbitrary argument
	@Nonnull
	IStructuredLogger put(@Nonnull LogEntryKey key, @Nullable Object value)
			throws IllegalStateException;

	// Writes the accumulated message to a log.
	// scope -- log type enumeration (for example, APP_DEV -- defines a log entry both for user and dev logs)
	// severity -- another logging threshold enumeration (ERROR, INFO and so forth)
	// message -- arbitrary human-readable message
	@Nonnull
	IStructuredLogger log(@Nonnull Scope scope, @Nonnull Severity severity, @Nonnull String message)
			throws IllegalStateException;

	// The begin() counter-part
	@Nonnull
	IStructuredLogger end()
			throws IllegalStateException;

}
```

As it was said above, static methods fill up the test above, but does not guarantee that verifications of all kinds are performed.
And, for sure, such a test can fail.
I assume that the following criteria are successful if it's possible to:

* ... determine what class is a log event generator;
* ... determine the log event object and subject;
* ... determine the logged action arguments;
* ... determine the target logs;
* ... and check if the structured logged is not used for anything else.

Actually, these are well-defined requirements that should be executed in a strict order.
To resolve this issue, one might use the [Strategy](https://en.wikipedia.org/wiki/Strategy_pattern) design pattern that could declare an interface for every type of check, and each method would test its own loggin aspect.
[Template method](https://en.wikipedia.org/wiki/Template_method_pattern) is an alternative
But it's obiviously that such approachs are pretty cumbersome and are not very robust in separating the logging aspects across the methods.
This would also sacrifice the readability and this is what I would not do.

About five years ago I came across an article that describes a [Builder](https://en.wikipedia.org/wiki/Builder_pattern) design pattern implementation that really guaranteed that building a complex object will be done in strict order.
Shortly speaking, some builder object required the `setFoo()` method to be invoked first, then the next method would be `setBar()`, and the final method is `build()` only.
And any other order is prohibited because the order is controlled by the compiler!

A similar approach, that's implemented in a slightly different way, could be used to simplify the tests that must be executed in strict order, and this wouldn't require a template method imlpementation.
Having formal criteria described above, we could create a set of interfaces that could concatenate such transitions.
[Fluent interface](https://ru.wikipedia.org/wiki/Fluent_interface) is a convenient way to define a graceful test chain.

```java
// This step verifies which unit and which unit method a log message is associated to
@FunctionalInterface
public interface IOperationCallerVerificationStep {

	// unitMatcherSupplier -- returns a unit
	// methodMatcherSupplier -- returns a method that is expected to invoke a log
	// This method makes a few checks, and if there were no errors, then it creates an object for the next step
	@Nonnull
	IOperationTypeVerificationStep withOperationCaller(
			@Nonnull Supplier<?> unitMatcherSupplier,
			@Nonnull Supplier<Method> methodMatcherSupplier
	);

	// By default, let's consider that we don't really care about the method that invoked the logger. This is
	// pretty controversial, because it may even damage the idea of unit tests here. But for the systems with
	// automatic logging (using an annotation processing mechanism, not to be confused with [APT](http://docs.oracle.com/javase/7/docs/technotes/guides/apt/))
	// it does not really matter. At least, logs automatic generation allows to create the method object
	// in convenient way, and that is not that robust as manual logging.
	@Nonnull
	default IOperationTypeVerificationStep withOperationCaller(
			@Nonnull final Supplier<?> unitMatcherSupplier
	) {
		return withOperationCaller(unitMatcherSupplier, () -> any(Method.class));
	}

}
```

```java
// This step verifies the object and the subject
@FunctionalInterface
public interface IOperationTypeVerificationStep {

	// operationTypeMatcherSupplier -- returns the type of an operation
	// objectTypeMatcherSupplier -- returns athe type of an object that must be logged
	@Nonnull
	IValueVerificationStep withOperationType(
			@Nonnull Supplier<OperationType> operationTypeMatcherSupplier,
			@Nonnull Supplier<ObjectType> objectTypeMatcherSupplier
	);

}
```

```java
// A step verifying the log context data (those that were passed to the unit)
public interface IValueVerificationStep {

	// logEntryKeyMatcherSupplier -- возвращает тип ключа для структурированного сообщения
	// valueMatcherSupplier -- arbitrary argument
	// By the way, this method does not return a next step object, because it's variadic thus a few
	// invocations might be executed step by step since the number of such steps is not known before.
	@Nonnull
	IValueVerificationStep withValue(
			@Nonnull Supplier<LogEntryKey> logEntryKeyMatcherSupplier,
			@Nonnull Supplier<?> valueMatcherSupplier
	);

	// A pretty synthetic method. The only need is making a transition to the next step.
	@Nonnull
	ILogVerificationStep then();

}
```

```java
// The final step that verifies to which log a message was wrote to and what's the log level
@FunctionalInterface
public interface ILogVerificationStep {

	// scopeMatcherSupplier -- the logger type
	// severityMatcherSupplier -- the log message level/severity
	// messageMatcherSupplier -- arbitrary message
	// This is the final verification so this method does not need to return a value
	void withLog(
			@Nonnull Supplier<Scope> scopeMatcherSupplier,
			@Nonnull Supplier<Severity> severityMatcherSupplier,
			@Nonnull Supplier<String> messageMatcherSupplier
	);

	// We don't care the message here
	default void withLog(
			@Nonnull final Supplier<Scope> scopeMatcherSupplier,
			@Nonnull final Supplier<Severity> severityMatcherSupplier
	) {
		withLog(scopeMatcherSupplier, severityMatcherSupplier, () -> any(String.class));
	}

	// And this method considers that a log message is written to two loggers (APP and DEV)
	default void withLog(
			@Nonnull final Supplier<Severity> severityMatcherSupplier
	) {
		withLog(() -> eq(APP_DEV), severityMatcherSupplier, () -> any(String.class));
	}

}
```

Almost all interfaces are marked with `@FunctionalInterface`, but it's not a requirement though.
Nevertheless, the "variadic" interface has two methods, because there should be a way of terminating the logged operation arguments checks.
So, the original test code now can transform into:

```java
public abstract class AbstractStructuredLoggingTest<T> {

	private final IStructuredLogger mockStructuredLogger = mock(IStructuredLogger.class);

	private T unit;

	@Nonnull
	protected abstract T createUnit(@Nonnull IStructuredLogger logger);

	// Surprise-surprise! The method is no longer necessary, because all checks are encapsulated in this class.
	/*protected final IStructuredLogger getMockStructuredLogger() {
		return mockStructuredLogger;
	}*/

	protected final T getUnit() {
		return unit;
	}

	@Before
	public void initializeMockStructuredLogger() {
		when(mockStructuredLogger.begin()).thenAnswer(selfAnswer());
		when(mockStructuredLogger.put(any(LogEntryKey.class), any(Object.class))).thenAnswer(selfAnswer());
		when(mockStructuredLogger.log(any(Scope.class), any(Severity.class), any(String.class))).thenAnswer(selfAnswer());
		when(mockStructuredLogger.end()).thenAnswer(selfAnswer());
		unit = createUnit(mockStructuredLogger);
	}

	@After
	public void resetMockStructuredLogger() {
		try {
			verifyNoMoreInteractions(mockStructuredLogger);
		} finally {
			reset(mockStructuredLogger);
		}
	}

	// This is the method taht creates the very first step of logs verification. It encapsulates all
	// necessary checks. The code looks somewhat clumsy, but it works really good. Lambda expressions
	// can make the things somewhat simpler. Before a next verification step is invoked, the verify(...)
	// methods verify the mock. The last step does not invoke verifyNoMoreInteractions because the latter
	// is invoked for every test automatically.
	protected final IOperationCallerVerificationStep verifyLog() {
		return (unitMatcherSupplier, methodMatcherSupplier) -> {
			verify(mockStructuredLogger).put(eq(OPERATION_CALLER_CLASS), unitMatcherSupplier.get());
			verify(mockStructuredLogger).put(eq(OPERATION_CALLER_METHOD), methodMatcherSupplier.get());
			return (IOperationTypeVerificationStep) (operationTypeMatcherSupplier, objectTypeMatcherSupplier) -> {
				verify(mockStructuredLogger).put(eq(OPERATION_TYPE), operationTypeMatcherSupplier.get());
				verify(mockStructuredLogger).put(eq(OPERATION_OBJECT_TYPE), objectTypeMatcherSupplier.get());
				return new IValueVerificationStep() {
					@Nonnull
					@Override
					public IValueVerificationStep withValue(@Nonnull final Supplier<LogEntryKey> logEntryKeyMatcherSupplier,
							@Nonnull final Supplier<?> valueMatcherSupplier) {
						verify(mockStructuredLogger).put(logEntryKeyMatcherSupplier.get(), valueMatcherSupplier.get());
						return this;
					}

					@Nonnull
					@Override
					public ILogVerificationStep then() {
						return (scopeMatcherSupplier, severityMatcherSupplier, messageMatcherSupplier) -> verify(mockStructuredLogger).log(scopeMatcherSupplier.get(), severityMatcherSupplier.get(), messageMatcherSupplier.get());
					}
				};
			};
		};
	}

}
```

So, here is how a concrete test gets simpler and why the base test got more complex:

```java
public final class AdministratorServiceStructuredLoggingTest
		extends AbstractStructuredLoggingTest {

	private static final String USERNAME = "usr";
	private static final String PASSWORD = "qwerty";
	private static final String FIRST_NAME = "john";
	private static final String LAST_NAME = "doe";
	private static final String EMAIL = "usr@mail.com";

	@Nonnull
	protected IAdministratorService createUnit(@Nonnull final IStructuredLogger logger) {
		return createAdministratorService(logger);
	}

	@Test
	public void testCreate() {
		final T unit = getUnit();
		unit.create(USERNAME, PASSWORD, FIRST_NAME, LAST_NAME, EMAIL);
		verifyLog()
				.withOperationCaller(() -> any(IAdministratorService.class))
				.withOperationType(() -> eq(CREATE), () -> eq(ADMINISTRATOR))
				.withValue(() -> eq(VALUE_ADMINISTRATOR_NAME), () -> eq(USERNAME))
				.withValue(() -> eq(VALUE_FIRST_NAME), () -> eq(FIRST_NAME))
				.withValue(() -> eq(VALUE_LAST_NAME), () -> eq(LAST_NAME))
				.withValue(() -> eq(VALUE_EMAIL), () -> eq(EMAIL))
				.then()
				.withLog(() -> eq(INFO));
	}

}
```

I believe, the code is now more robust and pretty nicer.
And somewhat convenient as well.
And one of the main points -- any smart IDE can suggest the next step once the dot is pressed.
Thus, both compiler and IDE make some more confidence for the well-written test.
By the way, why is `Supplier` necessary along with the lambda expressions?
The thing is that Mockito detects if an object passed to a mock is a stub, and if it is not -- then it throws an exception.
Actually, as far as I know, Mockito check rules are more complex, and Mockito, for example, ignores anonymous classes.
And in the light of this fact, there is a little loophole: Mocktio does not track matchers passed via `return` making lambda expressions possible here.
This affects code and readability a little bit, but lambda expressions deal with it pretty good.

So we've got the following results:

* more look-alike tests;
* every next step of a test defines a step afterwards, and it's supported by compilers and IDEs really good, and this is not possible using static methods (at least in its original form);
* test initialization and their subsequent execution is performed in the abstract test, and a concrete test merely describes the verifications not even interacting with the original unit.
