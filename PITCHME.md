# `null` The Silent Killer

---

Java is **NOT** statically typed <!-- .element: class="fragment" -->

Note:
Java is statically typed… oh wait, not I mean it's NOT statically typed.
A statically typed language is one in which the type of every variable and expression
is known at compile time. In the case of Java, that's almost true, but that almost is
important.

---

```
final Foo value = getAFoo();
```

Note:
Consider this code snippet. What is the type of value? Some of you think the answer is
Foo. That's almost right. The correct answer is that we don't know, it could be one of
two possible types. It's either Foo (or a subclass), or else it's whatever type null
has.

---

```
null.aMethod();
```

Note:
Lets talk about null. null is special. null has no methods. null only supports a
single operator. The equality operator. You can check if something is equal to null
or not, but that's it. If you attempt to call a method on null, a NullPointerException
is thrown. Everyone has experienced NPEs, and they're no fun.

---

```
public Foo getAFoo(final Bar bar) {
  final Foo foo = new Foo();
  foo.setBar(bar);
  if(bar.aBusinessRule()) {
    foo.setBaz(getABaz(bar));
  } else {
    foo.setBaz(null);
  }
  return foo;
}
```

Note:
Lets consider this code. Looks pretty typical. But is it safe? Nope. This is just
chock full of opportunities to throw NullPointerExceptions. Lets rewrite it to be
safe!

---

```
public Foo getAFoo(final Bar maybeBar) {
  final Foo foo = new Foo();
  Bar bar = new Bar();
  if(maybeBar != null) {
    bar = maybeBar;
  }
  if(bar.aBusinessRule()) {
    final Baz baz = getABaz(bar);
    if(baz == null) {
      throw new IllegalStateException("failed to get a Baz!!!");
    } else {
      foo.setBaz(baz)
    }
  } else {
    foo.setBaz(null);
  }
  return foo;
}
```

Note:
Well, that's certainly safer. Not sure it's better, that's an awful lot of checks
just to avoid some exceptions. Lets look at how most people would ACTUALLY write this.

---

```
public Foo getAFoo(final Bar bar) {
  final Foo foo = new Foo();
  if(bar != null && bar.aBusinessRule()) {
    foo.setBaz(getABaz(bar));
  } else {
    foo.setBaz(null);
  }
  return foo;
}
```

Note:
This code is pretty close to what we started with. We do check to see if bar is null
before we invoke a method on it, but other than that it's pretty much the unsafe code.
So, that's that then right? No way to do this better, we just check the most common
sources of nulls, and otherwise just hope things aren't null. Right?

---

# Better living through ~~science~~ static analysis!

---

Our friends `@Nonnull` and `@Nullable`

Note:
Meet our friends the Nonnull and Nullable annotations. These annotations were introduced
as part of JSR-305 and are part of the javax.annotations package. By themselves they do nothing.
But when paired with the proper tools they can be an important tool in battling the scourge
of nulls. Lets see what our previous code looks like using these.

---

```
@Nonnull
public Foo getAFoo(@Nullable final Bar bar) {
  final Foo foo = new Foo();
  if(bar != null && bar.aBusinessRule()) {
    foo.setBaz(getABaz(bar));
  } else {
    foo.setBaz(null);
  }
  return foo;
}
```

Note:
Aside from the annotations, the code looks exactly the same, what gives? Well, the secret
is that the annotations are also applied to the getBaz method on the Foo class, and the
getABaz methods return types, specifically the Nullable and Nonnull anotations respectively.
By using those annotations we can use a static analysis tool like the one built into
intellij or FindBugs to statically assert at compile time that all possible sources of
null have been dealt with properly. Lets make a small change to the code, one that
will prevent it from passing the static analysis.

---

```
@Nonnull
public Foo getAFoo(@Nullable final Bar bar) {
  final Foo foo = new Foo();
  if(bar.aBusinessRule()) {
    foo.setBaz(getABaz(bar));
  } else {
    foo.setBaz(null);
  }
  return foo;
}
```

Note:
With the exception of the annotations, this code is exactly the same as our original
totally unsafe implementation. When we run this through static analysis however it flags
the line that calls the aBusinessRule method on bar as unsafe. We've told it that bar
MIGHT be null, but we've invoked a method on the bar instance without checking to see
if it was or was not null yet. By using these annotations we've provided the static
analysis tool the necessary metadata to find and prevent bugs before they crop up
at runtime.

---

# But Wait! There's More!

Note:
Once again, that's nice, but we can do better! Lets bring a new weapon into play.

---

```
public Foo getAFoo(final Optional<Bar> maybeBar) {
  final Foo foo = new Foo();
  if(maybeBar.map(Bar::aBusinessRule).orElse(false)) {
    foo.setBaz(maybeBar.map(this::getABaz));
  } else {
    foo.setBaz(Optional.empty());
  }
  return foo;
}
```

Note:
Now we've introduced Optional. Optional takes the concept of a nullable value and
lifts it into a type. Now we have a way to express in the type of a method parameter,
return value, or a class property, that this thing might be optional or not available.
Why is this better than the annotations we had previously? In this code the compiler
is checking to see if we've made a mistake, NOT some other tool that you need to remember
to run and check the result of. It's literally impossible to for instance call the
aBusinessRule method on maybeBar without first checking to see if we've got a bar
value or not. Likewise anyone who calls getBaz on our Foo instance will be forced
to handle the case where there is no Baz value on that instance of Foo.

---

`someOptional.get()` ಠ_ಠ

Note:
Optional isn't a silver bullet however. You can still get into trouble if you use it
carelessly. Optional provides a method, get, that you should be wary of. The method
exists because on a rare occasion, you HAVE to use it. That said, if you're not
careful with how you use it, you can end up right back where we started throwing the equivalent of
NullPointerExceptions. That's because when an Optional is empty, calling get on it
throws a NoSuchElementException, which is basically the same things as a 
NullPointerException. You can almost always avoid using get by using one of the other
methods that Optional provides like orElse or ifPresent.

---

`Optional`, the secret to never being woken up at 3 am to debug an outtage 
caused by a `NullPointerException`.

Note:
By using Optional everywhere you would have used null previously, and either wrapping
inputs to our code in Optional.ofNullable as soon as possible, or else annotating values
with Nullable annotations and performing static analysis you can effectively banish
NullPointerExceptions from your code. If you follow these conventions the only time
you'll ever see a NullPointerException is when some library you're using throws one.
