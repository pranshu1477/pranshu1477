#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#include <Adafruit_Sensor.h>
#include <Adafruit_BME280.h>
#include "MAX30105.h"
#include "spo2_algorithm.h"
 
Adafruit_BME280 bme; // I2C
 
#define DS18B20 5
OneWire oneWire(DS18B20);
DallasTemperature sensors(&oneWire);
 
#define REPORTING_PERIOD_MS     1000
uint32_t tsLastReport = 0;
 
MAX30105 particleSensor;
#define MAX_BRIGHTNESS 255 
uint32_t irBuffer[100]; //infrared LED sensor data
uint32_t redBuffer[100];  //red LED sensor data
int32_t bufferLength; //data length
int32_t spo2; //SPO2 value
int8_t validSPO2; //indicator to show if the SPO2 calculation is valid
int32_t heartRate; //heart rate value
int8_t validHeartRate; //indicator to show if the heart rate calculation is valid
byte pulseLED = 11; //Must be on PWM pin
byte readLED = 13; //Blinks with each data read 
 
float temperature, humidity, BPM, SpO2, bodytemperature;
 
/*Put your SSID & Password*/
const char* ssid = "circuitschools";  // Enter SSID here
const char* password = "password";  //Enter Password here
 
WebServer server(80);             
 
void setup() {
  Serial.begin(9600);  
 
  Serial.println("Connecting to ");
  Serial.println(ssid);
 
  //connect to your local wi-fi network
  WiFi.begin(ssid, password);
 
  //check wi-fi is connected to wi-fi network
  while (WiFi.status() != WL_CONNECTED) {
  delay(1000);
  Serial.print(".");
  }
  Serial.println("");
  Serial.println("WiFi connected..!");
  Serial.print("Got IP: ");  Serial.println(WiFi.localIP());
 
  server.on("/", handle_OnConnect);
  server.onNotFound(handle_NotFound);
 
  server.begin();
  Serial.println("HTTP server started");
  
  sensors.begin();  //initialize DS18B20 sensor
  
  if (!bme.begin(0x76)) {   //if still not found try another address or scan through I2C scanner from our prevoius posts
    Serial.println(F("Could not find a valid BME280 sensor, check wiring!"));
    //while (1);
  }
 
  Serial.print("Initializing pulse oximeter..");
 
  pinMode(pulseLED, OUTPUT);
  pinMode(readLED, OUTPUT);
 
  // Initialize max30102 sensor
  if (!particleSensor.begin(Wire, I2C_SPEED_FAST)) //Use default I2C port, 400kHz speed
  {
    Serial.println(F("MAX30102 was not found. Please check wiring/power."));
    //while (1);
  }
 
  Serial.println(F("Attach sensor to finger with rubber band. Press any key to start conversion"));
  while (Serial.available() == 0) ; //wait until user presses a key
  Serial.read();
 
  byte ledBrightness = 60; //Options: 0=Off to 255=50mA
  byte sampleAverage = 4; //Options: 1, 2, 4, 8, 16, 32
  byte ledMode = 2; //Options: 1 = Red only, 2 = Red + IR, 3 = Red + IR + Green
  byte sampleRate = 100; //Options: 50, 100, 200, 400, 800, 1000, 1600, 3200
  int pulseWidth = 411; //Options: 69, 118, 215, 411
  int adcRange = 4096; //Options: 2048, 4096, 8192, 16384
 
  particleSensor.setup(ledBrightness, sampleAverage, ledMode, sampleRate, pulseWidth, adcRange); //Configure sensor with these settings
 
}
void loop() {
  server.handleClient();
  }
