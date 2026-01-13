# STYRBAR Setup for Alexa Remote Control

**Version:** 4.0 (Alexa-Optimized)  
**Date:** 2026-01-12  
**Purpose:** Use IKEA STYRBAR as a smart remote for Alexa routines

---

## What This Guide Does

This RUNBOOK sets up your IKEA STYRBAR to control devices through **Alexa Routines**.

**How it works:**
1. Press button on STYRBAR
2. Home Assistant detects button press
3. Virtual switch appears in Alexa
4. Alexa routine triggers based on switch state
5. Alexa controls your devices

**Perfect for:** Controlling Alexa-compatible devices (lights, switches, scenes, etc.) with physical buttons.

---

## Compatibility

| Component | Version | Notes |
|-----------|---------|-------|
| Home Assistant | 2024.1+ | Required |
| **Zigbee Stack** | ZHA 0.58.0+ OR Zigbee2MQTT 1.35.0+ | Choose ONE |
| STYRBAR Firmware | 2.4.5 / 24.4.5 | Both tested |
| Alexa Smart Home | Any | Required |

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Choose Your Zigbee Stack](#choose-your-zigbee-stack)
3. [System Architecture](#system-architecture)
4. [Directory Setup](#directory-setup)
5. [Step 1: Find Your STYRBAR](#step-1-find-your-styrbar)
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
- [ ] STYRBAR paired with ZHA or Zigbee2MQTT
- [ ] Alexa Smart Home skill connected to Home Assistant
- [ ] Home Assistant Cloud (Nabu Casa) OR manual Alexa skill setup
- [ ] SSH or File Editor access to Home Assistant
- [ ] Decided on STYRBAR naming (alpha, beta, charlie, etc.)

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

### How STYRBAR → Alexa Works

```
┌──────────────┐
│   STYRBAR    │  Physical button press
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

### What Gets Created (Per STYRBAR)

**9 input_boolean helpers:**
- 1 master enabled toggle
- 8 status helpers (one per button)

**8 template switches:**
- Virtual switches exposed to Alexa
- Mirror status helpers

**8 automations:**
- Detect button presses
- Update status helpers
- Auto-reset after 2 seconds

**Total:** 17 entities per STYRBAR

---

## Directory Setup

### Create Directory Structure

```
config/
├── configuration.yaml
├── helpers/
│   ├── styrbar_alpha.yaml
│   ├── styrbar_beta.yaml
│   └── styrbar_charlie.yaml
├── templates/
│   ├── styrbar_alpha.yaml
│   ├── styrbar_beta.yaml
│   └── styrbar_charlie.yaml
└── automations/
    ├── styrbar_alpha.yaml
    ├── styrbar_beta.yaml
    └── styrbar_charlie.yaml
```

**Create directories:**

```bash
mkdir -p /config/helpers
mkdir -p /config/templates
mkdir -p /config/automations
```

Or use File Editor add-on to create folders.

---

## Step 1: Find Your STYRBAR

### 🔷 ZHA Users

**Method 1: Using Developer Tools**

1. Go to **Developer Tools** → **Events**
2. Listen to event type: `zha_event`
3. Click **"START LISTENING"**
4. Press any button on STYRBAR
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
2. Find your STYRBAR
3. Click on it
4. Look at URL: `.../device/DEVICE_ID_HERE`
5. Copy the device_id

✅ **Save this device_id** - you'll need it for automations.

### 🟠 Z2M Users

**Method 1: Zigbee2MQTT Web Interface**

1. Open Z2M web interface (usually `http://homeassistant.local:8099`)
2. Click **Devices** tab
3. Find your STYRBAR
4. Note the **friendly name** (e.g., `STYRBAR_Living_Room`)

**Method 2: Home Assistant**

1. **Developer Tools** → **States**
2. Search for: `sensor.` + your device name
3. Look for: `sensor.styrbar_living_room_action`
4. The prefix before `_action` is your friendly name

✅ **Save this friendly name** - you'll need it for automations.

---

## Step 2: Capture Button Events

**⚠️ CRITICAL: Do NOT assume command/action values. Capture from YOUR system.**

### 🔷 ZHA Users: Capture Commands

1. **Developer Tools** → **Events**
2. Listen to: `zha_event`
3. Press each button on STYRBAR
4. Record `command` and `args` values

**Capture sheet:**

| Button | Command | Args | Notes |
|--------|---------|------|-------|
| UP Short | | | |
| UP Long | | | |
| DOWN Short | | | |
| DOWN Long | | | |
| LEFT Short | | | |
| LEFT Long | | | May not work |
| RIGHT Short | | | |
| RIGHT Long | | | May not work |

**Common ZHA commands (reference only):**

| Button | Typical Command | Typical Args |
|--------|----------------|--------------|
| UP Short | `on` | - |
| UP Long | `move_with_on_off` | `[0, 83]` |
| DOWN Short | `off` | - |
| DOWN Long | `move` | `[1, 83, 0, 0]` |
| LEFT Short | `press` | `[257, 13, 0]` |
| LEFT Long | `hold` | `[3329, 0]` |
| RIGHT Short | `press` | `[256, 13, 0]` |
| RIGHT Long | `hold` | `[3328, 0]` |

### 🟠 Z2M Users: Capture Actions

1. **Developer Tools** → **States**
2. Find: `sensor.[your_styrbar]_action`
3. Press each button
4. Watch sensor value change
5. Record exact action name

**Capture sheet:**

| Button | Action Name | Notes |
|--------|-------------|-------|
| UP Short | | |
| UP Long | | |
| DOWN Short | | |
| DOWN Long | | |
| LEFT Short | | |
| LEFT Long | | May not work |
| RIGHT Short | | |
| RIGHT Long | | May not work |

**Common Z2M actions (reference only):**

| Button | Typical Action |
|--------|----------------|
| UP Short | `on` |
| UP Long | `brightness_move_up` |
| DOWN Short | `off` |
| DOWN Long | `brightness_move_down` |
| LEFT Short | `arrow_left_click` |
| LEFT Long | `arrow_left_hold` |
| RIGHT Short | `arrow_right_click` |
| RIGHT Long | `arrow_right_hold` |

**⚠️ Your values may be different! Use YOUR captured values.**

---

## Step 3: Create Helper File

### File Location

Create file: `helpers/styrbar_[NAME].yaml`

Replace `[NAME]` with: alpha, beta, charlie, etc.

### Template Content

```yaml
# STYRBAR [NAME] - Input Boolean Helpers
# Created: [DATE]
# Location: [ROOM/LOCATION]
# Device ID (ZHA) or Friendly Name (Z2M): [IDENTIFIER]

# ===== MASTER ENABLED TOGGLE =====
# Turn this OFF to disable entire STYRBAR
styrbar_[NAME]_enabled:
  name: "STYRBAR [NAME] - Enabled"
  initial: true
  icon: mdi:remote


# ===== STATUS TRACKING (For Alexa) =====
# These turn ON when button pressed, OFF after 2 seconds

styrbar_[NAME]_up_short_status:
  name: "STYRBAR [NAME] - Up Short"
  initial: false
  icon: mdi:arrow-up-circle

styrbar_[NAME]_up_long_status:
  name: "STYRBAR [NAME] - Up Long"
  initial: false
  icon: mdi:arrow-up-circle-outline

styrbar_[NAME]_down_short_status:
  name: "STYRBAR [NAME] - Down Short"
  initial: false
  icon: mdi:arrow-down-circle

styrbar_[NAME]_down_long_status:
  name: "STYRBAR [NAME] - Down Long"
  initial: false
  icon: mdi:arrow-down-circle-outline

styrbar_[NAME]_left_short_status:
  name: "STYRBAR [NAME] - Left Short"
  initial: false
  icon: mdi:arrow-left-circle

styrbar_[NAME]_left_long_status:
  name: "STYRBAR [NAME] - Left Long"
  initial: false
  icon: mdi:arrow-left-circle-outline

styrbar_[NAME]_right_short_status:
  name: "STYRBAR [NAME] - Right Short"
  initial: false
  icon: mdi:arrow-right-circle

styrbar_[NAME]_right_long_status:
  name: "STYRBAR [NAME] - Right Long"
  initial: false
  icon: mdi:arrow-right-circle-outline
```

### Example for "Alpha"

**File:** `helpers/styrbar_alpha.yaml`

```yaml
# STYRBAR Alpha - Input Boolean Helpers
# Created: 2026-01-12
# Location: Living Room
# Device ID: a1b2c3d4e5f6g7h8i9j0

styrbar_alpha_enabled:
  name: "STYRBAR Alpha - Enabled"
  initial: true
  icon: mdi:remote

styrbar_alpha_up_short_status:
  name: "STYRBAR Alpha - Up Short"
  initial: false
  icon: mdi:arrow-up-circle

# ... (7 more status helpers)
```

### Quick Create

**Using sed (Linux/Mac):**
```bash
sed 's/\[NAME\]/alpha/g; s/\[DATE\]/2026-01-12/g; s/\[ROOM\/LOCATION\]/Living Room/g; s/\[IDENTIFIER\]/a1b2c3d4e5f6g7h8i9j0/g' template.yaml > helpers/styrbar_alpha.yaml
```

**Using find & replace:**
1. Copy template above
2. Replace `[NAME]` with `alpha`
3. Replace `[DATE]` with today
4. Replace `[ROOM/LOCATION]` with room name
5. Replace `[IDENTIFIER]` with device_id or friendly name
6. Save as `helpers/styrbar_alpha.yaml`

---

## Step 4: Create Template Switches

### File Location

Create file: `templates/styrbar_[NAME].yaml`

### Template Content

```yaml
# STYRBAR [NAME] - Template Switches for Alexa
# Created: [DATE]
# These virtual switches are exposed to Alexa

- switch:
    - name: "STYRBAR [NAME] Up Short"
      unique_id: styrbar_[NAME]_up_short_switch
      state: "{{ is_state('input_boolean.styrbar_[NAME]_up_short_status', 'on') }}"
      turn_on:
        service: input_boolean.turn_on
        target:
          entity_id: input_boolean.styrbar_[NAME]_up_short_status
      turn_off:
        service: input_boolean.turn_off
        target:
          entity_id: input_boolean.styrbar_[NAME]_up_short_status
      icon: mdi:arrow-up-circle

    - name: "STYRBAR [NAME] Up Long"
      unique_id: styrbar_[NAME]_up_long_switch
      state: "{{ is_state('input_boolean.styrbar_[NAME]_up_long_status', 'on') }}"
      turn_on:
        service: input_boolean.turn_on
        target:
          entity_id: input_boolean.styrbar_[NAME]_up_long_status
      turn_off:
        service: input_boolean.turn_off
        target:
          entity_id: input_boolean.styrbar_[NAME]_up_long_status
      icon: mdi:arrow-up-circle-outline

    - name: "STYRBAR [NAME] Down Short"
      unique_id: styrbar_[NAME]_down_short_switch
      state: "{{ is_state('input_boolean.styrbar_[NAME]_down_short_status', 'on') }}"
      turn_on:
        service: input_boolean.turn_on
        target:
          entity_id: input_boolean.styrbar_[NAME]_down_short_status
      turn_off:
        service: input_boolean.turn_off
        target:
          entity_id: input_boolean.styrbar_[NAME]_down_short_status
      icon: mdi:arrow-down-circle

    - name: "STYRBAR [NAME] Down Long"
      unique_id: styrbar_[NAME]_down_long_switch
      state: "{{ is_state('input_boolean.styrbar_[NAME]_down_long_status', 'on') }}"
      turn_on:
        service: input_boolean.turn_on
        target:
          entity_id: input_boolean.styrbar_[NAME]_down_long_status
      turn_off:
        service: input_boolean.turn_off
        target:
          entity_id: input_boolean.styrbar_[NAME]_down_long_status
      icon: mdi:arrow-down-circle-outline

    - name: "STYRBAR [NAME] Left Short"
      unique_id: styrbar_[NAME]_left_short_switch
      state: "{{ is_state('input_boolean.styrbar_[NAME]_left_short_status', 'on') }}"
      turn_on:
        service: input_boolean.turn_on
        target:
          entity_id: input_boolean.styrbar_[NAME]_left_short_status
      turn_off:
        service: input_boolean.turn_off
        target:
          entity_id: input_boolean.styrbar_[NAME]_left_short_status
      icon: mdi:arrow-left-circle

    - name: "STYRBAR [NAME] Left Long"
      unique_id: styrbar_[NAME]_left_long_switch
      state: "{{ is_state('input_boolean.styrbar_[NAME]_left_long_status', 'on') }}"
      turn_on:
        service: input_boolean.turn_on
        target:
          entity_id: input_boolean.styrbar_[NAME]_left_long_status
      turn_off:
        service: input_boolean.turn_off
        target:
          entity_id: input_boolean.styrbar_[NAME]_left_long_status
      icon: mdi:arrow-left-circle-outline

    - name: "STYRBAR [NAME] Right Short"
      unique_id: styrbar_[NAME]_right_short_switch
      state: "{{ is_state('input_boolean.styrbar_[NAME]_right_short_status', 'on') }}"
      turn_on:
        service: input_boolean.turn_on
        target:
          entity_id: input_boolean.styrbar_[NAME]_right_short_status
      turn_off:
        service: input_boolean.turn_off
        target:
          entity_id: input_boolean.styrbar_[NAME]_right_short_status
      icon: mdi:arrow-right-circle

    - name: "STYRBAR [NAME] Right Long"
      unique_id: styrbar_[NAME]_right_long_switch
      state: "{{ is_state('input_boolean.styrbar_[NAME]_right_long_status', 'on') }}"
      turn_on:
        service: input_boolean.turn_on
        target:
          entity_id: input_boolean.styrbar_[NAME]_right_long_status
      turn_off:
        service: input_boolean.turn_off
        target:
          entity_id: input_boolean.styrbar_[NAME]_right_long_status
      icon: mdi:arrow-right-circle-outline
```

### Why Template Switches?

**Question:** Why not expose input_boolean directly to Alexa?

**Answer:** 
- Alexa Smart Home API works better with `switch` entities
- More reliable state change notifications
- Better device categorization in Alexa app
- Template switches act as Alexa-friendly wrapper

---

## Step 5: Create Automations

**Automation Purpose:** Detect button press → Turn ON status → Wait 2 seconds → Turn OFF status

**No custom actions needed** - Alexa routines will handle device control.

### 🔷 ZHA Automation Template

**File:** `automations/styrbar_[NAME].yaml`

```yaml
# STYRBAR [NAME] - ZHA Automations
# Created: [DATE]
# Location: [ROOM]
# Device ID: [DEVICE_ID]
# Stack: ZHA

# ===== UP BUTTON =====
- alias: "STYRBAR [NAME] - Up Short"
  id: styrbar_[NAME]_up_short
  mode: single
  trigger:
    - platform: event
      event_type: zha_event
      event_data:
        device_id: "[DEVICE_ID]"
        command: "on"  # ← YOUR captured command
  condition:
    - condition: state
      entity_id: input_boolean.styrbar_[NAME]_enabled
      state: "on"
  action:
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.styrbar_[NAME]_up_short_status
    - delay: 2
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.styrbar_[NAME]_up_short_status

- alias: "STYRBAR [NAME] - Up Long"
  id: styrbar_[NAME]_up_long
  mode: single
  trigger:
    - platform: event
      event_type: zha_event
      event_data:
        device_id: "[DEVICE_ID]"
        command: "move_with_on_off"  # ← YOUR captured command
        args: [0, 83]  # ← YOUR captured args
  condition:
    - condition: state
      entity_id: input_boolean.styrbar_[NAME]_enabled
      state: "on"
  action:
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.styrbar_[NAME]_up_long_status
    - delay: 2
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.styrbar_[NAME]_up_long_status

# ===== DOWN BUTTON =====
- alias: "STYRBAR [NAME] - Down Short"
  id: styrbar_[NAME]_down_short
  mode: single
  trigger:
    - platform: event
      event_type: zha_event
      event_data:
        device_id: "[DEVICE_ID]"
        command: "off"
  condition:
    - condition: state
      entity_id: input_boolean.styrbar_[NAME]_enabled
      state: "on"
  action:
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.styrbar_[NAME]_down_short_status
    - delay: 2
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.styrbar_[NAME]_down_short_status

- alias: "STYRBAR [NAME] - Down Long"
  id: styrbar_[NAME]_down_long
  mode: single
  trigger:
    - platform: event
      event_type: zha_event
      event_data:
        device_id: "[DEVICE_ID]"
        command: "move"
        args: [1, 83, 0, 0]
  condition:
    - condition: state
      entity_id: input_boolean.styrbar_[NAME]_enabled
      state: "on"
  action:
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.styrbar_[NAME]_down_long_status
    - delay: 2
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.styrbar_[NAME]_down_long_status

# ===== LEFT BUTTON =====
- alias: "STYRBAR [NAME] - Left Short"
  id: styrbar_[NAME]_left_short
  mode: single
  trigger:
    - platform: event
      event_type: zha_event
      event_data:
        device_id: "[DEVICE_ID]"
        command: "press"
        args: [257, 13, 0]
  condition:
    - condition: state
      entity_id: input_boolean.styrbar_[NAME]_enabled
      state: "on"
  action:
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.styrbar_[NAME]_left_short_status
    - delay: 2
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.styrbar_[NAME]_left_short_status

- alias: "STYRBAR [NAME] - Left Long"
  id: styrbar_[NAME]_left_long
  mode: single
  trigger:
    - platform: event
      event_type: zha_event
      event_data:
        device_id: "[DEVICE_ID]"
        command: "hold"
        args: [3329, 0]
  condition:
    - condition: state
      entity_id: input_boolean.styrbar_[NAME]_enabled
      state: "on"
  action:
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.styrbar_[NAME]_left_long_status
    - delay: 2
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.styrbar_[NAME]_left_long_status

# ===== RIGHT BUTTON =====
- alias: "STYRBAR [NAME] - Right Short"
  id: styrbar_[NAME]_right_short
  mode: single
  trigger:
    - platform: event
      event_type: zha_event
      event_data:
        device_id: "[DEVICE_ID]"
        command: "press"
        args: [256, 13, 0]
  condition:
    - condition: state
      entity_id: input_boolean.styrbar_[NAME]_enabled
      state: "on"
  action:
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.styrbar_[NAME]_right_short_status
    - delay: 2
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.styrbar_[NAME]_right_short_status

- alias: "STYRBAR [NAME] - Right Long"
  id: styrbar_[NAME]_right_long
  mode: single
  trigger:
    - platform: event
      event_type: zha_event
      event_data:
        device_id: "[DEVICE_ID]"
        command: "hold"
        args: [3328, 0]
  condition:
    - condition: state
      entity_id: input_boolean.styrbar_[NAME]_enabled
      state: "on"
  action:
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.styrbar_[NAME]_right_long_status
    - delay: 2
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.styrbar_[NAME]_right_long_status
```

### 🟠 Z2M Automation Template

**File:** `automations/styrbar_[NAME].yaml`

```yaml
# STYRBAR [NAME] - Zigbee2MQTT Automations
# Created: [DATE]
# Location: [ROOM]
# Z2M Friendly Name: [FRIENDLY_NAME]
# Stack: Zigbee2MQTT

# ===== UP BUTTON =====
- alias: "STYRBAR [NAME] - Up Short"
  id: styrbar_[NAME]_up_short
  mode: single
  trigger:
    - platform: state
      entity_id: sensor.[friendly_name]_action
      to: "on"  # ← YOUR captured action
  condition:
    - condition: state
      entity_id: input_boolean.styrbar_[NAME]_enabled
      state: "on"
  action:
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.styrbar_[NAME]_up_short_status
    - delay: 2
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.styrbar_[NAME]_up_short_status

- alias: "STYRBAR [NAME] - Up Long"
  id: styrbar_[NAME]_up_long
  mode: single
  trigger:
    - platform: state
      entity_id: sensor.[friendly_name]_action
      to: "brightness_move_up"  # ← YOUR captured action
  condition:
    - condition: state
      entity_id: input_boolean.styrbar_[NAME]_enabled
      state: "on"
  action:
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.styrbar_[NAME]_up_long_status
    - delay: 2
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.styrbar_[NAME]_up_long_status

# ===== DOWN BUTTON =====
- alias: "STYRBAR [NAME] - Down Short"
  id: styrbar_[NAME]_down_short
  mode: single
  trigger:
    - platform: state
      entity_id: sensor.[friendly_name]_action
      to: "off"  # ← YOUR captured action
  condition:
    - condition: state
      entity_id: input_boolean.styrbar_[NAME]_enabled
      state: "on"
  action:
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.styrbar_[NAME]_down_short_status
    - delay: 2
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.styrbar_[NAME]_down_short_status

- alias: "STYRBAR [NAME] - Down Long"
  id: styrbar_[NAME]_down_long
  mode: single
  trigger:
    - platform: state
      entity_id: sensor.[friendly_name]_action
      to: "brightness_move_down"  # ← YOUR captured action
  condition:
    - condition: state
      entity_id: input_boolean.styrbar_[NAME]_enabled
      state: "on"
  action:
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.styrbar_[NAME]_down_long_status
    - delay: 2
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.styrbar_[NAME]_down_long_status

# ===== LEFT BUTTON =====
- alias: "STYRBAR [NAME] - Left Short"
  id: styrbar_[NAME]_left_short
  mode: single
  trigger:
    - platform: state
      entity_id: sensor.[friendly_name]_action
      to: "arrow_left_click"  # ← YOUR captured action
  condition:
    - condition: state
      entity_id: input_boolean.styrbar_[NAME]_enabled
      state: "on"
  action:
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.styrbar_[NAME]_left_short_status
    - delay: 2
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.styrbar_[NAME]_left_short_status

- alias: "STYRBAR [NAME] - Left Long"
  id: styrbar_[NAME]_left_long
  mode: single
  trigger:
    - platform: state
      entity_id: sensor.[friendly_name]_action
      to: "arrow_left_hold"  # ← YOUR captured action (may not exist)
  condition:
    - condition: state
      entity_id: input_boolean.styrbar_[NAME]_enabled
      state: "on"
  action:
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.styrbar_[NAME]_left_long_status
    - delay: 2
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.styrbar_[NAME]_left_long_status

# ===== RIGHT BUTTON =====
- alias: "STYRBAR [NAME] - Right Short"
  id: styrbar_[NAME]_right_short
  mode: single
  trigger:
    - platform: state
      entity_id: sensor.[friendly_name]_action
      to: "arrow_right_click"  # ← YOUR captured action
  condition:
    - condition: state
      entity_id: input_boolean.styrbar_[NAME]_enabled
      state: "on"
  action:
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.styrbar_[NAME]_right_short_status
    - delay: 2
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.styrbar_[NAME]_right_short_status

- alias: "STYRBAR [NAME] - Right Long"
  id: styrbar_[NAME]_right_long
  mode: single
  trigger:
    - platform: state
      entity_id: sensor.[friendly_name]_action
      to: "arrow_right_hold"  # ← YOUR captured action (may not exist)
  condition:
    - condition: state
      entity_id: input_boolean.styrbar_[NAME]_enabled
      state: "on"
  action:
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.styrbar_[NAME]_right_long_status
    - delay: 2
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.styrbar_[NAME]_right_long_status
```

### Important Notes

**⚠️ Replace placeholders:**
- `[NAME]` → your STYRBAR name (alpha, beta, charlie)
- `[DEVICE_ID]` → your ZHA device_id
- `[friendly_name]` → your Z2M friendly name
- Command/action values → YOUR captured values

**⚠️ No custom actions needed:**
- Automations just update status helpers
- Alexa routines will control your devices
- Keep automations simple

---

## Step 6: Update Configuration

### Add to configuration.yaml

Add these lines **once** (if not already present):

```yaml
# Input Boolean Helpers
input_boolean: !include_dir_merge_named helpers/

# Template Switches
template: !include_dir_merge_list templates/

# Automations
automation: !include_dir_list automations/

# Alexa Integration
alexa:
  smart_home:
    endpoint: https://api.amazonalexa.com/v3/events
    client_id: YOUR_CLIENT_ID
    client_secret: YOUR_CLIENT_SECRET
    filter:
      include_entity_globs:
        - switch.styrbar_*  # Expose all STYRBAR switches
```

### If Using Home Assistant Cloud

Replace the `alexa:` section with:

```yaml
# Alexa via Home Assistant Cloud (Nabu Casa)
# Configuration is automatic, no manual setup needed
# Just expose entities in Configuration → Home Assistant Cloud → Alexa
```

Then go to **Configuration** → **Home Assistant Cloud** → **Alexa** tab and enable entity exposure.

---

## Step 7: Load and Verify

### Load Configuration

**Option 1: Selective Reload (Recommended)**

1. Go to **Developer Tools** → **YAML**
2. Click **"Reload Input Booleans"**
3. Click **"Reload Template Entities"**
4. Click **"Reload Automations"**

**Option 2: Full Restart**

1. Go to **Settings** → **System** → **Restart**
2. Wait 2-3 minutes

### Verify Entities Created

1. **Developer Tools** → **States**
2. Search for `styrbar_alpha`
3. Confirm 17 entities exist:

**Input Booleans (9):**
- `input_boolean.styrbar_alpha_enabled` ✓
- `input_boolean.styrbar_alpha_up_short_status` ✓
- `input_boolean.styrbar_alpha_up_long_status` ✓
- `input_boolean.styrbar_alpha_down_short_status` ✓
- `input_boolean.styrbar_alpha_down_long_status` ✓
- `input_boolean.styrbar_alpha_left_short_status` ✓
- `input_boolean.styrbar_alpha_left_long_status` ✓
- `input_boolean.styrbar_alpha_right_short_status` ✓
- `input_boolean.styrbar_alpha_right_long_status` ✓

**Template Switches (8):**
- `switch.styrbar_alpha_up_short` ✓
- `switch.styrbar_alpha_up_long` ✓
- `switch.styrbar_alpha_down_short` ✓
- `switch.styrbar_alpha_down_long` ✓
- `switch.styrbar_alpha_left_short` ✓
- `switch.styrbar_alpha_left_long` ✓
- `switch.styrbar_alpha_right_short` ✓
- `switch.styrbar_alpha_right_long` ✓

### Test Button Press

1. Ensure `input_boolean.styrbar_alpha_enabled` is **ON**
2. Go to **Developer Tools** → **States**
3. Watch `input_boolean.styrbar_alpha_up_short_status`
4. Press UP SHORT button on physical STYRBAR
5. Status should turn **ON**, then **OFF** after 2 seconds
6. Template switch `switch.styrbar_alpha_up_short` should mirror this

### Test All Buttons

Repeat test for all 8 buttons. Each should:
- Turn status ON when pressed
- Turn status OFF after 2 seconds
- Template switch mirrors the state

---

## Step 8: Expose to Alexa

### Using Home Assistant Cloud (Easiest)

1. Go to **Configuration** → **Home Assistant Cloud**
2. Click **Amazon Alexa** tab
3. Scroll to **Entities**
4. Find your STYRBAR switches (search "STYRBAR")
5. Enable each switch you want in Alexa
6. Click **Sync Entities** button
7. Wait 30 seconds

### Using Manual Alexa Skill

Your configuration.yaml already includes:

```yaml
alexa:
  smart_home:
    filter:
      include_entity_globs:
        - switch.styrbar_*
```

This automatically exposes all STYRBAR switches.

### Discover Devices in Alexa

1. Open **Alexa app** on phone
2. Tap **Devices** (bottom menu)
3. Tap **+** (top right)
4. Select **Add Device**
5. Choose **Other** → **Discover Devices**
6. Wait 20-45 seconds
7. Your STYRBAR switches should appear

**Expected in Alexa:**
- "STYRBAR Alpha Up Short"
- "STYRBAR Alpha Down Short"
- "STYRBAR Alpha Left Short"
- "STYRBAR Alpha Right Short"
- (and 4 long press variants)

### Test Voice Control

Try: "Alexa, turn on STYRBAR Alpha Up Short"

The switch should turn ON briefly, then OFF after 2 seconds. This confirms the connection works.

---

## Step 9: Create Alexa Routines

**This is where the magic happens!**

Alexa routines detect button presses and control your devices.

### How Alexa Routines Work

```
STYRBAR Button Press
    ↓
HA Automation runs
    ↓
Switch turns ON
    ↓
Alexa detects switch ON
    ↓
Alexa routine triggers
    ↓
Alexa controls your devices
    ↓
(Switch auto turns OFF after 2s)
```

### Create a Routine

**Example: UP SHORT = Turn on living room lights**

1. Open **Alexa app**
2. Tap **More** (bottom menu)
3. Tap **Routines**
4. Tap **+** (create routine)

**When:**
1. Tap **When this happens**
2. Choose **Smart Home**
3. Find **"STYRBAR Alpha Up Short"**
4. Select **Turns On**
5. Tap **Next**

**Add action:**
1. Tap **Add action**
2. Choose **Smart Home**
3. Select your device (e.g., "Living Room Lights")
4. Choose **Power** → **On**
5. Tap **Next**

**Enabled:**
1. Toggle **Enabled** to ON
2. Tap **Save**

Done! Now pressing UP SHORT will turn on your lights.

### Routine Ideas

**UP Buttons:**
- UP Short: Turn lights ON / Brightness UP
- UP Long: Set lights to 100% / Turn all lights ON

**DOWN Buttons:**
- DOWN Short: Turn lights OFF / Brightness DOWN  
- DOWN Long: Turn all lights OFF

**LEFT/RIGHT Buttons:**
- LEFT Short: Previous scene / Color warmer
- RIGHT Short: Next scene / Color cooler
- LEFT Long: Movie mode / Night mode
- RIGHT Long: Bright mode / Party mode

### Advanced Routine Example

**Multiple actions in one routine:**

When: STYRBAR Alpha Left Short turns ON
Actions:
1. Turn on TV (smart plug)
2. Dim living room lights to 30%
3. Close blinds
4. Set temperature to 72°F

### Testing Routines

1. Create a simple routine (one action)
2. Press the button on STYRBAR
3. Check if action executes
4. If works: Add more actions
5. If doesn't work: Check troubleshooting section

### Routine Best Practices

**Do:**
- Start with simple routines (one action)
- Test each button individually
- Use descriptive names
- Group related actions

**Don't:**
- Create routines that take >5 seconds
- Use same button for conflicting actions
- Forget to enable routines after creating

---

## Troubleshooting

### Problem: Button Press Does Nothing

**🔷 ZHA Diagnosis:**

1. **Developer Tools** → **Events** → Listen to `zha_event`
2. Press button
3. Check if event appears
4. If NO event: STYRBAR not paired or battery dead
5. If event appears: Check device_id matches automation

**🟠 Z2M Diagnosis:**

1. **Developer Tools** → **States** → Find `sensor.[styrbar]_action`
2. Press button
3. Watch sensor value change
4. If NO change: Check Z2M connection
5. If changes: Check action name matches automation

**Solutions:**
- Verify `styrbar_alpha_enabled` is ON
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
      entity_id: input_boolean.styrbar_alpha_up_short_status
```

2. Test manually:
   - Developer Tools → Services
   - Call `input_boolean.turn_on`
   - Target: `input_boolean.styrbar_alpha_up_short_status`
   - If this works: Automation issue
   - If doesn't work: Helper not created properly

### Problem: Status Stays ON Forever

**Symptoms:**
- Status turns ON
- Never turns OFF

**Solutions:**
1. Check automation has delay and turn_off:
```yaml
action:
  - service: input_boolean.turn_on
    target:
      entity_id: input_boolean.styrbar_alpha_up_short_status
  - delay: 2  # ← This line needed
  - service: input_boolean.turn_off  # ← This line needed
    target:
      entity_id: input_boolean.styrbar_alpha_up_short_status
```

2. Check automation `mode: single` (should be there)

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
2. Verify `include_entity_globs: - switch.styrbar_*`
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
   - Verify it's set to "When STYRBAR X turns ON"
   - NOT "turns OFF"

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

### Problem: LEFT/RIGHT Long Press Doesn't Work

**Symptoms:**
- Short presses work
- Long presses don't register

**Explanation:**
LEFT/RIGHT long press support varies by firmware and Zigbee stack.

**Solutions:**
1. **Accept limitation:** Use only 6 buttons (skip long press)
2. **Update firmware:** Check for STYRBAR updates
3. **Alternative mapping:** Use UP/DOWN long press instead

### 🟠 Z2M Specific: Action Sensor Shows "Unknown"

**Symptoms:**
- `sensor.[styrbar]_action` shows "unknown"
- Never updates

**Solutions:**
1. Press any button once (sensor updates on first press)
2. Check Z2M web interface shows device as online
3. Re-pair STYRBAR with Z2M
4. Check MQTT broker is running

### 🟠 Z2M Specific: Wrong Action Names

**Symptoms:**
- Some buttons work, others don't
- Random behavior

**Solution:**
You must use YOUR captured action names, not examples from guide.

1. Developer Tools → States
2. Watch `sensor.[styrbar]_action`
3. Press EACH button
4. Record EXACT action name
5. Update automation files with YOUR values

---

## Adding More STYRBARs

### Quick Checklist

For STYRBAR name: `______` (beta/charlie/delta)

- [ ] Paired with ZHA or Z2M
- [ ] Found device_id (ZHA) or friendly name (Z2M)
- [ ] Captured all button commands/actions
- [ ] Created `helpers/styrbar_[NAME].yaml`
- [ ] Created `templates/styrbar_[NAME].yaml`
- [ ] Created `automations/styrbar_[NAME].yaml`
- [ ] Replaced all placeholders with actual values
- [ ] Reload: Input Booleans, Templates, Automations
- [ ] Verified 17 entities created
- [ ] Tested all 8 buttons
- [ ] Exposed switches to Alexa
- [ ] Discovered devices in Alexa app
- [ ] Created Alexa routines

---

## Naming Consistency Rules

**Choose ONE spelling and use it everywhere.**

**Bad example:**
```
helpers/styrbar_jupiter.yaml     ← "jupiter"
automations/styrbar_jupitar.yaml  ← "jupitar" (WRONG!)
Entity IDs won't match! Nothing will work.
```

**Good example:**
```
helpers/styrbar_jupiter.yaml
automations/styrbar_jupiter.yaml
All entity IDs use "jupiter" consistently ✓
```

**Tips:**
- Use lowercase for names (alpha, beta, charlie)
- Use find & replace, not manual typing
- Double-check entity IDs match across files

---

## FAQ

**Q: Can I control HA devices instead of Alexa devices?**  
A: Yes, but that's not what this guide is optimized for. Add custom actions to automation files if needed.

**Q: Can I use both ZHA and Z2M?**  
A: No, you must choose one. They conflict.

**Q: How many STYRBARs can I add?**  
A: As many as you want. Each adds 17 entities.

**Q: Do I need Home Assistant Cloud?**  
A: No, but it's the easiest way to connect Alexa. Manual skill setup also works.

**Q: Can I customize switch names in Alexa?**  
A: Yes, in Alexa app: Devices → Your switch → Edit Name

**Q: Will this work with Google Home?**  
A: The HA setup is the same. Google Home routines work similarly to Alexa.

**Q: What if my STYRBAR has different buttons?**  
A: Follow the same process, just capture YOUR button events.

**Q: LEFT/RIGHT long press not working?**  
A: Common issue. Use only the 6 working buttons or update firmware.

---

## Summary

### What You Created

**Per STYRBAR:**
- 1 master enabled toggle
- 8 status helpers
- 8 template switches
- 8 automations

**Total:** 17 entities + 8 automations per STYRBAR

### How It Works

1. Press physical button
2. HA automation updates status helper
3. Template switch mirrors status
4. Alexa detects switch state change
5. Alexa routine executes
6. Your device is controlled
7. Status auto-resets after 2 seconds

### Key Benefits

✅ Physical buttons for Alexa  
✅ No voice commands needed  
✅ Works with any Alexa-compatible device  
✅ Flexible routine programming  
✅ Easy to add more STYRBARs  
✅ Master enable/disable per STYRBAR  

---

## Credits

**Created by:** Home Assistant Community  
**Version:** 4.0 (Alexa-Optimized)  
**License:** MIT  
**Support:** Home Assistant Forums

---

**END OF RUNBOOK**

Enjoy your smart button setup! 🎉
