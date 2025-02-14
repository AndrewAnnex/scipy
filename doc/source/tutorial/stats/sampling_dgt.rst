
.. _sampling-dgt:

Discrete Guide Table (DGT)

.. currentmodule:: scipy.stats.sampling

* Required: probability vector (PV) or the PMF along with a finite domain
* Speed:

    * Set-up: slow (linear with the vector-length)
    * Sampling: very fast


DGT samples from arbitrary but finite probability vectors. Random numbers
are generated by the inversion method, i.e.

1. Generate a random number U ~ U(0,1).
2. Find smallest integer I such that F(I) = P(X<=I) >= U.

Step (2) is the crucial step. Using sequential search requires O(E(X))
comparisons, where E(X) is the expectation of the distribution. Indexed
search, however, uses a guide table to jump to some I' <= I near I to find
X in constant time. Indeed the expected number of comparisons is reduced to
2, when the guide table has the same size as the probability vector
(this is the default). For larger guide tables this number becomes smaller
(but is always larger than 1), for smaller tables it becomes larger.

On the other hand the setup time for guide table is O(N), where N denotes
the length of the probability vector (for size 1 no preprocessing is
required). Moreover, for very large guide tables memory effects might even
reduce the speed of the algorithm. So we do not recommend to use guide
tables that are more than three times larger than the given probability
vector. If only a few random numbers have to be generated, (much) smaller
table sizes are better. The size of the guide table relative to the length
of the given probability vector can be set by the `guide_factor` parameter:

    >>> import numpy as np
    >>> from scipy.stats.sampling import DiscreteGuideTable
    >>>
    >>> pv = [0.18, 0.02, 0.8]
    >>> urng = np.random.default_rng()
    >>> rng = DiscreteGuideTable(pv, random_state=urng)
    >>> rng.rvs()
    2

By default, the probability vector is indexed starting at 0. However, this
can be changed by passing a ``domain`` parameter. When ``domain`` is given
in combination with the PV, it has the effect of relocating the
distribution from ``(0, len(pv))`` to ``(domain[0], domain[0] + len(pv))``.
``domain[1]`` is ignored in this case.

    >>> rng = DiscreteGuideTable(pv, random_state=urng, domain=(10, 13))
    >>> rng.rvs()
    10

The method also works when no probability vector but a PMF is given.
In that case, a bounded (finite) domain must also be given either by
passing the ``domain`` parameter explicitly or by providing a ``support``
method in the distribution object:

    >>> class Distribution:
    ...     def __init__(self, c):
    ...             self.c = c
    ...     def pmf(self, x):
    ...             return x ** self.c
    ...     def support(self):
    ...             return 0, 10
    ...
    >>> dist = Distribution(2)
    >>> rng = DiscreteGuideTable(dist, random_state=urng)
    >>> rng.rvs()
    9

.. note:: As :class:`~DiscreteGuideTable` expects PMF with signature
          ``def pmf(self, x: float) -> float``, it first vectorizes the
          PMF using ``np.vectorize`` and then evaluates it over all the
          points in the domain. But if the PMF is already vectorized,
          it is much faster to just evaluate it at each point in the domain
          and pass the obtained PV instead along with the domain.
          For example, ``pmf`` methods of SciPy's discrete distributions
          are vectorized and a PV can be obtained by doing:

          >>> from scipy.stats import binom
          >>> from scipy.stats.sampling import DiscreteGuideTable
          >>> dist = binom(10, 0.2)  # distribution object
          >>> domain = dist.support()  # the domain of your distribution
          >>> x = np.arange(domain[0], domain[1] + 1)
          >>> pv = dist.pmf(x)
          >>> rng = DiscreteGuideTable(pv, domain=domain)

          Domain is required here to relocate the distribution

The size of the guide table relative to the probability vector may be set using
the ``guide_factor`` parameter. Larger guide tables result in faster generation
time but require a more expensive setup.

    >>> guide_factor = 2
    >>> rng = DiscreteGuideTable(pv, random_state=urng, guide_factor=guide_factor)
    >>> rng.rvs()
    2

Unfortunately, the PPF is rarely available in closed form or too slow when
available. The user only has to provide the probability vector and the 
PPF (inverse CDF) can be evaluated using ``ppf`` method. This 
method calculates the (exact) PPF of the given distribution.

For example to calculate the PPF of a binomial distribution with :math:`n=4` and
:math:`p=0.1`: we can set up a guide table as follows:

    >>> n, p = 4, 0.1
    >>> dist = stats.binom(n, p)
    >>> rng = DiscreteGuideTable(dist, random_state=42)
    >>> rng.ppf(0.5)
    0.0

Please see [1]_ and [2]_ for more details on this method.

References
----------

.. [1] UNU.RAN reference manual, Section 5.8.4,
       "DGT - (Discrete) Guide Table method (indexed search)"
       https://statmath.wu.ac.at/unuran/doc/unuran.html#DGT

.. [2] H.C. Chen and Y. Asau (1974). On generating random variates from an
       empirical distribution, AIIE Trans. 6, pp. 163-166.
