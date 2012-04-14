.. _model:

===============
Arbitrary Model
===============

.. note:: The current document only shows example use of some methods, and
          does not discuss all the options available. To see these, do not
          hesitate to use the ``help`` command, for example ``help
          m.write`` will return more detailed instructions on using the
          ``write`` method.

To create a general model, you will need to first import the Model class
from the Python ``hyperion`` module::

    from hyperion.model import Model

it is then easy to set up a generic model using::

    m = Model('example')

``Model`` requires one argument, the name of the model (this is used later
on as a prefix for the model filename). The model can then be set up using
methods of the ``Model`` instance. These are described in the following
sections.

Once the model is set up, the user can write it out to the disk for use
with the Fortran radiation transfer code::

    m.write()

If the model name was set to ``example``, the ``write`` method will write
an HDF5 file named ``example.rtin`` that is ready for use in the Fortran
code.

.. _grid:

Grid
====

The code currently supports five types of grid, all 3D:

* Cartesian grids
* Spherical polar grids
* Cylindrical polar grids
* AMR grids
* Octree grids

The following sections show the different kinds of grids should be set up.

Regular 3D grids
----------------

In the case of the cartesian and polar grids, the user should define the wall
position in each of the three directions, using cgs units for the spatial
coordinates, and radians for the angular coordinates. These wall positions
should be stored in one 1D NumPy array for each dimension, with one element
more than the number of cells defined. The walls can then be used to create a
coordinate grid using methods of the form ``set_x_grid(walls_1, walls_2,
walls_3)``. The following examples demonstrate how to do this for the various
grid types

* A 10x10x10 cartesian grid from -1 pc to +1 pc in each direction::

    x = np.linspace(-pc, pc, 11)
    y = np.linspace(-pc, pc, 11)
    z = np.linspace(-pc, pc, 11)
    m.set_cartesian_grid(x, y, z)

* A 2D 399x199 spherical polar grid::

    r = np.logspace(np.log10(rsun), np.log10(100*au), 400)
    theta = np.linspace(0., pi., 199)
    phi = np.array([0., 2*pi])
    m.set_spherical_polar_grid(r, theta, phi)

* A 3D 100x100x10 cylindrical polar grid::

    w = np.logpsace(np.log10(rsun), np.log10(100*au), 101)
    z = np.linspace(-10*au, 10*au, 101)
    phi = np.linspace(0, 2*pi, 11)
    m.set_cylindrical_polar_grid(w, z, phi)

AMR grids
---------

AMR grids have to be specified using nested Python objects. The names of the classes used, and the origin of the AMR grid is unimportant, but an AMR object has to contain a ``levels`` attribute. The ``levels`` attribute should be iterable, and contain single levels that have a ``grids`` attribute. The ``grids`` attribute should be iterable and contain single grids that have the following attributes:

* ``xmin`` - lower x position of the grid
* ``xmax`` - upper x position of the grid
* ``ymin`` - lower y position of the grid
* ``ymax`` - upper y position of the grid
* ``zmin`` - lower z position of the grid
* ``zmax`` - upper z position of the grid
* ``nx`` - number of cells in x direction
* ``ny`` - number of cells in y direction
* ``nz`` - number of cells in z direction
* ``data`` - a NumPy array with shape (``nx``, ``ny``, ``nz``) containing a physical quantity (e.g. density or temperature)

Once we have an AMR grid object, which we call ``amr`` here, the geometry can be set using::

    m.set_amr_grid(amr)

The quantity contained in the grid is unimportant for this step, as long as the geometry is correct.

For more details on how to create or read in an AMR object, see :ref:`amr_indepth`.

Octree grids
------------

Coming soon...

Density and Specific Energy
===========================

Once a regular grid is set up, it is straightforward to add one or more density grids. In this step, a dust file in HDF5 format is also required. See :ref:`dustfile` for more details about creating and using dust files in HDF5
format.

Regular 3D grids
----------------

For regular cartesian and polar grids, a 3D NumPy array containing
the density array is required. A density grid is added with::

    m.add_density_grid(density_array, dust_file)

For example::

    m.add_density_grid(np.ones(100,100,10), 'kmh.hdf5')

This command can be called multiple times if multiple density arrays are
needed (for example if different dust sizes have different spatial
distributions).

Optionally, a specific energy distribution can also be specified using a 3D NumPy
array using the ``specific_energy=`` argument::

    m.add_density_grid(density_array, dust_file, specific_energy=specific_energy_array)

