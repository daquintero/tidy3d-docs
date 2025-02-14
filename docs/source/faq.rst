Frequently Asked Questions
==========================

How do I run a simulation and access the results?
-------------------------------------------------

Submitting and monitoring jobs, and donwloading the results, is all done 
through our `web API <api.html#web-api>`_. After a successful run, 
all data for all monitors can be downloaded in a single ``.hdf5`` file 
using :meth:`tidy3d.web.webapi.load`, and the
raw data can be loaded into a :class:`.SimulationData` object.

From the :class:`.SimulationData` object, one can grab and plot the data for each monitor with square bracket indexing, inspect the original :class:`.Simulation` object, and view the log from the solver run.  For more details, see `this tutorial <notebooks/VizSimulation.html>`_.

How is using Tidy3D billed?
---------------------------

The `Tidy3D client <https://pypi.org/project/tidy3d/>`_ that is used for designing 
simulations and analyzing the results is free and 
open source. We only bill the run time of the solver on our server, taking only the compute 
time into account (as opposed to overhead e.g. during uploading).
When a task is uploaded to our servers, we will print the maximum incurred cost in Flex units.
This cost is also displayed in the online interface for that task.
This value is determined by the cost associated with simulating the entire time stepping specified.
If early shutoff is detected and the simulation completes before the full time stepping period, this
cost will be pro-rated.
For more questions or to purchase Flex units, please contact us at ``support@flexcompute.com``.

What are the units used in the simulation?
------------------------------------------

We generally assume the following physical units in component definitions:

 - Length: micron (μm, 10\ :sup:`-6` meters)
 - Time: Second (s)
 - Frequency: Hertz (Hz)
 - Electric conductivity: Siemens per micron (S/μm)

Thus, the user should be careful, for example to use the speed of light 
in μm/s when converting between wavelength and frequency. The built-in 
speed of light :py:obj:`.C_0` has a unit of μm/s. 

For example:

.. code-block:: python

    wavelength_um = 1.55
    freq_Hz = td.C_0 / wavelength_um
    wavelength_um = td.C_0 / freq_Hz

Currently, only linear evolution is supported, and so the output fields have an 
arbitrary normalization proportional to the amplitude of the current sources, 
which is also in arbitrary units. In the API Reference, the units are explicitly 
stated where applicable. 

Output quantities are also returned in physical units, with the same base units as above. For time-domain outputs
as well as frequency-domain outputs when the source spectrum is normalized out (default), the following units are
used:

 - Electric field: Volt per micron (V/μm)
 - Magnetic field: Ampere per micron (A/μm)
 - Flux: Watt (W)
 - Poynting vector: Watt per micron squared (W/μm\ :sup:`2`)
 - Modal amplitude: Sqare root of watt (W\ :sup:`1/2`)

If the source normalization is not applied, the electric field, magnetic field, and modal amplitudes are divided by
Hz, while the flux and Poynting vector are divided by Hz\ :sup:`2`.

How are results normalized?
---------------------------

In many cases, Tidy3D simulations can be run and well-normalized results can be obtained without normalizing/empty runs.
This is because care is taken internally to normalize the injected power, as well as the output results, in a
meaningful way. To understand this, there are two separate normalizations that happen, outlined below. Both of those are
discussed with respect to frequency-domain results, as those are the most commonly used.

Source spectrum normalization
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Every source has a spectrum associated to its particular time dependence that is imprinted on the fields injected
in the simulation. Usually, this is somewhat arbitrary and it is most convenient for it to be taken out of the
frequency-domain results. By default, after a run, Tidy3D normalizes all frequency-domain results by the spectrum of the first source
in the list of sources in the simulation. This choice can be modified using the :py:obj:`.Simulation.normalize_index` attribute, or
normalization can be turned off by setting that to ``None``. Results can even be renoramlized after the simulation run using
:meth:`.SimulationData.renormalize`. If multiple sources are used, but they all have the same
time dependence, the default normalization is still meaningful. However, if different sources have a different time dependence,
then it may not be possible to obtain well-normalized results without a normalizing run.

This type of normalization is applied directly to the frequency-domain results. The custom pulse amplitude and phase defined in
:py:obj:`.SourceTime.amplitude` and :py:obj:`.SourceTime.phase`, respectively, are **not** normalized out. This gives the user control
over a (complex) prefactor that can be applied to scale any source.
Additionally, the power injected by each type of source may have some special normalization, as outlined below.

