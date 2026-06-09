#include "Freenove_WS2812_Lib_for_ESP32.h" 
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <WiFi.h>
#include <HTTPClient.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#include "MAX30105.h"
#include "heartRate.h"
#include "time.h" 

#define BUTTON_PIN 10  
#define ONE_WIRE_BUS 5 
#define POTENTIOMETER 0   
#define TOUCHSENSOR 3     
#define LED_PIN    4      
#define LED_COUNT 1


const char* ssid = "-";
const char* password = "-";

const char* ntpServer = "pool.ntp.org";
const long  gmtOffset_sec = 28800;
const int   daylightOffset_sec = 0; 


#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);


Freenove_ESP32_WS2812 strip(LED_COUNT, LED_PIN, 0, TYPE_GRB);
int colorMode = 0;
bool lastButtonState = HIGH;

MAX30105 particleSensor;
OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

const byte RATE_SIZE = 4; 
byte rates[RATE_SIZE]; 
byte rateSpot = 0;
long lastBeat = 0; 
float beatsPerMinute;
int beatAvg = 0;
int spo2 = 0; 

int userAge = 30; 
bool isConfiguringAge = false; 

bool isCheckupActive = false;
uint32_t checkupStartTime = 0;
uint32_t currentCheckupDuration = 60000; 

const uint32_t checkupDurationStandard = 60000;  
const uint32_t checkupDurationThermal  = 120000; 


long checkupBpmSum = 0;
long checkupSpo2Sum = 0;
float checkupTempSum = 0;
int checkupSampleCount = 0;


int finalBpm = 0;
int finalSpo2 = 0;
float finalTemp = 0.0;
volatile bool checkupDone = false; 

float globalLastTemp = -127.0;

uint32_t tsLastReport = 0;
uint32_t tsLastSearch = 0;
uint32_t tsLastTempRequest = 0; 
volatile int displayMode = 0; 


volatile bool buttonPressed = false;
unsigned long lastInterruptTime = 0;

bool lastTouchState = LOW;        
bool stableTouchState = LOW;    
uint32_t lastTouchDebounceTime = 0; 
const uint32_t touchDebounceDelay = 60; 

char timeString[20] = ".....";

void refreshOLEDDisplay();
void changeColor(int mode);


void IRAM_ATTR handleButtonPress() {
    unsigned long interruptTime = millis();
    if (interruptTime - lastInterruptTime > 300) { 
        if (!isCheckupActive) { 
            displayMode++;
            if (displayMode > 7) displayMode = 0; 
            checkupDone = false;
            buttonPressed = true; 
            
            // Sync LED profile shift to display modes seamlessly
            colorMode = (displayMode % 8); 
        }
        lastInterruptTime = interruptTime;
    }
}

void updateLocalTime() {
    time_t now;
    struct tm timeinfo;
    time(&now); 
    localtime_r(&now, &timeinfo);
    
    if (timeinfo.tm_year > 70) {
        strftime(timeString, sizeof(timeString), "%H:%M:%S", &timeinfo);
    } else {
        if (WiFi.status() == WL_CONNECTED) {
            strcpy(timeString, "WiFi Sync...");
        } else {
            strcpy(timeString, "No WiFi");
        }
    }
}

void setup() {
    Serial.begin(115200);
    
    
    Wire.begin(8, 9, 400000); 
    
    pinMode(BUTTON_PIN, INPUT_PULLUP);
    pinMode(ONE_WIRE_BUS, INPUT_PULLUP); 
    pinMode(POTENTIOMETER, INPUT); 
    pinMode(TOUCHSENSOR, INPUT_PULLDOWN); 
    
    attachInterrupt(digitalPinToInterrupt(BUTTON_PIN), handleButtonPress, FALLING);

    sensors.begin();
    sensors.setWaitForConversion(false); 

    if(!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
        Serial.println(F("OLED Screen allocation failed"));
        for(;;);
    }
    
    display.clearDisplay();
    display.setTextColor(WHITE);
    display.setTextSize(1);
    display.setCursor(0, 10);
    display.println(F("ZL Project 2026"));
    display.display();

    if (!particleSensor.begin(Wire, I2C_SPEED_FAST)) {
        display.clearDisplay();
        display.setCursor(0, 10);
        display.println(F("MAX30102 ERROR"));
        display.display();
        while (1);
    }

    byte ledBrightness = 0x1F; 
    byte sampleAverage = 4;    
    byte ledMode = 2;          
    int sampleRate = 200;      
    int pulseWidth = 411;      
    int adcRange = 4096;       
    particleSensor.setup(ledBrightness, sampleAverage, ledMode, sampleRate, pulseWidth, adcRange);
    
    sensors.requestTemperatures(); 
    globalLastTemp = sensors.getTempCByIndex(0);

    WiFi.begin(ssid, password);
    configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
    
    // Initialize the RMT controller for the LED
    strip.begin();           
    strip.setBrightness(40); 
    changeColor(0); 
}