.. note:: Specifying a specific energy distribution is only useful if the
          number of initial iterations for the RT code is set to zero (see
          `Specific Energy Calculation`_), otherwise the input specific energy
          will be overwritten with the self-consistently computed one.

AMR grids
---------

The density can be added using an AMR object (as described in :ref:`grid`)::

    m.add_density_grid(amr_object, dust_file)

for example::

    m.add_density_grid(amr, 'kmh.hdf5')

Specific energies can be specified using the same kinds of objects and using the `specific_energy` argument::

    m.add_density_grid(amr, dust_file, specific_energy=amr_specific_energy)

If one wants to set a preliminary specific energy based e.g. on density or a constant temperature, then one can do for example::

    # Set the AMR object
    amr = ...

    # Create a constant temperature grid
    from copy import deepcopy
    amr_specific_energy = deepcopy(amr)
    for level in amr_specific_energy.levels:
        for grid in level.grids:
            grid.data[:, :, :] = 100.  # Set to 100K

    m.add_density_grid(amr, 'kmh.hdf5', specific_energy=amr_specific_energy)

For more details on how to create or read in an AMR object, see :ref:`amr_indepth`.

Octree grids
------------

Coming soon...

Sources
=======

General notes
-------------

Sources can be added to the model using methods of the form
``m.add_*_source(arguments)``. For example adding a point source can be
done with::

    m.add_point_source(luminosity=lsun, temperature=10000.)

These methods return a handle to the source object, which if captured allow
the user to set and modify the source parameters. The following example is equivalent to the previous command::

    source = m.add_point_source()
    source.luminosity = lsun
    source.temperature = 10000.

In the rest of this section, the second notation will be used, as although it is not as concise, it is easier to read.

All sources require a luminosity, given by the ``luminosity=`` argument or the
``luminosity`` attribute, and the emission spectrum can be defined in one of
three ways:

* by specifying a spectrum using the ``spectrum=`` argument or ``spectrum``
  attribute. The spectrum should either be a tuple of (nu, fnu) or an instance
  of an atpy.Table with two columns named 'nu' and 'fnu'. For example, given a
  file ``spectrum.txt`` with two columns listing frequency and flux, the
  spectrum can be set using::

    import numpy
    spectrum = np.loadtxt('spectrum.txt', dtype=[('nu', float), ('fnu', float)])
    source.spectrum = (spectrum['nu'], spectrum['fnu'])

* by specifying a blackbody temperature using the ``temperature=`` argument or
  ``temperature`` attribute. This should be a floating point value.

* by using the local dust emissivity if neither a spectrum or temperature are
  specified.

Point Sources
-------------

A point source is defined by a luminosity, a 3D cartesian position (set to
the origin by default), and a spectrum or temperature. The following
examples demonstrate adding different point sources:

* Set up a 1 solar luminosity 10,000K point source at the origin::

    source = m.add_point_source()
    source.luminosity = lsun  # [ergs/s]
    source.temperature = 10000.  # [K]

* Set up two 0.1 solar luminosity 1,300K point sources at +/- 1 AU in the x direction::

    # Set up the first source
    source1 = m.add_point_source()
    source1.luminosity = 0.1 * lsun  # [ergs/s]
    source1.position = (au, 0, 0)  # [cm]
    source1.temperature = 1300.  # [K]

    # Set up the second source
    source2 = m.add_point_source()
    source2.luminosity = 0.1 * lsun  # [ergs/s]
    source2.position = (-au, 0, 0)  # [cm]
    source2.temperature = 1300.  # [K]

* Set up a 10 solar luminosity source at the origin with a Kurucz spectrum read in from a file with two columns giving wav (in microns) and fnu::

    # Use NumPy to read in the spectrum
    import numpy as np
    data = np.loadtxt('spectrum.txt', dtype=[('wav', float), ('fnu', float)])

    # Convert to nu, fnu
    nu = c / (data['wav'] * 1.e-4)
    fnu = data['fnu']

    # Set up the source
    source = m.add_point_source()
    source.luminosity = 10 * lsun  # [ergs/s]
    source.spectrum = (nu, fnu)

Spherical Sources
-----------------

Adding spherical sources is very similar to adding point sources, with the
exception that a radius can be specified::

    source = m.add_spherical_source()
    source.luminosity = lsun  # [ergs/s]
    source.radius = rsun  # [cm]
    source.temperature = 10000.  # [K]

It is possible to add limb darkening, using::

    source.limb_darkening = True

Spots
-----