Source power normalization
^^^^^^^^^^^^^^^^^^^^^^^^^^

Source power normalization is applied depending on the source type. In the cases where normalization is applied,
the actual injected power may differ slightly from what is described below due to finite grid effects. The normalization
should become exact with sufficiently high resolution. That said, in most cases the error is negligible even at default resolution.

The injected power values described below assume that the source spectrum normalization has also been applied.

- :class:`.PointDipole`: Normalization is such that the power injected by the source in a homogeneous material of
  refractive index :math:`n` at frequency :math:`\omega = 2\pi f` is given by

  .. math::
      \frac{\omega^2}{12\pi}\frac{\mu_0 n}{c}.

- :class:`.UniformCurrentSource`: No extra normalization applied.
- :class:`.CustomFieldSource`: No extra normalization applied.
- :class:`.ModeSource`, :class:`.PlaneWave`, :class:`.GaussianBeam`, :class:`.AstigmaticGaussianBeam`:
  Normalized to inject 1W power at every frequency. If supplied :py:obj:`.SourceTime.num_freqs` is ``1``, this normalization is
  only exact at the central frequency of the associated :class:`.SourceTime` pulse, but should still be
  very close to 1W at nearby frequencies too. Increasing ``num_freqs`` can be used to make sure the normalization
  works well for a broadband source.

  The correct usage for a :class:`.PlaneWave` source is to span the whole simulation domain for a simulation with
  periodic (or Bloch) boundaries, in which
  case the normalization of this technically infinite source is equivalent to 1W per unit cell. For the other sources
  which have a finite extent, the normalization is correct provided that the source profile decays by the boundaries
  of the source plane. Verifying that this is the case is always advised, as otherwise results may be spurious
  beyond just the normalization (numerical artifacts will be present at the source boundary).
- :class:`.TFSFSource`: Normalized to inject 1W/μm\ :sup:`2` in the direction of the source injection axis. This is convenient
  for computing scattering and absorption cross sections without the need for additional normalization. Note that for angled incidence,
  a factor of :math:`1/\cos(\theta)` needs to be applied to convert to the power carried by the plane wave in the propagation direction,
  which is at an angle :math:`\theta` with respect to the injection axis. Note also that when the source spans the entire simulation
  domain with periodic or Bloch boundaries, the conversion between the normalization of a :class:`.TFSFSource` and a :class:`.PlaneWave`
  is just the area of the simulation domain in the plane normal to the injection axis.

Why is a simulation diverging?
------------------------------

Sometimes, a simulation is numerically unstable and can result in divergence. All known cases where
this may happen are related to PML boundaries and/or dispersive media. Below is a checklist of things
to consider.

- For dispersive materials with :math:`\epsilon_{\infty} < 1`, decrease the value of the Courant stability factor to
  below :math:`\sqrt{\epsilon_{\infty}}`.
- Move PML boundaries further away from structure interfaces inside the simulation domain, or from sources that may be injecting
  evanescent waves, like :class:`.PointDipole`, :class:`.UniformCurrentSource`, or :class:`.CustomFieldSource`.
- Make sure structures are translationally invariant into the PML, or if not possible, use :class:`.Absorber` boundaries.
- Remove dispersive materials extending into the PML, or if not possible, use :class:`.Absorber` boundaries.
- If using our fitter to fit your own material data, make sure you are using the :class:`.plugins.StableDispersionFitter`.
- If none of the above work, try using :class:`.StablePML` or :class:`.Absorber` boundaries anyway
  (note: these may introduce more reflections than in usual simulations with regular PML).

How do I include material dispersion?
-------------------------------------

Dispersive materials are supported in Tidy3D and we provide an extensive 
`material library <api.html#material-library>`_ with pre-defined materials. 
Standard `dispersive material models <api.html#dispersive-mediums>`_ can also be defined. 
If you need help inputting a custom material, let us know!

