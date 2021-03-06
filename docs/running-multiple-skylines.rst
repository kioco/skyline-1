.. role:: skyblue
.. role:: red
.. role:: brow

Running multiple Skyline instances
==================================

It is possible to run Skyline in a somewhat distributed manner in terms of
running multiple Skyline instances and those Skyline instances sharing a MySQL
database but analysing different metrics and querying the Redis remotely where
required, e.g. Luminosity.

Some organisations have multiple Graphite instances for sharding, geographic or
latency reasons.  In such a case it is possible that each Graphite instance
would pickle to a Skyline instance and for there to be multiple Skyline
instances.

The following settings pertain to running multiple Skyline instances:

- :mod:`settings.ALTERNATIVE_SKYLINE_URLS`
- :mod:`settings.REMOTE_SKYLINE_INSTANCES`

With the introduction of Luminosity a requirement for Skyline to pull the time
series data from remote Skyline instances was added to allow for cross
correlation of all the metrics in the population.  Skyline does this via a api
endpoint and preprocesses the time series data for Luminosity on the remote
Skyline instance and returns only the fragments (gzipped) of time series
required for analysis, by default the previous 12 minutes, to minimise bandwidth
and ensure performance is maintained.

Hot standby configuration
-------------------------

Although there are many possible methods and configurations to ensure that
single points of failure are mitigated in infrastructure, this can be difficult
to achieve with both Graphite and Skyline.  This is due to the nature and volume
of the data being dealt with, especially if you are interested in ensuring
that you have redundant storage for disaster recovery.

With Graphite it is difficult to ensure the whisper data is redundant, due to
volume and real time nature of the whisper data files.

With Skyline it is difficult to ensure the real time Redis data is redundant
given that you DO NOT want to run a Redis slave as the ENTIRE key store
constantly changes.  Slaving a Skyline Redis instance is not an option
as it will use mountains of network bandwidth and just would not work.

One possible configuration to achieve redundancy of Graphite and Skyline data is
to run a Graphite and a Skyline instance as hot standbys in a different data
center.  Where the primary Graphite is pickling to a primary Skyline instance
and a standby Graphite instance.  With the standby Graphite instance pickling
data to the standby Skyline instance.

.. code-block

                        graphite-1
                            |
                       carbon-relay
          __________________|____
          |            |         |
      carbon-cache   pickle    pickle
                       |         |
                       |         +-->--> data-center-2
                       |                      |
                   skyline-1              graphite-2
                                              |
                                         carbon-relay
                                        ______|______
                                        |            |
                                  carbon-cache     pickle
                                                     |
                                                  skyline-2

In terms of the Skyline configuration of the hot standby you configure skyline-2
the same as skyline-1 in terms of alerts, etc, but you set
:mod:`settings.ANALYZER_ENABLED` and :mod:`settings.LUMINOSITY_ENABLED` to
`False`.

In the event of a failure of graphite-1 you reconfigure your things to send
their metrics to graphite-2 and set skyline-2 :mod:`settings.ANALYZER_ENABLED`
and :mod:`settings.LUMINOSITY_ENABLED` to `True`.

In the event of a failure of skyline-1 you set skyline-2
:mod:`settings.ANALYZER_ENABLED` and :mod:`settings.LUMINOSITY_ENABLED` to
`True`.

The setting up of a hot standby Graphite instance requires pickling AND periodic
flock rsyncing of all the whisper files from graphite-1 to graphite-2 to ensure
that any data that graphite-2 may have been lost in any `fullQueueDrops`
experienced with the pickle from graphite-1 to graphite-2 due to network
partitioning, etc, are updated.  flock rsyncing all the whisper files daily
mostly handles this and ensures that you have no gaps in the whisper data on
your backup Graphite instance.

Webapp UI
---------

In terms of the functionality in webapp, the webapp is multiple instance aware.
Where any "not in Redis" UI errors are found, webapp responds to the request
with a 301 redirect to the alternate Skyline URL.
