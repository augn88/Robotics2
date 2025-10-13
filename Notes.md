# **Projekto Pastabos: Temperatūros Monitorius**

## **1\.Design**

Sistema veikia asinchroniniu, įvykiais valdomu modeliu, kad būtų išvengta blokuojančių operacijų. Architektūrą sudaro:

* **Interrupts:** button_isr (išorinis) ir timer_isr (periodinis) vykdo visą darbą su volatile vėliavėliamis.  
* **State Machine:** Valdo tris režimus (NORMAL, SET_HIGH, SET_LOW), reaguodama į vėliavėles ir veiksmus.

## **2\. Timing Budget**

Sistemos pagrindinis taktas yra 10 ms, generuojamas Timer1. Kitos operacijos užima neženklius laikus.

## **3\. Test/Accuracy Results**

Visi pagrindiniai moduliai buvo sėkmingai patikrinti: asinchroninis veikimas, būsenų perjungimas, aliarmo suveikimas ir EEPROM atminties išsaugojimas.

## **4\. Žinomos Problemos ir Patobulinimai (Known Issues)**

2. **Ribos:** Sistema leidžia nustatyti žemą ribą didesnę už aukštą. 
3. **LCD:** Galima pridėti LCD ekraną temperatūros rodymui.