Adding spots to a spherical source is straightforward. Spots behave the same as other sources, requiring a luminosity, spectrum, and additional geometrical parameters::

    source = m.add_spherical_source()
    source.luminosity = lsun  # [ergs/s]
    source.radius = rsun  # [cm]
    source.temperature = 10000.  # [K]

    spot = source.add_spot()
    spot.luminosity = 0.1 * lsun  # [ergs/s]
    spot.longitude = 45.  # [degrees]
    spot.latitude = 30.  # [degrees]
    spot.radius = 5.  # [degrees]
    spot.temperature = 20000.  # [K]

Map Sources
-----------

Map sources are diffuse sources that are defined by a total luminosity, and a
probability distribution map for the emission, defined on the same grid as the
density. For example, if the grid is defined on a 10x10x10 grid, the following
will add a source which emits photons from all cells equally::

    source = m.add_map_source()
    source.luminosity = lsun  # [ergs/s]
    source.map = np.ones((10, 10, 10))

.. note:: The ``map`` array does not need to be normalized.

Configuration
=============

To configure the parameters for the model, such as number of photons or number of iterations, the following methods are available::

Number of photons
-----------------

The number of photons to run in various iterations is set using the
following method::

    m.set_n_photons(...)

This method can take the following arguments, which depend on the type of radiation transfer calculations requested:

* ``initial=`` - number of photons per initial iteration to compute the
  specific energy of the dust
* ``imaging=`` - number of photons emitted in the SED/image iteration.
* ``raytracing_sources=`` - number of photons emitted from sources in the
  raytracing iteration
* ``raytracing_dust=`` - number of photons emitted from dust in the raytracing
  iteration
* ``stats=`` - used to determine how often to print out statistics

If computing the radiation transfer in monochromatic mode, the ``imaging`` argument should be replaced by:

* ``imaging_sources=`` - number of photons emitted from sources in the
  SED/image iteration.
* ``imaging_dust=`` - number of photons emitted from dust in the SED/image
  iteration.

.. note:: Only the relevant arguments need to be specified - for example if no
          sources are present, the ``*_sources`` arguments can be ignored,
          while if no dust density grids are present, the ``*_dust`` arguments
          can be ignored.

.. note:: All the required arguments have to be specified in a single call to
          ``set_n_photons``.

Specific Energy calculation
---------------------------

To set the number of initial iterations used to compute the dust specific
energy, use::

    m.set_n_initial_iterations(10)

Raytracing
----------

To enable raytracing, simply use::

    m.set_raytracing(True)

Diffusion
---------

If the model density contains regions of very high density where photons
get trapped or do not enter, one can enable either or both the modified
random walk (MRW; Min et al. 2009, Robitaille et al. 2010) and the partial
diffusion approximation (PDA; Min et al. 2009). The MRW requires a
parameter ``gamma`` which is used to determine when to start using the MRW
(see Min et al. 2009 for more details). By default, this parameter is set
to one. The following examples show how to enable the PDA and MRW respectively:

* Enable the partial diffusion approximation::

    m.set_pda(True)

* Enable the modified random walk, and set the gamma parameter to 2::

    m.set_mrw(True, gamma=2)

Dust sublimation
----------------

To set whether and how to sublimate dust, first the dust file needs to be read in, the sublimation parameters should be set, and the dust object should be passed directly to add_density::

    from hyperion.dust import SphericalDust

    dust = SphericalDust('kmh.hdf5')
    dust.set_sublimation_temperature('fast', temperature=1600)

    m.add_density_grid(density, dust)

The first argument of ``set_sublimation_temperature`` can be ``none`` (dust sublimation does not occur), ``cap`` (temperatures in excess of the one specified will be reset to the one given), ``slow`` (dust with temperatures in excess of the one specified will be gradually destroyed), or ``fast`` (dust with temperatures in excess of the one specified will be immediately destroyed).

Advanced Settings
-----------------

Set the maximum number of photon interactions::

    m.set_max_interactions(100000)

Kill all photons as soon as they are absorbed, in the imaging/SED iteration
(not in the temperature iterations)::

    m.set_kill_on_absorb(True)

Set a minimum temperature to which temperatures below this will be reset::

    m.add_density_grid(density, dust, minimum_temperature=100.)

and in terms of specific energy::

    m.add_density_grid(density, dust, minimum_specific_energy=100.)

Set the number of output bytes per floating point value (4 = 32-bit, 8 = 64-bit)::

    m.set_output_bytes(4)

