.. title: Spring Security @PreAuthorize annotation custom types and inspectable DSL support
.. date: 2016-08-07 00:00:00 +0300
.. tags: java, spring-framework
.. category: java
.. type: text

> This article was originally written in Russian and published on August 11, 2016 at [Habrahabr](https://habrahabr.ru/post/307558/)

[Spring Security](http://projects.spring.io/spring-security/) is a must-have component for Spring applications as it's responsible for user authentication and system activity authorization.
The use of `@PreAuthorize` is one of Spring Security methods allowing to define some authorization rules easily, and these rules can grant or deny some operation for a particular user.

The REST service I currently develop has to provide an endpoint to list all controller methods authorization rules.
And, if possible, avoid revealing the specifics of [SpEL](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/expressions.html) expressions (so, use something like `anybody` instead of `permitAll`; avoid `principal` at all as an excessive expression), but return custom expressions we can process whatever we want.

<!-- TEASER_END -->

Let's start!
---

Let's create a small service and define some authorization rules for the service:

* `IGreetingService.java` -- this interface defines a small service with a minimum operations

```java
public interface IGreetingService {

	@Nonnull
	String sayHelloTo(@Nonnull String name);

	@Nonnull
	String sayGoodByeTo(@Nonnull String name);

}
```

* `GreetingService.java` -- probably the simplest service implementation that also features service methods authorization rules using the `@PreAuthorize` annotation

```java
@Service
public final class GreetingService
		implements IGreetingService {

	@Override
	@Nonnull
	@PreAuthorize("@A.maySayHelloTo(principal, #name)")
	public String sayHelloTo(
			@P("name") @Nonnull final String name
	) {
		return "hello " + name;
	}

	@Nonnull
	@Override
	@PreAuthorize("@A.maySayGoodByeTo(principal, #name)")
	public String sayGoodByeTo(
			@P("name") @Nonnull final String name
	) {
		return "good bye" + name;
	}

}
```

* `IAuthorizationComponent.java` -- a simple interface as well featuring a few authorization rules

```java
public interface IAuthorizationComponent {

	boolean maySayHelloTo(@Nonnull UserDetails principal, @Nonnull String name);

	boolean maySayGoodByeTo(@Nonnull UserDetails principal, @Nonnull String name);

}
```

* `AuthorizationComponent.java` -- the authorization rules implementation (we might also use more complex rules respecting input parameters instead of `true` and `false`, but we'll take a look below)

```java
@Component("A")
public final class AuthorizationComponent
		implements IAuthorizationComponent {

	@Override
	public boolean maySayHelloTo(@Nonnull final UserDetails principal, @Nonnull final String name) {
		return true;
	}

	@Override
	public boolean maySayGoodByeTo(@Nonnull final UserDetails principal, @Nonnull final String name) {
		return false;
	}

}
```

From `boolean` to expressions
---

A boolean expression result cannot provide the rule expression definition.
We should define how a rule that can affect authorization somehow.
Let's say, we decide to use an rules describing object rather than to use `boolean`.
Such an object must only meet two requirements:

* it must be able to affect authorization process, i.e. just to return a boolean value allowing or denying an operation;
* it must be able to be converted to a human-readable text view (although it can be also converted to another a more machine-readable view, but this is unnecessary so far).

According to the requirements we only need something like:

```java
public interface IAuthorizationExpression {

	boolean mayProceed();

	@Nonnull
	String toHumanReadableExpression();

}
```

And then just change the authorization component:

```java
public interface IAuthorizationComponent {

	@Nonnull
	IAuthorizationExpression maySayHelloTo(@Nonnull UserDetails principal, @Nonnull String name);

	@Nonnull
	IAuthorizationExpression maySayGoodByeTo(@Nonnull UserDetails principal, @Nonnull String name);

}
```

```java
@Component("A")
public final class AuthorizationComponent
		implements IAuthorizationComponent {

	@Nonnull
	@Override
	public IAuthorizationExpression maySayHelloTo(@Nonnull final UserDetails principal, @Nonnull final String name) {
		return simpleAuthorizationExpression(true);
	}

	@Nonnull
	@Override
	public IAuthorizationExpression maySayGoodByeTo(@Nonnull final UserDetails principal, @Nonnull final String name) {
		return simpleAuthorizationExpression(true);
	}

}
```

* `SimpleAuthorizationExpression.java` -- the simplest expression that depends on a single boolean value

```java
public final class SimpleAuthorizationExpression
		implements IAuthorizationExpression {

	private static final IAuthorizationExpression mayProceedExpression = new SimpleAuthorizationExpression(true);
	private static final IAuthorizationExpression mayNotProceedExpression = new SimpleAuthorizationExpression(false);

	private final boolean mayProceed;

	private SimpleAuthorizationExpression(final boolean mayProceed) {
		this.mayProceed = mayProceed;
	}

	public static IAuthorizationExpression simpleAuthorizationExpression(final boolean mayProceed) {
		return mayProceed ? mayProceedExpression : mayNotProceedExpression;
	}

	public boolean mayProceed() {
		return mayProceed;
	}

	@Nonnull
	public String toHumanReadableExpression() {
		return mayProceed ? "TRUE" : "FALSE";
	}

}
```

Unfortunately, by default, `@PreAuthorize` expressions can only return boolean values.
So we shall get the following exception trying to invoke the service methods:

```
Exception in thread "main" java.lang.IllegalArgumentException: Failed to evaluate expression '@A.maySayHelloTo(principal, #name)'
	at org.springframework.security.access.expression.ExpressionUtils.evaluateAsBoolean(ExpressionUtils.java:30)
	at org.springframework.security.access.expression.method.ExpressionBasedPreInvocationAdvice.before(ExpressionBasedPreInvocationAdvice.java:59)
	at org.springframework.security.access.prepost.PreInvocationAuthorizationAdviceVoter.vote(PreInvocationAuthorizationAdviceVoter.java:72)
	at org.springframework.security.access.prepost.PreInvocationAuthorizationAdviceVoter.vote(PreInvocationAuthorizationAdviceVoter.java:40)
	at org.springframework.security.access.vote.AffirmativeBased.decide(AffirmativeBased.java:63)
	at org.springframework.security.access.intercept.AbstractSecurityInterceptor.beforeInvocation(AbstractSecurityInterceptor.java:233)
	at org.springframework.security.access.intercept.aopalliance.MethodSecurityInterceptor.invoke(MethodSecurityInterceptor.java:65)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179)
	at org.springframework.aop.framework.JdkDynamicAopProxy.invoke(JdkDynamicAopProxy.java:213)
	at com.sun.proxy.$Proxy38.sayHelloTo(Unknown Source)
	at test.springfx.security.app.Application.lambda$main$0(Application.java:23)
	at test.springfx.security.app.Application$$Lambda$7/2043106095.run(Unknown Source)
	at test.springfx.security.fakes.FakeAuthentication.withFakeAuthentication(FakeAuthentication.java:32)
	at test.springfx.security.app.Application.main(Application.java:23)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:497)
	at com.intellij.rt.execution.application.AppMain.main(AppMain.java:144)
Caused by: org.springframework.expression.spel.SpelEvaluationException: EL1001E:(pos 0): Type conversion problem, cannot convert from @javax.annotation.Nonnull test.springfx.security.app.auth.IAuthorizationExpression$1 to java.lang.Boolean
	at org.springframework.expression.spel.support.StandardTypeConverter.convertValue(StandardTypeConverter.java:78)
	at org.springframework.expression.common.ExpressionUtils.convertTypedValue(ExpressionUtils.java:53)
	at org.springframework.expression.spel.standard.SpelExpression.getValue(SpelExpression.java:301)
	at org.springframework.security.access.expression.ExpressionUtils.evaluateAsBoolean(ExpressionUtils.java:26)
	... 18 more
Caused by: org.springframework.core.convert.ConverterNotFoundException: No converter found capable of converting from type [@javax.annotation.Nonnull test.springfx.security.app.auth.IAuthorizationExpression$1] to type [java.lang.Boolean]
	at org.springframework.core.convert.support.GenericConversionService.handleConverterNotFound(GenericConversionService.java:313)
	at org.springframework.core.convert.support.GenericConversionService.convert(GenericConversionService.java:195)
	at org.springframework.expression.spel.support.StandardTypeConverter.convertValue(StandardTypeConverter.java:74)
	... 21 more
```

Configuring `@PreAuthorize`
---

Firstly, to overcome the non-boolean values issue we have to configure `GlobalMethodSecurityConfiguration` since it allows to configure the evaluation context for `@PreAuthorize`.
`TypeConverter` can help and it can be reached pretty easily:

```java
public abstract class CustomTypesGlobalMethodSecurityConfiguration
		extends GlobalMethodSecurityConfiguration {

	protected abstract ApplicationContext applicationContext();

	protected abstract ConversionService conversionService();

	@Override
	protected MethodSecurityExpressionHandler createExpressionHandler() {
		final ApplicationContext applicationContext = applicationContext();
		final TypeConverter typeConverter = new StandardTypeConverter(conversionService());
		final DefaultMethodSecurityExpressionHandler handler = new DefaultMethodSecurityExpressionHandler() {
			@Override
			public StandardEvaluationContext createEvaluationContextInternal(final Authentication authentication, final MethodInvocation methodInvocation) {
				final StandardEvaluationContext decoratedStandardEvaluationContext = super.createEvaluationContextInternal(authentication, methodInvocation);
				return new ForwardingStandardEvaluationContext() {
					@Override
					protected StandardEvaluationContext standardEvaluationContext() {
						return decoratedStandardEvaluationContext;
					}

					@Override
					public TypeConverter getTypeConverter() {
						return typeConverter;
					}
				};
			}
		};
		handler.setApplicationContext(applicationContext);
		return handler;
	}

}
```

There should be a few notes.
First, we shall use standard `DefaultMethodSecurityExpressionHandler` that makes the business itself but we shall just override the returned context.
Second, the `DefaultMethodSecurityExpressionHandler` parent, `AbstractSecurityExpressionHandler`, does not let to create own context, but it's possible to create a so called internal context that will provide `TypeConverter` itself.
Third, we need a forwarding decorator for `StandardEvaluationContext` in order not to break the original context behavior:

* `ForwardingStandardEvaluationContext.java` -- `StandardEvaluationContext` forwarding decorator

```java
public abstract class ForwardingStandardEvaluationContext
		extends StandardEvaluationContext {

	protected abstract StandardEvaluationContext standardEvaluationContext();

	// @formatter:off
	@Override public void setRootObject(final Object rootObject, final TypeDescriptor typeDescriptor) { standardEvaluationContext().setRootObject(rootObject, typeDescriptor); }
	@Override public void setRootObject(final Object rootObject) { standardEvaluationContext().setRootObject(rootObject); }
	@Override public TypedValue getRootObject() { return standardEvaluationContext().getRootObject(); }
	@Override public void addConstructorResolver(final ConstructorResolver resolver) { standardEvaluationContext().addConstructorResolver(resolver); }
	@Override public boolean removeConstructorResolver(final ConstructorResolver resolver) { return standardEvaluationContext().removeConstructorResolver(resolver); }
	@Override public void setConstructorResolvers(final List<ConstructorResolver> constructorResolvers) { standardEvaluationContext().setConstructorResolvers(constructorResolvers); }
	@Override public List<ConstructorResolver> getConstructorResolvers() { return standardEvaluationContext().getConstructorResolvers(); }
	@Override public void addMethodResolver(final MethodResolver resolver) { standardEvaluationContext().addMethodResolver(resolver); }
	@Override public boolean removeMethodResolver(final MethodResolver methodResolver) { return standardEvaluationContext().removeMethodResolver(methodResolver); }
	@Override public void setMethodResolvers(final List<MethodResolver> methodResolvers) { standardEvaluationContext().setMethodResolvers(methodResolvers); }
	@Override public List<MethodResolver> getMethodResolvers() { return standardEvaluationContext().getMethodResolvers(); }
	@Override public void setBeanResolver(final BeanResolver beanResolver) { standardEvaluationContext().setBeanResolver(beanResolver); }
	@Override public BeanResolver getBeanResolver() { return standardEvaluationContext().getBeanResolver(); }
	@Override public void addPropertyAccessor(final PropertyAccessor accessor) { standardEvaluationContext().addPropertyAccessor(accessor); }
	@Override public boolean removePropertyAccessor(final PropertyAccessor accessor) { return standardEvaluationContext().removePropertyAccessor(accessor); }
	@Override public void setPropertyAccessors(final List<PropertyAccessor> propertyAccessors) { standardEvaluationContext().setPropertyAccessors(propertyAccessors); }
	@Override public List<PropertyAccessor> getPropertyAccessors() { return standardEvaluationContext().getPropertyAccessors(); }
	@Override public void setTypeLocator(final TypeLocator typeLocator) { standardEvaluationContext().setTypeLocator(typeLocator); }
	@Override public TypeLocator getTypeLocator() { return standardEvaluationContext().getTypeLocator(); }
	@Override public void setTypeConverter(final TypeConverter typeConverter) { standardEvaluationContext().setTypeConverter(typeConverter); }
	@Override public TypeConverter getTypeConverter() { return standardEvaluationContext().getTypeConverter(); }
	@Override public void setTypeComparator(final TypeComparator typeComparator) { standardEvaluationContext().setTypeComparator(typeComparator); }
	@Override public TypeComparator getTypeComparator() { return standardEvaluationContext().getTypeComparator(); }
	@Override public void setOperatorOverloader(final OperatorOverloader operatorOverloader) { standardEvaluationContext().setOperatorOverloader(operatorOverloader); }
	@Override public OperatorOverloader getOperatorOverloader() { return standardEvaluationContext().getOperatorOverloader(); }
	@Override public void setVariable(final String name, final Object value) { standardEvaluationContext().setVariable(name, value); }
	@Override public void setVariables(final Map<String, Object> variables) { standardEvaluationContext().setVariables(variables); }
	@Override public void registerFunction(final String name, final Method method) { standardEvaluationContext().registerFunction(name, method); }
	@Override public Object lookupVariable(final String name) { return standardEvaluationContext().lookupVariable(name); }
	@Override public void registerMethodFilter(final Class<?> type, final MethodFilter filter) throws IllegalStateException { standardEvaluationContext().registerMethodFilter(type, filter); }
	// @formatter:on

}
```

Yes, Java is overly verbose, and it would be great if Java could have something like [by](https://kotlinlang.org/docs/reference/delegation.html) in Kotlin in order to avoid too much code.
Lombok [@Delegate](https://projectlombok.org/features/Delegate.html) might be an option too.
Also, a delegated field might be used rather than an abstracted method, but I think that the abstract method is slightly more flexible (however I'm not sure if Kotlin and Lombok can delegate to decorated object supplier method).

The two classes above, I think, should be at the "library" layer, so that it could be used and configured in another applications.
And now, in the application layer, we can easily create a converter to map `IAuthorizationExpression` to `boolean`:

* `SecurityConfiguration.java` -- we just bind the application context and the conversion service here

```java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true, jsr250Enabled = false)
public class SecurityConfiguration
		extends CustomTypesGlobalMethodSecurityConfiguration {

	private final ApplicationContext applicationContext;
	private final ConversionService conversionService;

	public SecurityConfiguration(
			@Autowired final ApplicationContext applicationContext,
			@Autowired final ConversionService conversionService
	) {
		this.applicationContext = applicationContext;
		this.conversionService = conversionService;
	}

	@Override
	protected ApplicationContext applicationContext() {
		return applicationContext;
	}

	@Override
	protected ConversionService conversionService() {
		return conversionService;
	}

}
```

* `ConversionConfiguration.java` -- and this is just a configuration adding a new converter to the list of existing ones

```java
@Configuration
public class ConversionConfiguration {

	@Bean
	public ConversionService conversionService() {
		final DefaultConversionService conversionService = new DefaultConversionService();
		conversionService.addConverter(IAuthorizationExpression.class, Boolean.class, IAuthorizationExpression::mayProceed);
		return conversionService;
	}

}
```

Now `IAuthorizationExpression` can work as `boolean` and be an operand type for the logical operations.

More complex expressions
---

Since we have an expression object, we can enhance its base functionality and create expressions that are more complex than `SimpleAuthorizationExpression`.
It allows to combine expressions of any complexity still following the requirements defined earlier.

I've always liked fluent interfaces which methods could be composed in convenient way.
For example [Mockito](http://mockito.org/) and [Hamcrest](http://hamcrest.org/) use such an approach with might and main.
And Java now also uses this approach for interfaces like `Function`, `Supplier`, `Consumer`, `Comparator` and so forth.
Java 8 brought many nice features, and one of them, default methods, can be used to enhance the default expressions features.
For example, we can add a simple AND predicate to `IAuthorizationExpression`:

```java
default IAuthorizationExpression and(final IAuthorizationExpression r) {
	return new IAuthorizationExpression() {
		@Override
		public boolean mayProceed() {
			return IAuthorizationExpression.this.mayProceed() && r.mayProceed();
		}

		@Nonnull
		@Override
		public String toHumanReadableExpression() {
			return new StringBuilder("(")
					.append(IAuthorizationExpression.this.toHumanReadableExpression())
					.append(" AND ")
					.append(r.toHumanReadableExpression())
					.append(')')
					.toString();
		}
	};
}
```

As seen, the AND operation can be implemented very easy.
The `mayProceed` method merely provides the composition of two expressions using `&&`, and the `toHumanReadableExpression` method generates a string representating this expression.
Now we can combine expressions using the AND operation, for example:

```
simpleAuthorizationExpression(true).and(simpleAuthorizationExpression(true))
```

And the string view of the code above is:

```
(TRUE AND TRUE)
```

Pretty good.
The OR and unary NOT operations support can be added without any issues.
Moreover, we can create even more complex expressions without trivial operations because `SimpleAuthorizationExpression` does not make much sense.
For example, an expression that determines if a given user is the root:

```java
public final class IsRootAuthorizationExpression
		implements IAuthorizationExpression {

	private final UserDetails userDetails;

	private IsRootAuthorizationExpression(final UserDetails userDetails) {
		this.userDetails = userDetails;
	}

	public static IAuthorizationExpression isRoot(final UserDetails userDetails) {
		return new IsRootAuthorizationExpression(userDetails);
	}

	@Override
	public boolean mayProceed() {
		return Objects.equals(userDetails.getUsername(), "root");
	}

	@Nonnull
	@Override
	public String toHumanReadableExpression() {
		return "isRoot";
	}

}
```

Or an expression that determines if the given variable `name` is banned:

```java
public final class IsNamePermittedAuthorizationExpression
		implements IAuthorizationExpression {

	private static final Collection<String> bannedStrings = emptyList();

	private final String name;

	private IsNamePermittedAuthorizationExpression(final String name) {
		this.name = name;
	}

	public static IAuthorizationExpression isNamePermitted(final String name) {
		return new IsNamePermittedAuthorizationExpression(name);
	}

	@Override
	public boolean mayProceed() {
		return !bannedStrings.contains(name.toLowerCase());
	}

	@Nonnull
	@Override
	public String toHumanReadableExpression() {
		return new StringBuilder()
			.append("(name NOT IN (")
			.append(bannedStrings.stream().collect(joining()))
			.append("))")
			.toString();
	}

}
```

Now the authorization rules can be defined as follows:

```java
@Nonnull
@Override
public IAuthorizationExpression maySayHelloTo(@Nonnull final UserDetails principal, @Nonnull final String name) {
	return isNamePermitted(name);
}

@Nonnull
@Override
public IAuthorizationExpression maySayGoodByeTo(@Nonnull final UserDetails principal, @Nonnull final String name) {
	return isRoot(principal).and(isNamePermitted(name));
}
```

The code is well readable and it's quite clear what the expressions above do.
And this is how string views look like:

* `(name NOT IN ())`
* `(isRoot AND (name NOT IN ()))`

Looks recognizable, right?

Converting `@PreAuthorize` to `IAuthorizationExpression` and its string view
---

And now the last remaining thing is to obtain these expressions during runtime.
Let's assume we can obtain the list of all methods we need (it can be really specific, but we don't really care so far).
Having such a set of service methods, we can simply run `@PreAuthorize` expression "virtually" just substituting the variables with some values.
For example:

```java
@Service
public final class DiscoverService
		implements IDiscoverService {

	private static final UserDetails userDetailsMock = (UserDetails) newProxyInstance(
			DiscoverService.class.getClassLoader(),
			new Class<?>[]{ UserDetails.class },
			(proxy, method, args) -> {
				throw new AssertionError(method);
			}
	);

	private static final Authentication authenticationMock = (Authentication) newProxyInstance(
			DiscoverService.class.getClassLoader(),
			new Class<?>[]{ Authentication.class },
			(proxy, method, args) -> {
				switch ( method.getName() ) {
				case "getPrincipal":
					return userDetailsMock;
				case "isAuthenticated":
					return true;
				default:
					throw new AssertionError(method);
				}
			}
	);

	private final ApplicationContext applicationContext;
	private final ConversionService conversionService;

	public DiscoverService(
			@Autowired final ApplicationContext applicationContext,
			@Autowired final ConversionService conversionService
	) {
		this.applicationContext = applicationContext;
		this.conversionService = conversionService;
	}

	@Override
	@Nullable
	public <T> String toAuthorizationExpression(@Nonnull final T object, @Nonnull final Class<? extends T> inspectType, @Nonnull final String methodName,
			@Nonnull final Class<?>... parameterTypes)
			throws NoSuchMethodException {
		final Method method = inspectType.getMethod(methodName, parameterTypes);
		final DefaultMethodSecurityExpressionHandler expressionHandler = createMethodSecurityExpressionHandler();
		final MethodInvocation invocation = createMethodInvocation(object, method);
		final EvaluationContext evaluationContext = createEvaluationContext(method, expressionHandler, invocation);
		final Object value = evaluate(method, evaluationContext);
		return resolveAsString(value);
	}

	private DefaultMethodSecurityExpressionHandler createMethodSecurityExpressionHandler() {
		final DefaultMethodSecurityExpressionHandler expressionHandler = new DefaultMethodSecurityExpressionHandler();
		expressionHandler.setApplicationContext(applicationContext);
		return expressionHandler;
	}

	private <T> MethodInvocation createMethodInvocation(@Nonnull final T object, final Method method) {
		final Parameter[] parameters = method.getParameters();
		return new SimpleMethodInvocation(object, method, Stream.of(parameters).map(<...>).toArray(Object[]::new));
	}

	private EvaluationContext createEvaluationContext(final Method method, final SecurityExpressionHandler<MethodInvocation> expressionHandler,
			final MethodInvocation invocation) {
		final EvaluationContext decoratedExpressionContext = expressionHandler.createEvaluationContext(authenticationMock, invocation);
		final TypeConverter typeConverter = new StandardTypeConverter(conversionService);
		return new ForwardingEvaluationContext() {
			@Override
			protected EvaluationContext evaluationContext() {
				return decoratedExpressionContext;
			}

			@Override
			public TypeConverter getTypeConverter() {
				return typeConverter;
			}

			@Override
			public Object lookupVariable(final String name) {
				return <...>;
			}
		};
	}

	private static Object evaluate(final Method method, final EvaluationContext evaluationContext) {
		final ExpressionParser parser = new SpelExpressionParser();
		final PreAuthorize preAuthorizeAnnotation = method.getAnnotation(PreAuthorize.class);
		final Expression expression = parser.parseExpression(preAuthorizeAnnotation.value());
		return expression.getValue(evaluationContext, Object.class);
	}

	private static String resolveAsString(final Object value) {
		if ( value instanceof IAuthorizationExpression ) {
			return ((IAuthorizationExpression) value).toHumanReadableExpression();
		}
		return String.valueOf(value);
	}

}
```

The code above is slightly more complex, but it's not difficult in fact.
It might be not very simple to resolve missing variables for the expressions (like `#name` in the examples above).
`<...>` is a placeholder to map arguments to the parameters.
Generally speaking, `null` might be acceptable too, but it will not work in some cases for known reasons.
And one more not that good thing: it's necessary to create `ForwardingEvaluationContext` similarly to the way how `ForwardingStandardEvaluationContext` was created above.

```java
public abstract class ForwardingEvaluationContext
		implements EvaluationContext {

	protected abstract EvaluationContext evaluationContext();

	// @formatter:off
	@Override public TypedValue getRootObject() { return evaluationContext().getRootObject(); }
	@Override public List<ConstructorResolver> getConstructorResolvers() { return evaluationContext().getConstructorResolvers(); }
	@Override public List<MethodResolver> getMethodResolvers() { return evaluationContext().getMethodResolvers(); }
	@Override public List<PropertyAccessor> getPropertyAccessors() { return evaluationContext().getPropertyAccessors(); }
	@Override public TypeLocator getTypeLocator() { return evaluationContext().getTypeLocator(); }
	@Override public TypeConverter getTypeConverter() { return evaluationContext().getTypeConverter(); }
	@Override public TypeComparator getTypeComparator() { return evaluationContext().getTypeComparator(); }
	@Override public OperatorOverloader getOperatorOverloader() { return evaluationContext().getOperatorOverloader(); }
	@Override public BeanResolver getBeanResolver() { return evaluationContext().getBeanResolver(); }
	@Override public void setVariable(final String name, final Object value) { evaluationContext().setVariable(name, value); }
	@Override public Object lookupVariable(final String name) { return evaluationContext().lookupVariable(name); }
	// @formatter:on

}
```

So that's all: we have just added custom types support to `@PreAuthorize` expressions operands or results, and now we can provide convenient human-readable string representations for these expressions.

What's remaining behind the scenes
---

In fact, as it was told above, my application authorization rules are specified in the controller methods, not in the services.
Since I use [Springmvc-router](http://resthub.org/springmvc-router/), obtaining the full list of authorized methods is not an issue.
As far as I know, it's easy to do using standard facilities, and it would let to inspect services and other components rather than just controllers.
So the way of obtaining the `@PreAuthorize`-annotatated methods is up to you.

I don't like public constructors as well and I always prefer static factory methods because:

* hide the real return type;
* have just a single constructor that only assigns parameters to fields;
* return cached objects and not always create new objects.

“Effective Java” -- is a very nice book.
Using `final` and nullability annotaions is a matter of habit, and I would like to think it's a habit of good manners.

And the last thing.
I prefer to use merely method invocations in `@PreAuthorize` expressions rather than complex expressions because of several reasons.
First, it allows to associate a certain name to a specific authorization rule, and to avoid string constants for some cases which you'd face with (in my opinion, no string contants are necessary here).
Second, it allows to group authorization rules into some groups and use them respectively.
Third, authorization rules expressions may be too long and it would be harder to maintain `@PreAuthorize` expressions.
Moreover, the compiler will find trivial errors in the compile time.
And your favorite IDE will hightlight these methods better than string expressions.
