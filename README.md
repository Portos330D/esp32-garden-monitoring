# 🌿 ESP32 – Capteurs Jardin (Température, Humidité du sol, Débitmètre)

Ce projet utilise un **ESP32** comme module de mesure extérieur pour Home Assistant.  
Il permet de mesurer :

- 🌡️ Température (air, sol, eau) via **DS18B20**
- 🌱 Humidité du sol (capteur analogique)
- 🚰 Débit d’eau via un **débitmètre à impulsions**
- 🔵 Activation/désactivation Bluetooth (optionnel)

Le code est basé sur **ESPHome**.

---

## 🔧 Matériel compatible

- ESP32 DevKit V1  
- Sondes DS18B20 (1 à plusieurs sondes)  
- Capteur d’humidité du sol analogique (0–3.3V)  
- Débitmètre : YF-S401, YF-B5, YF-B6  
- Résistances :  
  - 4.7kΩ pour le bus DS18B20  
  - 1kΩ en série sur le débitmètre (anti-bruit recommandé)

---

## 🔌 Schéma de câblage (générique)

### **1️⃣ DS18B20**
VCC → 3.3V  
GND → GND  
DATA → GPIO4  
Résistance 4.7kΩ entre DATA et 3.3V

### **2️⃣ Humidité du sol (ADC)**
Signal → GPIO34  
VCC → 3.3V  
GND → GND  

### **3️⃣ Débitmètre**
Signal (jaune) → GPIO27  
VCC (rouge) → 5V  
GND (noir) → GND  
Résistance en série recommandée : 1kΩ

---

## 📘 Exemple complet de configuration ESPHome (à adapter)

```yaml
esphome:
  name: capteurs-jardin

esp32:
  board: esp32dev
  framework:
    type: esp-idf

logger:
api:
ota:

wifi:
  ssid: "VOTRE_WIFI"
  password: "VOTRE_MDP"

# Bus DS18B20
one_wire:
  - platform: gpio
    pin: GPIO4
    id: bus_jardin

sensor:

  # ------------------ TEMPERATURES DS18B20 ------------------
  - platform: dallas_temp
    bus_id: bus_jardin
    address: 0x0000000000000000
    name: "Température Air"
    update_interval: 60s

  - platform: dallas_temp
    bus_id: bus_jardin
    address: 0x0000000000000000
    name: "Température Sol"
    update_interval: 60s

  - platform: dallas_temp
    bus_id: bus_jardin
    address: 0x0000000000000000
    name: "Température Eau"
    update_interval: 60s

  # ------------------ HUMIDITÉ DU SOL ------------------
  - platform: adc
    pin: GPIO34
    id: humidite_brut
    name: "Humidité Sol (brut)"
    update_interval: 10s
    attenuation: 12db

  # Capteur transformé en % (0 = humide, 100 = sol sec)
  - platform: template
    name: "Humidité Sol (%)"
    unit_of_measurement: "%"
    accuracy_decimals: 0
    lambda: |-
      float v = id(humidite_brut).state;
      return (v / 3.3) * 100.0;

  # ------------------ DEBITMETRE ------------------
  - platform: pulse_meter
    pin: GPIO27
    name: "Débit Jardin (pulses/min)"
    id: pulses_jardin
    internal: true
    update_interval: 1s
    timeout: 5s

  # Débit converti en L/min (exemple générique)
  - platform: template
    name: "Débit Jardin"
    unit_of_measurement: "L/min"
    accuracy_decimals: 2
    lambda: |-
      float p = id(pulses_jardin).state;
      return p / 158.0;   # Exemple : 158 pulses = 1L (à calibrer)

  # Volume total pompé
  - platform: template
    name: "Volume Jardin Total (L)"
    unit_of_measurement: "L"
    accuracy_decimals: 2
    lambda: |-
      static float total = 0;
      float l_min = id(pulses_jardin).state / 158.0;
      total += (l_min / 60.0);
      return total;
    update_interval: 60s

switch:
  - platform: template
    name: "Bluetooth ESP32 Jardin"
    turn_on_action:
      - ble.enable:
    turn_off_action:
      - ble.disable:
    restore_mode: ALWAYS_OFF
