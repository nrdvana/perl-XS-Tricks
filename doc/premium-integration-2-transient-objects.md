Premium XS Integration: Wrapping Transient Objects
==================================================

One frequent and difficult problem you will encounter when writing XS wrappers around a
C library is what to do when the C library exposes a struct which the user needs to see,
but the lifespan of that struct is controlled by something other than the reference the
user is holding onto.

For example, consider the Display and Screen structs of libX11.  When you connect to an
X server, the library gives you a Display pointer.  Within that Display struct are Screen
structs.  Some of the X11 API uses those Screen pointers as parameters, and you need to
expose them in the Perl interface.  But, if you call XCloseDisplay on the Display pointer
those Screen structs get freed, and now accessing them will crash the program.  The Perl
user might still be holding onto a X11::Xlib::Screen Perl object, so how do you stop
them from crashing the program when they check an attribute of that object?

## Indirect References

For the case of X11 Screens there was an easy workaround: The Screen structs are
numbered, and a pair of `(Display, ScreenNumber)` can refer to the Screen struct without
needing the pointer to it.  Because the Perl Screen object references the
Perl Display object, the methods of Screen can check whether the display is closed
before resolving the pointer to a Screen struct, and die with a useful message instead
of a crash.

From another perspective, you can think of them like symlinks.  You reference one Perl
object which has control over its own struct's lifecycle and then a relative path from
that struct to whatever internal data structure you're wrapping with the current object.

While this sounds like a quick solution, there's one other detail to worry about:
cyclical references.  If the sub-object is referring to the parent object, and the
parent refers to a collection of sub-objects, perl will never free these objects.
For the case of X11 Screens, the list of screen structs is known at connection-time
and is almost always just one Screen, and doesn't change at runtime. [1]
An easy solution for a case like this is to have a strong reference from Display to Screen,
and weak references (Scalar::Util::weaken) from Screen to Display, and create all the Screen
objects as soon as the Display is connected.

1) *this API is from an era before people thought about connecting new monitors while the
computer was powered up, and these days can more accurately be thought of as a list of
graphics cards rather than "screens"*

## Lazy Cache of Wrapper Objects

If the list of Screens were dynamic, or if I just didn't want to allocate them all
upfront for some reason, another approach is to wrap the C structs on demand.
You could literally create a new wrapper object each time they access the struct, but you'd
probably want to return the same Perl object if they access two references to the same
struct.  One way to accomplish this is with a cache of weak references.

In Perl it would look like:

```
package MainObject {
  use Moo;
  use Scalar::Util 'weaken';
  
  has is_closed         => ( is => 'rwp' );
  
  # MainObject reaches out to invalidate all the SubObjects
  sub close($self) {
    ...
    $self->_set_is_closed(1);
  }
  
  has _subobject_cache => ( is => 'rw', default => sub {+{}} );
  
  sub _new_cached_subobject($self, $ptr) {
    my $obj= $self->_subobject_cache->{$ptr};
    unless (defined $obj) {
      $obj= SubObject->new(main_ref => $main, data_ptr => $ptr);
      weaken($self->_subobject_cache->{$ptr}= $obj);
    }
    return $obj;
  }
  
  sub find_subobject($self, $search_key) {
    my $data_ptr= _xs_find_subobject($self, $search_key);
    return $self->_new_cached_subobject($data_ptr);
  }
}

package SubObject {
  use Moo;
  
  has main_ref => ( is => 'ro' );
  has data_ptr => ( is => 'ro' );
  
  sub method1($self) {
    # If main is closed, stop all method calls
    croak "Object is expired"
      if $self->main_ref->is_closed;
    ... # operate on data_ptr
  }

  sub method2($self) {
    # If main is closed, stop all method calls
    croak "Object is expired"
      if $self->main_ref->is_closed;
    ... # operate on data_ptr
  }
}

```

Now, the caller of `find_subobject` gets a SubObject, and it has a strong reference to
MainObject, and MainObject's cache holds a weak reference to the SubObject.  If we call
that same method again with the same search key while the first SubObject still exists,
we get the same Perl object back.  As long as the user holds onto the SubObject, the MainObject
won't expire, but the SubObjects can get garbage collected as soon as they aren't needed.

