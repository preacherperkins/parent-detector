# Parent Detector

How to use a Raspberry Pi to find out who's been in your room: make a parent detector that use motion detection to trigger video recording via the Raspberry Pi camera module.

## About the Passive Infrared (PIR) motion sensor

In this project, we are going to be using a Passive Infrared (PIR) motion sensor. You have probably seen these before: they are often used in burglar alarm systems (the sensors placed in the corners of rooms are typically PIR ones). All objects whose temperatures are above absolute zero emit infrared radiation. Infrared wavelengths are not visible to the human eye, but they can be detected by the electronics inside one of these modules.

The sensor is regarded as passive because it doesn't send out any signal in order to detect movement. It adjusts itself to the infra red signature of the room it's in and then watches for any changes. Any object moving through the room will disturb the infra red signature, and will cause a change to be noticed by the PIR module.

![](images/pir_module.png)

We don't need to worry about the inner workings of the motion sensor. What we're interested in are the three pins on it: we can connect those to the Raspberry Pi GPIO pins. One pin is for +5 volts, one pin is for ground and the other is the sensor pin (the middle pin on our Pi). This sensor pin will receive power whenever motion is detected by the PIR module. We can then see that happening on the Raspberry Pi and take action accordingly.

## Connect the PIR motion sensor

Before booting your Raspberry Pi, connect the PIR module to the Raspberry Pi.

Using three female-to-female jumper cables, you'll need to connect each of the PIR sensor's connectors to the appropriate pins on the Raspberry Pi.

Connect the top one (labelled `VCC` on the PIR sensor) to the 5V pin on the Raspberry Pi, connect the middle one (labelled `OUT`) to GPIO pin 4, and connect the bottom one (labelled `GND`) to a ground pin marked `GND`. 

![](images/pir_wiring.png)

Now boot your Pi and log in.

## Test the PIR motion sensor

We're going to use the Python programming language to write some code that will detect movement and print out some text; we can extend the program to involve the camera board later on. When movement is detected the PIR motion sensor applies power to its OUT pin, which we have connected to GPIO pin 4 on the Pi. So in our code we just need to continually check pin 4 to see if it has power or not.

If a pin has power we call it HIGH and if not we call it LOW.

The program is pretty simple. We will first set up the Raspberry Pi GPIO pins to allow us to use pin 4 as an input; it can then detect when the PIR module sends power. We need to check the pin continually for any changes, so we use a `while True` loop for this. This is an infinite loop so the program will run continuously unless we stop it manually with `Ctrl + C`.

We then use two Boolean variables (True or False) for the previous and current states of the pin, the previous state being what the current state was the preceding time around the loop. Inside the loop we compare the previous state to the current state to detect when they're different. We don't want to keep displaying a message if there has been no change.

Firstly create a blank Python file with the following command:

```bash
nano pirtest.py
```

Enter or copy and paste the code below:

```python
import RPi.GPIO as GPIO
import time

sensor = 4

GPIO.setmode(GPIO.BCM)
GPIO.setup(sensor, GPIO.IN, GPIO.PUD_DOWN)

previous_state = False
current_state = False

while True:
    time.sleep(0.1)
    previous_state = current_state
    current_state = GPIO.input(sensor)
    if current_state != previous_state:
        new_state = "HIGH" if current_state else "LOW"
        print("GPIO pin %s is %s" % (sensor, new_state))
```

Press `Ctrl + O` to save and `Ctrl + X` to quit.

Now run the Python file:

```bash
sudo python3 pirtest.py
```

If you get an error saying `RuntimeError: No access to /dev/mem` it means you forgot to use `sudo`. You must run programs that access the GPIO as root and `sudo` does this for you (if you need help to remember this you can think of it as 'super-user-do').

If you start moving or waving the sensor pin will go HIGH. Keep on waving and it will stay HIGH, and only go back to LOW if you keep still again. If you see the sensor behave like this, then everything is working correctly. If not, something is wrong and you need to go back and troubleshoot.

```
GPIO pin 4 is HIGH
GPIO pin 4 is LOW
GPIO pin 4 is HIGH
```

Press `Ctrl + C` when you want to exit.

![](images/pir_potentiometers.png)

On the PIR module you should see two orange components with sockets that fit a Phillips screwdriver (see above). These are called potentiometers: they allow you to adjust the sensitivity of the sensor and the detection time. You should begin by setting the sensitivity to max and the time to min, but you can vary this later if you wish.


## Setting up the camera

Before you can begin the project, you'll need to connect the camera. You should do this before you boot the Pi, or, if your Pi is running, you should stop it by entering `sudo-halt`. 

