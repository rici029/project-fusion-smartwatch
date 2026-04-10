# InkTime - Smartwatch Open Source

InkTime este un smartwatch ieftin, open-source, construit in jurul microcontroller-ului **Nordic nRF52840** (BLE 5 + ARM Cortex-M4) si a unui display **e-paper de 1.54"**. Consumul redus al e-paper-ului combinat cu eficienta nRF-ului permite o autonomie de cateva zile pe o baterie LiPo mica.

---

## Cuprins

- [Diagrama bloc](#diagrama-bloc)
- [Descriere hardware](#descriere-hardware)
- [Bill of Materials (BOM)](#bill-of-materials-bom)
- [Pin mapping nRF52840](#pin-mapping-nrf52840)
- [Estimare consum energie](#estimare-consum-energie)
- [Design mecanic](#design-mecanic)
- [Design log](#design-log)
- [Structura repository-ului](#structura-repository-ului)
- [Licenta](#licenta)

---

## Diagrama bloc

Arhitectura dispozitivului se bazeaza pe nRF52840 ca hub central. Acesta comunica cu:

- **Display e-paper** prin SPI + semnale de control (DC/RST/BUSY)
- **Accelerometrul BMA423** prin I2C (pentru detectarea miscarii si step counter)
- **PMIC BQ25180** prin I2C (management incarcare baterie Li-Ion)
- **Driver haptic DRV2605** prin I2C (feedback vibrator)
- **USB-C** pentru incarcare si comunicatie seriala (USB nativ al nRF)
- **3 butoane** pe GPIO-uri dedicate
- **Antena chip 2.4GHz** pentru BLE

Alimentarea este furnizata de un buck-boost **RT6160** care genereaza 3.3V stabil din baterie (domeniu tipic 3.0-4.2V).

---

## Descriere hardware

### Microcontroller - Nordic nRF52840

- **Core:** ARM Cortex-M4F @ 64 MHz
- **Memorie:** 1 MB Flash + 256 KB RAM
- **Radio:** BLE 5.3, 2.4 GHz, puterea TX pana la +8 dBm
- **Interfete:** USB 2.0 Full Speed nativ, SPI, I2C, UART, I2S, PDM, QSPI
- **Justificare alegere:** consum redus (~5 uA in System OFF cu RAM retinuta), suport BLE complet, USB nativ care elimina nevoia de USB-to-Serial

### Power Management

**BQ25180 - Battery Charger (IC1)**
- Incarcator Li-Ion de 1 celula, pana la 550 mA
- Protectii integrate: OVP, SCP, termica
- Control prin I2C - poate raporta starea bateriei si curentul de incarcare
- Pin ALERT (interrupt) catre nRF pentru notificare evenimente

**RT6160 - Buck-Boost DC/DC (IC2)**
- Intrare: 2.5-5.5V (acopera atat VBAT cat si VBUS de la USB)
- Iesire: 3.3V (VREG -> 3V3 rail intern)
- Eficienta >90% la sarcini tipice
- Iesirea VSEL configurabila digital

### Senzori

**BMA423 - Accelerometru 3-axe (IC3)**
- 16-bit, range +/-2/+/-4/+/-8/+/-16 g
- Consum ultra-low: ~3 uA in low-power mode
- Features integrate: step counter, wake-on-tilt, activity recognition
- Comunicatie: I2C
- Doua interrupt-uri (INT1, INT2) pentru wake-up asincron al nRF

### Haptic Feedback

**DRV2605 - Haptic Driver (IC4)**
- Driver pentru motor ERM/LRA (shaker coin motor)
- Librarie integrata de 123 efecte preset
- Control I2C + enable digital (HAPTIC_EN)

### Display

**E-paper 1.54" (model WSH-12561, 200x200 pixel)**
- Conectat prin SPI (MOSI, SCK, EPD_CS) + 3 semnale control (DC, RST, BUSY)
- Circuit de boost integrat - foloseste componentele externe EPD_C1..EPD_C12 (condensatoare + inductor L2, L3 + diode D1-D3 pentru generarea PREVGL/PREVGH/GDR)
- Activare alimentare 3.3V prin MOSFET Q2 controlat de EPD_CS

### USB si protectie

**Conector USB-C (J3)** cu 16 pini (reversibil)
**USBLC6-2SC6Y (D4)** - ESD protection pe liniile D+/D-
Conectare directa la nRF52840 (USB nativ integrat)

### Circuit RF

Antena **2450AT18B100E** (Johanson) cu pi-network de matching:
- **C13, C15, C16** - condensatoare de acord
- **L4 (3.9 nH)** - inductor de matching
- **X1 (32 MHz)** - cristal pentru PLL radio, cu condensatoare de sarcina (12pF)
- **X2 (32.768 kHz)** - cristal pentru RTC (wake-up periodic low-power)

---

## Bill of Materials (BOM)

| Ref | Componenta | Valoare / Model | Capsula | Link JLC Parts | Datasheet |
|-----|------------|-----------------|---------|----------------|-----------|
| IC - nRF | Microcontroller | nRF52840-QIAA | aQFN73 | [JLC](https://jlcpcb.com/parts) | [PDF](https://infocenter.nordicsemi.com/pdf/nRF52840_PS_v1.7.pdf) |
| IC1 | Battery Charger PMIC | BQ25180 | WCSP-8 | [JLC](https://jlcpcb.com/parts) | [PDF](https://www.ti.com/lit/ds/symlink/bq25180.pdf) |
| IC2 | Buck-Boost DC/DC | RT6160A | WQFN-15 | [JLC](https://jlcpcb.com/parts) | [PDF](https://www.richtek.com/assets/product_file/RT6160A/DS6160A-00.pdf) |
| IC3 | Accelerometer | BMA423 | LGA-12 | [JLC](https://jlcpcb.com/parts) | [PDF](https://www.bosch-sensortec.com/media/boschsensortec/downloads/datasheets/bst-bma423-ds000.pdf) |
| IC4 | Haptic Driver | DRV2605 | WCSP-9 | [JLC](https://jlcpcb.com/parts) | [PDF](https://www.ti.com/lit/ds/symlink/drv2605.pdf) |
| U1 | Fuel Gauge | (MAX17048 / LC709203) | WCSP-8 | [JLC](https://jlcpcb.com/parts) | [PDF](https://www.analog.com/media/en/technical-documentation/data-sheets/MAX17048-MAX17049.pdf) |
| ANT1 | Chip Antenna 2.4GHz | 2450AT18B100E | SMD | [JLC](https://jlcpcb.com/parts) | [PDF](https://www.johansontechnology.com/datasheets/2450AT18B100/2450AT18B100.pdf) |
| X1 | Cristal | 32 MHz | 3.2x2.5 mm | [JLC](https://jlcpcb.com/parts) | - |
| X2 | Cristal | 32.768 kHz | 3.2x1.5 mm | [JLC](https://jlcpcb.com/parts) | - |
| L1 | Inductor DC/DC | MLP2016 470nH | 2016 | [JLC](https://jlcpcb.com/parts) | - |
| L2 | Inductor E-paper | 10 uH | 0603 | [JLC](https://jlcpcb.com/parts) | - |
| L3 | Inductor E-paper | 15 uH | 0603 | [JLC](https://jlcpcb.com/parts) | - |
| L4 | Inductor RF matching | 3.9 nH | 0201 | [JLC](https://jlcpcb.com/parts) | - |
| Q2 | MOSFET P-channel | (EPD power switch) | SOT-23 | [JLC](https://jlcpcb.com/parts) | - |
| D1-D3 | Dioda Schottky | (boost converter) | SOD-323 | [JLC](https://jlcpcb.com/parts) | - |
| D4 | ESD USB Protection | USBLC6-2SC6Y | SOT-23-6 | [JLC](https://jlcpcb.com/parts) | [PDF](https://www.st.com/resource/en/datasheet/usblc6-2.pdf) |
| J1 | SWD Programming | TC2030-IDC | THT | [JLC](https://jlcpcb.com/parts) | [PDF](https://www.tag-connect.com/wp-content/uploads/bsk-pdf-manager/TC2030-IDC_Datasheet_14.pdf) |
| J2 | FPC Connector | 503480-2400 (24p) | SMD | [JLC](https://jlcpcb.com/parts) | [PDF](https://www.molex.com/pdm_docs/sd/5034802400_sd.pdf) |
| J3 | USB-C | KH-TYPE-C-16P | SMD | [JLC](https://jlcpcb.com/parts) | - |
| SW1-3 | Buton tactil | SMD side-push | SMD | [JLC](https://jlcpcb.com/parts) | - |
| Baterie | LiPo 3.7V | AKY0106 | - | - | [PDF](https://www.tme.eu/Document/b9e12bf26ad0ba929a22ab5d58f022cd/AKY0106.pdf) |
| Display | E-paper 1.54" | WSH-12561 | FPC | - | [PDF](https://www.tme.eu/Document/0ca57a8ffbcd57b5bca53252eb9d6ec3/WSH-12561.pdf) |
| Shaker | Motor vibrator | DFRobot FIT0774 | coin | - | [link](https://www.tme.eu/ro/details/df-fit0774/motoare-dc/dfrobot/fit0774/) |

**Condensatoare si rezistente:** toate in capsule 0201 (pentru valori <= 100 nF) sau 0402 (pentru valori > 100 nF), conform cerintei proiectului. Vezi fisierul `Manufacturing/bom.txt` pentru lista completa cu referinte si valori.

---

## Pin mapping nRF52840

| Pin nRF52840 | Net / Semnal | Functie | Motivatie alegere |
|--------------|--------------|---------|-------------------|
| P0.26 | SCL | I2C Clock | Pin GPIO standard, aproape de IC-urile I2C (BMA423, BQ25180, DRV2605) pentru trase scurte |
| P0.27 | SDA | I2C Data | Pereche cu P0.26 pentru TWI0 |
| P0.04 | IMU_INT1 | Interrupt BMA423 | Pin cu capabilitate AIN (ADC) dar folosit ca GPIO - pozitie convenabila pe PCB |
| P0.13 | SCK | SPI Clock (E-paper) | High-speed GPIO, suport SPIM |
| P0.14 | MOSI | SPI Data (E-paper) | Pereche cu SCK pe aceeasi periferica SPIM |
| P0.19 | EPD_CS | Chip Select e-paper | GPIO simplu, control SPI CS |
| P0.20 | EPD_DC | Data/Command | Control semnal pentru display |
| P0.21 | EPD_RST | Reset e-paper | GPIO cu output drive standard |
| P0.22 | EPD_BUSY | Busy signal | Input de la display, pentru polling |
| P0.23 | HAPTIC_EN | Enable DRV2605 | GPIO simplu, activeaza driver-ul haptic |
| P0.24 | PMIC_INT | Interrupt BQ25180 | Interrupt la evenimente de charging |
| P0.25 | ALERT | Alert fuel gauge | Notificare low-battery |
| P0.09, P0.10 | SW1, SW2 (Buttons) | Butoane utilizator | Pini dedicati cu suport NFC dezactivat (configurabili GPIO) |
| P1.09 | SW3 | Al treilea buton | GPIO free pe portul P1 |
| P1.01 | IMU_INT2 | Interrupt secundar BMA423 | GPIO free |
| SWDIO, SWDCLK | TP + J1 | Programming / Debug | Pini dedicati SWD, accesibili prin TC2030 si test pad-uri |
| XL1, XL2 | X2 (32.768 kHz) | LFCLK pentru RTC | Pini dedicati oscilatorului low-frequency |
| XC1, XC2 | X1 (32 MHz) | HFCLK pentru radio | Pini dedicati oscilatorului high-frequency |
| D+/D- | USB | USB native | Pini dedicati, conectati direct la USB-C prin USBLC6 |
| ANT | ANT1 | Antena BLE | Pin RF dedicat, conectat prin reteaua de matching |

**Nota:** Pinii ADC (P0.02-P0.05, P0.28-P0.31) care nu sunt folositi ca AIN sunt disponibili pentru extensii viitoare (ex: senzori analogici aditionali).

---

## Estimare consum energie

| Mod de operare | Componente active | Consum estimat |
|----------------|-------------------|----------------|
| Sleep (System OFF + RAM) | nRF RAM retained + RTC + BMA423 low-power | **~10 uA** |
| Idle (display static, BLE advertising) | nRF BLE adv + RTC + IMU | **~50 uA** |
| Active (refresh display) | nRF MCU on + SPI + EPD boost | ~8 mA (~500 ms per refresh) |
| BLE connected (data transfer) | nRF radio TX/RX @ 0 dBm | ~5-8 mA |
| Charging | BQ25180 la 500 mA | de la USB, nu de la baterie |

**Autonomie estimata:**  
Baterie tipica: ~110 mAh (AKY0106).  
Cu consum mediu ~50 uA (majoritatea timpului in idle + cateva refresh-uri de display pe ora):

T_autonomie = 110 mAh / 0.05 mA = 2200 ore ~ 90 zile

In practica, cu uzura normala (notificari, interactiuni cu butoanele, sync BLE), **durata de viata realista pe baterie este de ~2-4 saptamani**.

---

## Design mecanic

Dispozitivul este construit din:
- **PCB 1 mm grosime**, 4 layere (Top + 2 Inner GND + Bottom)
- **Carcasa** cu TopCase (rama + fereastra display) + BottomCase (compartiment baterie)
- **Baterie LiPo** plasata sub PCB in compartimentul BottomCase
- **Display e-paper** montat deasupra PCB, aliniat cu fereastra TopCase
- **Shaker motor** plasat lateral, conectat la PCB prin pad-uri TP_OUT+/OUT-
- **3 butoane fizice** aliniate cu SW1-SW3 pe PCB

Bateria se conecteaza direct la 2 test pad-uri (TP_BAT / TP_GND) in loc sa foloseasca conectorul JST original, pentru a economisi spatiu.

---

## Design log

### Decizii importante luate

1. **Alegerea stack-up-ului 4 layere:** desi un design 2-layer ar fi fost posibil, am optat pentru 4 layere cu doua planuri GND interne (Optiunea B). Motivele:
   - Return path continuu pentru semnalele SPI (display) si USB D+/D-
   - Performanta RF mai buna pentru circuitul BLE 2.4 GHz
   - Simplificare rutare - nu mai trebuie evitate via-urile pe trasee de putere
   - Ecranare EMI imbunatatita intre TOP si BOTTOM

2. **Rutarea 3V3 ca trasee groase (nu plane):** pentru ca 3V3 e consumator redus (< 100 mA) am preferat trasee groase (0.3 mm) pe TOP/BOTTOM in loc de un plane intern. Inner layers raman ambele GND.

3. **Antena la marginea PCB-ului + decupare:** am plasat antena 2450AT18B100E la marginea placii, cu zona de sub ea **decupata mecanic** si fara cupru pe niciun layer (conform datasheet si cerintei). In jurul antenei am adaugat via stitching pentru a lega planurile GND.

4. **Keepout pe toate cele 4 layere sub antena:** am desenat dreptunghiuri pe tRestrict, bRestrict si echivalentele pentru inner layers, plus decupare pe Dimension/Milling.

5. **Via stitching in zona RF:** plasate via-uri GND (0.3 mm drill / 0.6 mm diametru) la ~1.5 mm distanta in jurul nRF52840 si antenei pentru continuitate planuri GND.

6. **Plasarea condensatoarelor de decuplare:** toate cele 100 nF sunt plasate in capsula 0201 direct langa pinii VDD ai nRF, BMA423, DRV2605, BQ25180 si RT6160.

### Probleme intampinate

- **Rutare sub BGA nRF52840:** pinii interni ai BGA-ului au spatiere foarte mica; traseele de intrare/iesire din zona BGA au fost ingustate la 0.127 mm pentru a se incadra intre bile. Au aparut erori DRC "width too small" dar sunt acceptate conform cerintei.
- **Spatiu limitat pentru rutare:** placa de smartwatch fiind compacta (~35x40 mm), am trecut la 4 layere pentru a avea suficient spatiu de rutare.
- **Orientarea FPC-ului display-ului:** conectorul J2 a fost orientat astfel incat cablul flat sa aiba lungime minima pana la display, fara bucle.
- **Net classes aplicate dupa ce am inceput rutarea:** am adaugat clasa "Power" cu width 0.3 mm dupa ce rutasem deja cateva trasee la 0.15 mm. Am folosit comanda CHANGE WIDTH pentru a le ingrosa.


---

## Structura repository-ului

```
InkTime/
├── Hardware/
│   ├── InkTime_Schematic.sch     # Schematicul Fusion
│   ├── InkTime_Schematic.brd     # Board-ul Fusion
│   └── InkTime_Schematic.pdf     # Print-out schematic
├── Manufacturing/
│   ├── gerbers.zip               # Fisiere Gerber + Drill
│   ├── bom.txt                   # Bill of Materials
│   └── cpl.txt                   # Pick and Place
├── Mechanical/
│   ├── complete_device.step      # Model 3D exploded
│   └── complete_device.f3z       # Fisier Fusion Design complet
├── Images/
│   ├── block_diagram.png
│   ├── pcb_top.png
│   ├── pcb_bottom.png
│   ├── pcb_3d_top.png
│   └── device_3d.png
├── LICENSE                       # Apache 2.0
└── README.md
```

---

## Licenta

Acest proiect este licentiat sub **Apache License 2.0**. Vezi fisierul [LICENSE](LICENSE) pentru detalii.