It is important to keep in mind that dispersive materials are inevitably slower to 
simulate than their dispersion-less counterparts, with complexity increasing with the 
number of poles included in the dispersion model. For simulations with a narrow range 
of frequencies of interest, it may sometimes be faster to define the material through 
its real and imaginary refractive index at the center frequency. This can be done by 
defining directly a value for the real part of the relative permittivity 
:math:`\mathrm{Re}(\epsilon_r)` and electric conductivity :math:`\sigma` of a :class:`.Medium`, 
or through a real part :math:`n` and imaginary part :math:`k`of the refractive index at a 
given frequency :math:`f`. The relationship between the two equivalent models is 

.. math::

    &\mathrm{Re}(\epsilon_r) = n^2 - k^2 

    &\mathrm{Im}(\epsilon_r) = 2nk

    &\sigma = 2 \pi f \epsilon_0 \mathrm{Im}(\epsilon_r)

In the case of (almost) lossless dielectrics, the dispersion could be negligible in a broad 
frequency window, but generally, it is importat to keep in mind that such a 
material definition is best suited for single-frequency results.

For lossless, weakly dispersive materials, the best way to incorporate the dispersion 
without doing complicated fits and without slowing the simulation down significantly is to 
provide the value of the refractive index dispersion :math:`\mathrm{d}n/\mathrm{d}\lambda` 
in :meth:`.Sellmeier.from_dispersion`. The value is assumed to be 
at the central frequency or wavelength (whichever is provided), and a one-pole model for the 
material is generated. These values are for example readily available from the 
`refractive index database <https://refractiveindex.info/>`_.

Why did my simulation finish early?
-----------------------------------

By default, Tidy3D checks periodically the total field intensity left in the simulation, and compares
that to the maximum total field intensity recorded at previous times. If it is found that the ratio
of these two values is smaller than 10\ :sup:`-5`, the simulation is terminated as the fields remaining
in the simulation are deemed negligible. The shutoff value can be controlled using the :py:obj:`.Simulation.shutoff`
parameter, or completely turned off by setting it to zero. In most cases, the default behavior ensures
that results are correct, while avoiding unnecessarily long run times. The Flex Unit cost of the simulation
is also proportionally scaled down when early termination is encountered.

Should I make sure that fields have fully decayed by the end of the simulation?
-------------------------------------------------------------------------------

Conversely to early termination, you may sometimes get a warning that the fields remaining in the simulation
at the end of the run have not decayed down to the pre-defined shutoff value. This should **usually** be avoided
(that is to say, :py:obj:`.Simulation.run_time` should be increased), but there are some cases in which it may
be inevitable. The important thing to understand is that in such simulations, frequency-domain results cannot
always be trusted. The frequency-domain response obtained in the FDTD simulation only accurately represents
the continuous-wave response of the system if the fields at the beginning and at the end of the time stepping are (very close to) zero.
That said, there could be non-negligible fields in the simulation yet the data recorded in a given monitor
can still be accurate, if the leftover fields will no longer be passing through the monitor volume. From the
point of view of that monitor, fields have already fully decayed. However, there is no way to automatically check this.
The accuracy of frequency-domain monitors when fields have not fully decayed is also discussed in one of our
`FDTD 101 videos <https://www.flexcompute.com/fdtd101/Lecture-3-Applying-FDTD-to-Photonic-Crystal-Slab-Simulation/>`_.

The main use case in which you may want to ignore this warning is when you have high-Q modes in your simulation that would require
an extremely long run time to decay. In that case, you can use the the :class:`.ResonanceFinder` plugin to analyze the modes,
as well as field monitors with apodization to capture the modal profiles. The only thing to note is that the normalization of
these modal profiles would be arbitrary, and would depend on the exact run time and apodization definition. An example of
such a use case is presented in our high-Q photonic crystal cavity `case study <notebooks/OptimizedL3.html>`_.


Why can I not change Tidy3D instances after they are created?
-------------------------------------------------------------

You may notice in Tidy3D versions 1.5 and above that it is no longer possible to modify instances of Tidy3D components after they are created.
Making Tidy3D components immutable like this was an intentional design decision intended to make Tidy3D safer and more performant.

For example, Tidy3D contains several "validators" on input data.
If models are mutated, we can't always guarantee that the resulting instance will still satisfy our validations and the simulation may be invalid.

Furthermore, making the objects immutable allows us to cache the results of many expensive operations.
For example, we can now compute and store the simulation grid once, without needing to worry about the value becoming stale at a later time, which significantly speeds up plotting and other operations.

