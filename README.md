# Smart-Tracker
Tracker like a Pedometer 


Code 

#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// --- OLED Configuration (I2C Bus 0) ---
const int OLED_SDA = 18;
const int OLED_SCL = 17;
Adafruit_SSD1306 display(128, 64, &Wire, -1);

// --- MPU-6050 Configuration (I2C Bus 1) ---
const int MPU_SDA = 8;
const int MPU_SCL = 9;
TwoWire MPUWire = TwoWire(1);

const int LED_PIN = LED_BUILTIN; 
const int MPU_ADDR = 0x68;
const long THRESHOLD = 12000; 

int16_t prevAx, prevAy, prevAz, prevGx, prevGy, prevGz;

int16_t read16() {
  uint8_t hi = MPUWire.read();
  uint8_t lo = MPUWire.read();
  return (int16_t)((hi << 8) | lo);
}

void readSensor(int16_t &ax, int16_t &ay, int16_t &az, int16_t &gx, int16_t &gy, int16_t &gz) {
  MPUWire.beginTransmission(MPU_ADDR);
  MPUWire.write(0x3B);
  MPUWire.endTransmission(false);
  MPUWire.requestFrom(MPU_ADDR, 6, true);
  ax = read16(); ay = read16(); az = read16();

  MPUWire.beginTransmission(MPU_ADDR);
  MPUWire.write(0x41);
  MPUWire.endTransmission(false);
  MPUWire.requestFrom(MPU_ADDR, 2, true);
  read16();

  MPUWire.beginTransmission(MPU_ADDR);
  MPUWire.write(0x43);
  MPUWire.endTransmission(false);
  MPUWire.requestFrom(MPU_ADDR, 6, true);
  gx = read16(); gy = read16(); gz = read16();
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);

  Wire.begin(OLED_SDA, OLED_SCL);
  if(!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) { 
    Serial.println("SSD1306 allocation failed");
    for(;;);
  }
  
  MPUWire.begin(MPU_SDA, MPU_SCL);
MPUWire.beginTransmission(MPU_ADDR);
  MPUWire.write(0x6B);
  MPUWire.write(0x00);
  MPUWire.endTransmission(true);

  // Your pasted filter code:
  MPUWire.beginTransmission(MPU_ADDR);
  MPUWire.write(0x1A); 
  MPUWire.write(0x05); 
  MPUWire.endTransmission(true);

  delay(200);
  readSensor(prevAx, prevAy, prevAz, prevGx, prevGy, prevGz);
  
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 20);
  display.println("System Ready!");
  display.println("Waiting for motion...");
  display.display();
  
  Serial.println("Dual I2C Active! OLED on 18/17 | MPU on 8/9");
}

void loop() {
  int16_t ax, ay, az, gx, gy, gz;
  readSensor(ax, ay, az, gx, gy, gz);

  long motionDelta = abs((long)ax - prevAx) + abs((long)ay - prevAy) + abs((long)az - prevAz) +
                     abs((long)gx - prevGx) + abs((long)gy - prevGy) + abs((long)gz - prevGz);

  // Update previous readings to track continuous changes
  prevAx = ax; prevAy = ay; prevAz = az;
  prevGx = gx; prevGy = gy; prevGz = gz;

  if (motionDelta > THRESHOLD) {
    digitalWrite(LED_PIN, HIGH);
    Serial.println("\n--- MOTION DETECTED: STARTING 30-SECOND WINDOW ---");

    unsigned long startTime = millis();

    // 30-second looping window (30000 milliseconds)
    while (millis() - startTime < 30000) {
      readSensor(ax, ay, az, gx, gy, gz);

      // Check for CONTINUOUS motion while inside the loop
      long currentMotionDelta = abs((long)ax - prevAx) + abs((long)ay - prevAy) + abs((long)az - prevAz) +
                                abs((long)gx - prevGx) + abs((long)gy - prevGy) + abs((long)gz - prevGz);

      if (currentMotionDelta > THRESHOLD) {
        // Motion detected again! Reset the timer to keep the loop going.
        startTime = millis(); 
        Serial.println("--- Motion re-triggered! Timer reset to 30s ---");
      }

      // Update baseline for the next rapid check
      prevAx = ax; prevAy = ay; prevAz = az;
      prevGx = gx; prevGy = gy; prevGz = gz;

      // Print to Serial
      Serial.print("ACCEL [X:"); Serial.print(ax); Serial.print(" Y:"); Serial.print(ay); Serial.print(" Z:"); Serial.print(az);
      Serial.print("] | GYRO [X:"); Serial.print(gx); Serial.print(" Y:"); Serial.print(gy); Serial.print(" Z:"); Serial.print(gz);
      Serial.println("]");

      // Print to OLED
      display.clearDisplay();
      display.setCursor(0, 0);
      display.println("!! MOTION DETECTED !!");
      
      display.print("Time Left: ");
      display.print(30 - ((millis() - startTime) / 1000));
      display.println("s\n");

      display.print("A:"); display.print(ax); display.print(","); display.print(ay); display.print(","); display.println(az);
      display.print("G:"); display.print(gx); display.print(","); display.print(gy); display.print(","); display.println(gz);
      display.display();

      delay(100); 
    }

    // Only reaches here if 30 full seconds pass with NO motion
    digitalWrite(LED_PIN, LOW);
    Serial.println("--- 30 SECONDS COMPLETE NO MOTION. RETURNING TO SILENT STATE ---\n");
    
    // Reset OLED
    display.clearDisplay();
    display.setCursor(0, 20);
    display.println("System Ready!");
    display.println("Waiting for motion...");
    display.display();
    
    // Re-sync baseline before returning to main loop
    readSensor(prevAx, prevAy, prevAz, prevGx, prevGy, prevGz);
  }

  delay(50);
}