One downside of this exact design is that *every* method of SubObject which uses
data_ptr will need to first check that `main_ref` isn't closed (like shown in `method1`).
If you have frequent method calls and you'd like them to be a little more efficient, here's
an alternate version of the same idea:

```
package MainObject {
  ...
  
  # MainObject reaches out to invalidate all the SubObjects
  sub close($self) {
    ...
    $_->data_ptr(undef)
      for grep defined, values $self->_subobject_cache->%*;
  }
  
  ...
}

package SubObject {
  ...

  sub method1($self) {
    my $data_ptr= $self->data_ptr
      // croak "SubObject belongs to a closed MainObject";
    ... # operate on data_ptr
  }
  
  sub method2($self) {
    my $data_ptr= $self->data_ptr
      // croak "SubObject belongs to a closed MainObject";
    ... # operate on data_ptr
  }

  ...
}
```

In this pattern, the subobject doesn't need to consult anything other than its own pointer
before getting to work, which comes in really handy with the XS Typemap.  The subobject also
doesn't need a reference to the main object (unless you want one to prevent the main object
from getting freed while a user holds SubObjects) so this design is a little more flexible.
The only downside is that closing the main object takes a little extra time as it invalidates
all of the SubObject instances, but in XS that time won't be noticeable.

## Lazy Cache of Wrapper Objects, in XS

So, what does the code above look like in XS?  Here we go...

```
/* First, the API for your internal structs */

struct MainObject_info {
  SomeLib_MainObject *obj;
  HV *wrapper;
  HV *subobj_cache;
  bool is_closed;
};

struct SubObject_info {
  SomeLib_SubObject *obj;
  SomeLib_MainObject *parent;
  HV *wrapper;
};

struct MainObject_info*
MainObject_info_create(HV *wrapper) {
  struct MainObject_info *info= NULL;
  Newxz(info, 1, struct MainObject_info);
  info->wrapper= wrapper;
  return info;
}

void MainObject_info_close(struct MainObject_info* info) {
  if (info->is_closed) return;
  /* All SubObject instances are about to be invalid */
  if (info->subobj_cache) {
    HE *pos;
    hv_iterinit(info->subobj_cache);
    while (pos= hv_iternext(info->subobj_cache)) {
      /* each value of the hash is a weak reference,
         which might have become undef at some point */
      SV *subobj_ref= hv_iterval(info->subobj_cache, pos);
      if (subobj_ref && SvROK(subobj_ref)) {
        struct SubObject_info *s_info =
          SubObject_from_magic(SvRV(subobj_ref), 0);
        if (s_info) {
          /* it's an internal piece of the parent, so
             no need to call a destructor here */
          s_info->obj= NULL;
          s_info->parent= NULL;
        }
      }
    }
  }
  SomeLib_MainObject_close(info->obj);
  info->obj= NULL;
  info->is_closed= true;
}

void MainObject_info_free(struct MainObject_info* info) {
  if (info->obj)
    MainObject_info_close(info);
  if (info->subobj_cache)
    SvREFCNT_dec((SV*) info->subobj_cache);
  /* The lifespan of 'wrapper' is handled by perl,
   * probably in the process of getting freed right now.
   * All we need to do is delete our struct.
   */
  Safefree(info);
}
```

The gist here is that MainObject has a set of all SubObject wrappers which are still held by the
perl script, and during "close" (which, in this hypothetical library, invalidates all SubObject
pointers) it can iterate that set and mark each wrapper as being invalid.

The Magic setup for MainObject goes just like in the previous article:

```
static int MainObject_magic_free(pTHX_ SV* sv, MAGIC* mg) {
  MainObject_info_free((struct MainObject_info*) mg->mg_ptr);
}
static MAGIC MainObject_magic_vtbl = {
  ...
};

struct MainObject_info *
MainObject_from_magic(SV *objref, int flags) {
  ...
}
```

The destructor for the magic will call the destructor for the info struct.  The "_from_magic"
function instantiates the magic according to 'flags', and so on.

Now, the Magic handling for SubObject works a little differently.  We don't get to decide when
to create or destroy SubObject, we just encounter these pointers in the return values of the
C library functions, and need to wrap them in order to show them to the perl script.

