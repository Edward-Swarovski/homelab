# RUNBOOK – IKEA STYRBAR → Home Assistant → Alexa Plug (Virtual Switch)

## Goal
Use an **IKEA STYRBAR Zigbee remote** to control an **Alexa smart plug** via **Home Assistant**, using a **Virtual Switch** and **Alexa Proactive Events**.

---

## Architecture Overview

IKEA STYRBAR  
→ Zigbee2MQTT  
→ Home Assistant Automation  
→ Virtual Switch (`switch.jupitar_virtual_switch`)  
→ Alexa Proactive Events  
→ Alexa Routine  
→ Physical Plug (`Jupitar`)

---

## Prerequisites

- Home Assistant running
- Zigbee2MQTT configured and connected
- IKEA STYRBAR paired to Zigbee2MQTT
- Alexa smart plug already added to Alexa app
- AWS account + Alexa Smart Home Skill (DIY)
- Lambda already connected to the skill
- Home Assistant externally accessible via HTTPS

---

## Step 1 – Create Virtual Switch in Home Assistant

### 1.1 Create Helper
Home Assistant → **Settings → Devices & Services → Helpers**

Create:
- Type: **Toggle**
- Name: `Jupitar Virtual Switch`
- Entity ID:
  ```
  input_boolean.jupitar_virtual_switch
  ```

---

### 1.2 Create Template Switch (modern syntax)

Add to `configuration.yaml`:

```yaml
template:
  - switch:
      - name: "Jupitar Virtual Switch"
        unique_id: jupitar_virtual_switch
        state: "{{ is_state('input_boolean.jupitar_virtual_switch', 'on') }}"
        turn_on:
          - service: input_boolean.turn_on
            target:
              entity_id: input_boolean.jupitar_virtual_switch
        turn_off:
          - service: input_boolean.turn_off
            target:
              entity_id: input_boolean.jupitar_virtual_switch
```

Restart Home Assistant (**full restart**).

---

## Step 2 – Automation: IKEA Remote → Virtual Switch

### Automation YAML

```yaml
alias: Jupitar Remote → Jupitar Virtual Switch
mode: single

trigger:
  - platform: device
    domain: mqtt
    device_id: YOUR_STYRBAR_DEVICE_ID
    type: action
    subtype: "on"
    id: on_action

  - platform: device
    domain: mqtt
    device_id: YOUR_STYRBAR_DEVICE_ID
    type: action
    subtype: "off"
    id: off_action

condition: []

action:
  - choose:
      - conditions:
          - condition: trigger
            id: on_action
        sequence:
          - service: switch.turn_on
            target:
              entity_id: switch.jupitar_virtual_switch

      - conditions:
          - condition: trigger
            id: off_action
        sequence:
          - service: switch.turn_off
            target:
              entity_id: switch.jupitar_virtual_switch
```

> Note: Trigger IDs (`on_action`, `off_action`) are **local to this automation only**.

---

## Step 3 – Expose Virtual Switch to Alexa

Add to `configuration.yaml`:

```yaml
alexa:
  smart_home:
    filter:
      include_entities:
        - switch.jupitar_virtual_switch
```

Restart Home Assistant.

---

## Step 4 – Enable Alexa Proactive Events (Required)

### 4.1 Alexa Developer Console
- Open: https://developer.amazon.com/alexa/console/ask
- Open your **Smart Home Skill**
- Go to **Build → Permissions**
- Enable:
  ```
  Send Alexa Events (alexa::async_event:write)
  ```
- Copy:
  - Client ID
  - Client Secret

---

### 4.2 Home Assistant Proactive Events Config

```yaml
alexa:
  smart_home:
    endpoint: https://api.amazonalexa.com/v3/events
    client_id: YOUR_SKILL_MESSAGING_CLIENT_ID
    client_secret: YOUR_SKILL_MESSAGING_CLIENT_SECRET
    filter:
      include_entities:
        - switch.jupitar_virtual_switch
```

Restart Home Assistant.

---

### 4.3 Re-link Skill
Alexa App:
- Skills & Games
- Disable the Home Assistant skill
- Wait 30 seconds
- Enable again

---

## Step 5 – Alexa Routine (Virtual → Physical)

Create **two routines** in Alexa:

### Routine ON
- When: `Jupitar Virtual Switch` → ON
- Action: Turn ON → Physical plug `Jupitar`

### Routine OFF
- When: `Jupitar Virtual Switch` → OFF
- Action: Turn OFF → Physical plug `Jupitar`

---

## Verification Checklist

- Press IKEA remote ON → HA virtual switch turns ON
- Alexa app shows virtual switch state update automatically
- Alexa routine fires
- Physical plug turns ON/OFF

---

## Security Cleanup (After Testing)

AWS Lambda:
- Set `DEBUG = False`
- Remove `LONG_LIVED_ACCESS_TOKEN`

---

## Notes / Best Practices

- One automation per physical remote
- Trigger IDs only need to be unique per automation
- Always use Proactive Events for Alexa routines based on HA state
- Avoid exposing helpers directly to Alexa

---

## END OF RUNBOOK

