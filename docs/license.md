---
title: License and data sources
description: The Apache-2.0 license terms of QPUTwin and the sources of its packaged calibration data.
---

# License & data sources

## License

QPUTwin is released under the **Apache License 2.0**. You may use, modify and redistribute it, including commercially, provided you keep the copyright and license notices (`NOTICE`) and state any changes you made. The license also grants a patent licence from contributors.

Copyright 2026 Koustav Kumar Mondal · erkoustavkumar@gmail.com

## Data sources

The packaged calibration values cite their sources in `calibration.provenance`. They include:

* Quantinuum (Helios launch) and the Sandia/Quantinuum Helios benchmark;
* IonQ (Forte system page);
* Microsoft Azure Quantum (IonQ provider, Aria timings);
* Rigetti (Ankaa-3 launch);
* IQM and Amazon Braket (Emerald launch);
* arXiv:2608.30176 (Emerald calibration snapshot).

When you report results, also state the calibration snapshot and the `assumed_parameters` your results depend on; see [Reproducibility](research/reproducibility.md).
