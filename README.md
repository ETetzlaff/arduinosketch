/*
Project: Evan Tetzlaff
Sources: Twitter API listed on arduino.cc written by Tom Igoe (see below)
         arduino.cc for basic motor operations
Purpose: Provide a tool to tweet and execute motor, rigged to open a latch to feed the cat.
*/

/*
  Twitter Client with Strings
 
 This sketch connects to Twitter using an Ethernet shield. It parses the XML
 returned, and looks for <text>this is a tweet</text>
 
 You can use the Arduino Ethernet shield, or the Adafruit Ethernet shield, 
 either one will work, as long as it's got a Wiznet Ethernet module on board.
 
 This example uses the DHCP routines in the Ethernet library which is part of the 
 Arduino core from version 1.0 beta 1
 
 This example uses the String library, which is part of the Arduino core from
 version 0019.  
 
 Circuit:
 * Ethernet shield attached to pins 10, 11, 12, 13
 
 created 21 May 2011
 modified 9 Apr 2012
 by Tom Igoe
 
 This code is in the public domain.
 
 */
#include <SPI.h>
#include <Ethernet.h>


// Enter a MAC address and IP address for your controller below.
// The IP address will be dependent on your local network:
byte mac[] = { 0x90, 0xA2, 0xDA, 0x0D, 0xC4, 0xC2 };
IPAddress ip(192,168,1,40);

// initialize the library instance:
EthernetClient client;

const unsigned long requestInterval = 60000;  // delay between requests

char serverName[] = "api.twitter.com";  // twitter URL
//char serverName[] = "https://twitter.com/itachireuhssure";

boolean requested;                   // whether you've made a request since connecting
unsigned long lastAttemptTime = 0;            // last time you connected to the server, in milliseconds

String currentLine = "";            // string to hold the text from server
String tweet = "";                  // string to hold the tweet
boolean readingTweet = false;       // if you're currently reading the tweet

int motorPin = 9; //pin the motor is connected to

void setup() {
  pinMode(motorPin, OUTPUT); 
  
  // reserve space for the strings:
  currentLine.reserve(256);
  tweet.reserve(150);

 // Open serial communications and wait for port to open:
  Serial.begin(9600);
   while (!Serial) {
    ; // wait for serial port to connect. Needed for Leonardo only
  }


  // attempt a DHCP connection:
  Serial.println("Attempting to get an IP address using DHCP:");
  if (!Ethernet.begin(mac)) {
    // if DHCP fails, start with a hard-coded address:
    Serial.println("failed to get an IP address using DHCP, trying manually");
    Ethernet.begin(mac, ip);
  }
  Serial.print("My address:");
  Serial.println(Ethernet.localIP());
  // connect to Twitter:
  connectToServer();
}



void loop()
{
  if (client.connected()) {
    if (client.available()) {
      // read incoming bytes:
      char inChar = client.read();

      // add incoming byte to end of line:
      currentLine += inChar; 

      // if you get a newline, clear the line:
      if (inChar == '\n') {
        currentLine = "";
      } 
      // if the current line ends with <text>, it will
      // be followed by the tweet:
      if ( currentLine.endsWith("<text>")) {
        // tweet is beginning. Clear the tweet string:
        readingTweet = true; 
        tweet = "";
      }
      // if you're currently reading the bytes of a tweet,
      // add them to the tweet String:
      if (readingTweet) {
        if (inChar != '<') {
          tweet += inChar;
        } 
        else {
          // if you got a "<" character,
          // you've reached the end of the tweet:
          readingTweet = false;
          Serial.println(tweet);   
          // close the connection to the server:
          client.stop(); 
          
          //I think this is where the motor stop/start will go.
          if(tweet == "feed cat"){
            motorOnThenOff();
            //Set tweet to not "feed cat" so it does not continue activating motor.
            tweet = "reset";
          }  
          Serial.println("Worked!");
        }
      }
    }   
  }
  else if (millis() - lastAttemptTime > requestInterval) {
    // if you're not connected, and two minutes have passed since
    // your last connection, then attempt to connect again:
    connectToServer();
  }
}

void connectToServer() {
  // attempt to connect, and wait a millisecond:
  Serial.println("connecting to server...");
  if (client.connect(serverName, 80)) {
    Serial.println("making HTTP request...");
    // make HTTP GET request to twitter:
    client.println("GET /1/statuses/user_timeline.xml?screen_name=itachireuhssure HTTP/1.1");
    client.println("HOST: api.twitter.com");
    client.println();
  }
  // note the time of this connect attempt:
  lastAttemptTime = millis();
}

void motorOnThenOff(){
  int onTime = 2500;  //the number of milliseconds for the motor to turn on for
  int offTime = 1000; //the number of milliseconds for the motor to turn off for
  
  digitalWrite(motorPin, HIGH); // turns the motor On
  delay(onTime);                // waits for onTime milliseconds
  digitalWrite(motorPin, LOW);  // turns the motor Off
  delay(offTime);               // waits for offTime milliseconds
}

void motorOnThenOffwithSpeed(){
  int onSpeed = 200; //a number between 0 (stopped) and 255 (full)
  
  int onTime = 2500;
  int offSpeed = 50;
  
  int offTime = 1000;
  analogWrite(motorPin, onSpeed); //turn the motor on
  
  delay(onTime); //waits for onTime milliseconds
  analogWrite(motorPin, offSpeed); //turn motor off
  
  delay(offTime); //wait for offTime milliseconds
}

void motorAcceleration(){
  int delayTime = 50; //time between each speed step
  for(int i = 0; i < 256; i++){
    analogWrite(motorPin, i);
    delay(delayTime);
  }
  for(int i = 255; i >= 0; i--){
    analogWrite(motorPin, i);
    delay(delayTime);
  }
}  
