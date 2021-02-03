
:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. note::

   With integration of the MTAOS with the Main Telescope components approaching we have identified some design decisions made in the past that needs revisiting.
   This technote explores some design considerations for improvements to the MTAOS operation.

The `MTAOS`_ is in charge of Main Telescope optics control loop, analyzing pairs of intra/extra focal images, computing and sending corrections for the mirror control systems (MTM1M3 and MTM2) and the Camera and M2 hexapods.
Some background and relevant information about the MTAOS can be found in the `AOS Workshop`_ confluence page.

In addition to being an extremely critical component for operating the Main Telescope (MT) the MTAOS is also unique in the sense that it interfaces with all other sub-systems, e.g.;

  - Camera, to acquire the intra/extra or corner rafts data for wavefront error processing.
  - Data Management, to process the data with instrument signature removal (ISR) and wavefront estimation pipeline (`WEP`_).
  - Telescope and Site, mainly to apply optical corrections.

Here we explore updates to the `MTAOS (xml) interface`_ and overall software architecture to align it with the evolution of the system design.
The most significant advancement that affect how the MTAOS operates are;

  - Better understanding of the observatory operation procedures.
  - Consolidation of the data access interface, through butler gen3, for local data processing of corner wavefront sensors and remote access for full array mode.
  - Consolidation of the OCS Controlled Pipeline System (`OCPS`_) for remote data processing.
  - Conversion of `WEP`_ into a butler gen3 pipeline task.

.. _MTAOS: https://ts-mtaos.lsst.io/index.html
.. _AOS Workshop: https://confluence.lsstcorp.org/x/zzTbAw
.. _MTAOS (xml) interface: https://ts-xml.lsst.io/sal_interfaces/MTAOS.html
.. _OCPS: https://dmtn-133.lsst.io
.. _WEP: https://github.com/lsst-ts/ts_wep

.. _Data-access-changes:

Data access changes
===================

The consolidation of data access through butler gen3 represents an important change in the interface and operation of the MTAOS.
The following is a series of improvements that can be implemented to improve the MTAOS usability.

Processing calibration products
-------------------------------

This is currently done via the ``processCalibrationProducts`` command, which expected a directory containing the raw calibration data.
Instead, the butler instances hosting the data will also contain processed calibration products suitable for all exposures.
This will have to be carefully managed by observatory personnel over time, but is a much more suitable approach than leaving the MTAOS in charge of the process.

Image ingestion
---------------

As part of the wavefront processing commands, the current version of the MTAOS receive files names and directory location.
This data is initially ingested to a butler instance to allow them to be ISR corrected and later WEP processed.

Since the data will now be ingested by a different component and will be made available to the MTAOS through butler, there is no more need for a local ingestion.

.. _Source-selection-command:

Source selection command
------------------------

In most cases, we will know the coordinates of the next visit field ahead of time.
That means some pre-processing activities can be done in parallel, while the telescope slews and the image is acquired.
One of these processes is to prepare a list of suitable targets for wavefront sensing.

This could be done by adding a command that would cause the MTAOS to run the source selection routine.
The command payload would minimally be;

  - RA: Right ascension of the field (in hours).
  - Dec: Declination of the field (in degrees).
  - PA: Position angle (in degrees).
  - filter: The filter used for the observation.
  - mode: An enumeration specifying the wfs mode; corner, full array, etc.

Issuing the command would look something like:

.. code-block:: py

  from lsst.ts.wep import CamType

  # Select sources for Main Camera Corner wavefront sensor mode
  await mtaos.cmd_selectSources.set_start(ra=14., dec=-10., pa=0., filter="r", type=CamType.LsstFamCam)

  # Select sources for Main Camera Full array mode
  await mtaos.cmd_selectSources.set_start(ra=14., dec=-10., pa=0., filter="r", type=CamType.LsstCam)

  # Select sources for ComCam mode
  await mtaos.cmd_selectSources.set_start(ra=14., dec=-10., pa=0., filter="r", type=CamType.ComCam)


Simplification of wfs process commands
--------------------------------------

As mentioned above, the wfs process commands receives file names and path.
With the data already ingested in the butler the images can be uniquely identified by the ``dataId`` entry in the database.
That means, all process commands need to be updated to receive the ``dataId`` instead of the files name and path.

.. note::

   For accessing the images from the butler it is also required to know the ``raft`` and ``sensor``.
   Nevertheless, this information can be derived from the sources selected for wavefront sensing (see :ref:`Source-selection-command`).

