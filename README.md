# 🌿 ESP32 – Capteurs Jardin (Température, Humidité du sol, Débitmètre)

Ce projet utilise un **ESP32** comme module de mesure extérieur pour Home Assistant.  
Il permet de mesurer :

- 🌡️ Température (air, sol, eau) via **DS18B20**
- 🌱 Humidité du sol (capteur analogique)
- 🚰 Débit d’eau via un **débitmètre à impulsions**
- 🔵 Activation/désactivation Bluetooth (optionnel)

Le code est basé sur **ESPHome**.

--- 
## 🔧 Matériel recommandé & liens    Sonde d’humidité de sol (analogique)

Voici des exemples de matériel utilisé pour ce projet — libre à vous d’adapter en fonction de vos capteurs ou de votre fournisseur :

| Matériel / usage | Lien / Référence |
|------------------|------------------|
| Débitmètre (pulse meter) pour arrosage / pompe | https://a.aliexpress.com/_Ew5A44U |
| Capteur DS18B20 étanche (température) | https://a.aliexpress.com/_EHGbenE |
| ESP32 DevKit ou module de base | https://a.aliexpress.com/_EG2qenS |
| Résistances, câbles, composants passifs | https://a.aliexpress.com/_EzdSZI4 , https://a.aliexpress.com/_EzNLtpi |
| Platine / support prototype pour ESP32 | https://a.aliexpress.com/_Ey54Ir2 |
| Sonde d’humidité de sol (analogique) | https://a.aliexpress.com/_ExRe2iY |
| Capteur de niveau de cuve Zigbee | https://a.aliexpress.com/_EHnzeVI |
| Support 3D / Abri météo (Stevenson Screen) pour capteurs extérieurs | https://makerworld.com/fr/models/936490-universal-stevenson-screen-temperature-humidy |

> 💡 **Note** : Le “Stevenson Screen” (abri météo) est utilisé pour protéger les sondes de température et d’humidité extérieure contre le soleil, la pluie et les radiations, tout en permettant une bonne circulation d’air — il améliore la précision des mesures en conditions réelles. 2
>
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
```

🎯 Calibration du débitmètre (méthode générique)

1. Faire passer 5 litres réels dans un récipient gradué.  
2. Noter le nombre de pulses mesurés.  
3. Calculer :

pulses_par_litre = pulses_mesurés / litres

4. Modifier la ligne du YAML :

return p / pulses_par_litre;

---

🎯 Calibration humidité du sol (méthode générique)

1. Mesurer la tension dans du terreau sec.  
2. Mesurer la tension dans du terreau 100% humide.  
3. Adapter la formule :

return (v - tension_humide) * 100 / (tension_sec - tension_humide);

---

🧪 Exemple d’utilisation dans Home Assistant

Représentation recommandée :

- Graphique d’humidité du sol  
- Courbe de température air / sol / eau  
- Suivi du volume pompé  
- Automatisation d’arrosage basée sur un seuil du sol  

---

📦 Partage GitHub

Ce README est prêt pour être placé dans un dépôt public GitHub.  
Ajoutez-y :

- `/esphome/capteurs-jardin.yaml`  
- Des schémas ou photos (optionnel)  
- Une section “Issues” pour aider les utilisateurs  

---

🤝 Contributions

Les utilisateurs peuvent :

- Adapter les GPIO  
- Ajouter des sondes  
- Modifier les filtres  
- Ouvrir des issues ou PR  

---

📝 Licence

Libre d’utilisation et de modification.
