---
layout: post
title: "What's the Opposite of \"nullable\"?"
categories: c-sharp code
---
I hate [duplicate code](http://en.wikipedia.org/wiki/DRY). [Hate it](http://c2.com/cgi/wiki?DontRepeatYourself) [hate it](http://www.codinghorror.com/blog/archives/000805.html) [hate it](http://www.thefrontside.net/blog/repeat_yourself). At the
same time, I *do* like having my code check its arguments.

After a while, I noticed that half of the functions I wrote looked like this

{% highlight csharp %}
void SomeMethod(Foo foo)
{
    if (foo == null) throw new ArgumentNullException("foo");

    // ...
}
{% endhighlight %}

That's good code in the sense that it bails on `null`, but it's got a couple
of things I don't like. First, I have to keep copying and pasting that exception line in every method. I know, it's just one line, but it started to
annoy me.

Worse is that to a *caller* of the function, there's no way to tell that
`SomeMethod()` doesn't allow a `null` value for `foo` without looking at the
method body (which may not be available to an API user).

The [`Nullable&lt;T&gt;`](http://msdn2.microsoft.com/en-us/library/b3h38hb0.aspx) type in the [BCL](http://msdn2.microsoft.com/en-us/netframework/aa569603.aspx) (which is automatically aliased
to it's [more familiar `?`](http://blogs.msdn.com/ericgu/archive/2004/05/27/143221.aspx), as in "`int?`") had me wondering. Can I make the opposite? A "not nullable?"

## NotNull&lt;T&gt;

Let's build it a bit at a time. The core not-very-clever idea is just a class
that wraps a reference. When you construct it, it throws our familiar
`ArgumentNullException` if the reference is `null`:

{% highlight csharp %}
public class NotNull<T>
{
    public T Value
    {
        get { return mValue; }
    }

    public NotNull(T maybeNull)
    {
        if (maybeNull == null)
        {
            throw new ArgumentNullException("maybeNull");
        }

        mValue = maybeNull;
    }

    private T mValue;
}
{% endhighlight %}

Definitely not rocket science. Now you can define methods like this:

{% highlight csharp %}
void SomeMethod(NotNull<Foo> foo) {}
{% endhighlight %}

By the time you get to your method body, you can be sure that `foo` is not
`null`. Unfortunately, your call sites just got uglier:

{% highlight csharp %}
SomeMethod(new NotNull(myFoo));
{% endhighlight %}

A conversion operator will fix that:

{% highlight csharp %}
// in NotNull<T>
public static implicit operator NotNull<T>(T maybeNull)
{
    return new NotNull<T>(maybeNull);
}
{% endhighlight %}

Now the call sites are unchanged:

{% highlight csharp %}
SomeMethod(mFoo);
{% endhighlight %}

and when you make the call, it will automatically call the conversion, which
will in turn bail if `myFoo` is `null`. We can make things a little easier by
providing a conversion the other way too:

{% highlight csharp %}
public static implicit operator T(NotNull<T> notNull)
{
    return notNull.Value;
}
{% endhighlight %}

Now you can do this:

{% highlight csharp %}
void SomeMethod(NotNull<Foo> foo)
{
    // will automatically convert on assign :)
    Foo someFoo = foo;

    // but not on member access :(
    // can't do foo.SomeFooProperty, have to do:
    foo.Value.SomeFooProperty;
}
{% endhighlight %}

Not too shabby. The bonus, and the real reason I like this is that you've now
**decorated the function signature itself with its requirements**. Any caller
of `SomeMethod()` will now see in Intellisense that it requires a
`NotNull<Foo>`. We've moved an *imperative* exception check to a
*[declarative](http://en.wikipedia.org/wiki/Declarative_programming) property* of the argument itself.

## Caveats

I'd like to say this is a perfect solution, but alas it's not. There's at
least one caveat to be aware of. This won't work with interface types. That's
because value types can implement interfaces too, and you can't compare a
value type to `null`. If you can figure out a way around this, holler.

**Edit:** as Brad points out below, this does work with interfaces… sort of. The limitation is that implicit conversion operators don't work with them. The actual wrapping and `null` checks are fine. It's just that to use it with an interface, you have to do:

{% highlight csharp %}
SomeMethod(new NotNull(myInterfaceFoo));
{% endhighlight %}

which is kind of lame. But aside from that, you can use interfaces with this.
Thanks, B-Rad!

## The Whole Shebang

Oh, and here's a prettier version with comments and stuff. Don't say I never
gave you nothing:

{% highlight csharp %}
/// <summary>
/// <para>
/// Wrapper around a reference that ensures the reference is not <c>null</c>.
/// Provides implicit cast operators to automatically wrap and unwrap
/// values.
/// </para>
/// <para>
/// NotNull{T} can be used as an argument to a method to ensure that
/// no <c>null</c> values are passed to the method in place of manually
/// throwing an <see cref="ArgumentNullException"/>. It has an added
/// benefit over that because using it as an argument type clearly
/// communicates to the caller the expectation of the method.
/// </para>
/// </summary>
/// <typeparam name="T">Type being wrapped.</typeparam>
public class NotNull<T>
{
    /// <summary>
    /// Automatically unwraps the non-<c>null</c> object being wrapped
    /// by this NotNull{T}.
    /// </summary>
    /// <param name="notNull">The wrapper.</param>
    /// <returns>The raw object being wrapped.</returns>
    public static implicit operator T(NotNull<T> notNull)
    {
        return notNull.Value;
    }

    /// <summary>
    /// Automatically wraps an object in a NotNull{T}. Will throw
    /// an <see cref="ArgumentNullException"/> if the value being
    /// wrapped is <c>null</c>.
    /// </summary>
    /// <param name="maybeNull">The raw reference to wrap.</param>
    /// <returns>A new NotNull{T} that wraps the value, provided the
    /// value is not <c>null</c>.</returns>
    /// <exception cref="ArgumentNullException">If <c>maybeNull</c> is <c>null</c>.</exception>
    public static implicit operator NotNull<T>(T maybeNull)
    {
        return new NotNull<T>(maybeNull);
    }

    /// <summary>
    /// Gets and sets the non-null reference being wrapped by this
    /// NotNull{T}.
    /// </summary>
    /// <exception cref="ArgumentNullException">If <c>value</c> is <c>null</c>.</exception>
    public T Value
    {
        get { return mValue; }
        set
        {
            if (value == null) throw new ArgumentNullException("value");
            mValue = value;
        }
    }

    /// <summary>
    /// Creates a new wrapper around the given reference.
    /// </summary>
    /// <remarks>Explicitly calling the constructor is rarely needed. Usually the
    /// implicit cast is simpler.</remarks>
    /// <param name="maybeNull">The reference to wrap.</param>
    /// <exception cref="ArgumentNullException">If <c>maybeNull</c> is <c>null</c>.</exception>
    public NotNull(T maybeNull)
    {
        if (maybeNull == null) throw new ArgumentNullException("maybeNull");

        mValue = maybeNull;
    }

    private T mValue;
}
{% endhighlight %}

and a unit test:

{% highlight csharp %}
[TestFixture]
public class NotNullFixture
{
    [Test]
    public void TestConstructor()
    {
        Foo foo = new Foo();
        NotNull<Foo> notNull = new NotNull<Foo>(foo);
    }

    [Test]
    [ExpectedException(typeof(ArgumentNullException))]
    public void TestConstructorThrowsOnNull()
    {
        Foo foo = null;
        NotNull<Foo> notNull = new NotNull<Foo>(foo);
    }

    [Test]
    public void TestGetValue()
    {
        Foo foo = new Foo();
        NotNull<Foo> notNull = new NotNull<Foo>(foo);

        Assert.AreEqual(foo, notNull.Value);
    }

    [Test]
    public void TestImplicitWrap()
    {
        Foo foo = new Foo();
        NotNull<Foo> notNull = foo;

        Assert.AreEqual(foo, notNull.Value);
    }

    [Test]
    public void TestImplicitUnwrap()
    {
        Foo foo = new Foo();
        NotNull<Foo> notNull = new NotNull<Foo>(foo);

        Foo unwrap = notNull;

        Assert.AreEqual(foo, unwrap);
    }

    public class Foo { }
}
{% endhighlight %}
