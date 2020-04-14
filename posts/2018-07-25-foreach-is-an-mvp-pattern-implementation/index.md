.. title: foreach is an MVP pattern implementation
.. date: 2018-07-25 14:16:00 +0300
.. tags: java
.. category: java
.. type: text

The `foreach`, enhanced `for` statement, is a perfect example of an [MVP](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93presenter) pattern implementation.

<!-- TEASER_END -->

From the MVP perspective, there are three kinds of components that construct the MVP.
These components are represented as follows:

* Model - an `Iterable<T>` one's going to iterate over, data supplier.
* Presenter - the `foreach` statement itself, also a [mediator](https://en.wikipedia.org/wiki/Mediator_pattern).
* View - a code block to process each element from the data source, data consumer.

Now let's go a bit into a more "enterprise" approach:

```java
interface IForEachModel<T>
	extends Supplier<Iterable<T>> {
}

interface IForEachPresenter<T>
	extends BiConsumer<IForEachModel<? extends T>, IForEachView<? super T>> {

	@Override
	default void accept(final IForEachModel<? extends T> model, final IForEachView<? super T> view) {
		for ( final T t : model.get() ) {
			view.accept(t);
		}
	}

}

interface IForEachView<T>
	extends Consumer<T> {
}
```

Now, the following code

```java
for ( final String s : ImmutableList.of("foo", "bar", "baz") ) {
	System.out.println(s);
}
```

is totally equivalent to the following "more enterprise" approach:

```java
final IForEachModel<String> model = () -> ImmutableList.of("foo", "bar", "baz");
final IForEachPresenter<String> presenter = new IForEachPresenter<String>() {};
final IForEachView<String> view = System.out::println;
presenter.accept(model, view);
```

Design patterns everywhere!