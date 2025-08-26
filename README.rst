.. image:: https://img.shields.io/badge/dmtn--219-lsst.io-brightgreen.svg
   :target: https://dmtn-219.lsst.io/
.. image:: https://github.com/lsst-dm/dmtn-219/workflows/CI/badge.svg
   :target: https://github.com/lsst-dm/dmtn-219/actions/

############################################
Proposal and Prototype for Prompt Processing
############################################

DMTN-219
========

The Prompt Processing framework has long been a gray area in the overall Rubin Observatory architecture.  This document attempts to fill that hole by proposing a concrete design with an accompanying prototype implementation that should meet minimum requirements.  Additional features and optimizations that could be incorporated later are also described.

**Links:**

- Publication URL: https://dmtn-219.lsst.io/
- Alternative editions: https://dmtn-219.lsst.io/v
- GitHub repository: https://github.com/lsst-dm/dmtn-219
- Build system: https://github.com/lsst-dm/dmtn-219/actions/

Build this technical note
=========================

You can clone this repository and build the technote locally if your system has Python 3.11 or later:

.. code-block:: bash

   git clone https://github.com/lsst-dm/dmtn-219
   cd dmtn-219
   make init
   make html

Repeat the ``make html`` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run ``make clean``.

The built technote is located at ``_build/html/index.html``.

Publishing changes to the web
=============================

This technote is published to https://dmtn-219.lsst.io/ whenever you push changes to the ``main`` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://dmtn-219.lsst.io/v.

Editing this technical note
===========================

The main content of this technote is in ``index.rst`` (a reStructuredText file).
Metadata and configuration is in the ``technote.toml`` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.
