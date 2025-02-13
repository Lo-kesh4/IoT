# IOT-BASED AIR QUALITY MONITORING SYSTEM using CAN
### Create an IoT-based air quality monitoring system using gas sensors, a PM Sensor, a Temperature and Humidity sensor, and a microcontroller with CAN Bus. The system should monitor pollutants in the air and send the data to an IoT platform for remote access.
#### Air pollution has become a significant concern affecting human health and the environment. To address this issue, we propose an IoT-based Air Quality Monitoring System utilizing the Controller Area Network (CAN) protocol for reliable and efficient data transmission. The system comprises two ESP32 microcontrollers, where the first ESP32 collects real-time air quality data using multiple sensors, including the Nova PM SDS011 (for particulate matter), DHT22 (for temperature and humidity), and MQ135 (for detecting harmful gases). The collected data is displayed on a 0.96-inch OLED screen and transmitted via the MCP2515 CAN module to the second ESP32. The second ESP32 receives the data over CAN communication, processes it, and sends it to ThingSpeak, a cloud-based IoT analytics platform, for remote monitoring and analysis. This system ensures low-latency, robust, and interference-free communication using the CAN protocol, making it suitable for industrial and environmental applications. The proposed system provides real-time monitoring, remote accessibility, and improved data reliability, making it a cost-effective solution for air quality assessment.
## System Architecture & Components
The IoT-Based Air Quality Monitoring System using CAN Protocol is designed with a distributed two-node architecture to efficiently collect, transmit, and display real-time air quality data. The system consists of two ESP32 microcontroller-based nodes:
  1. **Sensor Node** – Collects air quality data from various sensors and transmits it using the CAN protocol.
  2. **Receiver Node** – Receives the sensor data, processes it, and uploads it to ThingSpeak for remote monitoring.
This architecture ensures efficient, reliable, and noise-resistant communication while allowing scalability for future expansion.
### Components of the System Architecture
#### Sensor Node (ESP32 + Sensors + CAN Transmitter)
The Sensor Node is responsible for collecting air quality parameters and transmitting the data using the MCP2515 CAN module. It consists of:
* **ESP32 Microcontroller** – Manages sensor data processing and CAN communication.
* **Nova PM Sensor SDS011** – Measures PM2.5 and PM10 particulate matter concentrations.
* **DHT22 Sensor** – Captures temperature and humidity data.
* **MQ135 Gas Sensor** – Detects harmful gases such as CO₂, NH₃, benzene, and VOCs.
* **0.96-inch OLED Display** – Displays real-time air quality readings for local monitoring.
* **MCP2515 CAN Controller** – Converts sensor data into CAN messages and transmits them to the Receiver Node.
#### Receiver Node (ESP32 + CAN Receiver + IoT Integration)
The Receiver Node is responsible for receiving data via CAN, processing it, and sending it to the cloud. It consists of:
* **ESP32 Microcontroller** – Decodes receive CAN messages and forward the data to the cloud.
* **MCP2515 CAN Controller** – Receives sensor data from the Sensor Node.
* **Wi-Fi Module (Built-in ESP32)** – Connects to the internet and uploads data to ThingSpeak.

