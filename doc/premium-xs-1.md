Premium XS Integrations, Part 1
===============================

There are two main schools of thought on how to interface external C code with Perl.
The first is that XS should be used to wrap all the messy low level work required for
using the C library.  The other is that the external C functions should be exposed to
perl using an extremely minimal XS layer, or the Foreign Function Interface (FFI) and
all the logic for working with the library should be written in Perl.

I subscribe to the former.  My reasons are runtime speed, and ease of debugging for
the perl scripts.  If you consider that the average C library is an awkward mess of
state machines and lightly-enforced state requirements that will segfault if not
carefully obeyed, wrapping that nicely for the perl developer is going to require a
lot of data translation and runtime sanity checks.  If you skip those runtime sanity
checks in your wrapper library, it drags down the efficiency of your subsequent Perl
development to the level of C development, which is to say, sitting around scratching
your head for hours wondering why the program keeps segfaulting.
If you write those runtime checks in Perl, your runtime performance can suffer
significantly.  If you write those runtime checks in XS, you can actually do quite a
lot of them before there's any notable decrease in the performance of the script.
C code runs an order of magntude faster than perl opcodes, so if you're going to
require the end user to use a compiled module, I feel you might as well go all the
way and put as much of the wrapper library code in XS as possible.

## Linking Perl Objects to C Objects

