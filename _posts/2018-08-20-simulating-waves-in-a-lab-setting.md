---
layout: post
title: "Simulating waves in a lab setting"
author: "Milan Curcic"
---

As we're getting ready for the re-do of the Donelan et al (2004) experiment
in the ASIST tank, I set up some preliminary numerical experiments
with the University of Miami Wave Model (UMWM, Donelan et al., 2012).
These will inform us about the wave properties that we can expect in the lab
environment, and hopefully point toward the parts of the model do not
capture realistic evolution of a growing wave state.
The model will also tell us about the spatial and temporal scales,
for example, how long does it take for the wave state to saturate,
and whether the fetch (tank size) is long enough to reach equilibrium
in a variety of wind conditions.

Wind-induced wave growth is directly related to momentum and energy loss
in the lower atmosphere due to surface drag. Among few other processes,
surface drag determines whether hurricanes and other extreme weather phenomena
will grow stronger or weaken and fall apart. It is thus crucial to fully
understand and simulate this process in coupled atmosphere-wave-ocean models
that are used to predict hurricane track and intensity and provide guidance to
the public.

Jump to:

* [Wave energy balance](#wave-energy-balance)
* [Jeffreys' sheltering hypothesis](#jeffreys-sheltering-hypothesis)
* [Connection to stress](#connection-to-stress)
* [Model Configuration](#model-configuration)
* [Results](#results)
* [Key points](#key-points)
* [References](references)

## Wave energy balance

The model I use here (UMWM) is a 3rd generation spectral ocean wave model.
_Spectral_ here means that the model solves for the statistical distribution
of wave energy in a range of frequencies (wavenumbers) and directions.
The energy spectrum can then be integrated to obtain various wave quantities
such as significant wave height, mean and dominant period and wavelength,
atmosphere- and ocean-side stresses, and so on.
Spectral approach to wave modeling also implies phase-averaging - individual
waves are not resolved, but the probability for a wave of certain frequency and
amplitude to appear is. In the phase-averaged framework,
the evolution of waves is described by the wave energy balance:

$$
\dfrac{\partial F(k, \theta, x, t)}{\partial t} +
\dfrac{\partial (\dot{\mathbf{x}} F)}{\partial x} +
\dfrac{\partial (\dot{\mathbf{k}} F)}{\partial k} +
\dfrac{\partial (\dot{\theta} F)}{\partial \theta} =
\rho_w g \left( S_{in} + S_{ds} + S_{nl} \right)
$$

where $F$ is the wave elevation variance as function of
wavenumber $k$, direction $\theta$, and space and time.
$\rho_w$ is the water density and $g$ is the gravitational acceleration.
The rightmost three terms on the left-hand side are advection terms in
in geographical, wavenumber, and directional space.
$S_{in}$, $S_{ds}$, and $S_{nl}$ are the wave growth, dissipation,
and non-linear interaction source terms.
These are empirical functions that determine how much should waves grow,
decay, and transform in spectral space depending on the current wave state
and external forcing such as surface wind and ocean currents.

If you're not familiar with the spectral approach to wave physics,
$F$ describes how much elevation variance there is in each wavenumber and
directional bin. For example, higher values of $F$ for high values of $k$
would indicate more energy in short waves, and vice versa.
While seemingly simplistic, the spectral wave energy approach is incredibly
powerful in describing the interactions in the coupled wind-wave-current system
on time scales longer the several wave periods.

The key term that connects the atmospheric boundary layer and the wave state
is the wave growth source term $S_{in}$. Perhaps the most established approach
to determine the wave growth is Jeffreys' sheltering hypothesis.

## Jeffreys' sheltering hypothesis

Jeffreys' sheltering hypothesis (Jeffreys, 1924; 1925) states that propagating
waves create fields of low and high pressure on the forward and rear faces
of the wave, respectively. This pressure gradient acts to displace the
water surface and generate waves, and is assumed proportional to the square of
the wind speed relative to the phase speed of the wave:

$$
S_{in}(k, \theta) = \mathcal{A} \dfrac{\rho_a}{\rho_w}
\left(U_{\lambda/2} \cos{\varphi} - C_p \right)
\left|U_{\lambda/2} \cos{\varphi} - C_p \right|
\dfrac{\omega k}{g} F(k, \theta)
$$

$\rho_a$ is the air density, $U_{\lambda/2}$ is the mean wind speed at the
height of half wavelength, $\varphi$ is the direction between wind and waves,
$C_p$ is the phase speed of the wave, and $\omega$ is its angular frequency.
Finally, $\mathcal{A}$ is the so-called _sheltering coefficient_.
This number controls how much energy and momentum is transferred from
wind to waves, given the input wind-wave state.

However, the sheltering coefficient $\mathcal{A}$ has historically been
elusive. Most researchers sought out a constant value for this coefficient,
and in most cases it fell between 0.1 and 0.3.
In the first iteration of the UMWM (Donelan et al., 2012), $\mathcal{A}$
($A_1$ in that paper) is constant and takes three different values depending
on the wind-wave conditions:

* 0.11 if $U_{\lambda/2} \cos{\varphi} > C_p$ -- wind forcing waves;
* 0.10 if $\cos{\varphi} < 0$ -- waves against wind;
* 0.01 if $0 < U_{\lambda/2} \cos{\varphi} < C_p$ -- swell outrunning wind.

Since then, we have learned that $\mathcal{A}$ should be closer to 0.01
when waves are against wind, and closer to 0.001 for swell outrunning wind,
for wave energy spectra to match up with the observed ones in hurricane
conditions (Chen and Curcic, 2016). Earlier this year and shortly before passing,
Mark Donelan proposed that $\mathcal{A}$ should actually be a function of
the waves Reynolds number resulting in drag coefficient that decays in winds
above 30 m/s or so, reaching a local minimum at about $U_{10} = 55$ m/s.
Mark's Reynolds number data originate from the ASIST tank experiments
described in Donelan et al. (2004), so they didn't go beyond 60 m/s winds.

Measuring detailed wave-state, including the wave Reynolds number
and relating it to measured stress is a key item for both our upcoming
ASIST and SUSTAIN experiments.

## Connection to stress

I mentioned above that the wave growth source function $S_{in}$ directly
determines how much energy do waves take from wind. Due to momentum and energy
conservation, this term also determines the atmospheric surface stress, which
is the rate at which the air loses momentum due to ocean surface roughness.
The stress vector is then exactly the integral of the wave growth function
over the spectrum:

$$
\left( \tau_x, \tau_y \right) =
\rho_w g \int \int \dfrac{S_{in}}{C_p} k\ dk\
\left( \cos{\theta}, \sin{\theta} \right) d\theta
$$

Drag coefficient can be obtained directly from stress,
while losing directional information in the process:

$$
C_D = \dfrac{\mathbf{\tau}}{\rho_a U_{10}^2}
$$

Uncoupled atmosphere and ocean models without exception prescribe
the drag coefficient (or friction velocity, or roughness length, depending
on the programmer) as surface boundary condition. This is also true for
most 3rd generation wave models, such as WAM, SWAN, or Wavewatch III, which
employ wave growth functions that depend on assumed stress. UMWM takes a bit
different approach in the sense that it doesn't make assumptions about what
the stress should be, at least not beyond the value of $\mathcal{A}$ itself.
Instead, it evaluates the full stress vector from the wave growth function,
and $C_D$ can be diagnosed if desired. Stress vectors can then be passed
to atmosphere and ocean models as surface boundary condition to achieve
atmosphere-wave-ocean momentum closure.
I did this as part of my PhD thesis (Curcic, 2015).

To date, no 3rd generation ocean wave model has been able to provide a
dynamically consistent relationship between surface wind,
stress, and wave state, without a priori assumptions about stress.
Most models assume stress as function of wind, and tune the source functions
to obtain correct wave state. UMWM assumes what the wave growth should be
(given input wind), tunes the other source functions to obtain correct wave state,
and diagnoses the stress. Given that of the three (wind, wave state, and stress),
the stress is the least well understood process and also most difficult to
measure, I think it's a good idea to not make a priori assumptions about it.
A full wind-wave-stress closure is one of the principal goals of this project.

## Model configuration

For these experiments, I approximately replicated the conditions in the
SUSTAIN tank itself. I used 37 frequency bins in the range from 0.5 to 100 Hz,
with logarithmic spacing (commonly used for spectral wave models). The upper
limit of 100 Hz is more than plenty -- capillary wave regime begins at about 50 Hz.

Parameter | Value
----------|------
Length | 23 m
Width |  6 m   
$\Delta x$ |  1 m  
$\Delta y$ |  1 m
Frequency range | 0.5 - 100 Hz
Directions | 36
Wind speed  | 5 - 60 m/s
Duration | 1 minute

I did a total of 12 runs, one for each value of constant wind speed, ranging
from 5 to 60 m/s, at 5 m/s increments.
Each experiment is run through equilibrium, which is the time it takes for
the shortest wave generated at the beginning of the tank to propagate to
the other end. In low wind speed runs (5 m/s), this is about one minute.
Higher wind speed runs reach equilibrium in less time.
The waves are initialized from calm state.

A shortcoming of the model for these experiments is the lack of reflective
boundary conditions along the side walls. While straightforward to implement,
it is not in the code yet. Lack of reflective boundary conditions likely
means a small underestimate of all wave parameters in these runs.

## Results

For the first look, I want to see the overall magnitudes of common wave
and stress parameters. I plotted significant wave height, mean and dominant
wave period, friction velocity, wave age (dominant phase speed over wind speed),
and drag coefficient, as function of fetch, after one minute:

![Fetch-dependent]({{ "/assets/sustain_multipanel_fetch.png" | absolute_url }})

Each line corresponds to a different run,
forced by constant winds of 10, 20, 30, 40, 50, and 60 m/s.
From the inlet on the left to the end of the tank on the right,
significant wave height and mean period increase downwind with a parabolic shape,
due to the compounding of propagating waves.
The discrete steps in dominant wave period and wave age are due to discrete
representation of the wave spectrum in the model. To calculate the peak period
in the model, we simply look for the spectral bin with most energy. As we
move downwind, this value increases in steps as the peak energy shifts to
lower frequency bins. Finally, the drag coefficient shows saturation as we
get to the 30 m/s wind speed mark. Currently, the model assumes the
wind-speed dependent form of the sheltering coefficient $\mathcal{A}$
to achieve this saturation. During this project we will also seek a dynamically
consistent form of $\mathcal{A}$ that yields observed stess.

With the range of constant wind speed runs from 5 to 60 m/s, we can also look
at the dependence of the wave state on wind speed:

![Wind speed-dependent]({{ "/assets/sustain_multipanel_wspd.png" | absolute_url }})

The values are sampled at equilibrium (after one minute) and at the fetch of
23 m (end of the tank)
All integral wave parameters, except wave age, increase with wind speed.
Wave age shows a curious local maximum at $U_{10} = 10$ m/s. I am not sure
yet why this is.
Finally, the drag coefficient shows saturation close to that observed by
Donelan et al. (2004), although at a slightly lower level.

## Key points

* Preliminary UMWM simulations at laboratory scale provide insight into
characteristic wave parameters from low to high wind speeds.
* First results more or less realistic looking. The fact that waves don't
grow higher than mean water depth is encouraging!
* Integral wave parameters such as significant wave height and mean and dominant
period increase monotonically with wind speed, as expected.
* Local maximum in wave age at $U_{10} = 10$ m/s. To be looked at in more details.
* $C_D$ saturates at a lower level than observed by Donelan et al. (2004).
This is a matter of turning a few knobs to tweak the form of $\mathcal{A}$.

Next steps will include:

1. Plotting the modeled wave spectrum at the end of the tank;
2. Comparing the integrated model parameters and wave spectra with the
measurements from the tank;
3. Add reflective boundary condition option to the model.

This is a work in progress, developed in [this repo](https://github.com/sustain-lab/umwm-sustain).

## References

* Chen, S. S. and M. Curcic, 2016: Ocean surface waves in Hurricane Ike (2008) and Superstorm Sandy (2012): Coupled modeling and observations, *Oce. Mod.*, **103**, 161-176. doi:10.1016/j.ocemod.2015.08.005.
* Curcic, M., 2015: Explicit air-sea momentum exchange in coupled atmosphere-wave-ocean modeling of tropical cyclones, *Ph.D. Thesis*, University of Miami. [Link](http://scholarlyrepository.miami.edu/oa_dissertations/1512)
* Donelan, M. A., B. K. Haus, N. Reul, W. J. Plant, M. Stiassnie, H. C. Graber, O. B. Brown, and E. S.  Saltzman, 2004: On the limiting aerodynamic roughness of the ocean in very strong winds, *Geophys. Res. Lett.*, **31**, L18306, doi:10.1029/2004GL019460.
* Donelan, M. A., M. Curcic, S. S. Chen, and A. K. Magnusson, 2012: Modeling waves and wind stress, *J. Geophys. Res. Oceans*, **117**, C00J23, doi:10.1029/2011JC007787.
* Donelan, M. A., 2018: On the decrease of the oceanic drag coefficient in high winds, *J. Geophys. Res. Oceans*, **123**, 1485–1501. doi:10.1002/2017JC013394.
* Jeffreys, H., 1924: On the formation of waves by wind, Part 1, *Proc. Royal Soc. London, Series A*, **107**, 189–206.
* Jeffreys, H., 1925: On the formation of waves by wind, Part 2, *Proc. Royal Soc. London, Series A*, **110**, 341–347.
