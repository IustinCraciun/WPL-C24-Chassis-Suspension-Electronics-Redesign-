#include <Servo.h>

//rc receptor pins 
#define CH1_pin 18
#define CH2_pin 19
#define CH3_pin 20
#define CH4_pin 21

//BTS7960 driver pins
const int motorRPWM = 5;
const int motorLPWM = 6;
const int motorEN1 = 7;
const int motorEN2 = 8;

//led pin
const int LED_pin = 10;

//Servo object
Servo myServo;
int neutralServoAngle = 90; //setting neutral angle
int servoAngle;

//variables for motor control
int intermediarMotorSpeed = 0;
int motorSpeed = 0;
bool motorDirection = true;

//variables for pwm signal stabilization
float alpha = 0.3; //smoothness coeficient !!change this for smoother or sharper signal variations!!
//variables for filtered values
float filteredCH1 = 0;
float filteredCH2 = 0;

//exponential smoothness function
int applyExponentialFilter(float &filteredValue, int rawValue) {
  filteredValue = alpha * rawValue + (1-alpha) * filteredValue;
  return (int)filteredValue;
}


void setup() {

  // rc receptor pins configuration
  pinMode(CH1_pin, INPUT);
  pinMode(CH2_pin, INPUT);
  pinMode(CH3_pin, INPUT);
  pinMode(CH4_pin, INPUT);

  //output pins for BTS7960
  pinMode(motorLPWM, OUTPUT);
  pinMode(motorRPWM, OUTPUT);
  pinMode(motorEN1, OUTPUT);
  pinMode(motorEN2, OUTPUT);

  //led pin
  pinMode(LED_pin, OUTPUT);


  //servo pin
  myServo.attach(3);

  //serial initialization for debugging
  //Serial.begin(9600);

}

void loop() {
  
  //extracting signals from the receptor
  int rawpwmCH1 = pulseIn(CH1_pin, HIGH, 25000);
  int rawpwmCH2 = pulseIn(CH2_pin, HIGH, 25000);
  int pwmCH3 = pulseIn(CH3_pin, HIGH, 25000);
  int pwmCH4 = pulseIn(CH4_pin, HIGH, 25000);

  int pwmCH1 = applyExponentialFilter(filteredCH1, rawpwmCH1);
  int pwmCH2 = applyExponentialFilter(filteredCH2, rawpwmCH2);
  //debugging
  //Serial.println(LED_pin);

  //control servo (CH1)
  if(pwmCH1 >= 930 && pwmCH1 <= 2070) {
    servoAngle = map(pwmCH1, 930, 2070, -40, 40);
    int servoPosition = neutralServoAngle + servoAngle;
    myServo.write(servoPosition);
  }

  //control motor (CH2)
  if(pwmCH2 >=900 && pwmCH2 <= 2130) {
    if(pwmCH2 > 1560) {
      motorDirection = true; //forward
      if(pwmCH2 <=2070) {
       intermediarMotorSpeed = map(pwmCH2, 1560, 2070, 0, 255 ); 
      } else if(pwmCH2 > 2070){
        intermediarMotorSpeed = 255;
      }
      
    } else if (pwmCH2 < 1520) {
      motorDirection = false; //backward
      if(pwmCH2 >= 950){
        intermediarMotorSpeed = map(pwmCH2, 950, 1520, 255, 0);
      } else if (pwmCH2 < 950) {
        intermediarMotorSpeed = 255;
      }
      
    } else {
      intermediarMotorSpeed = 0; //stop
    }
    
    //speed controll
    if(pwmCH3 < 1500){
      motorSpeed = intermediarMotorSpeed / 2;
    } else if(pwmCH3 > 1500){
      motorSpeed = intermediarMotorSpeed;
    }

    controlMotor(motorSpeed, motorDirection);

  } else {
    controlMotor(0, true); // invalid signal-> motor stop
  }
  
  //control led (CH4)
  if(pwmCH4 > 1500) {
    digitalWrite(LED_pin, HIGH);
  } else if (pwmCH4 < 1500) {
    digitalWrite(LED_pin, LOW);
  }
  
  delay(10); //delay for stabilization
}


//function for controlling the motor 
void controlMotor(int speed, bool direction) {
  if (direction) { // forward
    analogWrite(motorRPWM, speed); 
    analogWrite(motorLPWM, 0);   
    digitalWrite(motorEN1, HIGH); 
    digitalWrite(motorEN2, HIGH);  
  } else if(!direction){ // reverse
    analogWrite(motorRPWM, 0);    
    analogWrite(motorLPWM, speed); 
    digitalWrite(motorEN1, HIGH);  
    digitalWrite(motorEN2, HIGH); 
  } else {
    analogWrite(motorRPWM, 0);
    analogWrite(motorLPWM, 0);
    digitalWrite(motorEN1, LOW);
    digitalWrite(motorEN2, LOW);
  }
}