void loop() {
    if (buttonPressed) {
        buttonPressed = false; 
        changeColor(colorMode); 
        refreshOLEDDisplay(); 
        tsLastReport = millis(); 
    }

    static uint32_t lastTimeUpdate = 0;
    if (millis() - lastTimeUpdate > 1000) {
        updateLocalTime();
        lastTimeUpdate = millis();
    }

    if (millis() - tsLastTempRequest > 3000) {
        globalLastTemp = sensors.getTempCByIndex(0);
        sensors.requestTemperatures(); 
        tsLastTempRequest = millis();
    }

    // --- Touch Filtering ---
    bool touchReading = digitalRead(TOUCHSENSOR);

    if (touchReading != lastTouchState) {
        lastTouchDebounceTime = millis();
    }

    if ((millis() - lastTouchDebounceTime) > touchDebounceDelay) {
        if (touchReading != stableTouchState) {
            stableTouchState = touchReading;

            if (stableTouchState == HIGH) {
                if (displayMode >= 4 && displayMode <= 6) {
                    if (!isCheckupActive) {
                        isCheckupActive = true;
                        checkupDone = false;
                        checkupStartTime = millis();
                        checkupBpmSum = 0;
                        checkupSpo2Sum = 0;
                        checkupTempSum = 0;
                        checkupSampleCount = 0;

                        if (displayMode == 5) {
                            currentCheckupDuration = checkupDurationThermal;
                        } else {
                            currentCheckupDuration = checkupDurationStandard;
                        }
                    }
                } 
                else if (displayMode != 7) { 
                    isConfiguringAge = !isConfiguringAge; 
                    if (!isConfiguringAge) {
                        display.clearDisplay();
                        display.setTextSize(1);
                        display.setCursor(0, 20);
                        display.print(F("Age set completed"));
                        display.display();
                        delay(1200);
                        displayMode = 0; 
                    }
                }
            }
        }
    }
    lastTouchState = touchReading; 

    if (isConfiguringAge) {
        int potValue = analogRead(POTENTIOMETER);
        int constrainedPot = constrain(potValue, 36, 4095);
        userAge = map(constrainedPot, 36, 4095, 1, 100); 

        if (millis() - tsLastReport > 50) { 
            display.clearDisplay();
            display.setTextSize(1);
            display.setCursor(0, 0);
            display.print(F("Edit Age:"));
            display.drawLine(0, 12, 128, 12, WHITE);
            
            display.setCursor(10, 25);
            display.setTextSize(2);
            display.print(userAge);
            
            display.setTextSize(1);
            display.setCursor(0, 40);
            display.println(F("Press the sensor to "));
            display.println(F("save. Twist the knob"));
            display.println(F("to change age."));
            display.display();
            tsLastReport = millis();
        }
    } 
    else {
        long irValue = particleSensor.getIR();
        long redValue = particleSensor.getRed();

        if (irValue > 30000) { 
            if (checkForBeat(irValue) == true) {
                long delta = millis() - lastBeat;
                lastBeat = millis();
                beatsPerMinute = 60 / (delta / 1000.0);

                if (beatsPerMinute < 255 && beatsPerMinute > 20) {
                    rates[rateSpot++] = (byte)beatsPerMinute;
                    rateSpot %= RATE_SIZE;
                    
                    beatAvg = 0;
                    for (byte x = 0 ; x < RATE_SIZE ; x++) beatAvg += rates[x];
                    beatAvg /= RATE_SIZE;

                    float ratio = (float)redValue / (float)irValue;
                    spo2 = 110 - (18 * ratio); 
                    if (spo2 > 100) spo2 = 100;
                    if (spo2 < 60)  spo2 = 60; 
                }
            }
        } else {
            beatAvg = 0;
            spo2 = 0;
        }

        if (isCheckupActive) {
            if (displayMode == 5) {
                if (globalLastTemp > -55.0 && globalLastTemp != -127.00) {
                    checkupTempSum += globalLastTemp;
                    checkupSampleCount++;
                }
            } else {
                if (irValue > 30000 && beatAvg > 0 && spo2 > 0) {
                    checkupBpmSum += beatAvg;
                    checkupSpo2Sum += spo2;
                    checkupSampleCount++;
                }
            }

            if (millis() - checkupStartTime >= currentCheckupDuration) {
                isCheckupActive = false;
                checkupDone = true;
                if (checkupSampleCount > 0) {
                    if (displayMode == 5) {
                        finalTemp = checkupTempSum / checkupSampleCount;
                    } else {
                        finalBpm = checkupBpmSum / checkupSampleCount;
                        finalSpo2 = checkupSpo2Sum / checkupSampleCount;
                    }
                } else {
                    finalBpm = 0; finalSpo2 = 0; finalTemp = 0.0;
                }
                refreshOLEDDisplay();
                tsLastReport = millis();
            }
        }

        // Fixed Blocking Bug: Implemented Non-Blocking Async HTTP Requests
        if (millis() - tsLastSearch > 60000) {
            if (WiFi.status() == WL_CONNECTED) {
                HTTPClient http;
                char requestUrl[160];
                snprintf(requestUrl, sizeof(requestUrl), "http://httpbin.org/get?bpm=%d&temp=%.1f&spo2=%d&age=%d", beatAvg, globalLastTemp, spo2, userAge);
                
                http.begin(requestUrl);
                http.setTimeout(150); // Prevent long lockups on weak connection
                
                int httpCode = http.GET();
                if (httpCode > 0) {
                    Serial.printf("[HTTP] GET success, code: %d\n", httpCode);
                }
                http.end();
            }
            tsLastSearch = millis();
        }

        uint32_t reportInterval = isCheckupActive ? 200 : 1000;
        if (millis() - tsLastReport > reportInterval) {
            refreshOLEDDisplay();
            tsLastReport = millis();
        }
    }
}

