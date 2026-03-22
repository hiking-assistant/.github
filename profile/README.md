# Hiking Assistant

Hiking Assistant is a multi-repository embedded systems project built around a **LilyGo T-Watch 2020 V3**, a **Raspberry Pi receiver**, and a **web UI** for viewing synchronized hiking sessions.

The project is split into separate repositories so that firmware, backend synchronization, frontend development, and shared organization-level assets can evolve independently while still working together as one end-to-end system.

A core design choice across the system is **explicit state transition modeling**, especially on the watch side. Instead of relying on loosely coupled flags and implicit control flow, the watch firmware models user-visible and recovery-critical behavior as named states with defined transitions. This makes reboot recovery, time synchronization, session lifecycle handling, and Bluetooth synchronization easier to reason about and maintain.

## Repositories

### [`watch`](https://github.com/hiking-assistant/watch)
Firmware for the **LilyGo T-Watch 2020 V3**.

Responsibilities:

- run the watch UI
- count steps using the onboard accelerometer step counter
- start, checkpoint, resume, and end hiking sessions
- persist sessions locally in LittleFS
- recover unfinished sessions after reboot
- synchronize finished sessions to the Raspberry Pi over **Classic Bluetooth RFCOMM**
- receive wall-clock time from the Raspberry Pi after boot
- model session and recovery behavior through explicit state transitions

### [`receiver`](https://github.com/hiking-assistant/receiver)
Backend/service running on the **Raspberry Pi**.

Responsibilities:

- connect to the watch over Classic Bluetooth
- perform the sync handshake and time synchronization
- receive completed sessions from the watch
- acknowledge each stored session explicitly
- persist synchronized session data into the server-side database

### [`web-ui`](https://github.com/hiking-assistant/web-ui)
Frontend and web backend for browsing synchronized hiking sessions.

Responsibilities:

- store and serve synchronized hiking session data
- expose the Flask application and database migrations
- display session metrics such as steps, distance, duration, and derived values
- provide a dashboard for browsing hiking history
- support a fully offline deployment model on the Raspberry Pi

The web UI is intended to be usable **completely offline**. The target deployment model is that users connect directly to the Raspberry Pi through its local hotspot / access point and access the interface without any Internet connection. The hotspot-based access workflow is planned but not yet implemented.

### [`.github`](https://github.com/hiking-assistant/.github)
Shared organization-level assets and operational setup.

This repository should contain:

- organization-wide documentation
- Raspberry Pi service definitions such as:
  - `startup_receiver.service`
  - `startup_webserver.service`

## System architecture

```text
LilyGo T-Watch 2020 V3
        |
        |  Classic Bluetooth RFCOMM
        v
Raspberry Pi Receiver
        |
        |  local Flask app / database
        v
      Web UI
```

## High-level workflow

1. The watch records a hiking session locally.
2. Active sessions are checkpointed so they can survive reboot.
3. If the watch reboots, it restores the unfinished session.
4. After boot, the Raspberry Pi sends fresh wall-clock time to the watch.
5. The watch synchronizes finished sessions one by one over Classic Bluetooth.
6. The Raspberry Pi acknowledges each session only after it has been processed and stored.
7. The web UI serves the synchronized session data from the Raspberry Pi and displays it in the browser.

## Design principles

* **clear separation of concerns**
  Firmware, receiver logic, web application logic, and shared operational assets are kept in separate repositories.

* **explicit state transition modeling**
  Runtime behavior, especially on the watch, is modeled with explicit named states and transitions for flows such as idle, session start, active tracking, reboot recovery, pending time sync, session ending, and post-save confirmation.

* **reliable synchronization**
  Sessions are transferred one at a time and deleted from the watch only after explicit acknowledgment.

* **reboot recovery**
  Unfinished sessions survive watch reboot through local persistence and restore logic.

* **time correctness**
  The watch does not assume its clock remains valid across reboot; time is resynchronized by the Raspberry Pi.

* **offline-first deployment**
  The Raspberry Pi and web UI are intended to work locally without Internet access, with hotspot / access-point-based access planned as the main user-facing deployment model.

* **incremental refactoring**
  The watch firmware is organized into focused modules such as state machine, storage, Bluetooth, session logic, UI, and time handling.

## Raspberry Pi setup

The Raspberry Pi side requires **both** of these repositories to be present on the device:

- `receiver`
- `web-ui`

Both repositories must be cloned locally onto the Raspberry Pi, since the receiver process and the Flask web application run there on the same device.

