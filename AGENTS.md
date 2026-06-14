# Repository Instructions

## Project Snapshot

LaserVision / `laser-level-webcam` is a Python desktop application for real-time laser-level measurement using a webcam sensor. The main GUI reads webcam frames through PySide6 multimedia APIs, converts each frame to a 1D intensity profile, fits a Gaussian center, and converts the pixel delta from zero into physical units. The repo also contains an optional LinuxCNC remote driver GUI that connects to the main app's socket server, runs probing jobs, and displays measured surfaces with Plotly.

Start future sessions by reading this file and `README.md`.

## Runtime And Packaging

- Recommended local Python is 3.11. The README notes Python 3.12 is not compatible with the pinned PySide6 version. Packaging metadata says `>=3.7`, and `tox.ini` lists py37-py311, but the current dependency pins are the practical source of truth.
- Runtime dependencies are pinned in `requirements.txt`; test/dev tools are in `requirements-dev.txt`.
- Install for local development:

```powershell
py -3.11 -m venv venv
.\venv\Scripts\activate
python -m pip install -r requirements.txt -r requirements-dev.txt
```

- Run the main measurement GUI from source with:

```powershell
python .\laser-level-webcam.py
```

- Run the LinuxCNC remote driver GUI with:

```powershell
python .\linuxcnc_remote.py
```

- After package installation, the console script is:

```powershell
laser-level-webcam
```

## Code Structure

Think of the code in four main areas:

- Main measurement GUI: start at `laser-level-webcam.py` and `src/main.py`. This is the user's primary app surface: camera selection, sensor feed, analyser, sampling controls, graph/table display, CSV export, cyclic measurement, socket server menu, and persisted window/settings state.
- Measurement pipeline: start at `src/Core.py`, then `src/Workers.py`, `src/curves.py`, and `src/utils.py`. This area owns camera setup, Qt worker threads, frame-to-profile conversion, smoothing, Gaussian center fitting, subsample/outlier aggregation, zeroing, unit conversion, and linear-regression-derived shim/scrape values.
- Qt display widgets and dialogs: start at `src/Widgets.py`, `src/cycle.py`, and `src/s_server.py`. Use these when changing the camera pixmap view, analyser visualization, measurement graph, formatted table cells, cyclic measurement dialog, or the main app socket protocol (`ZERO\n` and `TAKE_SAMPLE\n`).
- LinuxCNC remote workflow: start at `linuxcnc_remote.py`, `src/linuxcnc_remote_driver.py`, and `src/CNC_jobs/probe.py`. This is the optional probing/control GUI that connects to the main app's socket server, runs a probing grid, sends machine commands when LinuxCNC is available, and renders the measured surface with bundled Plotly. Treat files under `src/CNC_jobs/` other than `probe.py` as experimental unless the task points to them.

Supporting areas:

- Tests live in `tests/`; they cover utility/curve behavior and basic Qt widget/window smoke tests.
- Packaging and tooling live in `setup.cfg`, `pyproject.toml`, `requirements*.txt`, `tox.ini`, and `.pre-commit-config.yaml`.
- `images/` contains README screenshots.

## Testing And QA

- Run tests with:

```powershell
python -m pytest tests
```

- The test suite includes pure utility/curve tests and pytest-qt smoke tests for widgets and the main window. Qt tests may need a working graphical environment and installed PySide6 dependencies.
- `tox.ini` is configured for py37, py38, py39, py310, and py311, with `pytest {posargs:tests}`. `tox` itself is not listed in `requirements-dev.txt`.
- Code quality tooling is configured through `.pre-commit-config.yaml`: trailing whitespace, EOF fixer, YAML check, large-file check, Black line length 120, strict mypy with ignored missing imports, flake8 line length 120, reorder-python-imports, and setup-cfg-fmt.
- Helper scripts in `testing/` shell out via `os.system()`. Check their working-directory assumptions before relying on them; direct `python -m pytest tests` and `pre-commit run --all-files` are clearer.

## Development Notes

- Prefer preserving the existing PySide6 signal/slot style and Qt worker-thread separation. Camera frame processing should stay off the main GUI path.
- Keep sample values internally in millimeters unless the surrounding code explicitly formats for display. Unit conversion for display goes through `get_units()` and `units_of_measurements`.
- Be careful around `QSettings` keys in `src/main.py` and `src/linuxcnc_remote_driver.py`; changing them affects persisted user layout and connection settings.
- Some files currently contain mojibake for the micro symbol in strings and README text. Do not do broad encoding churn unless the task is specifically about text cleanup.
- The LinuxCNC modules import `linuxcnc` only when `sys.platform == "linux"` and otherwise print simulated commands. Treat machine-motion changes as high risk.
- The main GUI's extra camera controls rely on `ffmpeg` being available on PATH.
- If changing UI or camera processing, consider both the live sensor feed and analyser display: `FrameWorker.OnPixmapChanged`, `FrameWorker.OnAnalyserUpdate`, and `FrameWorker.OnCentreChanged` drive different parts of the app.