void handle_OnConnect() { 
  sensors.requestTemperatures();
  
  
  bufferLength = 100; //buffer length of 100 stores 4 seconds of samples running at 25sps
 
  //read the first 100 samples, and determine the signal range
  for (byte i = 0 ; i < bufferLength ; i++)
  {
    while (particleSensor.available() == false) //do we have new data?
      particleSensor.check(); //Check the sensor for new data
 
    redBuffer[i] = particleSensor.getRed();
    irBuffer[i] = particleSensor.getIR();
    particleSensor.nextSample(); //We're finished with this sample so move to next sample
 
    Serial.print(F("red="));
    Serial.print(redBuffer[i], DEC);
    Serial.print(F(", ir="));
    Serial.println(irBuffer[i], DEC);
  }
 
  //calculate heart rate and SpO2 after first 100 samples (first 4 seconds of samples)
  maxim_heart_rate_and_oxygen_saturation(irBuffer, bufferLength, redBuffer, &spo2, &validSPO2, &heartRate, &validHeartRate);
 
  //Continuously taking samples from MAX30102.  Heart rate and SpO2 are calculated every 1 second
  while (1)
  {
    //dumping the first 25 sets of samples in the memory and shift the last 75 sets of samples to the top
    for (byte i = 25; i < 100; i++)
    {
      redBuffer[i - 25] = redBuffer[i];
      irBuffer[i - 25] = irBuffer[i];
    }
 
    //take 25 sets of samples before calculating the heart rate.
    for (byte i = 75; i < 100; i++)
    {
      while (particleSensor.available() == false) //do we have new data?
        particleSensor.check(); //Check the sensor for new data
 
      digitalWrite(readLED, !digitalRead(readLED)); //Blink onboard LED with every data read
 
      redBuffer[i] = particleSensor.getRed();
      irBuffer[i] = particleSensor.getIR();
      particleSensor.nextSample(); //We're finished with this sample so move to next sample
 
      //send samples and calculation result to terminal program through UART
      Serial.print(F("red="));
      Serial.print(redBuffer[i], DEC);
      Serial.print(F(", ir="));
      Serial.print(irBuffer[i], DEC);
 
      Serial.print(F(", HR="));
      Serial.print(heartRate, DEC);
 
      Serial.print(F(", HRvalid="));
      Serial.print(validHeartRate, DEC);
 
      Serial.print(F(", SPO2="));
      Serial.print(spo2, DEC);
 
      Serial.print(F(", SPO2Valid="));
      Serial.println(validSPO2, DEC);
    }
 
    //After gathering 25 new samples recalculate HR and SP02
    maxim_heart_rate_and_oxygen_saturation(irBuffer, bufferLength, redBuffer, &spo2, &validSPO2, &heartRate, &validHeartRate);
  }
  
  
  temperature = bme.readTemperature();
  humidity = bme.readHumidity()/100; 
  BPM = heartRate;
  SpO2 = spo2;
  bodytemperature = (sensors.getTempCByIndex(0)*9.0)/5.0+32.0; //Converting into Farenheit
 
  
  if (millis() - tsLastReport > REPORTING_PERIOD_MS) 
  {
    Serial.print("Room Temperature: ");
    Serial.print(bme.readTemperature());
    Serial.println("°C");
    
    Serial.print("Room Humidity: ");
    Serial.print(bme.readPressure()/100);
    Serial.println("%");
    
    Serial.print("BPM: ");
    Serial.println(heartRate);
    
    Serial.print("SpO2: ");
    Serial.print(spo2);
    Serial.println("%");
 
    Serial.print("Body Temperature: ");
    Serial.print(bodytemperature);
    Serial.println("°C");
    
    Serial.println("*********************************");
    Serial.println();//for more details visit circuitschools.com
 
    tsLastReport = millis();
  }
  
  server.send(200, "text/html", SendHTML(temperature, humidity, BPM, SpO2, bodytemperature)); 
}
 
