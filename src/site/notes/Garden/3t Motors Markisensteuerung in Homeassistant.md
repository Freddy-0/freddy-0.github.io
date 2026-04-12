---
{"dg-publish":true,"permalink":"/garden/3t-motors-markisensteuerung-in-homeassistant/","dg-note-properties":{}}
---

Ich habe in **ESPHome** einen RF-Transmitter eingerichtet, der mithilfe eines **ESP32** und eines **CC1101-Transceivers** die Markise auf meiner Terrasse steuert. Dafür habe ich einen **RAW-Dump** der übertragenen Daten aufgenommen, der direkt von einer gekoppelten Fernbedienung gesendet wurde. Den **RAW-Dump** habe ich unverändert in ein `transmit_raw`-Statement und einen Cover integriert.

Außerdem habe ich einen Decoder für **Dooya Commands** integriert, der die Signale der Original-Fernbedienung empfängt und sie mit der richtigen Aktion in ESPHome verknüpft. Der ESP32 erkennt nun, wenn die Markise mit der externen Fernbedienung geschlossen wird, und passt die Öffnung in Prozent an den tatsächlichen Stand der Markise an.

Problem ist, dass die Motoren in etwas das Dooya Protokol nutzen, aber beim Empfangen von Befehlen nicht das als Dooya Empfangen, was ich sende, da eine besondere Logik zum empfangen implementiert ist. Die Fernbedienung sendet Dooya Commands, wenn ich den selben Command sende, klappt es aber nicht. Die Fernbedienung nutzt zum glück statische Codes zur Kommunikation mit dem Motor.

## Idee zur Automatisierung

Eine mögliche Automatisierung ist:

- Sonnenstandsgesteuert: Angepasste Position abhängig von den auf der Terrasse befindlichen Solarmodulen
- Wenn es an einem Tag über **25 °C** warm ist, soll die Markise ausgefahren werden, sobald die Sonne eine Elevation von mehr als **20 °** erreicht

## Bill of Materials

|Bauteil|Zweck|
|---|---|
|ESP32|Zentrale Steuerung und ESPHome-Plattform|
|CC1101-Transceiver|433,92-MHz-Funkmodul für Senden und Empfangen|
|433-MHz-Fernbedienung|Original-Fernbedienung zum Mitschneiden der Signale|
|Markisenmotor / Markisensteuerung|Zielgerät der Funksteuerung|
|Jumper-Kabel|Verbindung zwischen ESP32 und CC1101|
|Stromversorgung|Versorgung des ESP32 und des Funkmoduls|

## Pinout

Das Pinout zwischen ESP32 und CC1101 ist wie folgt:

1. GND
2. 3V3
3. D26
4. D27
5. D14
6. D13
7. D12
8. D25