### **Flow Chart**
![image](https://github.com/user-attachments/assets/49e1ab26-20b9-4c06-b387-b4c295a2814e)
### **Block Diagram**
![image](https://github.com/user-attachments/assets/43bac73e-c3c0-4c82-a5ef-6e201ed7f235)
### **Circuit Diagram**
![image](https://github.com/user-attachments/assets/effc88c2-818e-4777-a44c-d3d693c6a3c3)

## Transmitter Code
```
#include <Wire.h>
#include <Adafruit_SSD1306.h>
#include <DHT.h>
#include <mcp_can.h>
#include <SPI.h>
#include <HardwareSerial.h>
#include <SDS011.h>
#include <Arduino.h>
// OLED Display Setup
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

// DHT22 Sensor
#define DHTPIN 4
#define DHTTYPE DHT22
DHT dht(DHTPIN, DHTTYPE);

// MQ135 Sensor
#define MQ135_PIN 34
#define RL_VALUE 10.0 // Load resistance in kΩ
#define CLEAN_AIR_RATIO 3.6 // Rs/R0 ratio in clean air

float R0 = 6.67; // Manually calibrated R0 based on clean air Rs readings

// SDS011 Sensor
HardwareSerial mySerial(1);
SDS011 my_sds;
#define SDSG1 26
#define SDSG2 27
// MCP2515 CAN Module
#define SPI_CS_PIN 5
MCP_CAN CAN(SPI_CS_PIN);

// Global Sensor Variables
float pm25, pm10, temperature, humidity;

void setup() {
    Serial.begin(115200);
    Wire.begin();

    // Initialize OLED
    if (!display.begin(SSD1306_BLACK, 0x3C)) {
        Serial.println("SSD1306 allocation failed");
        for (;;);
    }
    display.clearDisplay();

    // Initialize DHT Sensor
    dht.begin();

    // Initialize SDS011
mySerial.begin(9600, SERIAL_8N1, SDSG1, SDSG2);  // Initialize UART1 with RX=16, TX=17

    my_sds.begin(SDSG1, SDSG2);

    // Initialize CAN
    if (CAN.begin(MCP_ANY, CAN_500KBPS, MCP_16MHZ) == CAN_OK) {
        Serial.println("CAN Module Initialized Successfully!");
    } else {
        Serial.println("CAN Module Initialization Failed!");
        while (1);
    }
    CAN.setMode(MCP_NORMAL);
}

void loop() {
    // Read SDS011 Data
    int status = my_sds.read(&pm25, &pm10);
    pm25=pm25/10;
    pm10=pm10/10;
    // Read DHT22 Data
    temperature = dht.readTemperature();
    humidity = dht.readHumidity();

    // Read MQ135 Sensor
    //airQuality = analogRead(MQ135_PIN);
    float Rs = getResistance();
    float airQuality = getPPM();
    

    // Display Data on OLED
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(0, 0);
    display.printf("PM2.5: %.1f ug/m3\nPM10: %.1f ug/m3\nTemp: %.1f C\nHumidity: %.1f%%\nAQ: %.1f\n", pm25, pm10, temperature, humidity, airQuality);
   display.setCursor(0, 40);
    if( pm25 < 30.00 && pm10 < 50.00 && airQuality < 400.00){
      display.printf("Air Quality is Good");
      }
    else{
      display.printf("Air Quality is not Good");
      }

     display.display();
    // Transmit Data via CAN
  int airQualityInt = (int)airQuality;  // Convert float to integer
byte canData[8] = { (byte)pm25, (byte)pm10, (byte)temperature, (byte)humidity, (byte)(airQualityInt / 256), (byte)(airQualityInt % 256), 0, 0 };

    CAN.sendMsgBuf(0x100, 0, 8, canData);
    Serial.println("Data sent");
    delay(2000);  // Update every 2 seconds
}

// Function to read ADC and calculate Rs
float getResistance() {
    int adc_value = analogRead(MQ135_PIN);
    float voltage = adc_value * (3.3 / 4095.0); // Convert ADC to voltage
    if (voltage == 0) return 999999; // Avoid division by zero
    float Rs = ((3.3 * RL_VALUE) / voltage) - RL_VALUE;
    return Rs;
}

// Function to calculate CO2 PPM using the improved formula
float getPPM() {
    float Rs = getResistance();
    float ratio = Rs / R0; // Rs/R0 ratio
    float ppm = 1000 * pow(ratio, -1.5); // Updated formula for CO2
    return ppm;
}
```

## Receiver Code
```
#include <SPI.h>
#include <mcp_can.h>
#include <WiFi.h>
#include <HTTPClient.h>

#define SPI_CS_PIN 5
MCP_CAN CAN(SPI_CS_PIN);

// Wi-Fi Credentials
const char* ssid = "<username>";
const char* password = "<passed>";

// ThingSpeak API Key
const char* server = "http://api.thingspeak.com/update";
const char* apiKey = "V2U3S1E00TEBB8TQ";

void setup() {
    Serial.begin(115200);
    
    // Connect to Wi-Fi
    WiFi.begin(ssid, password);
    while (WiFi.status() != WL_CONNECTED) {
        Serial.print(".");
        delay(500);
    }
    Serial.println("\nWi-Fi Connected");




    // Initialize CAN Module
    if (CAN.begin(MCP_ANY, CAN_500KBPS, MCP_16MHZ) == CAN_OK) {
        Serial.println("CAN Module Initialized Successfully!");
    } else {
        Serial.println("CAN Module Initialization Failed!");
        while (1);
    }
    CAN.setMode(MCP_NORMAL);
}

void loop() {
    long unsigned int rxId;
    unsigned char len = 0;
    unsigned char rxBuf[8];

    if (CAN.checkReceive() == CAN_MSGAVAIL) {
        CAN.readMsgBuf(&rxId, &len, rxBuf);

        float pm25 = rxBuf[0];
        float pm10 = rxBuf[1];
        float temperature = rxBuf[2];
        float humidity = rxBuf[3];
        float airQuality = (rxBuf[4] << 8) | rxBuf[5];

        Serial.printf("Received: PM2.5=%.1f, PM10=%.1f, Temp=%.1f, Humidity=%.1f, AQ=%.1f\n", pm25, pm10, temperature, humidity, airQuality);

        // Send Data to ThingSpeak
        if (WiFi.status() == WL_CONNECTED) {
            HTTPClient http;
            String url = String(server) + "?api_key=" + apiKey + "&field1=" + pm25 + "&field2=" + pm10 + "&field3=" + temperature + "&field4=" + humidity + "&field5=" + airQuality;
            http.begin(url);
            int httpCode = http.GET();
            http.end();
        }
    }
}
```

### Working Principle
The IoT-Based Air Quality Monitoring System using CAN Protocol operates through a structured data collection, processing, transmission, and cloud integration approach. The system consists of two main nodes Sensor Node and Receiver Node which communicate using the Controller Area Network (CAN) protocol. The Sensor Node collects environmental data from various sensors, transmits it via CAN, and the Receiver Node processes this data before sending it to the ThingSpeak cloud platform for remote monitoring.
  1. **Sender Node Operation**
     * **Sensor Data Collection**:
         * The Nova PM SDS011 sensor measures PM2.5 and PM10 particulate matter concentrations in the air.
         * The DHT22 sensor records temperature and humidity levels.
         * The MQ135 gas sensor detects air pollutants such as CO₂, NH₃,NO₂, benzene, and VOCs.
     * **Data Processing**:
         * The ESP32 reads the analog and digital outputs from the sensors.
         * The sensor values are converted into readable digital data.
         * The formatted sensor readings are displayed on a 0.96-inch OLED screen for local monitoring.
     * **Data Transmission via CAN Protocol**:
         * The ESP32 sends the collected data to the MCP2515 CAN Controller, which converts the sensor readings into CAN messages.
         * The MCP2515 CAN transceiver transmits these messages over the CAN bus to the Receiver Node.
         * Each CAN message consists of a unique identifier, sensor values, and error-checking bits to ensure reliable transmission.
  2. **Receiver Node Operation**
     * **Receiving CAN Data**:
         * The Receiver Node consists of another ESP32 microcontroller and an MCP2515 CAN module.
         * The MCP2515 transceiver receives the transmitted CAN messages from the Sensor Node.
         * The ESP32 extracts the sensor values from the CAN frame and processes them.
     * **Data Processing & Validation**:
         * The ESP32 checks for error bits and message integrity to ensure accurate data reception.
         * If errors are detected, the system requests retransmission using built-in CAN error-handling mechanisms.
     * **Wi-Fi-Based Cloud Transmission**:
         * The Receiver Node connects to the internet via Wi-Fi (built-in ESP32 module).
         * Sensor values are formatted into HTTP requests and sent to ThingSpeak.
         * The ThingSpeak platform processes and displays the real-time air quality data in graphical form, accessible from any web browser or mobile device.