```
/* Return a new ref to an existing wrapper, or
 * create a new wrapper and cache it.
 */
SV * SubObject_wrap(SomeLib_SubObject *sub_obj) {
  /* If your library doesn't have a way to get the main object
   * from the sub object, this gets more complicated.
   */
  SomeLib_MainObject *main_obj= SomeLib_SubObject_get_main(sub_obj);
  SV **subobj_entry= NULL;
  SubObject_info *s_info= NULL;
  HV *wrapper= NULL;
  SV *objref= NULL;
  MAGIC *magic;

  /* lazy-allocate the cache */
  if (!main_obj->subobj_cache) {
    main_obj->subobj_cache= newHV();

  /* See if the SubObject has already been wrapped.
   * Use the pointer as the key
   */
  subobj_entry= hv_fetch(
    main_obj->subobj_cache,
    &sub_obj, sizeof(void*), 1
  );
  if (!subobj_entry)
    croak("lvalue hv_fetch failed"); /* should never happen */

  /* weak references may have become undef */
  if (*subobj_entry && SvROK(*subobj_entry))
    /* we can re-use the existing wrapper */
    return newRV_inc( SvRV(*subobj_entry) );
  
  /* Not cached. Create the struct and wrapper. */
  Newxz(s_info, 1, struct SubObject_info);
  s_info->obj= sub_obj;
  s_info->wrapper= newHV();
  s_info->parent= main_obj;
  objref= newRV_noinc((SV*) s_info->wrapper);
  sv_bless(objref, gv_stashpv("YourProject::SubObject", GV_ADD));

  /* Then attach the struct pointer to its wrapper via magic */
  magic= sv_magicext((SV*) s_info->wrapper, NULL, PERL_MAGIC_ext,
      &SubObject_magic_vtbl, (const char*) s_info, 0);
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
struct SubObject_info *
SubObject_from_magic(SV *objref, int flags) {
  struct SubObject_info *ret= NULL;

  ... /* inspect magic */

  if (flags & OR_DIE) {
    if (!ret)
      croak("Not an instance of SubObject");
    if (!ret->obj)
      croak("SubObject belongs to a closed MainObject");
  }
  return ret;
}

```

Now, the Typemap:

```
TYPEMAP
struct MainObject_info *   O_SomeLib_MainObject_info
SomeLib_MainObject*        O_SomeLib_MainObject
struct SubObject_info *    O_SomeLib_SubObject_info
SomeLib_SubObject*         O_SomeLib_SubObject

INPUT
O_SomeLib_MainObject_info
  $var= MainObject_from_magic($arg, OR_DIE);

INPUT
O_SomeLib_MainObject
  $var= MainObject_from_magic($arg, OR_DIE)->obj;

INPUT
O_SomeLib_SubObject_info
  $var= SubObject_from_magic($arg, OR_DIE);

INPUT
O_SomeLib_SubObject
  $var= SubObject_from_magic($arg, OR_DIE)->obj;

OUTPUT
O_SomeLib_SubObject
  sv_setsv($arg, sv_2mortal(SubObject_wrap($var)));

```

This time I added an "OUTPUT" entry for SubObject, because we can safely wrap *any* SubObject
pointer that we see in *any* of the SomeLib API calls, and get the desired result.

There's nothing stopping you from automatically wrapping MainObject pointers with an OUTPUT
typemap, but that's prone to errors because sometimes an API returns a pointer to the
already-existing MainObject, and you don't want perl to put a second wrapper on the same
MainObject.  This problem doesn't apply to SubObject, because we re-use any existing wrapper
by checking the cache.  (of course, you could apply the same trick to MainObject and have
a global cache of all the known MainObject instances, and actually I do this in X11::Xlib)

But in general, for objects like MainObject I prefer to special-case my constructor
(or whatever method initializes the instance of SomeLib_MainObject) with a call to
`_from_magic(..., AUTOCREATE)` on the INPUT typemap rather than returning the pointer and
letting perl's typemap wrap it on OUTPUT.

After all that, it pays off when you add a bunch of methods in the rest of the XS file.

Looking back to the `find_subobject` method of the original Perl example, all you need in the
XS is basically the prototype for that function of SomeLib:

```
SomeLib_SubObject *
find_subobject(main, search_key)
  SomeLib_MainObject *main
  char *key

```
and XS translation handles the rest!

## Reduce Redundancy in your Typemap

