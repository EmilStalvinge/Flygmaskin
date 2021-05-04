# Flygmaskin
Flight controller based on ATmega328p microcontroller and MPU-6050 accererometer and gyro


## Communication protocols
- Receiver - SBUS		 - input
- Gyro - I2C			 - input
- ESC -Oneshot 125	 -output

<p align="center">
<img src="https://user-images.githubusercontent.com/50867974/116965879-34cc7180-acaf-11eb-8e6a-1029a5222c77.png" width="500">
</p>

<p align="center">
<img src="https://user-images.githubusercontent.com/50867974/116965821-05b60000-acaf-11eb-9771-891ab2aab0d9.png" width="500">
</p>

<p align="center">
<img src="https://user-images.githubusercontent.com/50867974/116965911-4ada3200-acaf-11eb-883f-aa3006a2c6ca.png" width="500">
</p>



## Receiver - SBUS
The receiver we have is using SBUS, a popular protocol for rc receivers, that uses serial communication. It’s like UART but is inverted on most receivers,
so we had first to find the uninverted output of the receiver to be able to use ATmegas UART function. SBUS uses a baud rate of 100 000. This baud rate does
not fit with the ATmegas standard clock frequency of 1MHz. But running at 8Mhz gives zero error according to the WormFood baudrate calulator 


<p align="center">
<img src="https://user-images.githubusercontent.com/50867974/116966375-51b57480-acb0-11eb-8035-237c590d4e7e.png" width="200">
</p>

### Sbus bit mapping
A problem was as seen in the SBUS channel layout the channels values are 11 bit numbers and UART sends packets of 8 bits. 
Which leads to an unsymmetrical remapping of the bits corresponding to the bytes. 
This was solved by after reading the bytes shifting them into 16 bit memory but only using the last 11 bits.

## Gyro - I2C
The MCU-6050 communicates via I2c and can output new gyro values at a rate of 1000Hz. 
For the I2C communication we used Peter Fleury's library. To initialize the I2C connection some set bits was needed to be sent.

We then configure the inbuilt low pass filter, which we choose to set at 20Hz (seen in  the register map below) as it should eliminate
the vibrations from the motors to get a more stable reading.

<p align="center">
<img src="https://https://user-images.githubusercontent.com/50867974/116966685-05b6ff80-acb1-11eb-8d19-f51e82a8703b.png" width="300">
</p>

The read function is called in the beginning of the main loop. This was an easy way to get new numbers,
but as the mainloop was running at unknown frequency then either the same gyroscope value could be read several times or if the loop was too slow,
gyroscope values could be missed and never used. To only read new values we could have used the interrupt pin “int” from the MPU-6050 that tells
when a new value is ready.


## ESC -Oneshot 125
An upgrade from normal PWM with a period of 20ms is the protocol Oneshot 125 with a period of 2ms. This means that every 2 ms the ESC require a pulse with length 125us to 250ms. 
This will be interpreted by the ESC to a motor speed of 0% to 100%.

<p align="center">
<img src="https://user-images.githubusercontent.com/50867974/116966868-60505b80-acb1-11eb-9bee-13fd66c0d097.png" width="300">
</p>


## PID TUNING
We started with only P gain value until we got a good response and steady oscillations. Then the D value was raised until the oscillations disappeared. The quadcopter was then flyable but had a hard time keep its angle without drifting. So last the I value was added until the quad had a good feeling and could keep the input angle without drifting.
The derivative for the D term was calculated each loop with a timer that counts the elapsed time since last loop. 

As we used only the gyroscope all we cared about was the quadcopter to keep the angular velocity that we desired from the stick inputs. The stick in the middle corresponds to zero angle velocity(quadcopter should stay at the angle it has). The stick at full tilt corresponds to a angular velocity of 90 degrees/second.  The error term was calculated by:
desired angular velocity  - measured angular velocity
