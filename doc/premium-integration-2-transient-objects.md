Premium XS Integration: Wrapping Transient Objects
==================================================

One frequent and difficult problem you will encounter when writing XS wrappers around a
C library is what to do when the C library exposes a struct which the user needs to see,
but the lifespan of that struct is controlled by something other than the reference the
user is holding onto.

For example, consider the Display and Screen objects of libX11.  When you connect to an
X server, the library gives you a Display object.  Within that Display object are Screen
objects.  Some of the X11 API uses those Screen objects as parameters, and you need to
expose them to the Perl users.  But, if you call XCloseDisplay on the Display object,
those Screen objects get freed and now accessing them will crash the program.  The Perl
user might still be holding onto a X11::Xlib::Screen Perl object, so how do you stop
them from crashing the program when they check an attribute of that object?

## Indirect References

For the case of X11 Screen structs, there was an easy workaround: The Screen structs are
numbered, and you can store a pair of `(Display, ScreenNumber)` to refer to the Screen
struct without storing the pointer to it.  Because the Perl Screen object references the
Perl Display object, the methods of Screen can check whether the display is closed
before resolving the pointer to a Screen struct, and die with a useful message instead
of a crash.

From another perspective, you can think of them like symlinks.  You reference one Perl
object which has control over its own struct's lifecycle and then a relative path from
that struct to whatever internal data structure you're wrapping with the current object.

While this sounds like a quick solution, there's one other detail to worry about:
cyclical references.  If the sub-object is referring to the parent object, and the
parent refers to a collection of sub-objects, perl will never free these objects.
For the case of X11::Xlib::Screen, I decided to use strong references from Display to
Screen, and weak references (Scalar::Util::weaken) from Screen to Display, and create all
the Screen objects during the Display constructor.

## Lazy Cache of Wrapper Objects

If the list of Screens were dynamic, or if I just didn't want to allocate them all
upfront for some reason, another approach is to wrap the C structs on demand.
You could literally create a new Object each time they access the struct, but you'd
probably want to return the same object if they access two references to the same
struct.  One way to accomplish this is with a hash of weak references.

In Perl it would look like:

```
package MainObject {
  use Moo;
  sub is_closed { ... }
}

package SubObject {
  use Moo;
  has main_ref => ( is => 'ro', weak_ref => 1 );
  has data_ptr => ( is => 'ro' );
  sub method1($self) {
    # If main is closed, stop all method calls
    croak "Object is expired"
      unless $self->main_ref && !$self->main_ref->is_closed;
    ... # operate on data_ptr
  }

  our %sub_objects;
  sub _cached_from_ptr($ptr, $main) {
    my $obj= $sub_objects{$ptr};
    unless (defined $obj) {
      $obj= SubObject->new(main_ref => $main, data_ptr => $ptr);
      Scalar::Util::weaken($sub_objects{$ptr}= $obj);
    }
    return $obj;
  }
}

sub function_that_returns_subobjects {
  # Normally in Perl you'd be creating these from a method of MainObject,
  # but to simulate the C library experience, lets say we just suddenly
  # have a pointer to SubObject data '$ptr' pop out of nowhere while
  # working with '$main'
  ...
  return SubObject::_cached_from_ptr($ptr, $main);
}
```

Now, the caller of `function_that_returns_subobjects` gets a SubObject, and it has
a weak reference to MainObject, and the global hash also holds a weak reference to
the SubObject.  If we call that same function again and it gets the same $ptr while
the first SubObject object still exists, we get the same object back.
Meanwhile, if the caller releases that reference, the SubObject gets garbage
collected, and likewise for MainObject if they release that reference.

One downside of this exact design is that *every* method of SubObject which uses
data_ptr will need to first check that main_ref isn't closed.  If you have frequent
method calls and you'd like them to be a little more efficient, here's an alternate
variation on the pattern:

```
package MainObject {
  use Moo;
  has _sub_object_cache => ( is => 'rw', default => sub {+{}} );

  # MainObject reaches out to invalidate all the remaining SubObjects
  sub close($self) {
    $_->data_ptr(undef)
      for grep defined, values %{ %self->_sub_object_cache };
  }
  
  sub DESTROY { shift->close }
  
  sub new_cached_subobject($self, $ptr) {
    my $obj= $self->_sub_object_cache->{$ptr};
    unless (defined $obj) {
      $obj= SubObject->new(data_ptr => $ptr);
      Scalar::Util::weaken($self->_sub_object_cache->{$ptr}= $obj);
    }
    return $obj;
  }
}

package SubObject {
  use Moo;
  has data_ptr => ( is => 'rw' );
  sub method1($self) {
    croak "Object is expired" unless $self->data_ptr;
    ... # operate on data_ptr
  }
}

sub function_that_returns_subobjects {
  ...
  return $main->new_cached_subobject($ptr);
}
```

In this pattern, the subobject doesn't need to consult anything other than its own
pointer before getting to work.  The subobject doesn't need a weak reference to the main
object, but closing or destroying the main object takes a little extra time as it
invalidates all of the SubObject instances.

When these are implemented in XS, the weak references still need to be allocated as Perl
RV instances, with weak-ref magic applied.  So, I think the second pattern will generally
be a bit faster and lighter on resources.  The methods of SubObject can jump right into
their external library function without accessing a weak reference, and it uses half as
many weak references overall.

## Lazy Cache of Wrapper Objects, in XS

So, what does the code above look like in XS?  Here we go...