I should mention that you don't need a new typemap INPUT/OUTPUT macro for every single data
type.  The macros for a typemap provide you with a `$type` variable (and others, see
`perldoc xstypemap`) which you can use to construct function names, as long as you name your
functions consistently.  If you have lots of different types of sub-objects, you could extend
the previous typemap like this:

```
TYPEMAP
struct MainObject_info *    O_INFOSTRUCT_MAGIC
SomeLib_MainObject*         O_LIBSTRUCT_MAGIC

struct SubObject1_info *    O_INFOSTRUCT_MAGIC
SomeLib_SubObject1*         O_LIBSTRUCT_MAGIC_INOUT

struct SubObject2_info *    O_INFOSTRUCT_MAGIC
SomeLib_SubObject2*         O_LIBSTRUCT_MAGIC_INOUT

struct SubObject3_info *    O_INFOSTRUCT_MAGIC
SomeLib_SubObject3*         O_LIBSTRUCT_MAGIC_INOUT

INPUT
O_INFOSTRUCT_MAGIC
  $var= @{[ $type =~ / (\w+)/ ]}_from_magic($arg, OR_DIE);

INPUT
O_LIBSTRUCT_MAGIC
  $var= @{[ $type =~ /_(\w*)/ ]}_from_magic($arg, OR_DIE)->obj;

INPUT
O_LIBSTRUCT_MAGIC_INOUT
  $var= @{[ $type =~ /_(\w*)/ ]}_from_magic($arg, OR_DIE)->obj;

OUTPUT
O_LIBSTRUCT_MAGIC_INOUT
  sv_setsv($arg, sv_2mortal(@{[ $type =~ /_(\w*)/ ]}_wrap($var)));

```
Of course, you can choose your function names and type names to fit more conveniently into
these patterns.

## Finding the MainObject for a SubObject

Now, you maybe noticed that I made the convenient assumption that the C library has a function
that looks up the MainObject of a SubObject:

```
SomeLib_MainObject *main= SomeLib_SubObject_get_main(sub_obj);
```

That isn't always the case.  Sometimes the library authors assume you have both pointers handy
and don't bother to give you a function to look one up from the other.

The easiest workaround is if you can assume that any function which returns a SubObject also
took a parameter of the MainObject as an input.  Then, just standardize the variable name given
to the MainObject and use that variable name in the typemap macro.  (and yes, this breaks the
cool trick I just showed for parameterizing the typemap rules to handle multiple types)

```
OUTPUT
O_SomeLib_SubObject
  sv_setsv($arg, sv_2mortal(SubObject_wrap(main, $var)));
```

This macro blindly assumes that "main" will be in scope where the macro gets expanded, which is
true for my example:

```
SomeLib_SubObject *
find_subobject(main, search_key)
  SomeLib_MainObject *main
  char *key
```

But, what if it isn't?  What if the C API is basically walking a linked list, and
you want to expose it to Perl in a way that the user can write:

```
for (my $subobj= $main->first; $subobj; $subobj= $subobj->next) {
  ...
}
```

The problem is that the "next" method is acting on one SubObject and returning another
SubObject, with no reference to "main" available.

Well, if a subobject wrapper exists, then it knows the main object, so you just need to look at
that SubObject info's pointer to `parent` (the MainObject) and make that available for the
SubObject's OUTPUT typemap:

```
SomeLib_SubObject *
next(prev_obj_info)
  struct SubObject_info *prev_obj_info;
  INIT:
    SomeLib_MainObject *main= prev_obj_info->parent;
  CODE:
    RETVAL= SomeLib_SubObject_next(prev_obj_info->obj);
  OUTPUT:
    RETVAL
```

So, now there is a variable 'main' in scope when it's time for the typemap to construct a
wrapper for the SomeLib_SubObject.

## Conclusion

In Perl, the lifespan of objects is nicely defined: the destructor runs when the last reference
is lost, and you use a pattern of strong and weak references to control the order the
destructors run.  In C, the lifespan of objects is dictated by the underlying library, and you
might need to go to some awkward lengths to track which ones the Perl user is holding onto,
and then flag those objects when they become invalid.  While somewhat awkward, it's very
*possible* thanks to weak references and hashtables keyed on the C pointer address, and the
users of your XS library will probably be thankful when they get a useful error message about
violating the lifecycle of objects, instead of a mysterious segfault.
