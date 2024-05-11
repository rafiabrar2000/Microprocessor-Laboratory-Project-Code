import cv2
import numpy as np
import RPi.GPIO as GPIO
import picamera
import picamera.array
import time

def follow_line(color):
    in1 = 4
    in2 = 17
    in3 = 27
    in4 = 22
    en1 = 23
    en2 = 24

    GPIO.setmode(GPIO.BCM)
    GPIO.setup(en1, GPIO.OUT)
    GPIO.setup(en2, GPIO.OUT)
    GPIO.setup(in1, GPIO.OUT)
    GPIO.setup(in2, GPIO.OUT)
    GPIO.setup(in3, GPIO.OUT)
    GPIO.setup(in4, GPIO.OUT)

    p1 = GPIO.PWM(en1, 100)
    p2 = GPIO.PWM(en2, 100)
    p1.start(50)
    p2.start(50)

    GPIO.output(in1, GPIO.LOW)
    GPIO.output(in2, GPIO.LOW)
    GPIO.output(in3, GPIO.LOW)
    GPIO.output(in4, GPIO.LOW)

    with picamera.PiCamera() as camera:
        camera.resolution = (160, 120)
        camera.framerate = 30
        time.sleep(2)  # Allow camera to warm up
        with picamera.array.PiRGBArray(camera) as stream:
            while True:
                camera.capture(stream, format='bgr', use_video_port=True)
                frame = stream.array
                
                if color == "green":
                    low_color = np.array([35, 50, 50])
                    high_color = np.array([90, 255, 255])
                elif color == "red":
                    low_color = np.array([0, 50, 50])
                    high_color = np.array([10, 255, 255])
                elif color == "blue":
                    low_color = np.array([100, 100, 100])
                    high_color = np.array([140, 255, 255])
                elif color == "yellow":
                    low_color = np.array([20, 100, 100])
                    high_color = np.array([30, 255, 255])
                else:
                    break
                
                mask = cv2.inRange(cv2.cvtColor(frame, cv2.COLOR_BGR2HSV), low_color, high_color)
                
                contours, hierarchy = cv2.findContours(mask, 1, cv2.CHAIN_APPROX_NONE)
                
                if len(contours) > 0:
                    c = max(contours, key=cv2.contourArea)
                    M = cv2.moments(c)
                    
                    if M["m00"] != 0:
                        cx = int(M['m10'] / M['m00'])
                        cy = int(M['m01'] / M['m00'])
                        print("CX : " + str(cx) + "  CY : " + str(cy))
                        
                        if cx <= 40:
                            print("Turn Left")
                            GPIO.output(in1, GPIO.HIGH)
                            GPIO.output(in2, GPIO.LOW)
                            GPIO.output(in3, GPIO.LOW)
                            GPIO.output(in4, GPIO.HIGH)
                            
                        if 40 < cx < 120:
                            print("On Track!")
                            GPIO.output(in1, GPIO.HIGH)
                            GPIO.output(in2, GPIO.LOW)
                            GPIO.output(in3, GPIO.HIGH)
                            GPIO.output(in4, GPIO.LOW)
                            
                        if cx >= 120:
                            print("Turn Right")
                            GPIO.output(in1, GPIO.LOW)
                            GPIO.output(in2, GPIO.HIGH)
                            GPIO.output(in3, GPIO.HIGH)
                            GPIO.output(in4, GPIO.LOW)
                            
                        cv2.circle(frame, (cx, cy), 5, (255, 255, 255), -1)
                else:
                    print("I don't see the line")
                    GPIO.output(in1, GPIO.LOW)
                    GPIO.output(in2, GPIO.LOW)
                    GPIO.output(in3, GPIO.LOW)
                    GPIO.output(in4, GPIO.LOW)
                
                cv2.drawContours(frame, c, -1, (0, 255, 0), 1)
                cv2.imshow("Mask", mask)
                cv2.imshow("Frame", frame)
                
                if cv2.waitKey(1) & 0xff == ord('q'):
                    GPIO.output(in1, GPIO.LOW)
                    GPIO.output(in2, GPIO.LOW)
                    GPIO.output(in3, GPIO.LOW)
                    GPIO.output(in4, GPIO.LOW)
                    break
                
                stream.seek(0)
                stream.truncate()

    cv2.destroyAllWindows()

# Main code
while True:
    user_input = input()
    follow_line(user_input)
