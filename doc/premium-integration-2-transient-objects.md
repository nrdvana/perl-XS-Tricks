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
    croak "SubObject belongs to a MainObject which was closed"
      unless $self->data_ptr;
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

```
/* First, the API for your internal structs */

struct YourProject_MainObject_info {
  SomeLib_MainObject *obj;
  HV *wrapper;
  HV *subobj_cache;
  bool is_closed;
};

struct YourProject_SubObject_info {
  SomeLib_SubObject *obj;
  SomeLib_MainObject *parent;
  HV *wrapper;
};

struct YourProject_MainObject_info*
YourProject_MainObject_info_create(HV *wrapper) {
  struct YourProject_MainObject_info *info= NULL;
  Newxz(info, 1, struct YourProject_MainObject_info);
  info->wrapper= wrapper;
  return info;
}

void YourProject_MainObject_info_close(struct YourProject_MainObject_info* info) {
  if (info->is_closed) return;
  /* All SubObject instances are about to become invalid */
  if (info->subobj_cache) {
    HE *pos;
    hv_iterinit(info->subobj_cache);
    while (pos= hv_iternext(info->subobj_cache)) {
      /* each value of the hash is a weak reference, which might have become undef at some point */
      SV *subobj_ref= hv_iterval(info->subobj_cache, pos);
      if (subobj_ref && SvROK(subobj_ref)) {
        struct YourProject_SubObject_info *s_info= YourProject_SubObject_from_magic(SvRV(subobj_ref), 0);
        if (s_info) {
          s_info->obj= NULL;    /* it's an internal piece of the parent, so no destructor */
          s_info->parent= NULL;
        }
      }
    }
  }
  if (info->obj)
    SomeLib_MainObject_close(info->obj);
  info->is_closed= true;
}

void YourProject_MainObject_info_free(struct YourProject_MainObject_info* info) {
  if (info->obj)
    YourProject_MainObject_info_close(info);
  if (info->subobj_cache)
    SvREFCNT_dec((SV*) info->subobj_cache);
  /* The lifespan of 'wrapper' is handled by perl.
   * Probably in the process of getting freed right now. */
  Safefree(info);
}

```
The gist here is that MainObject has a set of all SubObject wrappers which are still held by the
perl script, and during "close" (which, in this hypothetical library, invalidates all SubObject
pointers) it can iterate that set and mark each wrapper as being invalid.

The Magic setup for MainObject goes just like in the previous article:

```
static int YourProject_MainObject_magic_free(pTHX_ SV* sv, MAGIC* mg) {
  YourProject_MainObject_info_free((struct YourProject_MainObject_info*) mg->mg_ptr);
}
static MAGIC YourProject_MainObject_magic_vtbl = {
  ...
};

struct YourProject_MainObject_info *
YourProject_MainObject_from_magic(SV *objref, int flags) {
  ...
}
```
The destructor for the magic will call the destructor for the info struct.  The "_from_magic"
function instantiates the magic according to 'flags', and so on.

Now, the Magic handling for SubObject works a little differently.  We don't get to decide when
to create or destroy SubObject, we just encounter these pointers in the return values of the
C library functions, and need to wrap them in order to show them to the perl script.

```

/* Return a new ref to an existing wrapper, or create a new wrapper and cache it.
 */
SV * YourProject_SubObject_wrap(SomeLib_SubObject *sub_obj) {
  /* If your library doesn't have a way to get the main object
   * from the sub object, this gets more complicated.
   */
  SomeLib_MainObject *main_obj= SomeLib_SubObject_get_main(sub_obj);
  SV **subobj_entry= NULL;
  YourProject_SubObject_info *s_info= NULL;
  HV *wrapper= NULL;
  SV *objref= NULL;
  MAGIC *magic;

  /* lazy-allocate the cache */
  if (!main_obj->subobj_cache) {
    main_obj->subobj_cache= newHV();

  /* See if the SubObject has already been wrapped.
   * Use the pointer as the key
   */
  subobj_entry= hv_fetch(main_obj->subobj_cache, &sub_obj, sizeof(void*), 1);
  if (!subobj_entry) croak("lvalue hv_fetch failed"); /* should never happen */

  /* weak references may have become undef */
  if (*subobj_entry && SvROK(*subobj_entry) && SvOK(SvRV(*subobj_entry)))
    /* we can re-use the existing wrapper */
    return newRV_inc( SvRV(*subobj_entry) );
  
  /* Not cached. Create the struct and wrapper. */
  Newxz(s_info, 1, struct YourProject_SubObject_info);
  s_info->obj= sub_obj;
  s_info->wrapper= newHV();
  s_info->parent= main_obj;
  objref= newRV_noinc((SV*) s_info->wrapper);
  sv_bless(objref, gv_stashpv("YourProject::SubObject", GV_ADD));

  /* Then attach the struct pointer to its wrapper via magic */
  magic= sv_magicext((SV*) s_info->wrapper, NULL, PERL_MAGIC_ext,
      &YourProject_SubObject_magic_vtbl, (const char*) s_info, 0);
#ifdef USE_ITHREADS
  magic->mg_flags |= MGf_DUP;
#else
  (void)magic; // suppress warning
#endif
  
  /* Then add it to the cache as a weak reference */
  *subobj_entry= sv_rvweaken( newRV_inc((SV*) s_info->wrapper) );

  /* Then return a strong reference to it */
  return objref;
}

```
Again, this is roughly equivalent to the Perl implementation of `new_cached_subobject` above.

Now, when methods are called on the SubObject wrapper, we want to throw an exception if the
SubObject is no longer valid.  We can do that in the function that the Typemap uses:

```
struct YourProject_SubObject_info *
YourProject_SubObject_from_magic(SV *objref, int flags) {
  struct YourProject_SubObject_info *ret= NULL;

  ... /* inspect magic */

  if (flags & OR_DIE) {
    if (!ret) croak("Not an instance of SubObject");
    if (!ret->obj) croak("SubObject belongs to a MainObject which was closed");
  }
  return ret;
}

```

Now, the Typemap:

```
TYPEMAP
SomeLib_MainObject*      O_SomeLib_MainObject
SomeLib_SubObject*       O_SomeLib_SubObject

INPUT
O_SomeLib_MainObject
  $var= YourProject_MainObject_from_magic($arg, OR_DIE);

INPUT
O_SomeLib_SubObject
  $var= YourProject_SubObject_from_magic($arg, OR_DIE);

OUTPUT
O_SomeLib_SubObject
  sv_setsv($arg, sv_2mortal(YourProject_SubObject_wrap($var)));

```

This time I added an "OUTPUT" entry for SubObject, because we can safely wrap *any* SubObject
pointer that we see in *any* of the SomeLib API calls, and get the desired result.

There's nothing stopping you from automatically wrapping MainObject pointers with an OUTPUT
typemap, but that's prone to errors because sometimes an API returns a pointer to the
already-existing MainObject, and you don't want perl to put a second wrapper on the same
MainObject.  For objects like MainObject, I prefer to special-case my constructor (or whatever
method initializes the instance of SomeLib_MainObject) with a call to
`_from_magic(..., AUTOCREATE)` on the INPUT typemap rather than returning the pointer and
letting perl's typemap wrap it on OUTPUT.

After all that, it pays off when you add a bunch of methods in the rest of the XS file.
For a C function with a signature of

```
SomeLib_SubObject * function_that_returns_subobjects(SomeLib_MainObject *);
```

all you need in the XS is:

```
SomeLib_SubObject *
function_that_returns_subobjects(main)
  SomeLib_MainObject *main

```

and XS translation handles the rest!