void handle_NotFound(){
  server.send(404, "text/plain", "Not found");
}
 
  String SendHTML(float temperature,float humidity,float BPM,float SpO2, float bodytemperature){
  String ptr = "<!DOCTYPE html>";
  ptr +="<html>";
  ptr +="<head>";
  ptr +="<title>IoT based Patient Health Monitoring system</title>";
  ptr +="<meta name='viewport' content='width=device-width, initial-scale=1.0'>";
  ptr +="<link href='https://fonts.googleapis.com/css?family=Open+Sans:300,400,600' rel='stylesheet'>";
  ptr +="<style>";
  ptr +="html { font-family: 'Open Sans', sans-serif; display: block; margin: 0px auto; text-align: center;color: #444444;}";
  ptr +="body{margin: 0px;} ";
  ptr +="h1 {margin: 50px auto 30px;} ";
  ptr +=".side-by-side{display: table-cell;vertical-align: middle;position: relative;}";
  ptr +=".text{font-weight: 600;font-size: 19px;width: 200px;}";
  ptr +=".reading{font-weight: 300;font-size: 50px;padding-right: 25px;}";
  ptr +=".temperature .reading{color: #F29C1F;}";
  ptr +=".humidity .reading{color: #3B97D3;}";
  ptr +=".BPM .reading{color: #FF0000;}";
  ptr +=".SpO2 .reading{color: #955BA5;}";
  ptr +=".bodytemperature .reading{color: #F29C1F;}";
  ptr +=".superscript{font-size: 17px;font-weight: 600;position: absolute;top: 10px;}";
  ptr +=".data{padding: 10px;}";
  ptr +=".container{display: table;margin: 0 auto;}";
  ptr +=".icon{width:65px}";
  ptr +="</style>";
  ptr +="</head>";
  ptr +="<body>";
  ptr +="<h1>ESP32 Patient Health Monitoring</h1>";
  ptr +="<h3>For more details visit www.circuitschools.com</h3>";
  ptr +="<div class='container'>";
  
  ptr +="<div class='data bodytemperature'>";
  ptr +="<div class='side-by-side icon'>";
  ptr +="<svg enable-background='new 0 0 19.438 54.003'height=54.003px id=Layer_1 version=1.1 viewBox='0 0 19.438 54.003'width=19.438px x=0px xml:space=preserve xmlns=http://www.w3.org/2000/svg xmlns:xlink=http://www.w3.org/1999/xlink y=0px><g><path d='M11.976,8.82v-2h4.084V6.063C16.06,2.715,13.345,0,9.996,0H9.313C5.965,0,3.252,2.715,3.252,6.063v30.982";
  ptr +="C1.261,38.825,0,41.403,0,44.286c0,5.367,4.351,9.718,9.719,9.718c5.368,0,9.719-4.351,9.719-9.718";
  ptr +="c0-2.943-1.312-5.574-3.378-7.355V18.436h-3.914v-2h3.914v-2.808h-4.084v-2h4.084V8.82H11.976z M15.302,44.833";
  ptr +="c0,3.083-2.5,5.583-5.583,5.583s-5.583-2.5-5.583-5.583c0-2.279,1.368-4.236,3.326-5.104V24.257C7.462,23.01,8.472,22,9.719,22";
  ptr +="s2.257,1.01,2.257,2.257V39.73C13.934,40.597,15.302,42.554,15.302,44.833z'fill=#F29C21 /></g></svg>";
  ptr +="</div>";
  ptr +="<div class='side-by-side text'>Body Temperature</div>";
  ptr +="<div class='side-by-side reading'>";
  ptr +=(int)bodytemperature;
  ptr +="<span class='superscript'>&deg;F</span></div>";
  ptr +="</div>";
  
  ptr +="<div class='data Heartrate'>";
  ptr +="<div class='side-by-side icon'>";
  ptr +="<svg enable-background='new 0 0 40.542 40.64'height=49.079px id=Layer_1 version=1.1 viewBox=0 0 49.079 49.079 width=49.079px x=0px xml:space=preserve xmlns=http://www.w3.org/2000/svg xmlns:xlink=http://www.w3.org/1999/xlink y=0px><path d='M35.811,2.632c0.694,0,1.433,0.039,2.223,0.113C42.84,3.218,48.4,7.62,49.079,16.054v2.807";
  ptr +="c-0.542,6.923-5.099,15.231-17.612,25.333c-3.722,3.004-10.135,3.004-13.856,0C5.097,34.092,0.541,25.784,0,18.861v-2.807";
  ptr +="C0.676,7.62,6.236,3.218,11.046,2.745c0.79-0.074,1.528-0.113,2.222-0.113c2.682,0,4.691,0.561,6.395,1.549";
  ptr +="c3.181,1.846,6.569,1.846,9.752,0C31.119,3.193,33.128,2.632,35.811,2.632'";
  ptr +="fill=#FF0000 /></svg>";
  ptr +="</div>";
  ptr +="<div class='side-by-side text'>Heart Rate(BPM)</div>";
  ptr +="<div class='side-by-side reading'>";
  ptr +=(int)BPM;
  ptr +="<span class='superscript'>BPM</span></div>";
  ptr +="</div>";
  
  ptr +="<div class='data Blood Oxygen'>";
  ptr +="<div class='side-by-side icon'>";
  ptr +="<svg enable-background='new 0 0 40.542 40.541'height=40.541px id=Layer_1 version=1.1 viewBox='0 0 40.542 40.541'width=40.542px x=0px xml:space=preserve xmlns=http://www.w3.org/2000/svg xmlns:xlink=http://www.w3.org/1999/xlink y=0px><g><path d='M16.458.584A1.3957 1.3957 0 0 0 15.323 0c-.451 0-.874.217-1.137.584-2.808 3.919-9.843 14.227-9.843 19.082 ";
  ptr +="0 6.064 4.915 10.98 10.979 10.98 6.065 0 10.981-4.916 10.981-10.98.001-4.855-7.037-15.163-9.845-19.082zm-4.991 25.297c-.3.357-.732.542-1.167.542-.345 0-.695-.118-.981-.358-4.329-3.646-2.835-9.031-2.769-9.26.234-.809 1.073-1.273 1.886-1.042.808.231 1.274 1.075 1.045 1.881-.047.175-.982 3.743 1.804 6.089.642.542.725 1.503.182 2.148zm2.997 3.029c-.893 ";
  ptr +="0-1.62-.727-1.62-1.62 0-.896.727-1.621 1.62-1.621.896 0 1.62.726 1.62 1.621s-.725 1.62-1.62 1.62z'";
  ptr +="fill=#FF0000 /></g></svg>";
  ptr +="</div>";
  ptr +="<div class='side-by-side text'>Blood Oxygen</div>";
  ptr +="<div class='side-by-side reading'>";
  ptr +=(int)SpO2;
  ptr +="<span class='superscript'>%</span></div>";
  ptr +="</div>";
  
  ptr +="<div class='data Room Temperature'>";
  ptr +="<div class='side-by-side icon'>";
  ptr +="<svg enable-background='new 0 0 19.438 54.003'height=54.003px id=Layer_1 version=1.1 viewBox='0 0 19.438 54.003'width=19.438px x=0px xml:space=preserve xmlns=http://www.w3.org/2000/svg xmlns:xlink=http://www.w3.org/1999/xlink y=0px><g><path d='M11.976,8.82v-2h4.084V6.063C16.06,2.715,13.345,0,9.996,0H9.313C5.965,0,3.252,2.715,3.252,6.063v30.982";
  ptr +="C1.261,38.825,0,41.403,0,44.286c0,5.367,4.351,9.718,9.719,9.718c5.368,0,9.719-4.351,9.719-9.718";
  ptr +="c0-2.943-1.312-5.574-3.378-7.355V18.436h-3.914v-2h3.914v-2.808h-4.084v-2h4.084V8.82H11.976z M15.302,44.833";
  ptr +="c0,3.083-2.5,5.583-5.583,5.583s-5.583-2.5-5.583-5.583c0-2.279,1.368-4.236,3.326-5.104V24.257C7.462,23.01,8.472,22,9.719,22";
  ptr +="s2.257,1.01,2.257,2.257V39.73C13.934,40.597,15.302,42.554,15.302,44.833z'fill=#F29C21 /></g></svg>";
  ptr +="</div>";
  ptr +="<div class='side-by-side text'>Room Temperature</div>";
  ptr +="<div class='side-by-side reading'>";
  ptr +=(int)temperature;
  ptr +="<span class='superscript'>&deg;C</span></div>";
  ptr +="</div>";
 
  ptr +="<div class='data Room Humidity'>";
  ptr +="<div class='side-by-side icon'>";
  ptr +="<svg enable-background='new 0 0 29.235 40.64'height=40.64px id=Layer_1 version=1.1 viewBox='0 0 29.235 40.64'width=29.235px x=0px xml:space=preserve xmlns=http://www.w3.org/2000/svg xmlns:xlink=http://www.w3.org/1999/xlink y=0px><path d='M14.618,0C14.618,0,0,17.95,0,26.022C0,34.096,6.544,40.64,14.618,40.64s14.617-6.544,14.617-14.617";
  ptr +="C29.235,17.95,14.618,0,14.618,0z M13.667,37.135c-5.604,0-10.162-4.56-10.162-10.162c0-0.787,0.638-1.426,1.426-1.426";
  ptr +="c0.787,0,1.425,0.639,1.425,1.426c0,4.031,3.28,7.312,7.311,7.312c0.787,0,1.425,0.638,1.425,1.425";
  ptr +="C15.093,36.497,14.455,37.135,13.667,37.135z'fill=#3C97D3 /></svg>";
  ptr +="</div>";
  ptr +="<div class='side-by-side text'>Room Humidity</div>";
  ptr +="<div class='side-by-side reading'>";
  ptr +=(int)humidity;
  ptr +="<span class='superscript'>%</span></div>";
  ptr +="</div>";
  
  ptr +="</div>";
  ptr +="</body>";
  ptr +="</html>";
  return ptr; 
}//
