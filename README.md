# freefire_esp_aimbot.py  |  Python 3.x  |  Android ADB  |  External ESP + aimbot
# Overlay ESP + aimbot for Garena Free Fire (mobile, Android)
# ESP: entity scan via memory read on a rooted device or rootless ADB with accessibility.
# Aimbot: send calibrated tap events through ADB when a target is centered.
#
# No signed drivers, no internal injection. External only.
#
# The offsets below are community-derived and may need tuning per build.
# Verified offsets and process name are the operator's job on their build.

import subprocess
import re
import time
import struct
import ctypes
import math
import threading
import logging
import signal
import sys

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
log = logging.getLogger("ff_esp_aimbot")

# ---------------------------------------------------------------------------
# Build-time constants (operator to verify on target build)
# ---------------------------------------------------------------------------
TARGET_PROCESS = "com.dts.freefirebr"  # may differ per build
SCREEN_W = 720  # change to your device width
SCREEN_H = 1280  # change to your device height

# Memory offsets for Free Fire (community-derived, may need per-build tuning)
# Base process pointer for local player
OFF_LOCAL_PLAYER = 0xD0B4B8  # example placeholder, operator to verify
# Entity list offset
OFF_ENTITY_LIST = 0xD123C0  # example placeholder, operator to verify
# Entity structure offsets (relative to entity base)
OFF_ENTITY_POS_X = 0x7C       # float
OFF_ENTITY_POS_Y = 0x80       # float
OFF_ENTITY_POS_Z = 0x84       # float
OFF_ENTITY_HP = 0x1E0         # float
OFF_ENTITY_NAME = 0x9B0       # pointer
OFF_ENTITY_ID = 0x1C          # int32
# Team offset
OFF_TEAM = 0xC8               # int32

# Aimbot constants
AIMBOT_SMOOTHING = 0.5       # lerp between current crosshair and target (0 to 1)
AIMBOT_MAX_OFFSET_X = 30.0   # max px movement per tick on X
AIMBOT_MAX_OFFSET_Y = 20.0   # max px movement per tick on Y
AIMBOT_FOV_RADIUS = 150.0    # pixels from screen center to activate
AIMBOT_TEAMMATE_IGNORE = True
AIMBOT_MIN_HP_THRESHOLD = 0.01  # ignore entities below this HP ratio

# ---------------------------------------------------------------------------
# ADB helpers
# ---------------------------------------------------------------------------
class ADB(object):
    """Thin ADB wrapper for memory read and input tap."""
    def __init__(self, serial=None):
        self.serial = serial

    def _adb(self, *args):
        parts = ["adb"]
        if self.serial:
            parts += ["-s", self.serial]
        parts += list(args)
        r = subprocess.run(parts, capture_output=True, text=True)
        return r.stdout.strip(), r.returncode

    def read_memory(self, addr, count=4):
        """Read count bytes from the Free Fire process as hex string via ADB memory read."""
        # ADB has no direct memory read on non-rooted devices.
        # On rooted devices, "su -c cat /proc/pid/mem" can be used.
        # This is a stub that calls the shell read helper if available.
        raise NotImplementedError("ADB memory read not available on non-rooted device.")

    def tap(self, x, y):
        """Send a tap event via ADB."""
        self._adb("shell", "input", "tap", str(int(x)), str(int(y)))

    def swipe(self, x1, y1, x2, y2, duration_ms=100):
        """Swipe from (x1,y1) to (x2,y2)."""
        self._adb("shell", "input", "swipe",
                  str(int(x1)), str(int(y1)),
                  str(int(x2)), str(int(y2)),
                  str(duration_ms))


# ---------------------------------------------------------------------------
# Memory scanner stubs
# ---------------------------------------------------------------------------
class MemoryReader(object):
    """
    Memory reading for the Free Fire process.
    On rooted Android: uses /proc/pid/mem through ADB with root shell.
    On non-rooted: not possible without a rootless rootkit or an internal module.
    """
    def __init__(self, adb: ADB, pid):
        self.adb = adb
        self.pid = pid

    def _proc_mem_path(self):
        """Build the command to read from /proc/<pid>/mem via root shell."""
        # This is a conceptual stub: the real call must use a root shell
        # and seek the virtual memory map from /proc/<pid>/maps.
        raise NotImplementedError("Needs /proc/<pid>/maps parsing on the target build.")

    def read_float(self, offset):
        """Read a 4-byte float at offset from the Free Fire process base address."""
        raise NotImplementedError("Needs process base address from maps.")

    def read_int32(self, offset):
        """Read a 4-byte signed int32 at offset."""
        raise NotImplementedError("Needs process base address from maps.")

    def read_bytes(self, offset, length):
        """Read raw bytes at offset."""
        raise NotImplementedError("Needs process base address from maps.")


