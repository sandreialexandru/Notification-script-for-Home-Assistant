# 🔔 Notificări acționabile pentru Home Assistant (v3)

Blueprint de **script** pentru Home Assistant care trimite notificări acționabile în aplicația [Home Assistant Companion](https://companion.home-assistant.io/) pe **Android și iOS**. Poți trimite pe mai multe telefoane deodată, ai un singur selector de urgență, ore de liniște, filtru după prezență, TTS și imagini atașate.

> [!NOTE]
> ### 🙏 Credit
> Acest blueprint este o **versiune modificată** a blueprint-ului
> **[🔔 Notifications v2.0.2](https://github.com/samuelthng/t-house-blueprints/blob/main/notifications.yaml)**
> creat de **[@samuelthng](https://github.com/samuelthng)** în repo-ul
> [t-house-blueprints](https://github.com/samuelthng/t-house-blueprints).
>
> Ideea, structura de bază și mare parte din logică (butoanele acționabile, timeout-ul, snapshot-urile de cameră, diferențele Android/iOS) îi aparțin autorului original. Proiectul lui are și o [discuție pe forumul Home Assistant](https://community.home-assistant.io/t/notifications-actionable-mobile-notifications-script-with-optional-timeout-feature-and-camera-snapshots-works-with-ios-android/551552).
> Dacă îți place, lasă-i o ⭐ și lui.

[![Importă blueprint-ul în Home Assistant](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FUTILIZATOR%2FREPO%2Fblob%2Fmain%2Fnotifications_v3.yaml)

<!-- Înlocuiește UTILIZATOR/REPO în linkul de mai sus cu calea repo-ului tău. -->

---

## Cuprins

- [Ce aduce nou față de original](#-ce-aduce-nou-față-de-original)
- [Cerințe](#-cerințe)
- [Instalare](#-instalare)
- [Cum funcționează](#-cum-funcționează)
- [Setări, pe secțiuni](#️-setări-pe-secțiuni)
- [Exemple de folosire](#-exemple-de-folosire)
- [Răspunsul scriptului](#-răspunsul-scriptului)
- [Limitări cunoscute](#️-limitări-cunoscute)
- [Depanare](#-depanare)
- [Changelog](#-changelog)

---

## ✨ Ce aduce nou față de original

| | Original v2.0.2 | v3 |
|---|---|---|
| Dispozitive | unul singur | **unul sau mai multe**, Android și iOS amestecate |
| Răspuns pe mai multe telefoane | — | primul răspuns câștigă, iar pe celelalte telefoane notificarea **se actualizează** cu cine a răspuns |
| Urgență | setări separate pentru Android (canal + importanță) și iOS | **un singur selector**: Info / Normal / Important / Critic |
| Canale Android | un singur canal `General`, așa că schimbarea importanței era ignorată | **câte un canal pentru fiecare nivel**, iar Critic folosește `alarm_stream` |
| Ore de liniște | — | ✅ |
| Filtru după prezență | — | ✅ (acasă / plecați) |
| TTS pe Android | — | ✅ |
| Imagine | doar cameră | cameră **sau URL** |
| Opțiuni Android | de bază | + numărătoare inversă, `alert_once`, `sticky`, vibrație, LED |
| Sunet iOS | implicit | **sunet personalizat** + volum la Critic |
| Interfață | listă lungă de setări | **secțiuni pliabile**, descrieri în română cu exemple |
| Mod de rulare | `restart`, așa că o notificare nouă o anula pe cea veche | `parallel`, fiecare notificare ascultă independent |
| Sintaxă | `service:` / `platform:` | `action:` / `trigger:` (sintaxa actuală) |

Reparat și:
- titlurile butoanelor 2 și 3 nu puteau fi suprascrise din `fields`;
- câmpul butonului 3 avea ca exemplu valoarea butonului 2;
- descrierile butoanelor 2 și 3 erau copiate de la butonul 1;
- scriptul nu returna niciun rezultat la timeout când acțiunile de timeout erau dezactivate.

---

## 📋 Cerințe

- **Home Assistant 2024.10** sau mai nou
- Aplicația **Home Assistant Companion** pe fiecare telefon sau tabletă ([Android](https://play.google.com/store/apps/details?id=io.homeassistant.companion.android) / [iOS](https://apps.apple.com/app/home-assistant/id1099568401)), cu notificările permise
- Pentru afișarea numelui celui care a răspuns: fiecare telefon trebuie legat de o **persoană** în *Settings → People*

---

## 📥 Instalare

**Varianta 1: butonul de import.** Apasă butonul **Importă blueprint-ul** de sus.

**Varianta 2: manual.**
1. Copiază `notifications_v3.yaml` în `config/blueprints/script/<folder_ales>/`.
2. *Developer Tools → YAML → Scripts* → **Reload**. Poți reporni și HA.

**Crearea unui script din blueprint:**
1. *Settings → Automations & Scenes → Blueprints*.
2. Alege **🔔 Notificări (v3.0…)** → **Create script**.
3. Completează setările și salvează. Numele ales devine entitatea `script.<nume>`.

> [!TIP]
> Fă câte un script pentru fiecare tip de notificare (interfon, ușă deschisă, mașina de spălat…). Fiecare script are butoanele, urgența și sunetul lui.

---

## 🧠 Cum funcționează

```
Scriptul pornește
   │
   ├─ calculează urgența → aplică orele de liniște (poate retrograda sau opri)
   ├─ alege dispozitivele → aplică filtrul de prezență
   ├─ trimite notificarea pe fiecare dispozitiv (format Android sau iOS, automat)
   │     └─ opțional: TTS pe Android
   │
   ├─ așteaptă: apăsare buton │ ștergere cu swipe │ timeout
   │
   ├─ mai multe dispozitive + cineva a răspuns?
   │     └─ pe celelalte telefoane notificarea devine „✅ <buton> — <persoană>, <ora>”
   │
   ├─ rulează acțiunile butonului apăsat (sau pe cele de timeout)
   └─ returnează rezultatul (vezi „Răspunsul scriptului”)
```

### Mai multe dispozitive

Toate dispozitivele primesc **aceeași** notificare. Când cineva apasă un buton:
- acțiunea butonului se execută **o singură dată**;
- pe celelalte telefoane notificarea **nu dispare**, ci se înlocuiește cu, de exemplu, `✅ Deschide ușa — Ana, 23:15`, fără butoane, ca nimeni să nu mai apese degeaba;
- telefoanele persoanei care a răspuns nu primesc actualizarea.

### Niveluri de urgență

| Nivel | Android | iOS | Folosește pentru |
|---|---|---|---|
| 🔕 **Info** | canal `<prefix> Info`, fără sunet | `passive`: nu sună, nu aprinde ecranul | rapoarte, „mașina de spălat a terminat” |
| 🔔 **Normal** | canal `<prefix> Normal`, sunet normal | `active` | majoritatea notificărilor |
| ⚠️ **Important** | canal `<prefix> Important`, banner peste ecran | `time-sensitive`: trece de Focus | interfon, ușă lăsată deschisă |
| 🚨 **Critic** | canal `alarm_stream`: sună ca alarma, trece de *Nu deranja* și de silențios | `critical`: trece de Mute și Focus | inundație, fum, alarmă |

> [!IMPORTANT]
> Pe **Android**, sunetul, vibrația și LED-ul unui canal se fixează **la prima notificare** trimisă pe acel canal. După aceea le poți schimba doar din telefon (*Setări → Aplicații → Home Assistant → Notificări*) sau dând alt **prefix de canal**, ca să se creeze canale noi.
>
> Pe **iOS**, nivelul Critic cere permisiunea **Critical Alerts**, pe care aplicația o solicită la prima notificare critică.

---

## ⚙️ Setări, pe secțiuni

<details>
<summary><b>📲 Dispozitive și prezență</b></summary>

| Setare | Ce face |
|---|---|
| **Dispozitive de notificat** | Unul sau mai multe dispozitive cu aplicația Companion. Televizoarele nu apar aici, pentru că au alt tip de notificare. |
| **Filtru după prezență** | *Toți* / *Doar cei acasă* / *Doar cei plecați*. Prezența se ia din `person.*` sau din `device_tracker`-ul telefonului. Un dispozitiv fără tracker primește mereu notificarea. |
| **Critic ignoră filtrul** | Implicit activ: o alertă critică ajunge la toată lumea. |
</details>

<details>
<summary><b>💬 Conținut</b></summary>

| Setare | Ce face |
|---|---|
| **Titlu / Subtitlu / Mesaj** | Textul notificării. Pe Android mesajul acceptă HTML simplu (`<b>`, `<i>`, `<font color>`). Pe iOS tag-urile apar ca text. |
| **Link la apăsarea notificării** | `/lovelace/interfon`, `entityId:lock.usa` (Android), `app://com.spotify.music` (Android), `https://…` |
| **Iconiță / culoare** (Android) | Iconița din bara de stare, de exemplu `mdi:doorbell`, și culoarea ei. |
</details>

<details>
<summary><b>🖼️ Imagine</b></summary>

| Tip | Ce face |
|---|---|
| **Fără** | — |
| **Cameră** | Snapshot din momentul trimiterii. Pe iOS, apăsarea lungă arată stream live. |
| **URL** | `/local/…` (din `config/www`), `/media/local/…` sau `https://…`. Linkurile relative se încarcă doar dacă telefonul ajunge la HA. |
</details>

<details>
<summary><b>🚦 Urgență și ore de liniște</b></summary>

| Setare | Ce face |
|---|---|
| **Nivel de urgență** | Vezi tabelul de mai sus. |
| **Ore de liniște** | În intervalul ales (poate trece peste miezul nopții, de exemplu 22:30 → 07:00), notificările **Info și Normal** sunt fie trimise silențios (ca Info, fără TTS), fie deloc. Important și Critic trec mereu. |
</details>

<details>
<summary><b>1️⃣ 2️⃣ 3️⃣ Butoane</b></summary>

| Setare | Ce face |
|---|---|
| **Afișează / Text** | Până la 3 butoane. Textul trebuie să fie scurt, pentru că Android îl taie. |
| **Ce face butonul** (Android) | **Acțiuni**: rulează acțiuni în HA. **Link**: deschide un link. Pe Android un buton nu le poate face pe amândouă (limitare a aplicației). Pe iOS setarea se ignoră și se fac ambele. |
| **Acțiuni** | Ce rulează la apăsare. Pentru un buton care doar închide notificarea, lasă lista goală. |
| **Link** | Același format ca la linkul notificării; pe iOS și `tel:`, `mailto:`. |
| **Iconiță / Roșu / Cere deblocare** (iOS) | SF Symbol, de exemplu `door.left.hand.open`; text roșu pentru acțiuni periculoase; cere Face ID / cod. Deblocarea e recomandată pentru deschiderea ușii. |
</details>

<details>
<summary><b>⌛️ Timeout</b></summary>

| Setare | Ce face |
|---|---|
| **Activează timeout** | Cât timp ascultă scriptul. Dacă e dezactivat, ascultă până răspunde cineva. |
| **Acțiuni la timeout** | Ce se întâmplă dacă nu răspunde nimeni, de exemplu trimiterea unei notificări Critice. |
| **Ștergerea = timeout** (Android) | Un swipe pe oricare telefon rulează imediat acțiunile de timeout. |
| **Șterge la timeout** | Notificarea dispare de pe toate dispozitivele la expirare. |
</details>

<details>
<summary><b>🤖 Android avansat</b></summary>

| Setare | Ce face |
|---|---|
| **Prefix canal** | Canalele create: `<prefix> Info / Normal / Important`. Un prefix propriu per script înseamnă un sunet propriu per script. |
| **Livrare imediată** | `priority: high` + `ttl: 0`, ca economia de baterie să nu întârzie notificarea. Nu schimbă sunetul. |
| **Sună o singură dată** | Actualizările aceleiași notificări nu mai sună. |
| **Notificare fixă** | Nu poate fi ștearsă cu swipe. |
| **Rămâne după apăsare** | Nu dispare când apeși pe ea. |
| **Numărătoare inversă** | Arată cât timp a mai rămas până la timeout. |
| **Model vibrație** | De exemplu `0, 500, 200, 500` (ms: pauză, vibrație, …). |
| **Culoare LED** | Doar pe telefoanele care mai au LED. |
| **Pe ecranul blocat** | Public / Privat / Secret. |
| **Android Auto** | Afișează notificarea și în mașină. |
| **TTS** | Telefonul citește mesajul. *Media* nu se aude pe silențios; *Alarmă* se aude; *Alarmă la maxim* urcă temporar volumul. Nu se citește la nivelul Info. |
</details>

<details>
<summary><b> iOS</b></summary>

| Setare | Ce face |
|---|---|
| **Sunet** | Numele unui sunet din aplicație (*Settings → Companion App → Notifications → Sounds*), de exemplu `US-EN-Alexa-Doorbell.wav`. `none` = fără sunet. |
| **Volum la Critic** | 0–1. Se aude chiar și cu telefonul pe Mute. |
</details>

<details>
<summary><b>⚙️ Diverse</b></summary>

| Setare | Ce face |
|---|---|
| **Tag** | Gol (recomandat) înseamnă tag unic per rulare. Un tag fix, de exemplu `interfon`, face ca o notificare nouă s-o **înlocuiască** pe cea veche. |
| **Grup** | Grupează vizual notificările. Pe iOS, cele critice nu se grupează. |
</details>

---

## 🧪 Exemple de folosire

### 1. Interfon: deschide ușa de pe orice telefon

Creezi un script din blueprint cu:
- **Dispozitive:** telefonul tău + telefonul partenerului
- **Urgență:** ⚠️ Important
- **Butonul 1:** `Deschide` → acțiune `switch.turn_on` pe releul ușii; pe iOS activezi *Cere deblocarea*
- **Butonul 2:** `Ignoră` → fără acțiuni
- **Imagine:** camera de la intrare
- **Timeout:** 2 minute

Apoi îl apelezi dintr-o automație:

```yaml
triggers:
  - trigger: state
    entity_id: binary_sensor.interfon_suna
    to: "on"
actions:
  - action: script.notificare_interfon
```

### 2. Suprascrierea setărilor la apelare (`fields`)

Același script poate fi apelat cu text, urgență sau dispozitive diferite:

```yaml
- action: script.notificare_generala
  data:
    field_title: "💧 Senzor inundație"
    field_message: "S-a detectat apă în baie!"
    field_urgency: critical
    field_tts_text: "Atenție, apă în baie"
```

Câmpuri disponibile: `field_notify_devices`, `field_urgency`, `field_title`, `field_subtitle`, `field_message`, `field_notification_link`, `field_attachment_type`, `field_attachment_camera_entity`, `field_attachment_image_url`, `field_option_{one,two,three}_enabled`, `field_option_{one,two,three}_text`, `field_enable_timeout`, `field_timeout`, `field_run_timeout_actions`, `field_tts_text`.

> Acțiunile butoanelor **nu** pot fi suprascrise din `fields`, din cauza unei limitări a blueprint-urilor. Pentru acțiuni diferite, folosește răspunsul scriptului (exemplul 3) sau creează alt script.

### 3. Folosirea răspunsului într-o automație

```yaml
- action: script.notificare_generala
  data:
    field_message: "Ai lăsat geamul deschis. Îl închid?"
    field_option_one_text: "Închide"
    field_option_two_text: "Lasă"
  response_variable: raspuns

- if: "{{ raspuns.result == 'option_one' }}"
  then:
    - action: cover.close_cover
      target:
        entity_id: cover.geam_dormitor
    - action: logbook.log
      data:
        name: Geam
        message: "Închis la cererea lui {{ raspuns.responded_by }}"
```

> [!WARNING]
> Ca să primești răspunsul, scriptul trebuie apelat cu `action: script.<nume>`, nu cu `script.turn_on`. Automația așteaptă până răspunde cineva sau până expiră timeout-ul.

---

## 📤 Răspunsul scriptului

```yaml
result: option_one          # vezi tabelul de mai jos
responded_by: Ana           # numele persoanei care a răspuns (sau "cineva")
responder_person: person.ana
level: important            # urgența efectivă, după orele de liniște
quiet_hours: false          # true dacă s-au aplicat orele de liniște
devices: [Pixel 8, iPhone Ana]
```

| `result` | Înseamnă |
|---|---|
| `option_one` / `option_two` / `option_three` | Cineva a apăsat butonul respectiv |
| `timeout` | Nu a răspuns nimeni în timpul setat |
| `notification_cleared` | Notificarea a fost ștearsă cu swipe (cu opțiunea activă) |
| `cleared_ignored` | Ștearsă cu swipe, dar opțiunea e dezactivată, deci nu s-a rulat nimic |
| `no_response` | Fără butoane și fără timeout, deci nu s-a așteptat răspuns |
| `not_sent` | Nu s-a trimis nimic; `reason` este `quiet_hours` sau `no_devices` |

---

## ⚠️ Limitări cunoscute

- **Restartul HA** oprește ascultarea. Butoanele notificărilor trimise înainte de restart nu mai fac nimic.
- **Butoanele-link pe Android** nu trimit eveniment în HA, deci **nu opresc timeout-ul**.
- **Redenumirea telefonului:** serviciul `notify.mobile_app_<nume>` e construit din numele dispozitivului. Dacă redenumești telefonul în aplicație, verifică în *Developer Tools → Actions* că serviciul există.
- **Televizoarele** (LG webOS, Android TV) nu sunt suportate. Au servicii de notificare proprii, fără butoane.
- **HTML în mesaj** apare ca text pe iOS.
- **Numele persoanei** care a răspuns apare doar dacă utilizatorul HA al telefonului e legat de o entitate `person.*`.

---

## 🛠 Depanare

**Nu se întâmplă nimic când apăs un buton:**
1. *Developer Tools → Events* → ascultă `mobile_app_notification_action`.
2. Rulează scriptul și apasă un buton pe telefon.
3. Dacă nu apare niciun eveniment, problema e la rețea sau la aplicație, nu la script. Vezi [FAQ-ul Companion](https://companion.home-assistant.io/docs/troubleshooting/faqs).

**Pe Android nu se schimbă sunetul sau importanța:** canalul există deja cu setările vechi. Schimbă **prefixul de canal** sau modifică canalul din setările telefonului.

**Notificarea nu ajunge deloc:** verifică *Settings → Automations & Scenes → Scripts → (scriptul) → Traces*. Pasul *Notificare* arată serviciul folosit și eventualele erori.

---

## 📝 Changelog

### v3.0
- Notificări pe mai multe dispozitive; primul răspuns câștigă, iar pe celelalte telefoane notificarea se actualizează
- Selector de urgență unificat Android/iOS, cu câte un canal Android pentru fiecare nivel
- Ore de liniște, filtru după prezență, TTS pe Android, imagine din URL
- Opțiuni Android avansate: numărătoare inversă, `alert_once`, `sticky`, vibrație, LED
- Sunet personalizat și volum critic pe iOS
- Secțiuni pliabile, descrieri în română, sintaxă `action:` / `trigger:`, `mode: parallel`
- Răspunsul scriptului include `responded_by`, `level`, `quiet_hours`
- Reparații față de v2.0.2: suprascrierea titlurilor butoanelor 2 și 3, exemplul greșit al câmpului 3, descrieri copiate, rezultat lipsă la timeout

### v2.0.2 și anterioare
Vezi [istoricul original al lui @samuelthng](https://github.com/samuelthng/t-house-blueprints/blob/main/notifications.yaml) și [postarea de pe forum](https://community.home-assistant.io/t/notifications-actionable-mobile-notifications-script-with-optional-timeout-feature-and-camera-snapshots-works-with-ios-android/551552).

---

## 🙏 Mulțumiri

- **[@samuelthng](https://github.com/samuelthng)**, autorul blueprint-ului original [Notifications](https://github.com/samuelthng/t-house-blueprints/blob/main/notifications.yaml), pe care se bazează integral această versiune
- Contribuitorii proiectului original, printre care [@HNKNTA](https://github.com/HNKNTA) (linkul notificării, v2.0.2)
- Echipa [Home Assistant Companion](https://companion.home-assistant.io/), pentru documentația notificărilor

Toate drepturile asupra codului original aparțin autorului său.
