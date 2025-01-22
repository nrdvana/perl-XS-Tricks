# Perl to XS Cheatsheet

This table is a list of the most common Perl operations, and how to accomplish
roughly the same thing in XS.  These are not exact difinitive translations!
You should look up the documentation for each C function in `perldoc perlapi`
and read the caveats involved.  In particular, any time you write to a SV that
you didn't create yourself, you should use `sv_setsv_mg` to handle the write-magic
that might be associated with it, if it were tied, for example.  But, if you
created the SV yourself and know it isn't tied, you can save time by skipping that
check.  Much of the Perl API follows that pattern, where you *could* use a very
robust API call, but it is much faster to use a less-safe API call based on your
knowledge of the situation.

## Basic operations

### Scalars
```
  defined $x                          SvOK(x)
  !!$x                                SvTRUE(x)
  undef $x                            sv_setsv(x, &PL_sv_undef)
                                      sv_set_undef(x)            // since 5.26
  Scalar::Util::looks_like_number($x) looks_like_number(x)
  int($x)                             SvIV(x), SvUV(x)
  $x+0                                SvNV(x)
  "$x"                                SvPV_nolen(x)
  my $x= 1                            newSViv(1)
  my $x= 1.5                          newSVnv(1.5)
  my $x= "test"                       newSVpvn("test", 4)
  $x= 1                               sv_setiv(x, 1)
  $x= 1.5                             sv_setnv(x, 1.5)
  $x= !0;                             sv_setsv(x, &PL_sv_yes)
  $x= !1;                             sv_setsv(x, &PL_sv_no)
  $x= !$y                             sv_setbool(x, !SvTRUE(y))
  length $x                           SvCUR(x)
  $x= "test"                          sv_setpvn(x, "test", 4)
  $x= Scalar::Util::dualvar("x",1)    {
                                        SV *x= newSVpvs("x");
                                        SvUPGRADE(x, SVt_PVIV);
                                        SvIV_set(x, 1);
                                        SvIOK_on(x);
                                      }
```

### References
```
  if (ref $x)                         if (SvROK(x))
  $$x                                 SvRV(x)
  \$x                                 newRV_inc(x)
  Scalar::Util::weaken($x)            sv_rvweaken(x)
```

### Arrays
```
  if (ref $x eq 'ARRAY')              if (SvROK(x) && SvTYPE(SvRV(x)) == SVt_PVAV)
  my @x                               x= newAV();
  $#x                                 av_len(x)
  scalar @x                           1+av_len(x)
  $x[5]                               SV **element= av_fetch(x, 5, 0)
  my $x= []                           x= newRV_noinc((SV*)newAV())
  $#$x                                av_len((AV*)SvRV(x))
  $x->[5]                             SV **element= av_fetch((AV*)SvRV(x), 5, 0)
  
  # get a list of array elements      /* see "Advanced Recipe: unwrap_array" below */
  # even if tied:                     IV len;
  ()= @x                              SV **list= unwrap_array(x, &len);
```

### Hashes
```
  if (ref $x eq 'HASH')               if (SvROK(x) && SvTYPE(SvRV(x)) == SVt_PVHV)
  my %x                               x= newHV()
  my %x= %y                           x= newHVhv(y)
  $x{1}                               SV **field= hv_fetchs(x, "1", 0)
  $x{1}= 1                            hv_stores(x, "1", newSViv(1))
  exists $x{1}                        hv_exists(x, "1", 1)
  %x= ()                              hv_clear(x)
  delete $x{1}                        hv_delete(x, "1", 1, G_DISCARD)
  $y= delete $x{1}                    sv_setsv(y, hv_delete(x, "1", 1, 0))
  my $x= {}                           x= newRV_noinc((SV*)newHV())
  $x->{1}                             SV **field= sv_fetchs((HV*) SvRV(x), "1", 0)
  for (my ($k, $v)= each %h)          for (hv_iterinit(hv); ent= hv_iternext(hv);) {
                                        SV *k= hv_iterkeysv(ent);
                                        SV *v= hv_iterval(hv, ent);
                                      }
  # get a list of hash key/val        /* see "Advanced Recipe: unwrap_hash" below */
  # even if tied:                     IV len;
  ()= %x                              SV **list= unwrap_hash(x, &len);
```

### Subs

```
                                      /* see "Advanced Recipe: make_sub_an_lvalue" below
                                       * for complete implementation. */
  sub fn :lvalue { ... }              CvLVALUE_on(GvCV(gv_fetchmethod(stash, "fn")));
```

### Objects
```
  if (Scalar::Util::blessed($x))      if (sv_isobject(x))
  ref $x                              sv_ref(NULL, SvRV(x), 1)
  $y= ref $x                          sv_ref(y, SvRV(x), 1)
  bless $x, "Class"                   sv_bless(x, gv_stashpv("Class", GV_ADD))
                                      /* careful, bypasses overridden 'isa' methods */
  $x->isa("Class")                    sv_derived_from(x, "Class")
```

