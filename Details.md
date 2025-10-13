# **Konfigūruojamas Temperatūros Monitorius su Aliarmu**


Įrenginys realiu laiku matuoja aplinkos temperatūrą, lygina ją su vartotojo nustatytomis ribomis ir aktyvuoja aliarmą, jei temperatūra peržengia šias ribas. Sistema naudoja pertrauktis (interrupts) ir būsenų mašiną, kad būtų išvengta blokuojančių operacijų.

Visi nustatymai yra išsaugomi EEPROM atmintyje, todėl išlieka net ir atjungus maitinimą.

## **Techninė Įranga (Komponentai)**

* 1x Arduino Uno  
* 1x LM35 Temperatūros jutiklis  
* 1x Potenciometras (B50K)  
* 1x Mygtukas 
* 1x Žalias LED  
* 1x Raudonas LED  
* 2x ~220Ω Rezistoriai  
* 1x 1KΩ Rezistorius
* 1x Montažinė plokštė (Breadboard)  
* Jungiamieji laidai

## **Sujungimo Schema (Wiring)**

* **LM35 Temperatūros Jutiklis:**  
  * kairė kojelė -> 5V 
  * vidurinė kojelė -> A0
  * dešinė kojelė -> GND
* **Mygtukas:**  
  * Viena kojelė -> D2  
  * Antra kojelė -> GND per 1KΩ Rezistorių
  * Trečia kojelė -> 5V  
* **Potenciometras:**  
  * Viena kraštinė kojelė -> 5V
  * Kita kraštinė kojelė -> GND  
  * Vidurinė kojelė -> A1  
* **LED:** 
  * Žalias LED: Anodas (+) per ~220Ω rezistorių -> D4. Katodas (-) -> GND.  
  * Raudonas LED: Anodas (+) per ~220Ω rezistorių -> D5. Katodas (-) -> GND.

## **Sistemos Architektūra**

### **Taimerio Konfigūracija (Timer1)**

* Inicializavimas: Timer1.initialize(10000);  
* Paskirtis: Ši komanda sukonfigūruoja Timer1, kad vykdytu interrupt'a kas 10 milisekundžių.  
* Tikslas: Kiekvieną kartą suveikus šiam pertraukimui, yra iškviečiama timer_isr() funkcija.

### **ISR Rolės**


1. **void button_isr():**  
   * **Suveikimas:** Aktyvuojamas, kai mygtukas yra nuspaudžiamas (signalo kritimas iš HIGH į LOW).  
   * **Rolė:** Nedelsiant nustato vėliavėlę g_flag_button_pressed = true;. Praneša (loop()), kad įvyko paspaudimas.  
2. **void timer_isr():**  
   * **Suveikimas:** Aktyvuojamas periodiškai kas 10 ms.  
   * **Rolė:** Veikia kaip tvarkaraštis. Jis didina vidinį skaitiklį ir nustato skirtingas vėliavėles pagal intervalus:  
     * g_flag_read_temp (kas 500 ms): Signalizuoja, kad laikas nuskaityti temperatūrą.  
     * g_flag_toggle_led (kas 1000 ms): Signalizuoja, kad reikia perjungti LED būseną lėtam mirksėjimui.  
     * g_flag_fast_toggle_led (kas 250 ms): Signalizuoja, kad reikia perjungti LED būseną greitam mirksėjimui.

### **EEPROM Atminties Išdėstymas (Layout)**

Duomenys atmintyje išdėstomi tokia tvarka:

* Nuo adreso 0: Saugomas EEPROM_MAGIC_NUMBER. Tai yra unikalus skaičius, kuris patikrina, ar atmintis buvo anksčiau paruošta. Jis užima sizeof(int) baitų.  
* Po magiško skaičiaus: Saugoma high_threshold reikšmė (aukšta aliarmo riba). Ji užima sizeof(float) (2) baitų.  
* Po aukštos ribos: Saugoma low_threshold reikšmė (žema aliarmo riba). Ji taip pat užima sizeof(float) (4) baitų.