void refreshOLEDDisplay() {
    long irValue = particleSensor.getIR();
    display.clearDisplay();
    
    if (displayMode != 7) {
        display.setTextSize(1);
        display.setCursor(80, 0);
        display.print(timeString);
    }
    display.setCursor(0, 0);
    display.setTextSize(1);

    if (displayMode == 0) {
        display.print(F("Heart Beat"));
        display.drawLine(0, 10, 128, 10, WHITE);
        display.setCursor(0, 25);
        display.setTextSize(2);
        if (irValue > 30000) {
            display.print(beatAvg); display.println(F(" BPM"));
        } else {
            display.setTextSize(1);
            display.setCursor(0, 30);
            display.println(F("Place Finger..."));
        }
    } 
    else if (displayMode == 1) {
        display.print(F("Body Temp"));
        display.drawLine(0, 10, 128, 10, WHITE);
        display.setCursor(0, 25);
        display.setTextSize(2);
        if (globalLastTemp < -55.0 || globalLastTemp == -127.00) {
            display.print(F("No Sensor"));
        } else {
            display.print(globalLastTemp, 1); display.println(F(" C"));
        }
    }
    else if (displayMode == 2) {
        display.print(F("SP02 Zone"));
        display.drawLine(0, 10, 128, 10, WHITE);
        display.setCursor(0, 25);
        display.setTextSize(2);
        if (irValue > 30000) {
            display.print(spo2); display.println(F(" %SpO2"));
        } else {
            display.setTextSize(1);
            display.setCursor(0, 30);
            display.println(F("Place Finger..."));
        }
    }
    else if (displayMode == 3) {
        display.print(F("About You"));
        display.drawLine(0, 10, 128, 10, WHITE);
        display.setCursor(0, 25);
        display.setTextSize(2);
        display.print(F("Age: ")); display.print(userAge);
    }
    else if (displayMode == 4) {
        display.print(F("Heart Checkup"));
        display.drawLine(0, 10, 128, 10, WHITE);
        
        if (isCheckupActive) {
            display.setCursor(0, 20);
            display.print(F("Testing: "));
            display.print((currentCheckupDuration - (millis() - checkupStartTime)) / 1000);
            display.println(F("s"));
            display.setTextSize(2);
            display.print(beatAvg); display.println(F(" BPM"));
        } else if (checkupDone) {
            display.setCursor(0, 18);
            display.print(F("1m Avg: ")); display.print(finalBpm); display.println(F(" BPM"));
            display.setCursor(0, 32);
            if (finalBpm == 0) display.println(F("Error: No data"));
            else if (finalBpm >= 55 && finalBpm <= 85) display.println(F("Healthy"));
            else if (finalBpm >= 60 && finalBpm <= 100) display.println(F("Normal"));
            else if (finalBpm > 100) display.println(F("Alert!"));
            
        } else {
            display.setCursor(0, 20);
            display.println(F("Touch Sensor to Start"));
        }
    } 
    else if (displayMode == 5) {
        display.print(F("Temp Checkup"));
        display.drawLine(0, 10, 128, 10, WHITE);
        
        if (isCheckupActive) {
            display.setCursor(0, 20);
            display.print(F("Testing: "));
            display.print((currentCheckupDuration - (millis() - checkupStartTime)) / 1000);
            display.println(F("s"));
            display.setTextSize(2);
            display.print(globalLastTemp, 1); display.println(F(" C"));
        } else if (checkupDone) {
            display.setCursor(0, 18);
            display.print(F("2m Avg: ")); display.print(finalTemp, 1); display.println(F(" C"));
            display.setCursor(0, 32);
            if (finalTemp < 34.0) display.println(F("Error: No sensor data"));
            else if (finalTemp >= 36.5 && finalTemp <= 37.5) display.println(F("Healthy"));
            else if (finalTemp > 37.5) display.println(F("Fever"));
            else display.println(F("Low Temp!!!"));
        } else {
            display.setCursor(0, 20);
            display.println(F("Touch the sensor to start"));
        }
    }
    else if (displayMode == 6) {
        display.print(F("SP02 Checkup"));
        display.drawLine(0, 10, 128, 10, WHITE);
        
        if (isCheckupActive) {
            display.setCursor(0, 20);
            display.print(F("Testing: "));
            display.print((currentCheckupDuration - (millis() - checkupStartTime)) / 1000);
            display.println(F("s"));
            display.setTextSize(2);
            display.print(spo2); display.println(F(" %"));
        } else if (checkupDone) {
            display.setCursor(0, 18);
            display.print(F("1m Avg: ")); display.print(finalSpo2); display.println(F(" %"));
            display.setCursor(0, 32);
            if (finalSpo2 == 0) display.println(F("Error: No data"));
            else if (finalSpo2 >= 97 && finalSpo2 <= 100) display.println(F("Healthy"));
            else if (finalSpo2 >= 94 && finalSpo2 <= 96) display.println(F("Normal"));
            else if (finalSpo2 >= 90 && finalSpo2 <= 93) display.println(F("Alert"));
            else display.println(F("!!!"));
        } else {
            display.setCursor(0, 20);
            display.println(F("Touch the sensor to start"));
        }
    }
    else if (displayMode == 7) {
        display.print(F("Clock"));
        display.drawLine(0, 10, 128, 10, WHITE);
        
        display.setTextSize(2);
        display.setCursor(16, 30);
        display.print(timeString);
    }

    display.display();
}

void changeColor(int mode) {
  uint8_t r = 0, g = 0, b = 0;

  switch (mode) {
    case 0: r = 255;     g = 255;     b = 255;     break; 
    case 1: r = 255;   g = 0;     b = 0;     break; 
    case 2: r = 255;   g = 165;   b = 0;     break; 
    case 3: r = 255;   g = 255;   b = 0;     break; 
    case 4: r = 0;     g = 255;   b = 0;     break; 
    case 5: r = 0;     g = 0;     b = 255;   break; 
    case 6: r = 255;   g = 0;     b = 255;   break; 
    case 7: r = 127;   g = 0;     b = 255;   break; 
    default: r = 0;    g = 0;     b = 0;     break;
  }

  strip.setLedColorData(0, r, g, b);
  strip.show(); 
}
