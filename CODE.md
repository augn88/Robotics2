#include <EEPROM.h>
#include <TimerOne.h> 

volatile bool g_flag_button_pressed = false;
volatile bool g_flag_read_temp = false;
volatile bool g_flag_toggle_led = false; 
volatile bool g_flag_fast_toggle_led = false; 

enum State { STATE_NORMAL, STATE_SET_HIGH, STATE_SET_LOW };
State currentState = STATE_NORMAL;

const int LM35_PIN = A0;
const int POTENTIOMETER_PIN = A1; 
const int GREEN_LED_PIN = 4;
const int RED_LED_PIN = 5;
const int BUTTON_PIN = 2;

unsigned long last_button_press_time = 0;
const long debounce_delay_ms = 100;

float current_temp_celsius = 0.0;
float high_threshold = 25.0; 
float low_threshold = 15.0;  

const int EEPROM_MAGIC_ADDR = 0;
const int EEPROM_HIGH_THRES_ADDR = sizeof(int);
const int EEPROM_LOW_THRES_ADDR = sizeof(int) + sizeof(float);
const int EEPROM_MAGIC_NUMBER = 0xACDC;


void button_isr() {
  g_flag_button_pressed = true;
}

void timer_isr() {
  static unsigned long timer_ms_counter = 0;
  timer_ms_counter += 10;

  if (timer_ms_counter % 500 == 0) {
    g_flag_read_temp = true;
  }
  if (timer_ms_counter % 1000 == 0) {
    g_flag_toggle_led = true;
  }
  if (timer_ms_counter % 250 == 0) {
    g_flag_fast_toggle_led = true;
  }
  if (timer_ms_counter >= 2000) {
      timer_ms_counter = 0;
  }
}

float read_lm35_celsius() {
  int analog_val = analogRead(LM35_PIN);
  float voltage = analog_val * (5.0 / 1024.0);
  return voltage * 100.0;
}

void check_and_set_alarm() {
  if (current_temp_celsius > high_threshold || current_temp_celsius < low_threshold) {
    digitalWrite(RED_LED_PIN, HIGH);
  } else {
    digitalWrite(RED_LED_PIN, LOW);
  }
}

void update_led_indicators() {
  static bool led_state = LOW;
  if (currentState == STATE_NORMAL && g_flag_toggle_led) {
    led_state = !led_state;
    digitalWrite(GREEN_LED_PIN, led_state);
    g_flag_toggle_led = false;
  } else if ((currentState == STATE_SET_HIGH || currentState == STATE_SET_LOW) && g_flag_fast_toggle_led) {
    led_state = !led_state;
    digitalWrite(GREEN_LED_PIN, led_state);
    g_flag_fast_toggle_led = false;
  }
}

void save_settings_to_eeprom() {
  EEPROM.put(EEPROM_HIGH_THRES_ADDR, high_threshold);
  EEPROM.put(EEPROM_LOW_THRES_ADDR, low_threshold);
  Serial.println("EEPROM: Nauji nustatymai išsaugoti.");
}



void setup() {
  Serial.begin(9600);
  Serial.println("Pradedamas darbas");

  pinMode(GREEN_LED_PIN, OUTPUT);
  pinMode(RED_LED_PIN, OUTPUT);
  pinMode(BUTTON_PIN, INPUT_PULLUP);

  attachInterrupt(digitalPinToInterrupt(BUTTON_PIN), button_isr, FALLING);
  Timer1.initialize(10000);
  Timer1.attachInterrupt(timer_isr);

  int magic_number_read;
  EEPROM.get(EEPROM_MAGIC_ADDR, magic_number_read);

  if (magic_number_read == EEPROM_MAGIC_NUMBER) {
    EEPROM.get(EEPROM_HIGH_THRES_ADDR, high_threshold);
    EEPROM.get(EEPROM_LOW_THRES_ADDR, low_threshold);
    Serial.print("EEPROM: Nustatymai rasti. Auksta riba: ");
    Serial.print(high_threshold);
    Serial.print(" C, Zema riba: ");
    Serial.print(low_threshold);
    Serial.println(" C");
  } else {
    Serial.println("EEPROM: Nerasti nustatymai. Naudojami numatytieji nustatymai.");
    EEPROM.put(EEPROM_MAGIC_ADDR, EEPROM_MAGIC_NUMBER);
    EEPROM.put(EEPROM_HIGH_THRES_ADDR, high_threshold);
    EEPROM.put(EEPROM_LOW_THRES_ADDR, low_threshold);
  }
}

void loop() {
  if (g_flag_read_temp) {
    g_flag_read_temp = false;
    current_temp_celsius = read_lm35_celsius();
    if (currentState == STATE_NORMAL) {
        check_and_set_alarm();
    }
  }

  if (g_flag_toggle_led || g_flag_fast_toggle_led) {
    update_led_indicators();
  }

  if (g_flag_button_pressed) {
    g_flag_button_pressed = false;

    unsigned long current_time = millis();

    if (current_time - last_button_press_time > debounce_delay_ms) {
      last_button_press_time = current_time;
      if (currentState == STATE_SET_HIGH || currentState == STATE_SET_LOW) {
        save_settings_to_eeprom();
      } 

      switch (currentState) {
       case STATE_NORMAL:   currentState = STATE_SET_HIGH; break;
       case STATE_SET_HIGH: currentState = STATE_SET_LOW;  break;
       case STATE_SET_LOW:  currentState = STATE_NORMAL;   break;
      }
      while(Serial.read() >= 0); 
    }
  }

  switch (currentState) {
    case STATE_NORMAL:
      Serial.print("Dabartine temp: ");
      Serial.print(current_temp_celsius, 1);
      Serial.print(" C (Ribos: ");
      Serial.print(low_threshold, 1);
      Serial.print("...");
      Serial.print(high_threshold, 1);
      Serial.println(" C)");
      delay(1000); 
      break;

    case STATE_SET_HIGH:
      { 
        int potValue = analogRead(POTENTIOMETER_PIN);
        high_threshold = map(potValue, 0, 1023, 0, 500) / 10.0;
        Serial.print("Nustatoma AUKSTA riba: ");
        Serial.print(high_threshold, 1);
        Serial.println(" C");
        delay(200); 
      }
      break;

    case STATE_SET_LOW:
      {
        int potValue = analogRead(POTENTIOMETER_PIN);
        low_threshold = map(potValue, 0, 1023, 0, 500) / 10.0;
        Serial.print("Nustatoma ZEMA riba: ");
        Serial.print(low_threshold, 1);
        Serial.println(" C");
        delay(200);
      }
      break;
  }
}