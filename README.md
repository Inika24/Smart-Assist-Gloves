#include <TinyGPS++.h>
#include <SoftwareSerial.h>
#include <pcmConfig.h>
#include <pcmRF.h>
#include <SD.h>   // need to include the SD library
#define SD_ChipSelectPin 10   //using digital pin 4 on arduino nano 328
#include <TMRpcm.h>   //also need to include this library...
#define trigPin1 6
#define echoPin1 7
#define motor1 8
#define buzzer A0
#define button A1

long duration1;
int distance1;

float gpslat;
float gpslon;

TinyGPSPlus gps;
SoftwareSerial gsmSerial(2,3); //SIM800L Tx & Rx is connected to Arduino #3 & #2
SoftwareSerial gpsSerial(4,5); //gps Tx & Rx is connected to Arduino #5 & #4
TMRpcm tmrpcm;   // create an object for use in this sketch
void setup() {
  // put your setup code here, to run once:
  Serial.begin(9600); // sets the serial port to 9600
  gsmSerial.begin(9600); // set the mySerial port to 9600
  gpsSerial.begin(9600);  // set the gpsSerial port to 9600
  
  pinMode(trigPin1,OUTPUT); // Sets the trigPin as an Output
  pinMode(echoPin1,INPUT); // Sets the echoPin as an Input
  pinMode(buzzer,OUTPUT);
  digitalWrite(buzzer,HIGH);
  pinMode(motor1,OUTPUT);
  pinMode(button,INPUT_PULLUP);

  tmrpcm.speakerPin = 9;   //11 on Mega, 9 on Uno, Nano, etc

  Serial.begin(9600);
  if (!SD.begin(SD_ChipSelectPin)) { // see if the card is present and can be initialized:
   Serial.println("SD fail"); 
   return;   // don't do anything more if not
  }else{
   Serial.println("SD read");
  }

  Serial.println("Initializing...");
  delay(10000);
  gsmSerial.println("AT"); //Once the handshake test is successful, it will back to OK
  updateSerial();
  gsmSerial.println("AT+CSQ"); //Signal quality test, value range is 0-31 , 31 is the best
  updateSerial();
  gsmSerial.println("AT+CCID"); //Read SIM information to confirm whether the SIM is plugged
  updateSerial();
  gsmSerial.println("AT+CREG?"); //Check whether it has registered in the network
  updateSerial();

  
}

void updateSerial(){
  delay(500);
  while(Serial.available()) 
  {
    gsmSerial.write(Serial.read());//Forward what Serial received to Software Serial Port
  }
  while(gsmSerial.available()) 
  {
    Serial.write(gsmSerial.read());//Forward what Software Serial received to Serial Port
  }
  delay(100);
}

void loop() {
  obstacleDetect();
  emergencyCheck();
  // This sketch displays information every time a new sentence is correctly encoded.
  while (gpsSerial.available() > 0)
  {
    if (gps.encode(gpsSerial.read()))
    {
      displayInfo();
    }
  }
  delay(100);
  
  // If 5000 milliseconds pass and there are no characters coming in
  // over the software serial port, show a "No GPS detected" error
  if (millis() > 5000 && gps.charsProcessed() < 10)
  {
    Serial.println("No GPS detected, retrying...");
    return;  // Continue execution instead of halting
  } 
}

void displayInfo(){
  if (gps.location.isValid())
  {
    Serial.print("Latitude: ");
    gpslat = gps.location.lat();
    Serial.println(gpslat, 6);
    Serial.print("Longitude: ");
    gpslon = gps.location.lng();
    Serial.println(gpslon, 6);
    Serial.print("Altitude: ");
    Serial.println(gps.altitude.meters());
  }
  else
  {
    Serial.println("Location: Not Available");
  }
  
  Serial.print("Date: ");
  if (gps.date.isValid())
  {
    Serial.print(gps.date.month());
    Serial.print("/");
    Serial.print(gps.date.day());
    Serial.print("/");
    Serial.println(gps.date.year());
  }
  else
  {
    Serial.println("Not Available");
  }

  Serial.print("Time: ");
  if (gps.time.isValid())
  {
    if (gps.time.hour() < 10) Serial.print(F("0"));
    Serial.print(gps.time.hour());
    Serial.print(":");
    if (gps.time.minute() < 10) Serial.print(F("0"));
    Serial.print(gps.time.minute());
    Serial.print(":");
    if (gps.time.second() < 10) Serial.print(F("0"));
    Serial.print(gps.time.second());
    Serial.print(".");
    if (gps.time.centisecond() < 10) Serial.print(F("0"));
    Serial.println(gps.time.centisecond());
  }
  else
  {
    Serial.println("Not Available");
  }
  Serial.println();
  Serial.println();
}

void moveright(){
  noInterrupts();
  // put your main code here, to run repeatedly:
  digitalWrite(trigPin1, LOW);// Clears the trigPin
  delayMicroseconds(5);
  digitalWrite(trigPin1, HIGH);// Sets the trigPin on HIGH state for 10 micro seconds
  delayMicroseconds(12);
  digitalWrite(trigPin1, LOW);
  duration1 = pulseIn(echoPin1, HIGH);// Reads the echoPin, returns the sound wave travel time in microseconds
  distance1= duration1*0.034/2;//  distance= (Time x Speed of Sound in Air (340 m/s))/2
  interrupts();
  Serial.print("ULT:00002:cm1: ");
  Serial.println(distance1);
  delay(100);
  if(distance1<=15)
  {
    //digitalWrite(buzzer,LOW);
    digitalWrite(motor1,HIGH);
    delay(100);
    tmrpcm.play("obstacle.wav");
    while (tmrpcm.isPlaying()) {
      delay(100);
    }
  }
  if(distance1>=15)
  {
    //digitalWrite(buzzer,HIGH);
    digitalWrite(motor1,LOW);
  }
}

void emergencyCheck(){
  int button_state = digitalRead(button);
  if(button_state == LOW)
  { 
    Serial.println("button pressed");
    delay(1000);
    tmrpcm.play("panic.wav");
    while (tmrpcm.isPlaying()) {
      delay(100);
    }
    makeCall();
    sendSMS();    
  }
}

void sendSMS()
{
  Serial.println("Sending SMS..."); 
  gsmSerial.println("AT"); //Once the handshake test is successful, it will back to OK
  updateSerial();
  gsmSerial.println("AT+CMGF=1"); // Configuring TEXT mode
  updateSerial();
  gsmSerial.println("AT+CMGS=\"+9196002177730\"");//change ZZ with country code and xxxxxxxxxxx with phone number to sms
  updateSerial();
  gsmSerial.println("Emergency Help Needed at:"); //text content
  gsmSerial.print("https://www.google.com/maps/?q=");
  gsmSerial.print(gpslat, 6);
  gsmSerial.print(",");
  gsmSerial.print(gpslon, 6);
  updateSerial();
  gsmSerial.write(26);
  delay(1000);
}

// Make a call function
void makeCall() {
  Serial.println("Making Call...");
  gsmSerial.println("AT"); //Once the handshake test is successful, i t will back to OK
  updateSerial();
  gsmSerial.println("ATD+ +919600217773;"); //  change ZZ with country code and xxxxxxxxxxx with phone number to dial
  updateSerial();
  delay(30000); // wait for 20 seconds...
  gsmSerial.println("ATH"); //hang up
  updateSerial();
  delay(1000);
}