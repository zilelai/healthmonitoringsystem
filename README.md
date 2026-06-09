Here's the step of making a portable health monitoring watch

1. Components:
   -ESP32C3 Supermini
   -MAX30102/MAX30105
   -DS18B20
   -TP4056
   -LiPo Battery
   -Slideswitch 3 pins
   -10K Potentiometer
   -6mm Switch
   -4.7K resistor
   -Touch Sensor Red Version
   -PCB Board
   -Female Header Pins
   -OLED Screen

2. Wire them (Breadboard/Perfboard)
   -Wire OLED Screen SCL/SCK to GPIO Pin 9 and SDA to GPIO Pin 8
   -Wire MAX30102/30105 Pins SCL and SDA like the OLED
   -Button to GPIO 10
   -WS2812B DIN Pin to Pin 4
   -DS18B20 to Pin 5
   -Potentiometer Wiper to GPIO 0
   -Touch Sensor SIG to GPIO 3
   -------- Battery ----------
   -LiPo Battery VCC to TP4056 B+ and GND to TP4056 B-
   -TP4056 OUT+ to 3.3V and OUT- to GND
   -Slideswitch middle to  GND and Terminal 1 to 5V and Terminal 2 to nothing!!!

3. PCB Board
   -Use the image in this repository to build yours (might include the file)
   -After receiving the board, solder them base on the image file

4. Happy using it :)

5. Might include a 3D Model