A typical layout could look like this:

```text
/home/user/hiking-assistant/
├── receiver/
├── web-ui/
└── .venv/
```

### 1. Install system packages

```bash
sudo apt update
sudo apt install python3-virtualenv python3-pip python3-bluez bluez
```

### 2. Create and activate a virtual environment

```bash
virtualenv --system-site-packages -p python3 .venv
source .venv/bin/activate
```

### 3. Verify Bluetooth Python support

```bash
python3 -c "import bluetooth; print(bluetooth.__file__)"
```

### 4. Install Python dependencies

```bash
python3 -m pip install -r web-ui/requirements.txt
```

## First-time database setup

If the migrations directory is not already present in the `web-ui` repository, initialize it with:

```bash
cd web-ui
flask db init
flask db migrate -m "initial"
flask db upgrade
```

If migrations are already tracked in the repository, running the upgrade is enough:

```bash
cd web-ui
flask db upgrade
```

## Normal development startup

After the environment and database are already set up, start the two Raspberry Pi components in separate terminal windows.

### Terminal 1: web UI

```bash
cd web-ui
source ../.venv/bin/activate
flask run
```

### Terminal 2: receiver

```bash
cd receiver
source ../.venv/bin/activate
python3 receiver.py
```

## Using the systemd service files

The shared `.github` repository should contain the Raspberry Pi service definitions:

- `startup_receiver.service`
- `startup_webserver.service`

These are intended to run the receiver and the web server automatically when the Raspberry Pi boots.

Before using them, make sure that **both the `receiver` and `web-ui` repositories have already been downloaded to the Raspberry Pi**, and that the paths inside the service files match their actual locations.

### Install the service files

Run the following commands from the directory containing the service files, or replace the filenames with their full paths.

Copy the service files into the systemd unit directory:

```bash
sudo cp startup_receiver.service /lib/systemd/system/
sudo cp startup_webserver.service /lib/systemd/system/
```

Then reload systemd so it detects the new units:

```bash
sudo systemctl daemon-reload
```

### Enable the services at boot

```bash
sudo systemctl enable startup_receiver.service
sudo systemctl enable startup_webserver.service
```

### Start the services immediately

```bash
sudo systemctl start startup_receiver.service
sudo systemctl start startup_webserver.service
```

### Check service status

```bash
sudo systemctl status startup_receiver.service
sudo systemctl status startup_webserver.service
```

### View service logs

For the receiver:

```bash
journalctl -u startup_receiver.service -f
```

For the web server:

```bash
journalctl -u startup_webserver.service -f
```

### Stop or restart the services

Stop:

```bash
sudo systemctl stop startup_receiver.service
sudo systemctl stop startup_webserver.service
```

Restart:

```bash
sudo systemctl restart startup_receiver.service
sudo systemctl restart startup_webserver.service
```

### Important notes

* Both `receiver` and `web-ui` must already exist on the Raspberry Pi.
* The paths inside the service files must match the actual locations of the repositories and virtual environment.
* If the repositories are moved, the `WorkingDirectory`, `ExecStart`, or virtual environment paths in the service files must be updated.
* The service files assume the Python environment and project dependencies have already been installed.
* During development, it is often easier to run the receiver and web UI manually first before enabling the services.

## Repository scope

Per-repository details such as:

* build instructions for the watch firmware
* flashing and upload steps
* Bluetooth protocol implementation details
* Flask routes and database models
* frontend structure
* watch state machine details

should remain in each repository’s own README.

The organization README is intended to explain how the repositories fit together and how to set up the full system at a high level.

## Suggested reading order

If you are new to the project:

1. read the watch repository to understand session creation, persistence, recovery, and state transitions
2. read the receiver repository to understand synchronization and Bluetooth handling
3. read the web UI repository to understand storage and visualization
4. read the `.github` repository for shared deployment and operational instructions

## Technologies used

Across the repositories, the project uses:

* **C++ / Arduino framework**
* **PlatformIO**
* **LilyGo T-Watch 2020 V3**
* **Classic Bluetooth RFCOMM**
* **Python**
* **Flask**
* **SQLite / database migrations**
* **Raspberry Pi**

## Goals

The project aims to provide:

* a watch-based hiking tracker with local resilience
* a Raspberry Pi hub for synchronization and persistence
* a browser-based interface for viewing synchronized hiking sessions
* a fully local, offline-capable Raspberry Pi deployment model
* a clean multi-repository structure separating firmware, backend, frontend, and operational assets