Follow the instructions [here](http://www.raspberrypi.org/camera-module-setup/) to set up and test the camera module. Stop once you have successfully used a few of the example commands.

Next, if you have it, set up the camera mount. This will enable you to aim the camera at the right part of the room. 

If you have not done so already, test the camera is working using the following command:

```bash
raspivid -t 0
```

Press `Ctrl + C` to exit.

## Program the camera to preview on movement

Now we're ready to extend our previous program to give it the ability to control the camera module. To start with, let's just make our program display what the camera can see when movement is detected; we can set up recording to a file later.

Make a copy of the previous program and we'll use that for this step:

```bash
cp pirtest.py pirCamera.py
```

Now use the following command to edit the file:

```bash
nano pirCamera.py
```

We first need to add the `import picamera` statement at the top; this allows your program to access the pre-made code which can control the camera module. We then declare the camera object `cam`, which provides all the camera control functions that we need to use. Then inside the `while` loop where we print the HIGH or LOW message, we can test to see if `current_state` is HIGH or True (meaning that movement is detected); we can then start or stop the camera preview accordingly.

Either copy and paste the code below, or enter it manually:

```python
import RPi.GPIO as GPIO
import time
import picamera  # new

sensor = 4

GPIO.setmode(GPIO.BCM)
GPIO.setup(sensor, GPIO.IN, GPIO.PUD_DOWN)

previous_state = False
current_state = False

cam = picamera.PiCamera()  # new

while True:
    time.sleep(0.1)
    previous_state = current_state
    current_state = GPIO.input(sensor)
    if current_state != previous_state:
        new_state = "HIGH" if current_state else "LOW"
        print("GPIO pin %s is %s" % (sensor, new_state))
        if current_state:  # new
            cam.start_preview()
        else:
            cam.stop_preview()
```

Press `Ctrl + O` to save and `Ctrl + X` to quit. To run the program use the following command:

```bash
sudo python3 pirCamera.py
```

Press `Ctrl + C` when you want to exit.

## Recording to a file and playing back

We can now add a bit more code to allow us to record to a file for playback at a later stage. If there are multiple intruders in your room, you want to be able to capture them all and not just the most recent one. To do that, we need a way to generate a new file name automatically each time movement is detected. The easiest and safest way to do this is to make a file name out of the date and time.

For example, if the time now was the 11th of February 2014 at 10:24 AM and 18 seconds the file name would be something like this: `2014-02-11_10.24.18.h264`. This uses the format of `YEAR-MONTH-DAY_HOUR.MINUTE.SECOND.h264`; h264 refers to the format the video will be recorded in. It's the same as the format used by YouTube.

We need to import the `datetime` Python module and write a function to generate the filename. See `get_file_name` below: this uses the string from time function to insert the values from the current time into the specified string format. Then you simply use the commands to start and stop the recording using the generated file name. These should happen at the same time as the preview commands respectively.

```bash
nano pirCamera.py
```

Either copy and paste the code below, or enter it manually:

```python
import RPi.GPIO as GPIO
import time
import picamera
import datetime  # new

def get_file_name():  # new
    return datetime.datetime.now().strftime("%Y-%m-%d_%H.%M.%S.h264")

sensor = 4

GPIO.setmode(GPIO.BCM)
GPIO.setup(sensor, GPIO.IN, GPIO.PUD_DOWN)

previous_state = False
current_state = False

cam = picamera.PiCamera()

while True:
    time.sleep(0.1)
    previous_state = current_state
    current_state = GPIO.input(sensor)
    if current_state != previous_state:
        new_state = "HIGH" if current_state else "LOW"
        print("GPIO pin %s is %s" % (sensor, new_state))
        if current_state:
            fileName = get_file_name()  # new
            cam.start_preview()
            cam.start_recording(fileName)  # new
        else:
            cam.stop_preview()
            cam.stop_recording()  # new
```

Press `Ctrl + O` to save and `Ctrl + X` to quit. To run the program use the following command:

```bash
sudo python3 pirCamera.py
```

Press `Ctrl + C` when you want to exit.

### Playback

If you now use the `ls` command you should see that a few files have been generated. You can use the following command to play back a file. Replace `<file>` with the filename you wish to play.

```
omxplayer <file> -o hdmi
```

For example: `omxplayer 2014-02-11_10.24.18.h264 -o hdmi`

## What next? 

You have completed your parent detector, but why not try taking it to another level by using it in stealth mode? 

- You could disable the red LED on the camera board which comes on when you start your Python program. This can be done by editing the Raspberry Pi configuration file. Enter the command below:

```bash
sudo nano /boot/config.txt
```

Add the following line to the end of the file:

```
disable_camera_led=1
```

Press `Ctrl + O` to save and `Ctrl + X` to quit. The changes will only take effect after a reboot, so enter the following command to do this:

```bash
sudo reboot
```

- If you want to leave the monitor connected and turned on while the program is running, it's a good idea to edit the Python code to disable the camera preview lines. Use the `#` sign at the start of a line to disable it.

- you could also start your Python program under a different login. To do this press `ALT - F2` before you log in; this will show you a new login prompt, so log in there and start the Python program. Now if you press `ALT - F1` to go back to the usual login prompt, it will appear as though the Raspberry Pi is innocently waiting for someone to log in.