If you have a Tidy3D component that you want to recreate with a new set of parameters, instead of ``obj.param1 = param1_new``, you can call ``obj_new = obj.copy(update=dict(param1=param1_new))``.
Note that you may also pass more key value pairs to the dictionary in ``update``.
Also, note you can use a convenience method ``obj_new = obj.updated_copy(param1=param1_new)``, which is just a shortcut to the ``obj.copy()`` call above.


What do I need to know about the numerical grid?
------------------------------------------------

Tidy3D tries to provide an illusion of continuity as much as possible, but at the level of the solver a finite numerical grid is used, which
can have some implications that advanced users may want to be aware of.


.. image:: img/yee_grid.png
  :width: 600
  :alt: Field components on the Yee grid

The FDTD method for electromagnetic simulations uses what's called the Yee grid, in which every field component is defined at a different spatial location, as illustrated in the figure, as well as in our FDTD video tutorial `FDTD 101 videos <https://www.flexcompute.com/fdtd101/Lecture-1-Introduction-to-FDTD-Simulation/>`_. On the left, we show one cell of the full 3D Yee grid, and where the various ``E`` and ``H`` field components live. On the right, we show a cross-section in the xy plane, and the locations of the ``Ez`` and ``Hz`` field components in that plane (note that these field components are not in the same cross-section along ``z`` but rather also offset by half a cell size). This illustrates a duality between the grids on which ``E`` and ``H`` fields live, which is related to the duality between the fields themselves. There is a primal grid, shown with solid lines, and a dual grid, shown with dashed lines, with the ``Ez`` and ``Hz`` fields living at the primal/dual vertices in the ``xy``-palne, respectively. In some literature on the FDTD method, the primal and dual grids may even be switched as the definitions are interchangeable. In Tidy3D, the primal grid is as defined by the solid lines in the Figure.

When computing results that involve multiple field components, like Poynting vector, flux, or total field intensity, it is important to use fields that are defined at the
same locations, for best numerical accuracy. The field components thus need to be interpolated, or colocated, to some common coordinates. All this is already done under the
hood when using Tidy3D in-built methods to compute such quantities. When using field data directly, Tidy3D provides several conveniences to handle this. Firstly, field monitors have a ``colocate`` option, set to ``True`` by default, which will automatically return the field data interpolated to the primal grid vertices. The data is then ready to be used directly for computing quantities derived from any combination of the field components. The ``colocate`` option can be turned off by advanced users, in which case each field component will have different coordinates as defined by the Yee grid. In some cases, this can lead to more accurate results, as discussed for example in the `custom source example <notebooks/CustomFieldSource.html>`_. In that example, when using data generated by one simulation as a source in another, it is best to use the fields as recorded on the Yee grid.

Regardless of whether the ``colocate`` option is on or off for a given monitor, the data can also be easily colocated after the solver run. In principle, if colocating to locations other than the primal grid in post-processing, it is more accurate to set ``colocate=False`` in the monitor to avoid double interpolation (first to the primal grid in the solver, then to new locations). Regardless, the following methods work for both Yee grid data and data that has already been previously colocated:

- ``data_at_boundaries = sim_data.at_boundaries(monitor_name)`` to colocate all fields of a monitor to the Yee grid cell boundaries (i.e. the primal grid vertexes).
- ``data_at_centers = sim_data.at_centers(monitor_name)`` to colocate all fields of a monitor to the Yee grid cell centers (i.e. the dual grid vertexes).
- ``data_at_coords = sim_data[monitor_name].colocate(x=x_points, y=y_points, z=z_points)`` to colocate all fields to a custom set of coordinates. Any or all of ``x``, ``y``, and ``z`` can be supplied; if some are not, the original data coordinates are kept along that dimension.


Can you convert a lumerical script file to Tidy3D?
--------------------------------------------------

We offer a limited ability to convert Lumerical .lsf project files to Tidy3D skeleton files in python. This can be done with the command ``tidy3d convert lumerical_project.lsf tidy3d_script.py``. This is an experimental feature, and not every command in lsf is covered, and lsf project files often have default values/conventions that are not specified, so the created Tidy3D script will often need additional specification. Always be sure to check over the created Tidy3D script to see if there are any values missing or if any Lumerical objects have not been parsed.


