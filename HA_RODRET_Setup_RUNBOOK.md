# RODRET Setup for Alexa Remote Control

**Version:** 1.0 (Toggle + Momentary Optimized)  
**Date:** 2026-01-14  
**Purpose:** Use IKEA RODRET as a smart remote for Alexa routines

**What's New in v1.0:**
- ✨ **Toggle behavior for short press** - Press once ON, press again OFF
- ⏱️ **Momentary for long press** - Auto-reset after 2 seconds
- 🔄 **Conflict prevention** - Long press clears short press state
- 🎯 **Better Alexa integration** - More flexible routine options
- 🎛️ **2-button design** - Compact remote with 4 button combinations
- ✅ **Based on proven STYRBAR pattern** - Tested and reliable architecture

---

## What This Guide Does

This RUNBOOK sets up your IKEA RODRET to control devices through **Alexa Routines**.

**How it works:**
1. Press button on RODRET
2. Home Assistant detects button press
3. Virtual switch appears in Alexa
4. Alexa routine triggers based on switch state
5. Alexa controls your devices

**Button behavior:**
- **Short press**: Toggle switch ON/OFF (persistent state)
- **Long press**: Momentary ON for 2 seconds (auto-reset)

**Perfect for:** Controlling Alexa-compatible devices (lights, switches, scenes, etc.) with a compact 2-button remote.

**✅ Z2M users:** This guide uses **device triggers** - the most reliable method for Zigbee2MQTT, tested and confirmed working.

---

## Compatibility

