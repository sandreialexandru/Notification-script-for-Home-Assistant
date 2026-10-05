# 🔔 Actionable Notifications for Home Assistant (v3)

A Home Assistant **script blueprint** for actionable notifications in the [Home Assistant Companion](https://companion.home-assistant.io/) app on **Android and iOS**. It can notify several phones at once and has a single urgency selector, quiet hours, a presence filter, text-to-speech and image attachments.

> [!NOTE]
> ### 🙏 Credit
> This blueprint is a **modified version** of
> **[🔔 Notifications v2.0.2](https://github.com/samuelthng/t-house-blueprints/blob/main/notifications.yaml)**
> by **[@samuelthng](https://github.com/samuelthng)**, from the
> [t-house-blueprints](https://github.com/samuelthng/t-house-blueprints) repository.
>
> The idea, base structure and most of the logic are the original author's work: actionable buttons, timeout handling, camera snapshots and the Android/iOS differences. See also his [Home Assistant Community thread](https://community.home-assistant.io/t/notifications-actionable-mobile-notifications-script-with-optional-timeout-feature-and-camera-snapshots-works-with-ios-android/551552).
> If you find this useful, please give his project a ⭐ too.

[![Import blueprint into Home Assistant](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fsandreialexandru%2FNotification-script-for-Home-Assistant%2Fblob%2Fmain%2Fnotifications_v3.yaml)

---

## Contents

- [What's new compared to the original](#-whats-new-compared-to-the-original)
- [Requirements](#-requirements)
- [Installation](#-installation)
- [How it works](#-how-it-works)
- [Settings by section](#️-settings-by-section)
- [Usage examples](#-usage-examples)
- [Script response](#-script-response)
- [Known limitations](#️-known-limitations)
- [Troubleshooting](#-troubleshooting)
- [Changelog](#-changelog)

---

## ✨ What's new compared to the original

| | Original v2.0.2 | v3 |
|---|---|---|
| Devices | one | **one or more**, Android and iOS mixed |
| Multi-device response | — | first answer wins, and the notification **is updated** on the other devices to show who answered |
| Urgency | separate Android (channel + importance) and iOS settings | **one selector**: Info / Normal / Important / Critical |
| Android channels | a single `General` channel, so importance changes were ignored | **one channel per level**, with `alarm_stream` for Critical |
| Quiet hours | — | ✅ |
| Presence filter | — | ✅ (home / away) |
| Android TTS | — | ✅ |
| Image | camera only | camera **or URL** |
| Android options | basic | + countdown, `alert_once`, `sticky`, vibration, LED |
| iOS sound | default only | **custom sound** + critical volume |
| UI | one long list | **collapsible sections**, descriptions with concrete examples |
| Run mode | `restart`, so a new notification cancelled the previous one | `parallel`, so each notification listens independently |
| Syntax | `service:` / `platform:` | `action:` / `trigger:` (current syntax) |

Also fixed:
- button 2 and button 3 titles could not be overridden through `fields`;
- the button 3 field showed button 2's value as its example;
- the button 2 and 3 descriptions were copied from button 1;
- the script returned no result on timeout when timeout actions were disabled.

---

## 📋 Requirements

- **Home Assistant 2024.10** or newer
- The **Home Assistant Companion** app on every phone or tablet ([Android](https://play.google.com/store/apps/details?id=io.homeassistant.companion.android) / [iOS](https://apps.apple.com/app/home-assistant/id1099568401)), with notifications allowed
- To show *who* answered, each phone must be linked to a **person** in *Settings → People*

---

## 📥 Installation

**Option 1: import button.** Click the **Import blueprint** button at the top of this page.

**Option 2: manual.**
1. Copy `notifications_v3.yaml` to `config/blueprints/script/<any_folder>/`.
2. Go to *Developer Tools → YAML → Scripts* and click **Reload**, or restart Home Assistant.

**Create a script from the blueprint:**
1. Go to *Settings → Automations & Scenes → Blueprints*.
2. Pick **🔔 Notifications (Version 3.0)** and click **Create script**.
3. Fill in the settings and save. The name you choose becomes the `script.<name>` entity.

> [!TIP]
> Create one script per notification type (intercom, door left open, washing machine…). Each one gets its own buttons, urgency and sound.

---

## 🧠 How it works

```
Script starts
   │
   ├─ resolve urgency → apply quiet hours (may demote or skip)
   ├─ resolve devices → apply presence filter
   ├─ send the notification to each device (Android/iOS payload built automatically)
   │     └─ optional: TTS on Android
   │
   ├─ wait for: button press │ swipe-away │ timeout
   │
   ├─ multiple devices and someone answered?
   │     └─ on the other phones the notification becomes "✅ <button> — <person>, <time>"
   │
   ├─ run the pressed button's actions (or the timeout actions)
   └─ return the result (see "Script response")
```

### Multiple devices

Every selected device receives the **same** notification. When someone presses a button:
- the button's actions run **exactly once**;
- on the other devices the notification is **not removed**. It is replaced with something like `✅ Open door — Ana, 23:15`, without buttons, so nobody presses one for nothing;
- the devices of the person who answered don't receive the update.

### Urgency levels

| Level | Android | iOS | Use it for |
|---|---|---|---|
| 🔕 **Info** | channel `<prefix> Info`, silent | `passive`: no sound, doesn't wake the screen | reports, "washing machine finished" |
| 🔔 **Normal** | channel `<prefix> Normal`, normal sound | `active` | most notifications |
| ⚠️ **Important** | channel `<prefix> Important`, heads-up banner | `time-sensitive`: breaks through Focus | intercom, door left open |
| 🚨 **Critical** | channel `alarm_stream`: alarm sound, bypasses Do Not Disturb and silent mode | `critical`: bypasses Mute and Focus | water leak, smoke, alarm |

> [!IMPORTANT]
> On **Android**, a channel's sound, vibration and LED are fixed **the first time** a notification is sent on it. After that you can change them only in the phone's settings (*Settings → Apps → Home Assistant → Notifications*) or by setting a different **channel prefix**, which creates new channels.
>
> On **iOS**, the Critical level needs the **Critical Alerts** permission. The app asks for it the first time it receives a critical notification.

---

## ⚙️ Settings by section

<details>
<summary><b>📲 Devices and presence</b></summary>

| Setting | What it does |
|---|---|
| **Devices to notify** | One or more devices with the Companion app. TVs don't appear here; they use a different kind of notification. |
| **Presence filter** | *Everyone* / *Only those at home* / *Only those away*. Presence comes from the `person.*` entity, or from the phone's `device_tracker` as a fallback. A device without a tracker, such as a wall tablet, is always notified. |
| **Critical bypasses the filter** | On by default, so a critical alert always reaches everyone. |
</details>

<details>
<summary><b>💬 Content</b></summary>

| Setting | What it does |
|---|---|
| **Title / Subtitle / Message** | The notification text. On Android the message accepts simple HTML (`<b>`, `<i>`, `<font color>`). On iOS the tags show up as plain text. |
| **Notification link** | What opens when the notification is tapped: `/lovelace/intercom`, `entityId:lock.front_door` (Android), `app://com.spotify.music` (Android), `https://…` |
| **Icon / color** (Android) | The status-bar icon, for example `mdi:doorbell`, and its color. |
</details>

<details>
<summary><b>🖼️ Image</b></summary>

| Type | What it does |
|---|---|
| **None** | — |
| **Camera** | A snapshot taken when the notification is sent. On iOS, long-pressing it shows the live stream. |
| **URL** | `/local/…` (from `config/www`), `/media/local/…` or `https://…`. Relative links only load when the phone can reach Home Assistant. |
</details>

<details>
<summary><b>🚦 Urgency and quiet hours</b></summary>

| Setting | What it does |
|---|---|
| **Urgency level** | See the table above. |
| **Quiet hours** | During the chosen window, which may cross midnight (e.g. 22:30 → 07:00), **Info and Normal** notifications are either sent silently (as Info, without TTS) or not sent at all. Important and Critical always go through. |
</details>

<details>
<summary><b>1️⃣ 2️⃣ 3️⃣ Buttons</b></summary>

| Setting | What it does |
|---|---|
| **Show / Text** | Up to 3 buttons. Keep the labels short, because Android truncates them. |
| **Button mode** (Android) | **Actions** runs actions in Home Assistant. **Link** opens a link. On Android a button can't do both (an app limitation). iOS ignores this setting and does both. |
| **Actions** | What runs when the button is pressed. For a button that only dismisses the notification, leave it empty. |
| **Link** | `/lovelace/…`, `https://…`; Android: `app://<package>`, `entityId:<entity>`, `deep-link://<link>`, `intent://…`, `settings://notification_history`; iOS: `tel:`, `mailto:` or any app URL scheme. |
| **Icon / Destructive / Require unlock** (iOS) | An SF Symbol name (e.g. `door.left.hand.open`), red text for dangerous actions, and Face ID / passcode before the action runs. Requiring unlock is recommended for door actions. |
</details>

<details>
<summary><b>⌛️ Timeout</b></summary>

| Setting | What it does |
|---|---|
| **Enable timeout** | How long the script listens for an answer. When disabled, it listens until someone answers. |
| **Timeout actions** | What happens when nobody answers, for example sending a Critical notification. |
| **Swipe-away = timeout** (Android) | A swipe on any device runs the timeout actions immediately. |
| **Clear on timeout** | Removes the notification from all devices when it expires. |
</details>

<details>
<summary><b>🤖 Advanced Android</b></summary>

| Setting | What it does |
|---|---|
| **Channel prefix** | Channels are created as `<prefix> Info / Normal / Important`. A separate prefix per script lets each script have its own sound. |
| **Fast delivery** | Sends with `priority: high` and `ttl: 0`, so battery saving doesn't delay the notification. It doesn't change the sound. |
| **Alert once** | Updates to the same notification don't sound again. |
| **Persistent** | The notification can't be swiped away. |
| **Sticky** | The notification stays after it is tapped. |
| **Countdown** | Shows the time left until the timeout. |
| **Vibration pattern** | For example `0, 500, 200, 500` (milliseconds of pause, vibrate, pause, vibrate…). |
| **LED color** | Only on phones that still have a notification LED. |
| **Lock screen** | Public / Private / Secret. |
| **Android Auto** | Also shows the notification in the car. |
| **TTS** | The phone reads the message aloud. *Media* can't be heard in silent mode; *Alarm* can; *Alarm (max)* temporarily raises the alarm volume to maximum. TTS never plays at the Info level. |
</details>

<details>
<summary><b> iOS</b></summary>

| Setting | What it does |
|---|---|
| **Sound** | The name of a sound in the app (*Settings → Companion App → Notifications → Sounds*), for example `US-EN-Alexa-Doorbell.wav`. `none` means no sound. |
| **Critical volume** | 0–1. Plays even when the phone is muted. |
</details>

<details>
<summary><b>⚙️ Misc</b></summary>

| Setting | What it does |
|---|---|
| **Tag** | Leave it empty (recommended) to get a unique tag per run. A fixed tag such as `intercom` makes each new notification **replace** the previous one. |
| **Group** | Groups notifications visually. Critical notifications on iOS are never grouped. |
</details>

---

## 🧪 Usage examples

### 1. Intercom: open the door from any phone

Create a script from the blueprint with:
- **Devices:** your phone and your partner's phone
- **Urgency:** ⚠️ Important
- **Button 1:** `Open` → `switch.turn_on` on the door relay, with *Require unlock* enabled on iOS
- **Button 2:** `Ignore` → no actions
- **Image:** the entrance camera
- **Timeout:** 2 minutes

Then call it from an automation:

```yaml
triggers:
  - trigger: state
    entity_id: binary_sensor.intercom_ringing
    to: "on"
actions:
  - action: script.intercom_notification
```

### 2. Override settings when calling the script (`fields`)

The same script can be called with a different text, urgency or set of devices:

```yaml
- action: script.general_notification
  data:
    field_title: "💧 Water leak"
    field_message: "Water detected in the bathroom!"
    field_urgency: critical
    field_tts_text: "Warning, water in the bathroom"
```

Available fields: `field_notify_devices`, `field_urgency`, `field_title`, `field_subtitle`, `field_message`, `field_notification_link`, `field_attachment_type`, `field_attachment_camera_entity`, `field_attachment_image_url`, `field_option_{one,two,three}_enabled`, `field_option_{one,two,three}_text`, `field_option_{one,two,three}_mode`, `field_option_{one,two,three}_uri`, `field_enable_timeout`, `field_timeout`, `field_run_timeout_actions`, `field_tts_text`.

> Button actions **cannot** be overridden through `fields`, because of a blueprint limitation. To run different actions, use the script response (example 3) or create another script.

### 3. Use the response in an automation

```yaml
- action: script.general_notification
  data:
    field_message: "The bedroom window is open. Close it?"
    field_option_one_text: "Close"
    field_option_two_text: "Leave it"
  response_variable: answer

- if: "{{ answer.result == 'option_one' }}"
  then:
    - action: cover.close_cover
      target:
        entity_id: cover.bedroom_window
    - action: logbook.log
      data:
        name: Window
        message: "Closed at {{ answer.responded_by }}'s request"
```

> [!WARNING]
> To receive the response, call the script with `action: script.<name>`, not `script.turn_on`. The automation waits until someone answers or the timeout expires.

---

## 📤 Script response

```yaml
result: option_one          # see the table below
responded_by: Ana           # name of the person who answered ("someone" if unknown)
responder_person: person.ana
level: important            # effective urgency, after quiet hours
quiet_hours: false          # true if quiet hours were applied
devices: [Pixel 8, iPhone Ana]
```

| `result` | Meaning |
|---|---|
| `option_one` / `option_two` / `option_three` | Someone pressed that button |
| `timeout` | Nobody answered in time |
| `notification_cleared` | The notification was swiped away, with swipe-away = timeout enabled |
| `cleared_ignored` | Swiped away, but swipe-away = timeout is disabled, so nothing ran |
| `no_response` | No buttons and no timeout, so no answer was expected |
| `not_sent` | Nothing was sent; `reason` is `quiet_hours` or `no_devices` |

---

## ⚠️ Known limitations

- **Restarting Home Assistant** stops all listeners. Buttons on notifications sent before the restart do nothing.
- **Link buttons on Android** don't send an event to Home Assistant, so they **don't stop the timeout**.
- **Renaming a phone:** the `notify.mobile_app_<name>` service is derived from the device name. If you rename the phone in the app, check in *Developer Tools → Actions* that the service still exists.
- **TVs** (LG webOS, Android TV) are not supported. They have their own notify services, without buttons.
- **HTML in the message** shows as plain text on iOS.
- **The responder's name** only appears if the phone's Home Assistant user is linked to a `person.*` entity.

---

## 🛠 Troubleshooting

**Nothing happens when I press a button:**
1. In *Developer Tools → Events*, listen to `mobile_app_notification_action`.
2. Run the script and press a button on the phone.
3. If no event appears, the problem is the network or the app, not the script. See the [Companion FAQ](https://companion.home-assistant.io/docs/troubleshooting/faqs).

**The sound or importance doesn't change on Android:** the channel already exists with the old settings. Change the **channel prefix**, or edit the channel in the phone's settings.

**The notification never arrives:** open *Settings → Automations & Scenes → Scripts → (your script) → Traces*. The *Notification* step shows which service was called and any error.

---

## 📝 Changelog

### v3.0
- Notifications to multiple devices; the first answer wins and the notification is updated on the other devices
- Unified Android/iOS urgency selector, with one Android channel per level
- Quiet hours, presence filter, Android TTS, image from URL
- Advanced Android options: countdown, `alert_once`, `sticky`, vibration, LED
- Custom sound and critical volume on iOS
- Collapsible sections, descriptions with examples, `action:` / `trigger:` syntax, `mode: parallel`
- Script response now includes `responded_by`, `level` and `quiet_hours`
- Fixes over v2.0.2: overriding button 2/3 titles, wrong example on the button 3 field, copied descriptions, missing result on timeout

### v2.0.2 and earlier
See [@samuelthng's original changelog](https://github.com/samuelthng/t-house-blueprints/blob/main/notifications.yaml) and the [community thread](https://community.home-assistant.io/t/notifications-actionable-mobile-notifications-script-with-optional-timeout-feature-and-camera-snapshots-works-with-ios-android/551552).

---

## 🙏 Acknowledgements

- **[@samuelthng](https://github.com/samuelthng)**, author of the original [Notifications](https://github.com/samuelthng/t-house-blueprints/blob/main/notifications.yaml) blueprint, on which this version is entirely based
- Contributors to the original project, including [@HNKNTA](https://github.com/HNKNTA) (notification link, v2.0.2)
- The [Home Assistant Companion](https://companion.home-assistant.io/) team, for the notification documentation

All rights to the original code belong to its author.