At the same time, most of the metadata that is now passed as part of the process commands (``fieldRA``, ``fieldDEC``, ``filter``, ``cameraRotation``) is also available as metadata in the butler and can be removed from the command payload.

Furthermore, through the ``dataId`` one can identify the source of the image (ComCam or LSSTCam), so there is no need for special commands for each one of them.

At the same time, we can differentiate between an Intra/Extra wavefront analysis process and a corner wavefront sensors by the input alone.
If only one input ``dataId`` then it can be assumed to be a corner wavefront sensor run.
If, in addition to ``dataId`` an ``extraId`` is given, it can be considered and Intra/Extra wavefront process.

The command would be used as follows:

.. code-block:: py

  from lsst.ts import salobj

  d = salobj.Domain()

  mtaos = salobj.Remote(d, "MTAOS")

  await mtaos.start_task

  # Process full array intra/extra focal images.
  # Frames can be ComCam or LSSTCam
  await mtaos.cmd_runWEP.set_start(
      dataId=2020103000040, extraId=2020103000041
  )

  # Process corner wavefront sensor
  # Frames should be LSSTCam only. If ComCam command will raise an exception.
  await mtaos.cmd_runWEP.set_start(
      dataId=2020103000040
  )

Pre-processing frames
---------------------

While obtaining Intra/Extra focal images for ComCam or Main Camera full array, one would have the first image ready while the second is exposing.
That means, it would be possible to get ahead and start the pre-processing of those frames (e.g. ISR).

This could be accomplished by adding a ``preProcess`` command to the MTAOS, which would take care of that initial step.
The command would only need to know the ``dataId`` of the frame to process, e.g.;

.. code-block:: py

  await mtaos.cmd_preProcess.set_start(dataId=2020103000040)

With all this considered, an Intra/Extra focus sequence with ``ComCam`` could be done like:

.. code-block:: py

  import asyncio
  from lsst.ts import salobj
  from lsst.ts.observatory.control.maintel import MTCS, ComCam

  domain = salobj.Domain()

  mtcs = MTCS(domain=domain)
  comcam = ComCam(domain=domain)

  asyncio.gather(mtcs.start_task, comcam.start_task)

  # slew/track target
  await mtcs.slew_icrs(ra=14., dec=-10., rot=0.)

  # Select sources for processing in the background
  select_sources_task = asyncio.create_task(
    mtcs.rem.mtaos.cmd_selectSources.set_start(
        ra=14.,
        dec=-10.,
        pa=0.,
        type=CamType.ComCam
      )
    )

  # piston camera hexapod
  await mtcs.rem.hexapod_1.cmd_move.set_start(z=1000.)

  # take first exposure
  data_id_1 = await comcam.take_eng(exptime=30)

  # piston hexapod
  await mtcs.rem.hexapod_1.cmd_move.set_start(z=-1000.)

  # tell MTAOS to start processing first image in the background
  preproc_task_1 = asyncio.create_task(mtcs.rem.mtaos.cmd_preProcess.set_start(dataId=data_id_1))

  # take second exposure
  data_id_2 = await comcam.take_eng(exptime=30)

  # tell MTAOS to start processing second image in the background
  preproc_task_2 = asyncio.create_task(mtcs.rem.mtaos.cmd_preProcess.set_start(dataId=data_id_2))

  # wait for background processes to finish
  await asyncio.gather(select_sources_task, preproc_task_1, preproc_task_2)

  # Process full array intra/extra focal images
  await mtcs.rem.mtaos.cmd_runWEP.set_start(
      intraId=data_id_1, extraId=data_id_2
  )

  # issue corrections to the components
  await mtcs.rem.mtaos.cmd_issueCorrections.start()

In the code above we used the already existing functionality of the `MTCS`_ and `ComCam`_ classes.
Note that the process would be similar if we where to use ``LSSTCam`` instead of ``ComCam``, simply replacing the `ComCam`_ class by the equivalent when that becomes available.
This means, SAL Scripts can be written for ``ComCam`` in a way that will be reusable for the main camera.

.. _MTCS: https://ts-observatory-control.lsst.io/user-guide/maintel/mtcs-user-guide.html
.. _ComCam: https://ts-observatory-control.lsst.io/user-guide/maintel/CCCamera-user-guide.html

.. note::

    It is very likely that there will be a background task automatically running ISR on images coming out of the summit.
    If this is the case, we may not need a command to pre-process the images.
    In any case, it is unlikely such system will be ready early or during commissioning, which makes this command a valuable addition.