| Component | Version | Notes |
|-----------|---------|-------|
| Home Assistant | 2024.1+ | Required |
| **Zigbee Stack** | ZHA 0.58.0+ OR Zigbee2MQTT 1.35.0+ | Choose ONE |
| RODRET Firmware | 1.0.35 / 1.1.0 | Both tested |
| Alexa Smart Home | Any | Required |

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Choose Your Zigbee Stack](#choose-your-zigbee-stack)
3. [System Architecture](#system-architecture)
4. [Directory Setup](#directory-setup)
5. [Step 1: Find Your RODRET](#step-1-find-your-rodret)
6. [Step 2: Capture Button Events](#step-2-capture-button-events)
7. [Step 3: Create Helper File](#step-3-create-helper-file)
8. [Step 4: Create Template Switches](#step-4-create-template-switches)
9. [Step 5: Create Automations](#step-5-create-automations)
10. [Step 6: Update Configuration](#step-6-update-configuration)
11. [Step 7: Load and Verify](#step-7-load-and-verify)
12. [Step 8: Expose to Alexa](#step-8-expose-to-alexa)
13. [Step 9: Create Alexa Routines](#step-9-create-alexa-routines)
14. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Components

- [ ] Home Assistant installed and running
- [ ] RODRET paired with ZHA or Zigbee2MQTT
- [ ] Alexa Smart Home skill connected to Home Assistant
- [ ] Home Assistant Cloud (Nabu Casa) OR manual Alexa skill setup
- [ ] SSH or File Editor access to Home Assistant
- [ ] Decided on RODRET naming (alpha, beta, etc.)

### Check Your Zigbee Stack

**You must know which Zigbee integration you're using:**

1. Go to **Settings** → **Devices & Services**
2. Look for either:
   - **ZHA** integration = You're using ZHA ✅
   - **MQTT** with Zigbee2MQTT = You're using Z2M ✅

**If unsure:** Check if you have a Zigbee2MQTT web interface (usually port 8099). If yes, you're using Z2M.

---

## Choose Your Zigbee Stack

This guide supports **BOTH** ZHA and Zigbee2MQTT. Follow the instructions for YOUR stack.

### Quick Stack Comparison

| Aspect | ZHA | Zigbee2MQTT |
|--------|-----|-------------|
| **Setup Complexity** | Easier | More complex |
| **Device Support** | Good | Excellent |
| **Integration** | Native HA | Requires MQTT |
| **Performance** | Fast | Fast |
| **This Guide** | Full support | Full support |

### Throughout This Guide

You'll see sections marked:

**🔷 ZHA Users:** Instructions for ZHA  
**🟠 Z2M Users:** Instructions for Zigbee2MQTT

Follow ONLY the sections for your stack.

---

## System Architecture

### How RODRET → Alexa Works

```
┌──────────────┐
│    RODRET    │  Physical button press
│   (Remote)   │
└──────┬───────┘
       │
       ↓ Zigbee signal
       
┌──────────────┐
│  ZHA or Z2M  │  Converts to HA event
└──────┬───────┘
       │
       ↓ Automation triggered
       
┌──────────────┐
│ input_boolean│  Status helper turns ON
│   *_status   │
└──────┬───────┘
       │
       ↓ Template mirrors state
       
┌──────────────┐
│ switch.*     │  Virtual switch for Alexa
└──────┬───────┘
       │
       ↓ State change notification
       
┌──────────────┐
│    Alexa     │  Detects switch ON
└──────┬───────┘
       │
       ↓ Routine executes
       
┌──────────────┐
│ Your Devices │  Lights, switches, scenes
└──────────────┘
```

### What Gets Created (Per RODRET)

**5 input_boolean helpers:**
- 1 master enabled toggle
- 4 status helpers (one per button combination)

**4 template switches:**
- Virtual switches exposed to Alexa
- Mirror status helpers

**4 automations:**
- Detect button presses
- Update status helpers
- Auto-reset after 2 seconds

**Total:** 9 entities per RODRET

---

## Directory Setup

### Create Directory Structure

```
config/
├── configuration.yaml
├── helpers/
│   ├── rodret_alpha.yaml
│   ├── rodret_beta.yaml
│   └── rodret_gamma.yaml
├── templates/
│   ├── rodret_alpha.yaml
│   ├── rodret_beta.yaml
│   └── rodret_gamma.yaml
└── automations/
    ├── rodret_alpha.yaml
    ├── rodret_beta.yaml
    └── rodret_gamma.yaml
```

**Create directories:**

```bash
mkdir -p /config/helpers
mkdir -p /config/templates
mkdir -p /config/automations
```

Or use File Editor add-on to create folders.

---

## Step 1: Find Your RODRET

### 🔷 ZHA Users

**Method 1: Using Developer Tools**

1. Go to **Developer Tools** → **Events**
2. Listen to event type: `zha_event`
3. Click **"START LISTENING"**
4. Press any button on RODRET
5. Copy the `device_id` from the event

**Example output:**
```json
{
  "event_type": "zha_event",
  "data": {
    "device_id": "a1b2c3d4e5f6g7h8i9j0",  ← COPY THIS
    "command": "on"
  }
}
```

**Method 2: From Device Page**

1. **Settings** → **Devices & Services** → **Devices**
2. Find your RODRET
3. Click on it
4. Look at URL: `.../device/DEVICE_ID_HERE`
5. Copy the device_id

✅ **Save this device_id** - you'll need it for automations.

### 🟠 Z2M Users

**You need the Z2M device_id (UUID format), not just the friendly name.**

**Method 1: Using Automation UI (EASIEST)**

1. **Settings** → **Automations & Scenes** → **Create New**
2. **Add Trigger** → **Device**
3. Find your RODRET in the device list
4. Select an action (e.g., "Turned on")
5. Click **"..."** (three dots) → **"Edit in YAML"**
6. Copy the `device_id` value

**Example:**
```yaml
trigger:
  - domain: mqtt
    device_id: 8fa17cc450fa42713be962ca8544782c  # ← COPY THIS
    type: action
    subtype: 'on'
```

7. **Cancel** the automation (don't save)

**Method 2: From Device Page**

1. **Settings** → **Devices & Services** → **Devices**
2. Find your RODRET
3. Click on it
4. Look at URL: `.../device/DEVICE_ID_HERE`
5. Copy the device_id (UUID format)

✅ **Save this device_id** - you'll need it for automations.

**Note:** The device_id is a long UUID like `8fa17cc450fa42713be962ca8544782c`, NOT the friendly name like "RODRET_Living_Room".

---

## Step 2: Capture Button Events

**⚠️ CRITICAL: Do NOT assume command/action values. Capture from YOUR system.**

### 🔷 ZHA Users: Capture Commands

1. **Developer Tools** → **Events**
2. Listen to: `zha_event`
3. Press each button on RODRET
4. Record `command` and `args` values

**Capture sheet:**

| Button | Command | Args | Notes |
|--------|---------|------|-------|
| POWER Short | | | Top button |
| POWER Long | | | Hold 2 seconds |
| DIMMER Short | | | Bottom button |
| DIMMER Long | | | Hold 2 seconds |

**Common ZHA commands for RODRET:**

| Button | Typical Command | Typical Args |
|--------|----------------|--------------|
| POWER Short | `toggle` | `{}` |
| POWER Long | `press` | `{"hold_time": 2}` OR `move_with_on_off` |
| DIMMER Short | `step_with_on_off` | `{"step_mode": 0}` |
| DIMMER Long | `move_with_on_off` | `{"move_mode": 0}` |

**⚠️ These are EXAMPLES. Your system may be different. Always capture YOUR actual values.**

**How to capture:**

```yaml
# Example event for POWER short press
{
  "event_type": "zha_event",
  "data": {
    "device_id": "YOUR_DEVICE_ID",
    "command": "toggle",      # ← Record this
    "args": {}                # ← And this
  }
}
```

### 🟠 Z2M Users: Capture Action Subtypes

**Use the Automation UI to get EXACT subtypes from YOUR device.**

1. **Settings** → **Automations & Scenes** → **Create New**
2. **Add Trigger** → **Device**
3. Select your RODRET
4. Try each action from the dropdown
5. For each action: **Edit in YAML** → Copy `subtype` value
6. **Cancel** automation (don't save)

**Capture sheet:**

| Button | Subtype | Notes |
|--------|---------|-------|
| POWER Short | | Top button |
| POWER Long | | Hold 2 seconds |
| DIMMER Short | | Bottom button |
| DIMMER Long | | Hold 2 seconds |

**Common Z2M subtypes for RODRET:**

| Button | Typical Subtype |
|--------|-----------------|
| POWER Short | `on` |
| POWER Long | `brightness_move_up` |
| DIMMER Short | `off` |
| DIMMER Long | `brightness_move_down` |

**⚠️ These are the most common values, but you should still verify using the dropdown method to confirm YOUR device uses these exact subtypes.**

**Example automation UI check:**

```yaml
trigger:
  - domain: mqtt
    device_id: "YOUR_UUID"
    type: action
    subtype: "on"           # ← YOUR captured value
```

---

## Step 3: Create Helper File

**File:** `helpers/rodret_alpha.yaml`

```yaml
# RODRET ALPHA - Alexa Remote Helpers
# Created: 2026-01-14
# Purpose: Status tracking for RODRET button presses

rodret_alpha_enabled:
  name: "RODRET Alpha Enabled"
  icon: mdi:toggle-switch

rodret_alpha_power_short_status:
  name: "RODRET Alpha Power Short"
  icon: mdi:power

rodret_alpha_power_long_status:
  name: "RODRET Alpha Power Long"
  icon: mdi:power-settings

rodret_alpha_dimmer_short_status:
  name: "RODRET Alpha Dimmer Short"
  icon: mdi:brightness-6

rodret_alpha_dimmer_long_status:
  name: "RODRET Alpha Dimmer Long"
  icon: mdi:brightness-7
```

**Key points:**
- ✅ Master toggle: `rodret_alpha_enabled`
- ✅ Four status helpers for buttons
- ✅ Descriptive names with icons
- ✅ **Replace `alpha` with your chosen name** (alpha/beta/gamma/etc.)

---

## Step 4: Create Template Switches

**File:** `templates/rodret_alpha.yaml`

```yaml
# RODRET ALPHA - Alexa Template Switches
# Created: 2026-01-14
# Purpose: Virtual switches for Alexa routines

- switch:
  - name: "RODRET Alpha Power Short"
    unique_id: rodret_alpha_power_short_switch
    icon: mdi:power
    value_template: "{{ is_state('input_boolean.rodret_alpha_power_short_status', 'on') }}"
    turn_on:
      - condition: state
        entity_id: input_boolean.rodret_alpha_enabled
        state: 'on'
      - service: input_boolean.turn_on
        target:
          entity_id: input_boolean.rodret_alpha_power_short_status
    turn_off:
      - condition: state
        entity_id: input_boolean.rodret_alpha_enabled
        state: 'on'
      - service: input_boolean.turn_off
        target:
          entity_id: input_boolean.rodret_alpha_power_short_status

  - name: "RODRET Alpha Power Long"
    unique_id: rodret_alpha_power_long_switch
    icon: mdi:power-settings
    value_template: "{{ is_state('input_boolean.rodret_alpha_power_long_status', 'on') }}"
    turn_on:
      - condition: state
        entity_id: input_boolean.rodret_alpha_enabled
        state: 'on'
      - service: input_boolean.turn_on
        target:
          entity_id: input_boolean.rodret_alpha_power_long_status
    turn_off:
      - condition: state
        entity_id: input_boolean.rodret_alpha_enabled
        state: 'on'
      - service: input_boolean.turn_off
        target:
          entity_id: input_boolean.rodret_alpha_power_long_status

  - name: "RODRET Alpha Dimmer Short"
    unique_id: rodret_alpha_dimmer_short_switch
    icon: mdi:brightness-6
    value_template: "{{ is_state('input_boolean.rodret_alpha_dimmer_short_status', 'on') }}"
    turn_on:
      - condition: state
        entity_id: input_boolean.rodret_alpha_enabled
        state: 'on'
      - service: input_boolean.turn_on
        target:
          entity_id: input_boolean.rodret_alpha_dimmer_short_status
    turn_off:
      - condition: state
        entity_id: input_boolean.rodret_alpha_enabled
        state: 'on'
      - service: input_boolean.turn_off
        target:
          entity_id: input_boolean.rodret_alpha_dimmer_short_status

  - name: "RODRET Alpha Dimmer Long"
    unique_id: rodret_alpha_dimmer_long_switch
    icon: mdi:brightness-7
    value_template: "{{ is_state('input_boolean.rodret_alpha_dimmer_long_status', 'on') }}"
    turn_on:
      - condition: state
        entity_id: input_boolean.rodret_alpha_enabled
        state: 'on'
      - service: input_boolean.turn_on
        target:
          entity_id: input_boolean.rodret_alpha_dimmer_long_status
    turn_off:
      - condition: state
        entity_id: input_boolean.rodret_alpha_enabled
        state: 'on'
      - service: input_boolean.turn_off
        target:
          entity_id: input_boolean.rodret_alpha_dimmer_long_status
```

**Key points:**
- ✅ Four switches mirror the status helpers
- ✅ Controlled by master enabled toggle
- ✅ Exposed to Alexa for routines
- ✅ **Replace `alpha` throughout** with your chosen name

---

## Step 5: Create Automations

### 🔷 ZHA Version

**File:** `automations/rodret_alpha.yaml`

```yaml
# RODRET ALPHA - Alexa Button Automations (ZHA)
# Created: 2026-01-14
# Purpose: Handle button presses and update status helpers
# Stack: ZHA

# ============================================
# POWER SHORT PRESS - Toggle behavior
# ============================================
- id: rodret_alpha_power_short
  alias: "RODRET Alpha - Power Short Press"
  description: "Toggle switch ON/OFF"
  mode: single
  triggers:
    - trigger: event
      event_type: zha_event
      event_data:
        device_id: YOUR_ZHA_DEVICE_ID_HERE
        command: YOUR_CAPTURED_COMMAND_HERE
        # args: {}  # Add if needed
      id: power_short
  conditions:
    - condition: state
      entity_id: input_boolean.rodret_alpha_enabled
      state: 'on'
  actions:
    - service: input_boolean.toggle
      target:
        entity_id: input_boolean.rodret_alpha_power_short_status

# ============================================
# POWER LONG PRESS - Momentary behavior
# ============================================
- id: rodret_alpha_power_long
  alias: "RODRET Alpha - Power Long Press"
  description: "Momentary ON for 2 seconds"
  mode: restart
  triggers:
    - trigger: event
      event_type: zha_event
      event_data:
        device_id: YOUR_ZHA_DEVICE_ID_HERE
        command: YOUR_CAPTURED_COMMAND_HERE
        # args: {"hold_time": 2}  # Add if needed
      id: power_long
  conditions:
    - condition: state
      entity_id: input_boolean.rodret_alpha_enabled
      state: 'on'
  actions:
    # Clear any short press state
    - if:
        - condition: state
          entity_id: input_boolean.rodret_alpha_power_short_status
          state: 'on'
      then:
        - service: input_boolean.turn_off
          target:
            entity_id: input_boolean.rodret_alpha_power_short_status
    # Momentary pulse
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.rodret_alpha_power_long_status
    - delay: 2
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.rodret_alpha_power_long_status

# ============================================
# DIMMER SHORT PRESS - Toggle behavior
# ============================================
- id: rodret_alpha_dimmer_short
  alias: "RODRET Alpha - Dimmer Short Press"
  description: "Toggle switch ON/OFF"
  mode: single
  triggers:
    - trigger: event
      event_type: zha_event
      event_data:
        device_id: YOUR_ZHA_DEVICE_ID_HERE
        command: YOUR_CAPTURED_COMMAND_HERE
        # args: {}  # Add if needed
      id: dimmer_short
  conditions:
    - condition: state
      entity_id: input_boolean.rodret_alpha_enabled
      state: 'on'
  actions:
    - service: input_boolean.toggle
      target:
        entity_id: input_boolean.rodret_alpha_dimmer_short_status

# ============================================
# DIMMER LONG PRESS - Momentary behavior
# ============================================
- id: rodret_alpha_dimmer_long
  alias: "RODRET Alpha - Dimmer Long Press"
  description: "Momentary ON for 2 seconds"
  mode: restart
  triggers:
    - trigger: event
      event_type: zha_event
      event_data:
        device_id: YOUR_ZHA_DEVICE_ID_HERE
        command: YOUR_CAPTURED_COMMAND_HERE
        # args: {}  # Add if needed
      id: dimmer_long
  conditions:
    - condition: state
      entity_id: input_boolean.rodret_alpha_enabled
      state: 'on'
  actions:
    # Clear any short press state
    - if:
        - condition: state
          entity_id: input_boolean.rodret_alpha_dimmer_short_status
          state: 'on'
      then:
        - service: input_boolean.turn_off
          target:
            entity_id: input_boolean.rodret_alpha_dimmer_short_status
    # Momentary pulse
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.rodret_alpha_dimmer_long_status
    - delay: 2
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.rodret_alpha_dimmer_long_status
```

**🔷 ZHA - Required changes:**
1. Replace `YOUR_ZHA_DEVICE_ID_HERE` with device_id from Step 1
2. Replace `YOUR_CAPTURED_COMMAND_HERE` with commands from Step 2
3. Add/remove `args:` as needed based on your captures
4. Replace `alpha` throughout with your chosen name

### 🟠 Z2M Version

**File:** `automations/rodret_alpha.yaml`

```yaml
# RODRET ALPHA - Alexa Button Automations (Z2M)
# Created: 2026-01-14
# Purpose: Handle button presses and update status helpers
# Stack: Zigbee2MQTT

# ============================================
# POWER SHORT PRESS - Toggle behavior
# ============================================
- id: rodret_alpha_power_short
  alias: "RODRET Alpha - Power Short Press"
  description: "Toggle switch ON/OFF"
  mode: single
  triggers:
    - domain: mqtt
      device_id: YOUR_Z2M_DEVICE_ID_HERE
      type: action
      subtype: "on"
      id: power_short
      trigger: device
  conditions:
    - condition: state
      entity_id: input_boolean.rodret_alpha_enabled
      state: 'on'
  actions:
    - service: input_boolean.toggle
      target:
        entity_id: input_boolean.rodret_alpha_power_short_status

# ============================================
# POWER LONG PRESS - Momentary behavior
# ============================================
- id: rodret_alpha_power_long
  alias: "RODRET Alpha - Power Long Press"
  description: "Momentary ON for 2 seconds"
  mode: restart
  triggers:
    - domain: mqtt
      device_id: YOUR_Z2M_DEVICE_ID_HERE
      type: action
      subtype: "brightness_move_up"
      id: power_long
      trigger: device
  conditions:
    - condition: state
      entity_id: input_boolean.rodret_alpha_enabled
      state: 'on'
  actions:
    # Clear any short press state
    - if:
        - condition: state
          entity_id: input_boolean.rodret_alpha_power_short_status
          state: 'on'
      then:
        - service: input_boolean.turn_off
          target:
            entity_id: input_boolean.rodret_alpha_power_short_status
    # Momentary pulse
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.rodret_alpha_power_long_status
    - delay: 2
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.rodret_alpha_power_long_status

# ============================================
# DIMMER SHORT PRESS - Toggle behavior
# ============================================
- id: rodret_alpha_dimmer_short
  alias: "RODRET Alpha - Dimmer Short Press"
  description: "Toggle switch ON/OFF"
  mode: single
  triggers:
    - domain: mqtt
      device_id: YOUR_Z2M_DEVICE_ID_HERE
      type: action
      subtype: "off"
      id: dimmer_short
      trigger: device
  conditions:
    - condition: state
      entity_id: input_boolean.rodret_alpha_enabled
      state: 'on'
  actions:
    - service: input_boolean.toggle
      target:
        entity_id: input_boolean.rodret_alpha_dimmer_short_status

# ============================================
# DIMMER LONG PRESS - Momentary behavior
# ============================================
- id: rodret_alpha_dimmer_long
  alias: "RODRET Alpha - Dimmer Long Press"
  description: "Momentary ON for 2 seconds"
  mode: restart
  triggers:
    - domain: mqtt
      device_id: YOUR_Z2M_DEVICE_ID_HERE
      type: action
      subtype: "brightness_move_down"
      id: dimmer_long
      trigger: device
  conditions:
    - condition: state
      entity_id: input_boolean.rodret_alpha_enabled
      state: 'on'
  actions:
    # Clear any short press state
    - if:
        - condition: state
          entity_id: input_boolean.rodret_alpha_dimmer_short_status
          state: 'on'
      then:
        - service: input_boolean.turn_off
          target:
            entity_id: input_boolean.rodret_alpha_dimmer_short_status
    # Momentary pulse
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.rodret_alpha_dimmer_long_status
    - delay: 2
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.rodret_alpha_dimmer_long_status
```

**🟠 Z2M - Required changes:**
1. Replace `YOUR_Z2M_DEVICE_ID_HERE` with UUID from Step 1
2. Subtypes are pre-filled with common values:
   - Power short: `"on"`
   - Power long: `"brightness_move_up"`
   - Dimmer short: `"off"`
   - Dimmer long: `"brightness_move_down"`
3. **⚠️ Verify these match YOUR device** using Step 2 capture method
4. Keep `domain: mqtt`, `type: action`, `trigger: device` - these are fixed
5. Replace `alpha` throughout with your chosen name

---

## Step 6: Update Configuration

**File:** `configuration.yaml`

Add these lines:

```yaml
# Input Boolean Helpers
input_boolean: !include_dir_merge_named helpers/

# Template Switches
template: !include_dir_list templates/

# Automations
automation: !include_dir_list automations/
```

**⚠️ Important:**
- If you already have these lines, DON'T duplicate them
- The `!include_dir_*` directives load ALL files in those folders
- Make sure paths match your directory structure

**Verify your configuration.yaml:**

```bash
# Check configuration
ha core check

# Or from Settings → System → Restart → Check Configuration
```

---

## Step 7: Load and Verify

### Reload Components

**Method 1: Developer Tools**

1. **Developer Tools** → **YAML**
2. Click **"Input Booleans"** → Reload ✓
3. Click **"Templates"** → Reload ✓
4. Click **"Automations"** → Reload ✓

**Method 2: Settings Menu**

1. **Settings** → **System**
2. **Controls** → Reload relevant components

### Verify Entities Created

**Check helpers:**
```
input_boolean.rodret_alpha_enabled
input_boolean.rodret_alpha_power_short_status
input_boolean.rodret_alpha_power_long_status
input_boolean.rodret_alpha_dimmer_short_status
input_boolean.rodret_alpha_dimmer_long_status
```

**Check switches:**
```
switch.rodret_alpha_power_short
switch.rodret_alpha_power_long
switch.rodret_alpha_dimmer_short
switch.rodret_alpha_dimmer_long
```

**Check automations:**
```
automation.rodret_alpha_power_short_press
automation.rodret_alpha_power_long_press
automation.rodret_alpha_dimmer_short_press
automation.rodret_alpha_dimmer_long_press
```

**Total:** 9 entities (5 helpers + 4 switches)

### Enable Master Toggle

1. Find `input_boolean.rodret_alpha_enabled`
2. **Turn it ON** ✓
3. Keep it ON during testing

### Test Buttons

**For each button:**

1. Press button on RODRET
2. Check **Developer Tools** → **States**
3. Look for status helper turning ON
4. Look for template switch mirroring the state
5. For short press: Status should toggle (ON → OFF → ON)
6. For long press: Status should reset to OFF after 2 seconds

**Test checklist:**

- [ ] POWER short press toggles
- [ ] POWER long press momentary (2s reset)
- [ ] DIMMER short press toggles
- [ ] DIMMER long press momentary (2s reset)
- [ ] Master toggle disables all buttons
- [ ] Long press clears short press state

**If buttons don't work:** See [Troubleshooting](#troubleshooting)

---

## Step 8: Expose to Alexa

### Option A: Home Assistant Cloud (Recommended)

1. **Configuration** → **Home Assistant Cloud**
2. **Amazon Alexa** section
3. Click **"Manage"** or **"Entity Settings"**
4. Find all `switch.rodret_*` entities
5. Enable each switch for Alexa
6. Click **"Sync Entities"**
7. Wait 30 seconds

### Option B: Manual Alexa Skill

Add to `configuration.yaml`:

```yaml
alexa:
  smart_home:
    filter:
      include_entity_globs:
        - switch.rodret_*
    entity_config:
      switch.rodret_alpha_power_short:
        name: "RODRET Alpha Power Short"
        display_categories: SWITCH
      switch.rodret_alpha_power_long:
        name: "RODRET Alpha Power Long"
        display_categories: SWITCH
      switch.rodret_alpha_dimmer_short:
        name: "RODRET Alpha Dimmer Short"
        display_categories: SWITCH
      switch.rodret_alpha_dimmer_long:
        name: "RODRET Alpha Dimmer Long"
        display_categories: SWITCH
```

Restart Home Assistant after adding.

### Discover in Alexa App

1. Open **Alexa app**
2. **Devices** → **+** (Add Device)
3. **Other** → **Discover Devices**
4. Wait **45 seconds** (don't skip this!)
5. Check if switches appear

Or just say: **"Alexa, discover devices"**

**Verify in Alexa:**
- All 4 switches should appear
- Named correctly
- Can toggle manually in app

---

## Step 9: Create Alexa Routines

### Routine Pattern

**For SHORT PRESS buttons (toggle behavior):**

Create **TWO routines per button:**
1. Routine when switch turns ON → Action 1
2. Routine when switch turns OFF → Action 2

**For LONG PRESS buttons (momentary):**

Create **ONE routine per button:**
1. Routine when switch turns ON → Scene/Mode change

### Example: POWER Short Press

**Routine 1: "RODRET Alpha Power Short ON"**
- **When:** RODRET Alpha Power Short turns ON
- **Then:** Turn on Living Room Lights to 100%

**Routine 2: "RODRET Alpha Power Short OFF"**
- **When:** RODRET Alpha Power Short turns OFF
- **Then:** Turn off Living Room Lights

### Example: POWER Long Press

**Routine: "RODRET Alpha Power Long"**
- **When:** RODRET Alpha Power Long turns ON
- **Then:** Activate "Movie Mode" scene

### Full Example Setup (4 buttons = 6 routines)

| Button | Behavior | Routines Needed |
|--------|----------|----------------|
| POWER Short | Toggle | 2 (ON action + OFF action) |
| POWER Long | Momentary | 1 (ON action only) |
| DIMMER Short | Toggle | 2 (ON action + OFF action) |
| DIMMER Long | Momentary | 1 (ON action only) |

**Total:** 6 Alexa routines per RODRET

### Create Routine (Step by Step)

1. **Alexa app** → **More** → **Routines**
2. **+** (Create new routine)
3. **Enter routine name**
4. **When this happens:**
   - Smart Home
   - Find your switch
   - Select "turns ON" (or "turns OFF" for toggle OFF action)
5. **Add action:**
   - Smart Home
   - Select device/group to control
   - Choose action (on/off/brightness/color/etc.)
6. **Save**

**Repeat for all buttons.**

---

## Troubleshooting

### Problem: Button Press Not Detected

**Symptoms:**
- Press RODRET button
- Nothing happens in HA

**Solutions:**
- Verify `rodret_alpha_enabled` is ON
- Check automation is enabled
- Confirm command/action names match YOUR system
- Check HA logs for errors

### Problem: Status Doesn't Turn ON

**Symptoms:**
- Button press detected
- But `input_boolean.*_status` stays OFF

**Solutions:**
1. Check automation action section has:
```yaml
action:
  - service: input_boolean.turn_on
    target:
      entity_id: input_boolean.rodret_alpha_power_short_status
```

2. Test manually:
   - Developer Tools → Services
   - Call `input_boolean.turn_on`
   - Target: `input_boolean.rodret_alpha_power_short_status`
   - If this works: Automation issue
   - If doesn't work: Helper not created properly

### Problem: Status Stays ON Forever (Long Press)

**Symptoms:**
- Long press status turns ON
- Never turns OFF

**Solutions:**
1. Check automation has delay and turn_off:
```yaml
action:
  - service: input_boolean.turn_on
    target:
      entity_id: input_boolean.rodret_alpha_power_long_status
  - delay: 2  # ← This line needed
  - service: input_boolean.turn_off  # ← This line needed
    target:
      entity_id: input_boolean.rodret_alpha_power_long_status
```

2. Check automation `mode: restart` for long press automations

### Problem: Alexa Doesn't See Switches

**Symptoms:**
- Switches exist in HA
- Not appearing in Alexa app

**Solutions:**

**For Home Assistant Cloud:**
1. Configuration → Home Assistant Cloud → Alexa
2. Check switches are enabled for exposure
3. Click **Sync Entities**
4. Wait 30 seconds
5. Discover devices in Alexa app

**For Manual Skill:**
1. Check `alexa:` configuration in configuration.yaml
2. Verify `include_entity_globs: - switch.rodret_*`
3. Restart HA
4. Discover devices in Alexa app (wait full 45 seconds)

### Problem: Alexa Routine Doesn't Trigger

**Symptoms:**
- Button press works in HA
- Switch turns ON/OFF
- Alexa routine doesn't execute

**Solutions:**
1. **Check routine trigger:**
   - Open routine in Alexa app
   - Verify it's set to "When RODRET X turns ON"
   - For short press OFF actions: Set to "turns OFF"

2. **Test manually:**
   - Alexa app → Devices → Your switch
   - Tap to turn ON
   - Check if routine triggers
   - If yes: HA issue
   - If no: Alexa routine issue

3. **Check routine is enabled:**
   - Open routine
   - Toggle at bottom should be ON

4. **Check Alexa connectivity:**
   - Say "Alexa, discover devices"
   - Wait for completion

### 🟠 Z2M Specific: Device Trigger Not Working

**Symptoms:**
- Automation doesn't trigger
- Button press does nothing

**Solutions:**
1. **Verify device_id format:**
   - Should be UUID: `8fa17cc450fa42713be962ca8544782c`
   - NOT friendly name: `RODRET_Living_Room`

2. **Check trigger format:**
```yaml
triggers:
  - domain: mqtt
    device_id: "YOUR_UUID_HERE"  # Must be quoted
    type: action
    subtype: "on"  # Must match YOUR device
    id: power_short
    trigger: device  # This line is required
```

3. **Test in Automation UI:**
   - Create test automation using UI
   - Select your RODRET device
   - If it works in UI, copy the exact YAML format
   - If doesn't work in UI, device may not support triggers

### Problem: Different Button Presses Trigger Same Action

**Symptoms:**
- Multiple buttons do the same thing
- Actions mixed up

**Solutions:**
- Verify each automation has unique `subtype` (Z2M) or `command` (ZHA)
- Verify each automation targets different status helper
- Check you captured button events correctly in Step 2
- Use **Edit in YAML** in automation UI to confirm trigger values

---

## Adding More RODRETs

### Quick Checklist

For RODRET name: `______` (beta/gamma/delta)

- [ ] Paired with ZHA or Z2M
- [ ] Found device_id (UUID for both ZHA and Z2M)
- [ ] Captured all button commands (ZHA) or subtypes (Z2M)
- [ ] Created `helpers/rodret_[NAME].yaml`
- [ ] Created `templates/rodret_[NAME].yaml`
- [ ] Created `automations/rodret_[NAME].yaml` (single file with all 4 buttons)
- [ ] Replaced all placeholders with actual values
- [ ] For Z2M: Used device trigger method with YOUR captured subtypes
- [ ] Reload: Input Booleans, Templates, Automations
- [ ] Verified 9 entities created
- [ ] Tested all 4 buttons
- [ ] Exposed switches to Alexa
- [ ] Discovered devices in Alexa app
- [ ] Created Alexa routines (6 total: 2+1+2+1)

---

## Naming Consistency Rules

**Choose ONE spelling and use it everywhere.**

**Bad example:**
```
helpers/rodret_jupiter.yaml     ← "jupiter"
automations/rodret_jupitar.yaml  ← "jupitar" (WRONG!)
Entity IDs won't match! Nothing will work.
```

**Good example:**
```
helpers/rodret_jupiter.yaml
automations/rodret_jupiter.yaml
All entity IDs use "jupiter" consistently ✓
```

**Tips:**
- Use lowercase for names (alpha, beta, gamma)
- Use find & replace, not manual typing
- Double-check entity IDs match across files

---

## FAQ

**Q: Why do short press buttons toggle instead of auto-reset?**  
A: Toggle behavior is better for most Alexa use cases. You can use one button for both ON and OFF actions. Long press buttons still auto-reset (momentary) for scene/mode changes.

**Q: How many Alexa routines do I need per button?**  
A: Short press = 2 routines (one for ON, one for OFF). Long press = 1 routine (only ON trigger needed). Total = 6 routines per RODRET.

**Q: Can I change short press to momentary like long press?**  
A: Yes, just change `input_boolean.toggle` to the long press pattern (turn_on → delay 2 → turn_off) in your automation file.

**Q: Can I control HA devices instead of Alexa devices?**  
A: Yes, but that's not what this guide is optimized for. Add custom actions to automation files if needed.

**Q: Can I use both ZHA and Z2M?**  
A: No, you must choose one. They conflict.

**Q: Why device triggers instead of state triggers for Z2M?**  
A: Device triggers are more reliable and are what the HA automation UI generates natively. They don't require an intermediate sensor entity and work better with Z2M.

**Q: How many RODRETs can I add?**  
A: As many as you want. Each adds 9 entities + 1 automation file with 4 automations.

**Q: Do I need Home Assistant Cloud?**  
A: No, but it's the easiest way to connect Alexa. Manual skill setup also works.

**Q: Can I customize switch names in Alexa?**  
A: Yes, in Alexa app: Devices → Your switch → Edit Name

**Q: Will this work with Google Home?**  
A: The HA setup is the same. Google Home routines work similarly to Alexa.

**Q: What's the difference between RODRET and STYRBAR?**  
A: RODRET has 2 physical buttons (4 combinations with short/long press). STYRBAR has 4 buttons (8 combinations). Otherwise, the setup is identical.

**Q: Can I mix RODRET and STYRBAR in the same setup?**  
A: Yes! They use the same architecture. Just follow the respective guides for each device.

---

## Summary

### What You Created

**Per RODRET:**
- 1 master enabled toggle
- 4 status helpers
- 4 template switches
- 1 consolidated automation (4 buttons)

**Total:** 9 entities + 1 automation per RODRET

### How It Works

1. Press physical button
2. HA automation detects via device trigger (Z2M) or event (ZHA)
3. Status helper toggles or pulses ON
4. Template switch mirrors status
5. Alexa detects switch state change
6. Alexa routine executes
7. Your device is controlled
8. Long press status auto-resets after 2 seconds

### Key Benefits

✅ Physical buttons for Alexa  
✅ No voice commands needed  
✅ Works with any Alexa-compatible device  
✅ Flexible routine programming  
✅ Easy to add more RODRETs  
✅ Master enable/disable per RODRET  
✅ **Z2M: Reliable device triggers** (tested and confirmed)  
✅ **Single automation per RODRET** (cleaner than 4 files)  
✅ **Compact 2-button design** - perfect for bedside/hallway use

---

## Credits

**Created by:** Home Assistant Community  
**Version:** 1.0 (Based on STYRBAR v4.2 pattern)  
**License:** MIT  
**Support:** Home Assistant Forums  
**Tested with:** ZHA 0.58.0+, Zigbee2MQTT 1.35.0+, RODRET firmware 1.0.35/1.1.0

**Special thanks:** 
- STYRBAR community for the proven architecture
- Users who tested device trigger methods
- Contributors of the toggle + momentary pattern

---

**END OF RUNBOOK v1.0**

Enjoy your compact smart button setup! 🎉
