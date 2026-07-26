# DISPatch

DISPatch is a small Qt5/C++ DIS6 Simulation Management controller. It sends
state-transition commands over UDP and displays received component responses.
The UI defaults to a dark theme and includes dark, light, Gruvbox, One Dark,
VS Code default, Tokyo Night, and Dracula themes.

## Build Dependencies

On RHEL 8 or newer, install the Qt5 development package and CMake toolchain:

```bash
sudo dnf install cmake gcc-c++ qt5-qtbase-devel
```

Then build:

```bash
cmake -S . -B build
cmake --build build
```

Install with the default CMake prefix:

```bash
sudo cmake --install build
```

This installs `dispatch` to `/usr/local/bin` and `dispatch.json` to
`/usr/local/etc`.

Tests are optional and use Catch2:

```bash
cmake -S . -B build-tests -DDISPATCH_WITH_TESTS=ON
cmake --build build-tests
ctest --test-dir build-tests
```

CLion can import the included `CMakePresets.json`. Select the `tests` preset
to configure with `DISPATCH_WITH_TESTS=ON`; CLion will discover the CTest
tests, and the `check` build preset/target builds and runs them with failure
output enabled.

The project also includes a Conan 2 recipe. The `tests` option controls the
same CMake variable:

```bash
conan build . -o tests=True
```

## Release Tarball

Build a RHEL 8 compatible release tarball with Docker or Podman:

```bash
docker build -f Dockerfile.release --output type=local,dest=dist .
```

The Dockerfile defaults to Rocky Linux 8.10 because the public UBI 8 repos do
not include all Qt build dependencies without extra Red Hat entitlements. To
build from a subscribed RHEL-derived image instead, override `BASE_IMAGE`.

The output is `dist/DISPatch-<version>-rhel8-<arch>.tar.gz` plus a SHA-256
checksum. The tarball contains the `dispatch` executable, the default config
file, Conan-deployed runtime libraries, Qt plugins, and a `run-dispatch`
launcher that sets `LD_LIBRARY_PATH` and `QT_PLUGIN_PATH`.

## DIS6 Command Mapping

The application uses standard DIS6 Simulation Management PDU layouts:

- `Initialize`: Action Request PDU, default action ID `39` for initialize internal parameters
- `Start`: Start/Resume PDU
- `Pause`: Stop/Freeze PDU, reason `recess`
- `Stop`: Stop/Freeze PDU, reason `termination`
- `Reset`: Stop/Freeze PDU, reason `stop_for_reset`

Responses received on the configured listen address and port are decoded enough
to show the sender, PDU type, request ID, and a summary. Acknowledge and Action
Response PDUs are matched back to the request ID sent by the manager.

Set command defaults in `dispatch.json` so they match the simulation
component interface control document. The Start command supports
`realWorldTimeOffsetSeconds` and `simulationTimeOffsetSeconds`, which default
to `0` and schedule the Start/Resume PDU clock-time fields relative to the
current UTC time. Set `useLiteralZero` to `true` in the `start` block when a
zero Start/Resume offset should be written as a literal zero clock-time value
instead of the current UTC time.

## Configuration

At startup, DISPatch looks for `dispatch.json` in your home directory, then the
configured system config directory such as `/usr/local/etc`, then the current
working directory, and then next to the executable. You can pass an explicit
path with `--config path/to/dispatch.json`.

The config file supplies startup defaults for theme, network addresses and
ports, DIS entity IDs, command settings, and frozen behavior. The theme can be
`dark`, `light`, `gruvbox`, `onedark`, `vscode`, `tokyonight`, or `dracula`.

The network section also controls UDP socket behavior. `shareAddress` and
`reuseAddress` allow multiple processes to bind the same UDP port on one
machine when the platform supports it. `interfaceName` can pin socket binding
and multicast sends/joins to a specific network interface; when it is blank,
DISPatch selects a usable IPv4 interface and shows that selection in the UI.
`multicastInterfaceName` is still accepted as a legacy alias for
`interfaceName`. `joinMulticast` makes the receive socket join the configured
`multicastGroupAddress`; when that field is blank, DISPatch uses
`destinationAddress` if it is multicast. `multicastLoopback` controls whether
multicast sent by this host is looped back to local sockets. It defaults to
`false`; enable it for same-machine multicast testing, including multiple local
federates or the built-in test federate on one multicast DIS port.

The Network section also has Broadcast and Localhost destination modes. They
are mutually exclusive shortcuts that set the destination to the selected
interface's IPv4 broadcast address or `127.0.0.1`, select an appropriate
interface, and adjust UDP bind flags for the selected mode. In Broadcast mode,
changing the interface immediately updates the displayed destination address.

The optional `heartbeat` block enables liveness tracking:

```json
"heartbeat": {
  "enabled": true,
  "timeout": 5
}
```

`timeout` is measured in seconds and must be at least `1`. When heartbeat
tracking is enabled, each received Entity State PDU updates the status of its
entity ID. A Comment PDU does the same when its receiving entity matches the
manager ID. Live entities appear with a heart that pulses on each update and
changes to a skull when no update arrives before the timeout. Entity State and
Comment PDUs are ignored when heartbeat tracking is disabled. Entity State PDUs
do not contain a receiving entity, so receipt on the configured DIS network is
treated as being addressed to this manager.

The message table traces commands sent by this manager and matched federate
responses for requests sent in the current session. Unexpected PDUs, unmatched
responses, and Entity State or Comment PDUs used as heartbeats are intentionally
omitted from both the table and the message log because their state is shown in
the DIS Identity heartbeat display.

The optional `log` section can mirror the UI logs to files. `logLevel` can be
`debug`, `warn`, or `error`; event log entries below that level are hidden from
the UI log and event log file. Warnings and errors are highlighted in the UI.
Set `logs` to true to append the filtered event log to `logFile`, and set
`messageLogs` to true to append the filtered PDU trace to `messageLogFile`.
Relative file paths are resolved next to the loaded config file.

Config validation warnings are written to the application log at startup.
DISPatch reports unknown JSON keys, invalid address strings, invalid multicast
groups, unknown multicast interfaces, and suspicious network combinations such
as joining multicast while binding to a specific listen address or sharing one
UDP port without address reuse enabled.

## Local Test Federate

Set `testFederate.enabled` in `dispatch.json` to run in-process UDP responders
for local testing. When enabled, the UI shows a Test Federates status line with
the bind state and editable site/application/entity controls for the first
configured federate. The entity IDs come from `testFederate.entityIds`. The
responders listen on the configured destination address and port, accept DIS6
Simulation Management state-transition requests addressed to one of those IDs
or to the broadcast entity ID, and send accepted responses back to the manager:

- `Initialize`: Action Response PDU
- `Start`, `Pause`, `Stop`, and `Reset`: Acknowledge PDU

When both test federates and heartbeat tracking are enabled, each configured
federate periodically sends an empty Comment PDU addressed to the manager.
Select `Dead (stop heartbeat)` in the Test Federate box to stop those messages
and exercise the configured timeout; clear it to resume heartbeats.

When localhost mode puts the manager and test federate on the same unicast
address and port, DISPatch routes test-federate requests through the manager
socket. This avoids platform `SO_REUSEPORT` behavior that can otherwise deliver
a localhost datagram to only one of the two sockets.