![Pasted image 20260412194231.png\|704](/img/user/X-Anh%C3%A4nge/Pasted%20image%2020260412194231.png)
[Bildquelle](https://zaitronics.com.au/blogs/guides/cc1101-guide-instruction-manual)
## Allgemeines Konzept

Das Projekt basiert auf einem einfachen, aber effektiven Prinzip:

- Die Original-Fernbedienung sendet ein Dooya-Funksignal.
- Der CC1101 empfängt dieses Signal über den ESP32.
- ESPHome interpretiert die empfangenen Dooya-Kommandos.
- Derselbe ESP32 kann zusätzlich die aufgezeichneten RAW-Signale wieder aussenden.
- Dadurch lassen sich die Funktionen **Auf**, **Zu** und **Stopp** zuverlässig nachbilden.
- Über einen `time_based`-Cover wird die Position der Markise logisch nachgeführt.

Der Vorteil dieses Ansatzes liegt darin, dass keine direkte Änderung an der eigentlichen Markisensteuerung nötig ist. Stattdessen wird die vorhandene Funktechnik genutzt und mit Home Assistant bzw. ESPHome intelligent erweitert.


![Pasted image 20260412194351.png\|654](/img/user/X-Anh%C3%A4nge/Pasted%20image%2020260412194351.png)
[Bildquelle](https://www.3t-motors.de/3T-Motors-Rollladenmotore)

## Technische Umsetzung

Die Konfiguration in ESPHome besteht aus drei Hauptteilen:

- **Senden der RAW-Codes**
- **Empfangen und Dekodieren der Dooya-Kommandos**
- **Zeitbasierte Positionsverwaltung**

### 1. RF-Senden per RAW-Dump

Für die Steuerung wurden die Funktelegramme der Original-Fernbedienung per RAW-Dump aufgezeichnet. Diese Sequenzen wurden direkt in `remote_transmitter.transmit_raw` übernommen. Dadurch sendet ESPHome exakt die gleichen Signale wie die originale Fernbedienung.

Dazu wurden drei interne Buttons angelegt:

- `shutter_up_btn`
- `shutter_down_btn`
- `shutter_stop_btn`

Diese Buttons sind intern und werden vom Cover aufgerufen.

### 2. Zeitbasierter Cover

Der eigentliche Cover-Eintrag nutzt `platform: time_based`. Damit wird die Position der Markise anhand der bekannten Laufzeit berechnet.

- `open_duration` definiert die Zeit für das vollständige Öffnen
- `close_duration` definiert die Zeit für das vollständige Schließen
- `assumed_state: true` erlaubt die Steuerung unabhängig vom exakten Ist-Zustand

Das ist besonders praktisch, wenn die Markise auch per Fernbedienung bewegt wird, da ESPHome dadurch den Positionsstatus logisch nachführen kann.

### 3. Empfang von Dooya-Kommandos

Der `remote_receiver` hört auf 433,92 MHz und dekodiert Dooya-Signale. Über drei `binary_sensor`-Einträge werden die Kommandos den jeweiligen Aktionen zugeordnet:

- Öffnen
- Schließen
- Stopp

Wenn also die Original-Fernbedienung benutzt wird, kann ESPHome das erkennen und den Cover-Status passend aktualisieren.

## Technische Notizen

- Der CC1101 arbeitet hier im **ASK/OOK-Modus**.
- Der Receiver ist auf **Dooya** konfiguriert.
- Die Toleranz wurde bewusst großzügig gewählt, um die Erkennung robuster zu machen.
- Die RAW-Sequenzen wurden unverändert übernommen, damit die Originalsignale möglichst exakt reproduziert werden.

## Ergebnis

Mit dieser Lösung lässt sich die Markise sowohl über Home Assistant als auch über die Original-Fernbedienung bedienen. Gleichzeitig bleibt der Positionsstatus der Markise in ESPHome weitgehend synchron. Das macht das System robust, alltagstauglich und erweiterbar für spätere Automatisierungen, etwa basierend auf Temperatur oder Sonnenstand.

## ESPHome-Konfiguration

```yaml
esphome:
  name: markisensteuerung
  friendly_name: Balkon-Markise

esp32:
  board: esp32dev
  framework:
    type: arduino

substitutions:
  # Duration for a full open/close cycle. Adjust these values to match the
  # amount of time your 3T shutter takes to fully open or close.  The
  # time_based cover below uses these variables to calculate the position.
  open_duration: "60s"
  close_duration: "60s"

# Enable logging
logger:

api:
  encryption:
    key: "redacted"

ota:
  - platform: esphome
    password: "redacted"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

# SPI bus for CC1101
spi:
  clk_pin: GPIO14
  miso_pin: GPIO12
  mosi_pin: GPIO13

# CC1101 radio
cc1101:
  cs_pin: GPIO27
  frequency: 433.92MHz
  modulation_type: ASK/OOK

# CC1101 dual-pin mode
remote_transmitter:
  pin: GPIO26        # CC1101 GDO0
  carrier_duty_percent: 100%
  on_transmit:
    then:
      - cc1101.begin_tx
  on_complete:
    then:
      - cc1101.begin_rx


## Internal buttons that send the raw pulse sequences captured from the
# original remote.  Marking these buttons as `internal: true` hides them
# from Home Assistant.  The cover below calls these buttons to perform
# open/close/stop actions.
button:
  - platform: template
    id: shutter_down_btn
    name: "Shutter DOWN"
    internal: true
    on_press:
      - remote_transmitter.transmit_raw:
          code: [
              4803, -1521, 340, -741, 337, -717, 343, -745, 316, -744, 698, -384,
              693, -368, 692, -368, 333, -760, 315, -747, 330, -741, 333, -733,
              340, -747, 705, -350, 714, -365, 712, -365, 326, -739, 707, -372,
              689, -388, 334, -741, 328, -737, 690, -368, 334, -740, 709, -369,
              314, -754, 338, -740, 687, -385, 321, -743, 714, -364, 336, -722,
              349, -717, 338, -752, 692, -389, 325, -739, 338, -728, 692, -366,
              700, -378, 341, -724, 337, -744, 695, -368, 692, -419, 4749, -1523,
              349, -719, 340, -724, 344, -745, 345, -718, 715, -365, 693, -366,
              698, -368, 344, -744, 326, -740, 340, -725, 342, -744, 326, -741,
              691, -369, 714, -348, 721, -366, 340, -723, 727, -369, 693, -367,
              327, -742, 341, -724, 717, -341, 356, -738, 690, -364, 358, -716,
              355, -735, 689, -370, 363, -715, 699, -366, 340, -747, 327, -741,
              338, -729, 691, -392, 328, -741, 338, -724, 713, -366, 702, -351,
              339, -750, 318, -738, 696, -379, 715, -7721, 4783, -1514, 345, -717,
              342, -719, 358, -725, 349, -716, 723, -364, 690, -368, 727, -344,
              345, -743, 326, -742, 339, -722, 358, -712, 345, -737, 691, -367,
              724, -344, 720, -340, 356, -738, 714, -346, 715, -371, 337, -734,
              339, -723, 727, -352, 341, -721, 720, -364, 345, -717, 342, -721,
              734, -343, 349, -717, 713, -366, 339, -733, 357, -713, 334, -743,
              714, -364, 338, -715, 359, -720, 716, -369, 692, -369, 334, -744,
              338, -734, 693, -364, 720, -364, 4789, -1516, 337, -731, 340, -724,
              349, -742, 338, -728, 716, -343, 728, -343, 720, -341, 355, -738,
              338, -734, 343, -717, 335, -746, 345, -717, 716, -339, 717, -367,
              702, -369, 345, -742, 699, -357, 716, -371, 343, -716, 356, -712,
              714, -371, 332, -741, 711, -341, 362, -722, 359, -714, 708, -371,
              339, -722, 710, -349, 363, -727, 340, -723, 348, -717, 716, -365,
              339, -724, 350, -740, 711, -341, 711, -366, 358, -714, 357, -712,
              713, -370, 708, -400, 4777, -1501, 341, -746, 344, -720, 327, -741,
              341, -722, 717, -366, 700, -358, 716, -371, 341, -716, 356, -714,
              364, -722, 330, -736, 335, -732, 716, -344, 729, -350, 716, -362,
              334, -735, 709, -347, 722, -366, 338, -731, 359, -713, 704, -375,
              340, -725, 718, -340, 356, -740, 340, -726, 718, -340, 358, -712
            ]

  - platform: template
    id: shutter_stop_btn
    name: "Shutter STOP"
    internal: true
    on_press:
      - remote_transmitter.transmit_raw:
          code: [
              4782, -1546, 313, -769, 314, -741, 321, -746, 338, -745, 686, -364,
              700, -394, 693, -366, 328, -737, 341, -721, 342, -742, 335, -745,
              318, -744, 692, -362, 710, -366, 700, -395, 318, -744, 698, -359,
              718, -360, 338, -744, 327, -740, 689, -369, 336, -741, 709, -370,
              314, -747, 359, -715, 707, -371, 339, -722, 710, -348, 362, -729,
              340, -723, 346, -735, 684, -376, 342, -746, 692, -366, 349, -718,
              717, -370, 339, -715, 707, -373, 340, -721, 710, -372, 4786, -1521,
              340, -725, 340, -740, 335, -719, 342, -721, 716, -364, 710, -365,
              696, -370, 344, -743, 326, -738, 338, -736, 343, -723, 325, -741,
              718, -343, 715, -371, 712, -342, 339, -748, 708, -348, 713, -367,
              339, -747, 328, -741, 688, -367, 339, -732, 710, -348, 362, -730,
              340, -722, 727, -369, 317, -742, 700, -357, 343, -747, 342, -720,
              325, -741, 716, -371, 338, -714, 705, -374, 341, -724, 718, -341,
              355, -738, 689, -364, 360, -715, 709, -382, 4780, -1517, 336, -730,
              340, -723, 349, -716, 363, -728, 692, -366, 699, -380, 690, -364,
              346, -745, 346, -717, 341, -723, 346, -744, 318, -744, 690, -363,
              712, -367, 699, -384, 335, -738, 689, -370, 713, -372, 339, -728,
              341, -723, 697, -369, 344, -745, 702, -352, 341, -748, 341, -717,
              709, -364, 347, -716, 715, -366, 340, -723, 348, -716, 340, -748,
              686, -393, 328, -741, 685, -366, 336, -746, 707, -348, 363, -728,
              691, -367, 351, -717, 715
            ]

  - platform: template
    id: shutter_up_btn
    name: "Shutter UP"
    internal: true
    on_press:
      - remote_transmitter.transmit_raw:
          code: [
              4788, -1546, 313, -746, 326, -740, 342, -725, 340, -744, 700, -354,
              718, -370, 691, -371, 336, -730, 340, -747, 328, -741, 335, -730,
              341, -723, 700, -395, 694, -367, 702, -355, 343, -747, 696, -366,
              698, -384, 319, -745, 340, -720, 720, -362, 321, -744, 691, -390,
              319, -745, 348, -743, 688, -365, 337, -734, 693, -368, 342, -742,
              341, -724, 349, -717, 714, -365, 340, -734, 340, -743, 327, -739,
              690, -369, 335, -740, 335, -738, 340, -722, 710, -398, 4756, -1526,
              341, -724, 341, -741, 334, -716, 341, -735, 723, -337, 735, -343,
              726, -358, 338, -731, 340, -723, 351, -741, 344, -707, 344, -746,
              694, -360, 714, -366, 699, -359, 344, -745, 714, -362, 710, -343,
              353, -716, 344, -731, 721, -363, 346, -718, 716, -338, 345, -748,
              348, -718, 714, -365, 340, -723, 703, -368, 345, -744, 327, -739,
              341, -721, 709, -373, 345, -709, 345, -746, 319, -744, 698, -366,
              347, -716, 343, -746, 342, -721, 697, -410, 4771, -1516, 335, -742,
              341, -722, 342, -741, 335, -720, 718, -358, 722, -339, 719, -366,
              348, -716, 344, -727, 360, -723, 349, -716, 339, -747, 710, -343,
              726, -359, 708, -353, 342, -746, 697, -366, 697, -385, 319, -743,
              340, -719, 710, -365, 350, -715, 722, -361, 336, -748, 328, -740,
              699, -366, 340, -723, 727, -368, 320, -745, 327, -740, 341, -721,
              708, -359, 369, -714, 336, -733, 330, -736, 709, -355, 717, -359,
              712, -365, 715, -366, 327, -8046, 4788, -1516, 346, -708, 344, -741,
              345, -717, 347, -728, 721, -363, 693, -370, 697, -367, 335, -733,
              362, -727, 338, -712, 358, -722, 341, -723, 721, -340, 732, -363,
              695, -370, 344, -716, 725, -369, 689, -371, 363, -714, 345, -713,
              718, -367, 349, -716, 724, -338, 345, -746, 346, -716, 727, -338,
              345, -740, 695, -370, 345, -716, 355, -714, 364, -722, 709, -370,
              343, -717, 347, -722, 344, -743, 699, -365, 724, -336, 719, -372,
              693, -370, 346, -8031, 4772, -1545, 312, -742, 343, -747, 315, -746,
              325, -740, 692, -394, 687, -372, 698, -367, 342, -746, 326, -741,
              343, -732, 319, -744, 348, -718, 714, -365, 692, -366, 701, -395,
              318, -745, 697, -373, 690, -369, 337, -741, 333, -731, 716, -368,
              331, -742, 687, -366, 340, -734, 360, -714, 709, -380, 318, -743
            ]

## Time-based cover.  Uses the internal buttons defined above for
# the actions and keeps track of the shutter position based on the
# configured open and close durations.  The durations are defined in
# the substitutions section.  Assumed state is enabled to allow
# controlling the cover regardless of its current state.
cover:
  - platform: time_based
    name: "Balkon Markise"
    id: shutter
    device_class: shutter
    open_action:
      - button.press: shutter_up_btn
    open_duration: ${open_duration}
    close_action:
      - button.press: shutter_down_btn
    close_duration: ${close_duration}
    stop_action:
      - button.press: shutter_stop_btn
    assumed_state: true

# Remote receiver to listen for Dooya RF remote codes.  When a code
# matching your remote is received, binary sensors below will call
# the appropriate cover action to keep the position and state in sync.
remote_receiver:
  pin: GPIO25
  dump:
    - dooya
  # Adjust tolerance and filter settings to improve decoding reliability.
  tolerance:
    # Allow ±50% variation in timings
    type: percentage
    value: 50%
  filter: 250us
  idle: 7ms
  buffer_size: 10kB

binary_sensor:
  # Closing command from original remote
  - platform: remote_receiver
    internal: true
    id: reciever_close
    dooya:
      id: 0x000E0ECA
      channel: 81
      button: 3
      check: 3
    filters:
      - delayed_off: 100ms
    on_press:
      - cover.close: shutter

  # Stop command from original remote
  - platform: remote_receiver
    internal: true
    id: reciever_stop
    dooya:
      id: 0x000E0ECA
      channel: 81
      button: 5
      check: 5
    filters:
      - delayed_off: 100ms
    on_press:
      - cover.stop: shutter

  # Opening command from original remote
  - platform: remote_receiver
    internal: true
    id: reciever_open
    dooya:
      id: 0x000E0ECA
      channel: 81
      button: 1
      check: 1
    filters:
      - delayed_off: 100ms
    on_press:
      - cover.open: shutter
```