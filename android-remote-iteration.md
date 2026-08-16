# How can I iterate faster on an Android APK built on a remote server, instead of manually downloading and installing it each time?

## Answer

The manual download-and-install loop is the bottleneck. Two things to optimize separately: (1) getting the built APK onto the device without a manual copy, and (2) making each build itself faster. The biggest single win is bridging `adb` from your local machine (where the device is plugged in) to the remote server (where Gradle runs), so `./gradlew installDebug` on the remote installs straight onto the device over an SSH tunnel.

### Recommended workflow: SSH reverse tunnel for adb

ADB uses a client/server model on TCP port 5037. The adb *server* owns the device connection; adb *clients* (and Gradle's install task) talk to that server. If you run the adb server on your local machine (where the device is attached) and forward the remote's port 5037 back to it, every `adb`/Gradle command on the remote controls your local device.

1. On the **local** machine, start the adb server and confirm the device is visible:
   ```
   adb start-server
   adb devices
   ```
2. Connect to the remote with a reverse tunnel that exposes local 5037 as remote 5037:
   ```
   ssh -R 5037:localhost:5037 user@remote-host
   ```
3. On the **remote**, verify it sees the device through the tunnel:
   ```
   adb devices
   ```
4. Now build and install in one step from the remote:
   ```
   ./gradlew installDebug
   ```
   Gradle builds the APK and runs `adb install -r` against the tunneled device. No manual download, no manual install. Relaunch the app with `adb shell am start -n your.package/.MainActivity` if Gradle does not auto-launch it.

Tip: add `RemoteForward 5037 localhost:5037` to the relevant `Host` block in `~/.ssh/config` so the tunnel is always up when you SSH in.

### Alternative: wireless adb (adb over TCP)

If the device and the remote server are on the same network (or you can route between them), put the device in TCP mode once:

```
adb tcpip 5555           # while device is USB-attached
adb connect <device-ip>:5555   # run from the remote, if reachable
```

Then `./gradlew installDebug` from the remote hits the device directly over WiFi. This avoids the SSH tunnel but requires the remote to reach the device's IP, which usually only works when they share a network.

### Alternative: automate the copy if you must keep adb local-only

If tunneling is not allowed, script the existing loop so it is one command instead of manual:

- On the remote, after building, `scp`/`rsync` the APK to a known local path and trigger `adb install -r` locally (e.g., via an SSH `LocalForward` plus a small script, or by pushing to a shared volume).
- Wrap it in a single shell script (`build-and-deploy.sh`) so each iteration is one invocation.

This still copies the file, but removes the manual steps.

### Making each build itself faster

Deployment is only half the loop. Speed up the build on the remote:

- Keep the Gradle daemon alive: it persists across builds by default; do not pass `--no-daemon`.
- Enable the build and configuration caches in `gradle.properties`:
  ```
  org.gradle.caching=true
  org.gradle.configuration-cache=true
  org.gradle.parallel=true
  ```
- Install only what changed. `installDebug` already reinstalls just the debug variant. For very large APKs on Android 11+, add incremental install: `adb install --incremental app-debug.apk` (streams the APK so the app launches before the transfer finishes).
- If you use APK splits or App Bundles, deploy a single ABI split matching the device instead of a universal APK to shrink transfer size.
- Avoid clean builds. Let Gradle's incremental compilation do its job; only `./gradlew clean` when something genuinely gets stuck.

### In short

Bridge adb to the remote over an SSH reverse tunnel (`ssh -R 5037:localhost:5037`) and then run `./gradlew installDebug` on the remote. That removes the manual download/install entirely. Layer Gradle daemon + caches on top to shrink the build step itself.

## Follow-up: skip the desktop entirely, go remote-to-phone directly

Yes. You can run adb on the remote server only and talk straight to the phone, with no desktop and no USB cable in the steady state. Two pieces are needed: a routable path from the server to the phone (the phone is normally behind NAT), and Android's wireless debugging for the actual adb link.

### The network problem, and the fix

A cloud server cannot normally reach a phone on home WiFi or mobile data, because the phone sits behind NAT. Solve this once by joining both devices to a mesh VPN such as Tailscale:

1. Install the Tailscale Android app on the phone and sign in.
2. Install Tailscale on the remote server (`curl -fsSL https://tailscale.com/install.sh | sh`) and sign in to the same tailnet.
3. Both now have stable, mutually routable IPs (the `100.x.y.z` range). The server can open TCP connections to the phone directly.

WireGuard or any other VPN that gives the phone a reachable IP works too. Tailscale is just the lowest-effort option on Android. If the server and phone already share a LAN (e.g., a server on your home network), skip the VPN entirely.

### Enable wireless debugging on the phone (Android 11+)

This is the cable-free path and needs no adb anywhere else:

1. Settings → System → Developer options → enable **Wireless debugging**.
2. Tap it → **Pair device with pairing code**. Note the IP:port (e.g., `100.x.y.z:43125`) and the 6-digit code.
3. On the remote, pair once:
   ```
   adb pair 100.x.y.z:43125      # enter the pairing code when prompted
   adb connect 100.x.y.z:43125   # connect port may differ from pair port
   adb devices                    # confirm the phone shows up
   ```
   The pair port and connect port are shown separately on the wireless debugging screen; use the "IP & port" value for `adb connect`.

Then build and install from the remote:
```
./gradlew installDebug
```
Gradle's install task uses the adb server on the remote, which is now paired to the phone over Tailscale. No desktop, no cable, no manual download.

Note: the pairing is persistent. You re-run `adb connect <ip>:<port>` if the connection drops, but you do not re-pair unless you revoke debugging authorization.

### Older Android (< 11)

Wireless debugging without a cable is an Android 11+ feature. On older versions you must run `adb tcpip 5555` once over USB, which requires adb on some machine for that one-time step. After that, `adb connect <phone-ip>:5555` from the remote works the same way. For a cable-free, old-Android setup your only real option is to upgrade or borrow a machine for the one-time `tcpip` command.

### Helpful extras on the remote

- Install adb: `sudo apt install adb` or download platform-tools and add to PATH.
- Install to a specific device when several are visible: `adb -s <serial-or-ip> install -r app-debug.apk`, or `./gradlew installDebug -PandroidBuilds>` if filtered.
- View the phone screen from the remote with `scrcpy` (`scrcpy -s <ip:port>`). It forwards video over the same adb connection, though you need an X display on the remote (or X forwarding to your desktop just for viewing).
- Keep the loop tight: `./gradlew installDebug && adb shell am start -n your.package/.MainActivity` builds, installs, and relaunches in one command.

### In short

Put the phone and server on Tailscale, enable Android 11+ wireless debugging, `adb pair` + `adb connect` from the remote, then `./gradlew installDebug`. The desktop drops out of the loop entirely.