# ---------------------------------------------------------------------------
# Camera / crosshair state
# ---------------------------------------------------------------------------
class Crosshair(object):
    """Track the current crosshair position on screen."""
    def __init__(self):
        self.cx = SCREEN_W / 2.0
        self.cy = SCREEN_H / 2.0

    def center(self):
        return (self.cx, self.cy)

    def move_to(self, x, y):
        self.cx = max(0, min(SCREEN_W - 1, x))
        self.cy = max(0, min(SCREEN_H - 1, y))


# ---------------------------------------------------------------------------
# ESP rendering (overlay)
# ---------------------------------------------------------------------------
class ESPOverlay(object):
    """Draw entity info as text on the screen through ADB accessibility or root overlay."""
    def __init__(self):
        self.entities = []

    def render(self, adb: ADB):
        """Render ESP text via adb shell dumpsys overlay or an accessibility service.
        This stub shows the approach: writing text to a floating window.
        On rooted devices, a root-level overlay window can be opened via ADB.
        """
        # This is a stub: the actual overlay must be opened by an
        # accessibility service or a root overlay module.
        lines = []
        for e in self.entities:
            lines.append(f"{e.get('name', '?')}  HP:{e.get('hp', 0):.1f}  X:{e.get('x', 0):.0f} Y:{e.get('y', 0):.0f}")
        log.info("ESP entities: %s", lines)
        # To actually paint text on screen, an accessibility service or a
        # root overlay window is needed. ADB shell dumpsys overlay is not a
        # standard Android surface for Free Fire.


# ---------------------------------------------------------------------------
# Aimbot core
# ---------------------------------------------------------------------------
class Aimbot(object):
    def __init__(self, crosshair: Crosshair, adb: ADB, smoothing=AIMBOT_SMOOTHING,
                 max_offset_x=AIMBOT_MAX_OFFSET_X, max_offset_y=AIMBOT_MAX_OFFSET_Y,
                 fov_radius=AIMBOT_FOV_RADIUS, teammate_ignore=AIMBOT_TEAMMATE_IGNORE,
                 min_hp=AIMBOT_MIN_HP_THRESHOLD):
        self.crosshair = crosshair
        self.adb = adb
        self.smoothing = smoothing
        self.max_offset_x = max_offset_x
        self.max_offset_y = max_offset_y
        self.fov_radius = fov_radius
        self.teammate_ignore = teammate_ignore
        self.min_hp = min_hp
        self.current_target = None

    def find_closest(self, entities):
        """Find the closest valid target within FOV, filtered by team and HP."""
        cx, cy = self.crosshair.center()
        best = None
        best_dist = self.fov_radius
        for e in entities:
            team = e.get("team", -1)
            if teammate_ignore and team == 0:  # team 0 is self (verify per build)
                continue
            hp = e.get("hp", 0)
            if hp < min_hp:
                continue
            ex = e.get("x", 0)
            ey = e.get("y", 0)
            dist = math.hypot(ex - cx, ey - cy)
            if dist < best_dist:
                best = e
                best_dist = dist
        return best

    def aim_at(self, target):
        """Send a swipe to move the crosshair toward the target's screen position."""
        if not target:
            return
        target_x = target.get("x", 0)
        target_y = target.get("y", 0)
        cx, cy = self.crosshair.center()
        dx = (target_x - cx) * self.smoothing
        dy = (target_y - cy) * self.smoothing
        dx = max(-self.max_offset_x, min(self.max_offset_x, dx))
        dy = max(-self.max_offset_y, min(self.max_offset_y, dy))
        new_x = cx + dx
        new_y = cy + dy
        self.crosshair.move_to(new_x, new_y)
        self.adb.swipe(cx, cy, new_x, new_y, duration_ms=50)

    def tick(self, entities):
        target = self.find_closest(entities)
        self.current_target = target
        if target:
            self.aim_at(target)
        return target


# ---------------------------------------------------------------------------
# Main loop
# ---------------------------------------------------------------------------
def main():
    adb = ADB()
    crosshair = Crosshair()
    aimbot = Aimbot(crosshair, adb)
    esp = ESPOverlay()

    # The memory reader would be initialized here if a root device is available.
    # Example: reader = MemoryReader(adb, pid)
    # Since this is a stub, we simulate entity data for structure.

    running = True
    def _stop(*_):
        nonlocal running
        running = False
    signal.signal(signal.SIGINT, _stop)

    log.info("Starting ESP/aimbot loop. Press Ctrl+C to stop.")
    tick_interval = 0.05  # 50 ms loop, about 20 Hz

    while running:
        # In a real build, read entities from the process memory.
        # Example: entities = reader.read_entities()
        # For now, this loop is a skeleton for the read/aim/render cadence.
        time.sleep(tick_interval)

        # Placeholder: in a real run, fetch entities and feed them in.
        # aimbot.tick(entities)
        # esp.entities = entities
        # esp.render(adb)

    log.info("Loop stopped.")

if __name__ == "__main__":
    main()
