/* TSL2591 Digital Light Sensor */

//include Libarys
//Adafruit basic libary
#include <Wire.h>
//Sensor
#include <Adafruit_Sensor.h>
#include "Adafruit_TSL2591.h"
//Modbus libary (Modbus Master-Slave library for Arduino by smarmengol(Github)
#include <ModbusRtu.h>
//Libary for the LCD-Display
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27,20,4); // set the LCD address to 0x27 for a 16 chars and 2 line display (Code by YWROBOT/Ec-yuan)

Adafruit_TSL2591 tsl = Adafruit_TSL2591(2591); // pass in a number for the sensor identifier (for your use later)

// Variabls
// Timers
unsigned long LooptimeMeasurement;
unsigned long LooptimeLCD;
// Pinnumber for Modbus
#define TXEN  4 
//Modbusobject
Modbus slave(1,0,TXEN);
//Array for Modbus
uint16_t photodata[3]={0,0,1};
/*
 * photodata[0]=Value now
 * photodata[1]=currentoutput
 * photodata[2]=blank
 */
//define Pins
//Pin for the Blankbutton
#define blankPin 3
// Pin für LED
#define LEDPIN 9
// Time intervals
int intervalltimeMeasurement=500;
int intervalltimeLCD=10000;
// Array for raw data
uint16_t data[100];
// conter1 for the data array
int i;
//Chekpoint variable to see if array has already reached 100 values
bool over100;
//variable to see if a blank given once
bool blankgiven;
//For calculation of sums in a loop
unsigned long sum;
int currentOutput;

//    Configures the gain and integration time for the TSL2591

void configureSensor(void)
{
  //Sets the the gain of the sensor to medium
  tsl.setGain(TSL2591_GAIN_MED);      // 25x gain
  
  //integration time set to 500 milliseconds
  tsl.setTiming(TSL2591_INTEGRATIONTIME_500MS);


}

void bootScreen(void)
{
  //initialize
  lcd.init(); 
  lcd.backlight();
  // infomation on display while booting
  lcd.setCursor(0,0);
  lcd.print("FermenTUM|PhoTUM");
  lcd.setCursor(0,1);
  lcd.print("Lukas+Max+Jan");
}

void setup(void) 
{
  // set bauderate to 19200 for Slave master communication
  slave.begin(19200); 
  //Bootscreen on LCD
  bootScreen();
  /* Configure the sensor */
  configureSensor();
  // get a starttime
  LooptimeMeasurement = millis();
  LooptimeLCD = millis();
  //set counter to 0
  i=0;
  int currentOutput=1;
  //inizilase LEDpin
  pinMode(LEDPIN,OUTPUT);
  digitalWrite (LEDPIN,1);
  //inizialase BlankButtonPin
  pinMode(blankPin,INPUT);
  //chekpoint Variable
  //variable to see if array is full
  over100=0;//or byte
  //variable to see if a blank given once
  blankgiven=0;
}

//simple read was used to get the intensety of visible light

void simpleRead(void)
{
  // Function to get intensety of visible light
  uint16_t x = tsl.getLuminosity(TSL2591_VISIBLE);
}


void loop(void) 
{ 
  // wait vor the preset time
 Start: slave.poll(photodata,3);
  if (millis()>LooptimeMeasurement + intervalltimeMeasurement){
    // get the intensety of the light at the sensor
    int16_t x = tsl.getLuminosity(TSL2591_VISIBLE);
    if (x<0) goto Start;
    photodata[0]=x;
    // reset counter if over hundret to get an array of hundret values
    if (i>=100){
      i=0; //to dont get a to high counter
      //to know if array was filled once
      over100=1;
    }
    
    if (over100==1){//Move each entry one position forward.
      for(int k=0; k<99;k++){
        data[k]=data[k+1];
      }
      data[99]=x;
    }
    else{//write the value into the array
      data[i]=x;
    }
  
    //data for the output is the mean of the 100 values
    //was done to get rid of noise in the data
    if (over100==1){
      for (int k=0; k<100;k++){
        //sum up the whole array
        sum=sum+data[k];
      }
      currentOutput=sum/100;
      sum=0;
    }
    else{
      for (int k=0; k<i;k++){
        sum=sum+data[k];
      }
      currentOutput=sum/i;
      sum=0;
    }
    //write averaged value into array
    photodata[1]=currentOutput;
    //raise the counter
    i++;
    //get the Starttime of the next Loop
    LooptimeMeasurement = millis();
  }
  /*

 
  Spaceholder for The bus system


  */
  if (photodata[2]!=1) blankgiven=1;

  if (digitalRead(blankPin)==1){
    blankgiven=1;
    photodata[2]=photodata[1];
  }
  
  
  if (millis()>LooptimeLCD + intervalltimeLCD){
    //display information on the LCD display
    lcd.clear();
    lcd.setCursor(0,0);
    lcd.print("NOW:");
    lcd.setCursor(4,0);
    lcd.print(currentOutput);
    LooptimeLCD=millis();
    if (blankgiven==1&&photodata[2]!=0){
      //calculate Trasmission
      float transmission=(float)photodata[1]/(float)photodata[2];
      lcd.setCursor(0,1);
      lcd.print("Transmis.:");
      lcd.setCursor(10,1);
      lcd.print(transmission*100,1);
      lcd.setCursor(15,1);
      lcd.print("%");
    }
  }
}
