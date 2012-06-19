Astrotime
==========

Summary
---------

The ``astrotime`` package is a candidate implementation of the ``astropy.time``
sub-package.  It uses Cython to wrap the C language SOFA time and calendar
routines.  All time system conversions are done by Cython vectorized versions
of the SOFA routines.  These transformations will be fast and memory efficient
(no temporary arrays).  

Other parts of the current implementation are pure Python but with a goal of
using Cython routines where possible.

Following the SOFA implementation, the internal representation of time is a
pair of doubles that sum up to the time JD in the current system.  The SOFA
routines take care throughout to maintain overall precision of the double pair.
The user is free to choose the way in which total JD is distributed between the
two values.  MOST IMPORTANTLY the user free to entirely ignore the whole issue
and just supply time values as strings or doubles and not even know what is
happening underneath.  After working with SOFA for a week I am convinced the
pair-of-doubles strategy is quite sound and useful.

Note: SOFA and most official references talk about time "scales" not time
"systems".  I find time "scale" confusing but we could easily change to that
terminology.

Build for testing
------------------

Build inplace with (requires Cython)::  

  % python setup.py build_ext --inplace

Examples
--------

::
  >>> import astrotime

  >>> times = ['1999-01-01 00:00:00.123456789', '2010-01-01 00:00:00']
  >>> t = astrotime.Time(times, format='iso', system='utc')
  >>> t
  <Time object: system='utc' format='iso' vals=['1999-01-01 00:00:00.123' '2010-01-01 00:00:00.000']>
  >>> t.jd1
  array([ 2451179.5,  2455197.5])
  >>> t.jd2
  array([  1.42889802e-06,   0.00000000e+00])

Set system to TAI::

  >>> t.set_system('tai')
  >>> t
  <Time object: system='tai' format='iso' vals=['1999-01-01 00:00:32.123' '2010-01-01 00:00:34.000']>
  >>> t.jd1
  array([ 2451179.5,  2455197.5])
  >>> t.jd2
  array([ 0.0003718 ,  0.00039352])

Get a new ``Time`` object which is referenced to the TT system (internal JD1 and JD1 are
now with respect to TT system):

  >>> t.tt  # system property returns a new Time object
  <Time object: system='tt' format='iso' vals=['1999-01-01 00:01:04.307' '2010-01-01 00:01:06.184']>

Get the representation of the ``Time`` object in a particular format (in this
case seconds since 1998.0).  This returns either a scalar or array, depending
on whether the input was a scalar or array::

  >>> t.cxcsec  # format property returns an array or scalar of that representation
  array([  3.15360643e+07,   3.78691266e+08])


Use properties to convert systems and formats.  Note that the UT1 to UTC
transformation requires a supplementary value (``delta_ut1_utc``) that can be
obtained by interpolating from a table supplied by IERS.  This will be included
in the package later.

  >>> t = astrotime.Time('2010-01-01 00:00:00', format='iso', system='utc')
  >>> t.set_delta_ut1_utc(0.3341)  # Explicitly set one part of the transformation
  >>> t.jd        # JD representation of time in current system (UTC)
  2455197.5
  >>> t.iso       # ISO representation of time in current system (UTC)
  '2010-01-01 00:00:00.000'
  >>> t.tt.iso    # ISO representation of time in TT system
  '2010-01-01 00:01:06.184'
  >>> t.tai.iso   # ISO representation of time in TAI system
  '2010-01-01 00:00:34.000'
  >>> t.utc.jd    # JD representation of time in UTC system
  2455197.5
  >>> t.ut1.jd    # JD representation of time in UT1 system
  2455197.500003867
  >>> t.tcg.isot  # ISO time with a "T" in the middle
  '2010-01-01T00:00:00.000'
  >>> t.unix      # seconds since 1970.0 (utc) excluding leapseconds
  1262304000.0
  >>> t.cxcsec    # SI seconds since 1998.0 (tt)
  378691266.184

Set the output precision which is used for some formats::

  >>> t.precision = 9
  >>> t.iso
  '2010-01-01 00:00:00.000000000'

Transform from UTC to all supported time systems (TAI, TCB, TCG, TDB, TT, UT1,
UTC).  This requires auxilliary information (latitude and longitude).

  >>> lat = 19.48125
  >>> lon = -155.933222
  >>> t = astrotime.Time('2006-01-15 21:24:37.5', format='iso', system='utc',
  ...                    precision=6, lat=lat, lon=lon)
  >>> t.set_delta_ut1_utc(0.3341)  # Explicitly set one part of the transformation
  >>> t.utc.iso
  '2006-01-15 21:24:37.500000'
  >>> t.ut1.iso
  '2006-01-15 21:24:37.834100'
  >>> t.tai.iso
  '2006-01-15 21:25:10.500000'
  >>> t.tt.iso
  '2006-01-15 21:25:42.684000'
  >>> t.tcg.iso
  '2006-01-15 21:25:43.322690'
  >>> t.tdb.iso
  '2006-01-15 21:25:42.683799'
  >>> t.tcb.iso
  '2006-01-15 21:25:56.893378'