### Error Handling
```
  warn "$action may have failed"      warn("%s may have failed", action)
  die "$action failed"                croak("%s failed", action)
```

## Advanced Recipes

### unwrap_array

'Unwrap' an array to get an array of SV*

```
/* Return an SV* vector of an AV or arrayref.
 * Returns NULL if it wasn't an AV or arrayref.
 * The return value is either the underlying vector of the AV,
 * or an allocated vector which will be freed automatically.
 * The vector could be destroyed if you modify the AV!
 * The returned SVs may still have get/set magic.
 */
static SV** unwrap_array(SV *array, IV *len) {
  AV *av;
  SV **vec;
  IV n;
  if (array && SvTYPE(array) == SVt_PVAV)
    av= (AV*) array;
  else if (array && SvROK(array) && SvTYPE(SvRV(array)) == SVt_PVAV)
    av= (AV*) SvRV(array);
  else {
    *len= 0;
    return NULL;
  }
  n= av_len(av) + 1;
  vec= AvARRAY(av);
  /* tied arrays and non-allocated empty arrays return NULL */
  if (!vec) {
    if (n == 0)
      /* don't return a NULL (false) for an empty array,
       * but doesn't need to be a real pointer
       * as long as caller respects len==0 */
      vec= (SV**) 1;
    else {
      /* in case of a tied array, extract the elements
       * into a temporary buffer, freed later by perl */
      IV i;
      Newx(vec, n, SV*);
      SAVEFREEPV(vec);
      for (i= 0; i < n; i++) {
        SV **el= av_fetch(av, i, 0);
        vec[i]= el? *el : NULL;
      }
    }
  }
  *len= n;
  return vec;
}
```

### unwrap_hashref

Dump key/value pairs of a hashref into SV* array

```
/* Return an SV* vector of the key/value pairs of a HV or hashref.
 * Returns NULL if it wasn't a HV or hashref.
 * The return value does not need cleaned up.
 * The returned SVs may still have get/set magic.
 */
static SV** unwrap_hashref(SV *hash, IV *len) {
  HV *hv;
  HE *ent;
  SV **vec;
  IV i, n;
  if (hash && SvTYPE(hash) == SVt_PVHV)
    hv= (HV*) hash;
  else if (hash && SvROK(hash) && SvTYPE(SvRV(hash)) == SVt_PVAV)
    hv= (HV*) SvRV(hash);
  else
    return NULL;
  n= hv_iterinit(hv) * 2;
  Newx(vec, n, SV*);
  SAVEFREEPV(vec);
  i= 0;
  while ((ent= hv_iternext(hv)) && i < len) {
    vec[i++]= hv_iterkeysv(ent);
    vec[i++]= hv_iterval(hv, ent);
  }
  *len= n;
  return vec;
}
```

### Iterating (k,v) pairs of either the stack, or hashref

This is the common pattern used in methods that take key/value parameters,
when you want to allow the caller to use a hashref *or* bare list of k,v.

```
  use v5.36;
  use experimental qw(for_list);
  sub process_params {
    for my ($key,$val) (
      @_ == 1 && ref $_[0] eq 'HASH'? %$_
      : (@_ & 1) == 0? @_
      : croak("Expected even-length list, or hashref")
    ) {
      ...
    }
  }
```
Note that you *could* assemble a hashref from the list, and look up each known key from the
hash, but that will likely be a lot slower in C than iterating the provided key/values and
matching the key against a switch() block and then populating a local variable.
Iterating a list also gives you a chance to report unknown keys.

```
void
process_params(self, ...)
  SV *self   /* or whatever type your object is */
  INIT:
    SV *sv_list= NULL;
    IV sv_count= 0;
  PPCODE:
    if (items == 2 && SvROK(ST(1)) && SvTYPE(SvRV(ST(1))) == SVt_PVHV) {
      /* See 'Advanced Patterns: unwrap_hashref' above */
      sv_list= unwrap_hashref(SvRV(ST(1)), &sv_count);
    }
    else {
      if (!(items & 1)) croak("Expected even-length list, or hashref");
      sv_list= PL_stack_base+ax + 1;
      sv_count= items - 1;
    }
    for (i= 0; i < sv_count; i+= 2) {
      SV *key= sv_list[i];
      SV *val= sv_list[i+1];
      STRLEN klen;
      const char *key_str= SvPV(key, klen);
      switch (klen) {
      ...
      }
    }

```

### make_sub_an_lvalue

This can be called during the BOOT section of the XS module, to convert some of your subs
into lvalue subs.  The macro assumes you have a variable named 'stash' in current scope.

```
#define MAKE_SUB_AN_LVALUE(x) make_sub_an_lvalue(aTHX_ stash, #x)
static void make_sub_an_lvalue(pTHX_ HV *stash, const char *name) {
  CV *method_cv;
  GV *method_gv;
  if (!(method_gv= gv_fetchmethod(stash, name))
    || !(method_cv= GvCV(method_gv)))
    croak("Missing method %s", name);
  CvLVALUE_on(method_cv);
}
```