One of the first things you'll need to do for any C library which allocates "objects"
is to link them to a matching Perl object, usually a blessed scalar ref or hash ref.
(The C language doesn't have official objects of course, but a library often allocates
a struct or opaque pointer with a lifespan and a destructor function that they expect
you to call when you're done with it.)

If you read through the common tutorials, you'll probably see a recipe like

```
SV*
new(class, some_data)
  SV *class;
  IV some_data;
  INIT:
    LibWhaever_obj *obj;
  CODE:
    obj= LibWhaever_create(some_data);
    if (!obj) croak("LibWhaever_create failed");
    RETVAL= (SV*) newRV_noinc(newSViv((IV)obj));
    sv_bless(RETVAL, gv_stashpv("YourProject::LibWhatever", GV_ADD));
  OUTPUT:
    RETVAL

DESTROY(self)
  SV *self;
  INIT:
    LibWhaever_obj *obj;
  PPCODE:
    obj= (LibWhaever_obj*) SvIV(SvRV(self));
    LibWhaever_destroy(obj);
    XSRETURN(0);

```

This is about the least effort/overhead you can have for binding a C data structure
to a perl blessed sclar ref, and freeing it when the Perl object goes out of scope.
(you can also move some of this code to the typemap, but I'll come back to that later)

I don't like this pattern for several reasons:

  * If someone passes the object to Storable's `dclone`, it happily makes a copy of
    your scalar ref and then when the first object goes out of scope it runs the
    destructor, and the other object is now referring to freed memory and will
    probably segfault during its next use.
  * When you create a new thread in a threaded perl, it clones objects, creating the
    same bug.
  * The pointer is stored as an integer visible to perl, and could get altered by
    sloppy/buggy perl code, and then you get a segfault.
  * A user could subclass the XS object, and write their own DESTROY method that
    forgets to call `$self->SUPER::DESTROY`, leaking the C object.
  * Sloppy/buggy perl code could re-bless the class, also bypassing the `DESTROY` call.
  * Sloppy/buggy perl code could call DESTROY on something which isn't the blessed
    scalar-ref containing a valid pointer.

While most of these scenarios *shouldn't* happen, if by unfortunate circumstances they
do happen, someone loses a bunch of hours debugging it, especially if they aren't the
XS author and don't know about these pitfalls.

## Magic

A much more reliable way to link the C structs to the Perl blessed refs is through
Perl's "magic" system.  Magic is the name for essentially a pointer within the
SV/AV/HV of your object which points to a linked list of C metadata.  This metadata
describes various things, like operator-overloading or ties or other low-level perl
features.  One type of magic is reserved for "extensions" (that's you!)

There is a fair amount of effort and boilerplate to set up magic on your objects,
but consider these benefits:

  * You are guaranteed that only the object your C code created will carry the pointer
    to your C struct, and no sloppy/buggy perl-level operations can break that.
  * If the magic-attached pointer isn't present, you can cleanly die with an error
    message to the user that somehow they have called your XS method on something that
    isn't your object.
  * Your C-function destructor is described by the magic metadata, and does not rely
    on a DESTROY perl method.  This also makes destruction faster if Perl doesn't
    need to call a perl-level DESTROY function.
  * Magic can be applied equally to any type of ref, so you can use one pattern for
    whaetever you are blessing, or even let the user choose what kind of ref it will be.
  * You can even use Moo or Moose to create the object, then attach your magic to
    whatever ref the object system created.
  * You get a callback when a new Perl thread starts and attempts to clone your object.
    (letting you clone it, or throw an exception that it can't be cloned which is at
    least nicer to the user than a segfault would be)

With that in mind, lets begin suffering through the boilerplate.

## Defining Magic

Magic is described with "struct MGVTBL":

```
static int YourProject_LibWhatver_magic_free(pTHX_ SV* sv, MAGIC* mg) {
  LibWhatever_obj *obj= (LibWhaever_obj*) mg->mg_ptr;
  LibWhaever_destroy(obj);
}

#ifdef USE_ITHREADS
static int YourProject_LibWhatever_magic_dup(pTHX_ MAGIC *mg, CLONE_PARAMS *param) {
  croak("This object cannot be shared between threads");
  return 0;
};
#else
#define YourProject_LibWhatever_magic_dup 0
#endif

// magic table for YourProject::LibWhatever
static MGVTBL YourProject_LibWhatever_magic_vtbl= {
  0, /* get */
  0, /* set */
  0, /* length */
  0, /* clear */
  YourProject_LibWhatever_magic_free, /* free */
#ifdef MGf_COPY
  0, /* copy magic to new variable */
#endif
#ifdef MGf_DUP
  YourProject_LibWhatever_magic_dup /* dup for new threads */
#endif
#ifdef MGf_LOCAL
  ,0 /* local */
#endif
};
```

You only need one static inctance for each type of magic your module creates.  It's just
metadata telling perl how to handle your particular type of extension magic.  The ifdefs
are from past versions of the struct that had fewer fields, though if your module is
requiring perl 5.8 you can assume 'copy' and 'dup' exist, and from 5.10 'local' always
exists as well.

Next, the recipe to attach it to a new Perl object:

```
SV * my_wrapper(LibWhatever_obj *cstruct) {
  SV *obj, *objref;
  MAGIC *magic;
  obj= newSV(0); // or newHV() or newAV()
  objref= newRV_noinc(obj);
  sv_bless(objref, gv_stashpv("YourProject::LibWhatever", GV_ADD));
  magic= sv_magicext(
    obj,                                 /* the internal SV/AV/HV, not the ref */
    NULL,
    PERL_MAGIC_ext,                      /* "extension magic" */
    &YourProject_LibWhatever_magic_vtbl, /* show perl your functions */
    (const char*) cstruct,               /* store the pointer to the C object */
    0);
#ifdef USE_ITHREADS
  magic->mg_flags |= MGf_DUP;
#else
  (void)magic; // suppress warning
#endif
  return objref;
}
```

The key there is 'sv_magicext'.  Note that you're applying it to the thing being
referred to, not the scalarref that you use for the call to `sv_bless`.
The messy `ifdef` part is due to the 'dup' field of the magic table only being
used when perl was compiled with threading support.  The reference to
`YourProject_LibWhatever_magic_vtbl` is both an instruction for Perl to know what
functions to call, but also a unique value used to identify *your* extension
magic from anyone else's.

To read your pointer back from an SV provided to you, the recipe is:

```
LibWhatever_obj* YourProject_LibWhatever_from_magic(SV *objref) {
  SV *sv;
  MAGIC* magic;

  if (SvROK(objref)) {
    sv= SvRV(objref);
    if (SvMAGICAL(sv)) {
      /* Iterate magic attached to this scalar, looking for one with our vtable */
      for (magic= SvMAGIC(sv); magic; magic = magic->mg_moremagic)
        if (magic->mg_type == PERL_MAGIC_ext
         && magic->mg_virtual == &YourProject_LibWhatever_magic_vtbl)
          /* If found, the mg_ptr points to the fields structure. */
          return (LibWhatever_obj*) magic->mg_ptr;
    }
  }
  return NULL;
}
```

This might look a little expensive, but there is likely only one type of magic on your
object, so the loop exits on the first iteration, and all you did was "SvROK", "SvRV",
"SvMAGICAL", and "SvMAGIC" followed by two comparisons.  It's actually quite a bit
faster than verifying the inheritance of the blessed package name.

So there you go - you can now attach your C structs with magic.

## Convenience via Typemap

In a typical wrapper around a C library, you're going to be writing a lot of methods
that need to call YourProject_LibWhatever_from_magic on the first argument.  To make
that easier, lets move this decoding step to the typemap.

Without a typemap:

```
IV
method1(self, param1)
  SV *self
  IV param1
  INIT:
    LibWhatever_obj *obj= YouProject_LibWhatever_from_magic(self);
  CODE:
    if (!obj) croak("Not an instance of LibWhatever");
    RETVAL= LibWhatever_method1(obj, param1);
  OUTPUT:
    RETVAL
```

With a typemap entry like:

```
TYPEMAP
LibWhatever_obj*        O_LibWhatever_obj

INPUT
O_LibWhatever_obj
  $var= YourProject_LibWhatever_from_magic($arg);
  if (!$var) croak("Not an instance of LibWhatever");
```
the XS method becomes

```
IV
method1(obj, param1)
  LibWhatever_obj *obj
  IV param1
  CODE:
    RETVAL= LibWhatever_method1(obj, param1);
  OUTPUT:
    RETVAL
```


If you have some functions that take an optional LibWhatever_obj pointer, try this trick:

```
typedef LibWhatever_obj Maybe_LibWhatever_obj;

...

void
show(obj)
  Maybe_LibWhatever_obj *obj
  PPCODE:
    if (obj) {
      printf("...", LibWhatever_get_attr1(obj));
    }
    else {
      printf("NULL");
    }

```

```
TYPEMAP
LibWhatever_obj*        O_LibWhatever_obj
Maybe_LibWhatever_obj*  O_Maybe_LibWhatever_obj

INPUT
O_LibWhatever_obj
  $var= YourProject_LibWhatever_from_magic($arg);
  if (!$var) croak("Not an instance of LibWhatever");

INPUT
O_Maybe_LibWhatever_obj
  $var= YourProject_LibWhatever_from_magic($arg);
```

If you want to save a bit of compiled .so file size, you can move the error message
into the 'from_magic' function, with a flag:

```
#define OR_DIE 1

LibWhatever_obj* YourProject_LibWhatever_from_magic(SV *objref, int flags) {
  SV *sv;
  MAGIC* magic;

  if (SvROK(objref)) {
    sv= SvRV(objref);
    if (SvMAGICAL(sv)) {
      /* Iterate magic attached to this scalar, looking for one with our vtable */
      for (magic= SvMAGIC(sv); magic; magic = magic->mg_moremagic)
        if (magic->mg_type == PERL_MAGIC_ext
         && magic->mg_virtual == &YourProject_LibWhatever_magic_vtbl)
          /* If found, the mg_ptr points to the fields structure. */
          return (LibWhatever_obj*) magic->mg_ptr;
    }
  }
  if (flags & OR_DIE)
    croak("Not an instance of LibWhatever");
  return NULL;
}
```

```
TYPEMAP
LibWhatever_obj*        O_LibWhatever_obj
Maybe_LibWhatever_obj*  O_Maybe_LibWhatever_obj

INPUT
O_LibWhatever_obj
  $var= YourProject_LibWhatever_from_magic($arg, OR_DIE);

INPUT
O_Maybe_LibWhatever_obj
  $var= YourProject_LibWhatever_from_magic($arg, 0);
```

You can play further games with this, like automatically initializing the SV to become
one of your blessed objects if it wasn't defined, in the style of Perl's
`open my $fh, ...`, or maybe an option to add the magic to an existing object created
by a pure-perl constructor.  Do whatever makes sense for your API.

## More Than One Pointer

In all the examples so far, I'm storing a single pointer to a type defined in the
external C library being wrapped.  Chances are, though, you need to store more than just
that one pointer.

Imagine a poorly-written C library where you need to call "somelib_create" to get the
object, then a series of "somelib_setup" calls before any other function can be used,
then if you want to call "somelib_go" you have to first call "somelib_prepare" or else
it segfaults.  You could track these states in perl variables in a hashref, but it would
just be easier if they were all present in a local C struct of your creation.

So, rather than attaching a pointer to the library struct with magic, you can attach
your own allocated struct, and your struct can have a pointer to all the library details.
For extra convenience, your struct can also have a pointer to the perl object which it
is attached to, which lets you access that object from other methods you write which
won't have access to the perl stack.

```
struct YourProject_objinfo {
  SomeLib_obj *obj;
  HV *wrapper;
  bool started_setup, finished_setup;
  bool did_prepare;
};

struct YourProject_objinfo*
YourProject_objinfo_create() {
  struct YourProject_objinfo *objinfo= NULL;
  Newxz(objinfo, 1, struct YourProject_objinfo);
  /* other setup here ... */
  return objinfo;
}

void
YourProject_objinfo_free(struct YourProject_objinfo *objinfo) {
  if (objinfo->obj) {
    SomeLib_obj_destroy(objinfo->obj);
  }
  /* other cleanup here ... */
  Safefree(objinfo);
}

static int YourProject_objinfo_magic_free(pTHX_ SV* sv, MAGIC* mg) {
  YourProject_objinfo_free((struct YourProject_objinfo *) mg->mg_ptr);
}


```

One other thing that has changed from the previous scenario is that you can allocate
this struct and attach it to the object whenever you want, instead of waiting for the
user to call the function that creates the instance of `SomeLib_obj`.  This gives you
more flexible ways to deal with creation of the magic.

Here's a pattern I like:

```
#define OR_DIE 1
#define AUTOCREATE 2

struct YourProject_objinfo*
YourProject_objinfo_from_magic(SV *objref, int flags) {
  SV *sv;
  MAGIC* magic;

  if (!sv_isobject(objref))
    /* could also check package hierarchy here, but that slows things down */
    croak("Not an instance of YourProject");

  sv= SvRV(objref);
  if (SvMAGICAL(sv)) {
    /* Iterate magic attached to this scalar, looking for one with our vtable */
    for (magic= SvMAGIC(sv); magic; magic = magic->mg_moremagic)
      if (magic->mg_type == PERL_MAGIC_ext
       && magic->mg_virtual == &YourProject_SomeLib_magic_vtbl)
        /* If found, the mg_ptr points to the fields structure. */
        return (SomeLib_obj*) magic->mg_ptr;
  }
  if (flags & AUTOCREATE) {
    struct YourProject_objinfo *ret= YourProject_objinfo_create();
    ret->wrapper= (HV*) sv;
    magic= sv_magicext(sv, NULL, PERL_MAGIC_ext,
      &YourProject_SomeLib_magic_vtbl, (const char*) ret, 0);
#ifdef USE_ITHREADS
    magic->mg_flags |= MGf_DUP;
#else
    (void)magic; // suppress warning
#endif
    return ret;
  }
  if (flags & OR_DIE)
    croak("Not an initialized instance of YourProject");
  return NULL;
}

typedef struct YourProject_objinfo Maybe_YourProject_objinfo;
typedef struct YourProject_objinfo Auto_YourProject_objinfo;

```
Then in the typemap:

```
TYPEMAP
struct YourProject_objinfo*  O_YourProject_objinfo
Maybe_YourProject_objinfo*   O_Maybe_YourProject_objinfo
Auto_YourProject_objinfo*    O_Auto_YourProject_objinfo

INPUT
O_YourProject_objinfo
  $var= YourProject_objinfo_from_magic($arg, OR_DIE);

INPUT
O_Maybe_YourProject_objinfo
  $var= YourProject_objinfo_from_magic($arg, 0);

INPUT
O_Auto_YourProject_objinfo
  $var= YourProject_objinfo_from_magic($arg, AUTOCREATE);
```

Then use it in your XS methods to conveniently implement your sanity checks for this
annoying C library:

```

# This is called by the pure-perl constructor, after blessing the hashref
void
_init(objinfo, param1, param2)
  Auto_YourProject_objinfo* objinfo
  IV param1
  IV param2
  PPCODE:
    if (objinfo->obj)
      croak("Already initialized");
    objinfo->obj= SomeLib_create(param1, param2);
    if (!objinfo->obj)
      croak("SomeLib_create failed: %s", SomeLib_get_last_error());
    XSRETURN(0);

bool
_is_initialized(objinfo)
  Maybe_YourProject_objinfo* objinfo
  CODE:
    RETVAL= objinfo != NULL && objinfo->obj != NULL;
  OUTPUT:
    RETVAL

void
setup(objinfo, key, val)
  struct YourProject_objinfo* objinfo
  const char *key
  const char *val
  PPCODE:
    if (objinfo->finished_setup)
      croak("Cannot call 'setup' after 'prepare'");
    if (!SomeLib_setup(objinfo->obj, key, val))
      croak("SomeLib_setup failed: %s", SomeLib_get_last_error());
    objinfo->setup_started= true;
    XSRETURN(0);

void
prepare(objinfo)
  struct YourProject_objinfo* objinfo
  PPCODE:
    if (!objinfo->started_setup)
      croak("Must call setup at least once before 'prepare'");
    objinfo->finished_setup= true;
    if (!SomeLib_prepare(objinfo->obj))
      croak("SomeLib_prepare failed: %s", SomeLib_get_last_error());
    objinfo->did_prepare= true;
    XSRETURN(0);

void
somelib_go(objinfo)
  struct YourProject_objinfo* objinfo
  PPCODE:
    if (!objinfo->did_prepare)
      croak("Must call 'prepare' before 'go'");
    if (!SomeLib_go(objinfo->obj))
      croak("SomeLib_go failed: %s", SomeLib_get_last_error());
    XSRETURN(0);
```

Like how clean the XS methods got?

## Conclusion

When you use the pattern above, your module becomes almost foolproof against misuse.
You provide helpful errors for the Perl coder to guide them toward correct usage of
the library with easy-to-understand errors (well, depending on how much effort you
spend on that) and they don't have to pull their hair out trying to log all the API
calls and compare to the C library documentation to figure out which one happened in
the wrong order resulting in a mysterious crash.

The code above is all assuming that the C library is providing objects whose lifespan
*you* are in control of.  Many times, the objects from a C library will have some
other lifespan that the user can't directly control with the perl objects.  I'll
cover some techniques for dealing with that in the next article.