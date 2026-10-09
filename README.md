# PixelOS-Volume-Panel-Toggle
PixelOS Volume Panel Left/Right Toggle - Ports volume panel position toggle from LineageOS to PixelOS
```bash
cd ~/pos

cat > volume_panel_patcher.py <<'PY'
#!/usr/bin/env python3
import os, sys, re

def patch_file(path, old, new):
    if not os.path.exists(path):
        print(f"[ERROR] Missing: {path}")
        return False
    with open(path, "r", encoding="utf-8") as f:
        s = f.read()
    if old not in s:
        print(f"[WARN] Pattern not found in {path}")
        return False
    s = s.replace(old, new)
    with open(path, "w", encoding="utf-8") as f:
        f.write(s)
    print(f"[OK] Patched: {path}")
    return True

def add_string_if_missing(path, needle, text):
    if not os.path.exists(path):
        print(f"[ERROR] Missing: {path}")
        return False
    with open(path, "r", encoding="utf-8") as f:
        s = f.read()
    if needle in s:
        print(f"[SKIP] Already present: {path}")
        return True
    s = s.replace("</resources>", text + "\n</resources>", 1)
    with open(path, "w", encoding="utf-8") as f:
        f.write(s)
    print(f"[OK] Updated: {path}")
    return True

def add_xml_pref(path, needle, xml_pref):
    if not os.path.exists(path):
        print(f"[ERROR] Missing: {path}")
        return False
    with open(path, "r", encoding="utf-8") as f:
        s = f.read()
    if needle in s:
        print(f"[SKIP] Already present: {path}")
        return True
    s = s.replace("</PreferenceScreen>", xml_pref + "\n</PreferenceScreen>", 1)
    with open(path, "w", encoding="utf-8") as f:
        f.write(s)
    print(f"[OK] Updated: {path}")
    return True

# 1) Patch VolumeDialog.kt
vp = "frameworks/base/packages/SystemUI/src/com/android/systemui/volume/dialog/VolumeDialog.kt"
old1 = '''val configuredGravity =
    Gravity.getAbsoluteGravity(
        context.resources.getInteger(R.integer.volume_dialog_gravity),
        context.resources.configuration.layoutDirection,
    )
val volumePanelOnLeft =
    (configuredGravity and Gravity.HORIZONTAL_GRAVITY_MASK) == Gravity.LEFT
'''
new1 = '''val volumePanelOnLeft = Settings.System.getInt(
    context.contentResolver,
    "volume_panel_on_left",
    1
) == 1
'''
if patch_file(vp, old1, new1):
    with open(vp, "r", encoding="utf-8") as f:
        s = f.read()
    if "import android.provider.Settings" not in s:
        s = s.replace("import android.view.Gravity\n", "import android.provider.Settings\nimport android.view.Gravity\n", 1)
        with open(vp, "w", encoding="utf-8") as f:
            f.write(s)
        print("[OK] Added Settings import to VolumeDialog.kt")

# 2) Create Settings controller
os.makedirs("packages/apps/Settings/src/com/android/settings/sound", exist_ok=True)
controller = """package com.android.settings.sound;

import android.content.Context;
import android.provider.Settings;

import androidx.preference.PreferenceScreen;
import androidx.preference.SwitchPreferenceCompat;

import com.android.settings.core.PreferenceControllerMixin;
import com.android.settingslib.core.AbstractPreferenceController;
import com.android.settingslib.core.lifecycle.Lifecycle;
import com.android.settingslib.core.lifecycle.LifecycleObserver;

public class VolumePanelPositionPreferenceController extends AbstractPreferenceController
        implements PreferenceControllerMixin, LifecycleObserver {

    private static final String KEY_VOLUME_PANEL_ON_LEFT = \"volume_panel_on_left\";
    private SwitchPreferenceCompat mPreference;

    public VolumePanelPositionPreferenceController(Context context, Lifecycle lifecycle) {
        super(context);
        if (lifecycle != null) {
            lifecycle.addObserver(this);
        }
    }

    @Override
    public String getPreferenceKey() {
        return KEY_VOLUME_PANEL_ON_LEFT;
    }

    @Override
    public boolean isAvailable() {
        return true;
    }

    @Override
    public void displayPreference(PreferenceScreen screen) {
        super.displayPreference(screen);
        mPreference = screen.findPreference(KEY_VOLUME_PANEL_ON_LEFT);
        if (mPreference != null) {
            updateState();
            mPreference.setOnPreferenceChangeListener((preference, newValue) -> {
                Settings.System.putInt(
                        mContext.getContentResolver(),
                        \"volume_panel_on_left\",
                        (Boolean) newValue ? 1 : 0
                );
                return true;
            });
        }
    }

    private void updateState() {
        if (mPreference != null) {
            mPreference.setChecked(
                    Settings.System.getInt(mContext.getContentResolver(),
                            \"volume_panel_on_left\", 1) == 1
            );
        }
    }
}
"""
controller_path = "packages/apps/Settings/src/com/android/settings/sound/VolumePanelPositionPreferenceController.java"
if not os.path.exists(controller_path):
    with open(controller_path, "w", encoding="utf-8") as f:
        f.write(controller)

# 3) Add preference in sound_settings.xml
sound_xml = "packages/apps/Settings/res/xml/sound_settings.xml"
pref = '''<SwitchPreferenceCompat
    android:key="volume_panel_on_left"
    android:title="@string/volume_panel_on_left_title"
    android:summary="@string/volume_panel_on_left_summary"
    android:order="-170"
    settings:controller="com.android.settings.sound.VolumePanelPositionPreferenceController" />
'''
add_xml_pref(sound_xml, "volume_panel_on_left", pref)

# 4) Add strings
strings_path = "packages/apps/Settings/res/values/strings.xml"
strings = '''    <!-- Volume panel position strings -->
    <string name="volume_panel_on_left_title">Volume panel on left</string>
    <string name="volume_panel_on_left_summary">Show the volume panel on the left side of the screen</string>
'''
add_string_if_missing(strings_path, "volume_panel_on_left_title", strings)

print("\\nDone. Now rebuild:")
print("  source build/envsetup.sh")
print("  lunch <device>-userdebug")
print("  make -j$(nproc) systemui settings")
PY

chmod +x volume_panel_patcher.py
python3 volume_panel_patcher.py
```
