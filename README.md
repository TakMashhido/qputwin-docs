# QPUTwin documentation

Source of the QPUTwin documentation site, published at **<https://koustavmondal.com/qputwin-docs/>**.

QPUTwin provides calibration-faithful digital twins of trapped-ion and superconducting quantum processors behind one OpenQASM 2.0/3.x API.

## Preview locally

```bash
pip install -r requirements.txt
mkdocs serve            # http://127.0.0.1:8088
mkdocs build --strict   # static site in ./site
```

Every push to `main` rebuilds the site and publishes it with GitHub Pages (`.github/workflows/deploy.yml`).

Koustav Kumar Mondal · koustav@koustavmondal.com · Apache-2.0