.. _user-provided-optical-aberration:

User-provided optical aberration
--------------------------------

In some conditions it may be useful to allow users to specify optical aberrations to be introduced to the system, e.g., add a certain amount of comma to the system.
The MTAOS can easily receive a list of Zernike coefficients, pass them through the Optical Feedback Control and compute the offsets and bending modes required.

The command would simply receive an array of wavefront errors and apply them to the system properly registering the correction internally.

.. code-block:: py

  import numpy as np

  # Assuming the wavefront correction is an array of 19 Zernike coefficients
  wf = np.zeros(19)

  # Assuming ANSI indexes, wf[7] is vertical comma
  # Adding 1mm of vertical comma
  wf[7] = 1.

  await mtaos.cmd_runOFC.set_start(wf = wf)

When the ``MTAOS`` is configured it will load appropriate configuration files for the OFC.
During commissioning it is likely we will want to run the ``OFC`` with a different set of configurations.
In order to allow quick turnaround, the command can accept an additional yaml string with overriding  parameters for the currently loaded configuration.

For instance, one could truncate the sensitivity matrix so that a wavefront error correction is applied to a specific component (or components), e.g.;

.. code-block:: py

  import yaml

  # switch off all corrections except m2 hexapod
  # See https://github.com/lsst-ts/ts_ofc/blob/master/policy/zkAndDofIdxArraySet.yaml
  ofc_config = {
    "dofIdx": {
        "m2Hex": [1, 1, 1, 1, 1],
        "camHex": [0, 0, 0, 0, 0],
        "m1m3Bend": [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
        "m2Bend": [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
      }
  }

  # convert dictionary to yaml string
  config = yaml.safe_dump(ofc_config)

  await mtaos.cmd_runOFC.set_start(wf = wf, config = config)

Similarly, the user can manipulate the configuration for Camera hexapod, M2 and M1M3.
Furthermore, this gives users the capability of customizing the configuration of the OFC on the fly, without the need to recycle the CSC state and the need for several different configurations.
By relying on a yaml string to parse the configuration it is also possible to expand the feature as needed without the need to update the xml interface, which considerably simplify the redeployment cycle.

.. _Architecture-updates:

Architecture
============

In terms of the MTAOS itself, there is practically no substantial architecture changes needed (see :numref:`fig-mtaosClass` bellow).
The current code-base is well organized in the sense that the "heavy lifting" is encapsulated by the `Model class`_.

.. figure:: /_static/mtaosClass.png
   :name: fig-mtaosClass
   :target: ../_images/mtaosClass.png
   :alt: MTAOS class diagram

   MTAOS class diagram (`original uml file from v0.4.5`_).


.. _Model class: https://ts-mtaos.lsst.io/py-api/lsst.ts.MTAOS.Model.html#lsst.ts.MTAOS.Model
.. _original uml file from v0.4.5: https://github.com/lsst-ts/ts_MTAOS/blob/v0.4.5/doc/uml/mtaosClass.uml

Nevertheless, the `Model class`_ itself will require substantial updates to incorporate the new functionality mentioned :ref:`above <Data-access-changes>` and the conversion of `WEP`_ into a gen3 pipeline task.

Updates to the Model class
--------------------------

The `Model class`_ has two main attributes it uses to perform the wavefront estimation (`WEP`_) and to convert the output to optical corrections (`OFC`_).

The ``wep`` attribute (and its uses) is probably the one that will require the most work.
First, we have to keep in mind that the ``wep`` will have two modes of operation; local and remote.

The local mode will be used for the closed loop, e.g., analysis of the corner wavefront sensors for the main camera.
Whereas, remote mode will be used (mostly) for full focal plane analysis (specially for ``LSSTCam``).

Although ComCam data could be analyzed in local mode, it is probably useful to allow users to select if they want ComCam to be processes locally or remotely.
This would allow us to test both use cases early on.

With respect to the ``ofc`` attribute, there is little that needs to be done.
The attribute should remain in the ``Model`` class and should continue to be used as is.
Most of what this class does, once configured with the correct instrument setup, is to receive a list of wavefront errors and convert them to corrections for each of the optical components.
Some updates to the ``Model`` class will have to be implemented to support the :ref:`user-provided-optical-aberration` feature, but should be straightforward to implement.

Fortunately, the use of ``wep`` and ``ofc`` is well abstracted in the ``Model`` class, being restricted to some specific methods that are called by the CSC.

.. _OFC: https://github.com/lsst-ts/ts_ofc
