Quickfeather [Simple Streaming Interface] AI Application Project
=================================

This project performs either data collection or recognition based on the 
build mode. Data collection uses [Simple Streaming Interface] to connect and 
stream sensor data to SensiML's [Data Capture Lab]. Adding external sensors to collect data, 
analyze and build models is made simple requiring only a sensor configuration
API function and data reading function API to be supplied for the new sensor.
This project provides an Arduino friendly Wire interface for easily integrating
Arduino sensor code.

Building and running the project for data collection mode:
---------------------

1. Verify that the following macros are set for data collection mode in the 
   source file [sensor\_ssss.h](inc/sensor\_ssss.h).
```
#define SENSOR_SSSS_RECOG_ENABLED      0    /* Enable SensiML recognition */
#define SENSOR_SSSS_LIVESTREAM_ENABLED 1    /* Enable live-streaming for data collection */
```
2. Use the provided [Makefile](GCC_Project/Makefile) and an appropriate ARM GCC 
toolchain to build the project

2. Use the flash programming procedure to flash the binary to Quickfeather board.

3. Reset the board to start running the qf\_ssi\_ai\_app application.

4. Use [Data Capture Lab] to connect, stream and capture the sensor data.

Building and running the project for recognition mode:
---------------------

1. Verify that the following macros are set for recognition mode in the 
   source file [sensor\_ssss.h](inc/sensor\_ssss.h).
```
#define SENSOR_SSSS_RECOG_ENABLED      1    /* Enable SensiML recognition */
#define SENSOR_SSSS_LIVESTREAM_ENABLED 0    /* Enable live-streaming for data collection */
```
2. Use the provided [Makefile](GCC_Project/Makefile) and an appropriate ARM GCC
toolchain to build the project

2. Use the flash programming procedure to flash the binary to Quickfeather board.

3. Reset the board to start running the qf\_ssi\_ai\_app application.

4. Connect a UART to the Quickfeather board. Open a terminal application, 
   set its baud to 4608000 bps to get the recognition results from the
   running application.

For details on data collection, building an AI model, and recognition please refer to [SensiML Getting Started]

## Adding a sensor

The default project uses the onboard Accelerometer sensor for data collection.
This section provides basic guideline on adding a new sensor to the project for 
data collection. Sensor data acquisition and processing or transfer to an 
external application such as [Data Capture Lab] uses [datablock manager] (qorc-sdk/Libraries/DatablockManager) and
[datablock processor] (qorc-sdk/Tasks/DatablockProcessor) for acquiring samples
and processing these acquired samples. Qorc-sdk uses [Simple Streaming Interface]
protocol to send the acquired sensor data over to the [DataCaptureLab]

FreeRTOS software timer is used to trigger a timer event to read 1 sample from
the sensor and fill the datablock. When enough samples are collected (determined 
by the sensor sample rate, and latency), the datablock is processed and the 
samples are sent over the UART using the [Simple Streaming Interface].

The [Wire interface] may be used to provide the requrired configuration and 
sample acquisition member functions to configure and read data from the new
sensor.

## Configure the sensor

To add a new sensor start with the sensor\_ssss.h and sensor\_ssss.c source files.
Sensor sampling rate and number of channels are specified in the macros
defined in the header file sensor\_ssss.h. Configuring and reading from the sensor
requires atleast 3 member functions:

- begin() this member function initializes and configures the new sensor.
- set\_sample\_rate() this member function sets the desired sample rate for this sensor
- read() reads 1 sample of data from this sensor. To make synchronizing and fusing
  multiple sensor data easier, this function simply retries the current sample 
  available in the sensor and returns the value.

### sensor\_ssss.h

Modify the header file sensor\_ssss.h and update the following macros
- SENSOR\_SSSS\_SAMPLE\_RATE to specify the desired sensor sampling rate 
- SENSOR\_SSSS\_CHANNELS\_PER\_SAMPLE to specify the desired number of channels for the new sensor
- SENSOR\_SSSS\_LATENCY default latency is set to 20ms. This value determines 
  how often the samples are processed and transmitted to the DCL ([Data Capture Lab]). 
  The default value may be left as is.

The above macros determine the number of samples held in one datablock. These 
datablocks are held in the array sensor\_ssss\_data_blocks\[\]. 

### sensor\_ssss.c

- Update the function sensor\_ssss\_configure() to initialize and setup the
  sensor configuration. 
  The example code uses the following code snippet to configure the onboard 
  accelerometer sensor.

  ```
  MC3635  qorc_ssi_accel;
  ```

  ```
  qorc_ssi_accel.begin();
  qorc_ssi_accel.set_sample_rate(sensor_ssss_config.rate_hz);
  qorc_ssi_accel.set_mode(MC3635_MODE_CWAKE);
  ```

## Output data description

Update the string value definition of json\_string\_sensor\_config in sensor\_ssss.cpp
for the new sensor added to this project. The example project which uses 3-channel
onboard accelerometer is described using the following string:

```
	{
	   sample_rate:100,
	   samples_per_packet:6,
	   column_location:{
		  AccelerometerX:0,
		  AccelerometerY:1,
		  AccelerometerZ:2
	   }
	}
```

Refer the SensiML [Data Capture Lab] for details

## Acquring and processing sensor samples

Based on the sensor sample rate, a FreeRTOS soft timer triggers requesting 
1 sensor sample to be filled-in the datablock.

### sensor_ssss.c

- Update the function sensor\_ssss\_acquisition\_buffer\_ready to read 1 sample 
  (16-bits per channel) into the current datablock. 
  This function returns 1 if datablock is ready for processing, returns 0 otherwise.

  The example code uses the following code snippet to configure the onboard 
  accelerometer sensor.

```c++
	xyz_t accel_data = qorc_ssi_accel.read();  /* Read accelerometer data from MC3635 */
	
	/* Fill this accelerometer data into the current data block */
	int16_t *p_accel_data = (int16_t *)p_dest;
	
	*p_accel_data++ = accel_data.x;
	*p_accel_data++ = accel_data.y;
	*p_accel_data++ = accel_data.z;
	
	p_dest += 6; // advance datablock pointer to retrieve and store next sensor data
```

## Capturing the sensor samples

- Sensor samples are sent using the [Simple Streaming Interface]. A 16-bit 
  little-endian data format is used for sending each channel's sample data.
  Quickfeather uses either an S3 UART or the USB serial to transmit these data.
  Sensor samples may be captured using [Data Capture Lab] 

## Accelerometer sensor example

An example accelerometer (mCube's MC3635) sensor available onboard is provided 
as part of this application. The MC3635 class interface to configure and read
data from the sensor is available in the source files mc3635\_wire.cpp and
mc3635\_wire.h. The sensor configuration function sensor\_ssss\_configure() 
uses the begin() function of the class MC3635 to configure and set up the accelerometer
for acquiring samples approximately at the chosen sampling rate (SENSOR\_SSSS\_SAMPLE\_RATE). 

To read samples the configured sampling rate, sensor data read is performed when 
the FreeRTOS soft timer triggers the function sensor\_ssss\_acquisition\_buffer\_ready(). 
The read() member function is used to read three 16-bit samples and fill-in the
current data block. When 20ms (= SENSOR\_SSSS\_LATENCY) samples are filled in the
data block, these samples are processed by the function sensor\_ssss\_livestream\_data_processor() 
to send these samples over UART using [Simple Streaming Interface].

## SparkFun ADS1015 Example

This section describes the steps to add [SparkFun Qwiic 12-bit ADC] sensor (ADS1015) 
to this project.

Obtain the [SparkFun ADS1015 Arduino Library] code and add these source files 
to the qf\_ssi\_ai\_app/src folder. Update the SparkFun\_ADS1015\_Arduino\_Library.cpp 
to resolve the missing function delay(), and provide definitions for the following
data types

- boolean
- byte

Update sensor\_ssss.h and sensor\_ssss.cpp as described in the above sections. 
For example, to replace the accelerometer with only the [SparkFun Qwiic 12-bit ADC] sensor
update following macro definition for SENSOR\_SSSS\_CHANNELS\_PER\_SAMPLE in sensor\_ssss.h 
with the following code snippet:

```
#define SENSOR_SSSS_CHANNELS_PER_SAMPLE  ( 1)  // Number of channels
```

Add a class instance of the ADS1015 to the source file sensor\_ssss.cpp as shown below:

```
	ADS1015 qorc_ssi_adc ;
```

Update the function sensor_ssss_configure in sensor\_ssss.cpp to replace the 
accelerometer initialization and sample readings with following code snippet:

```
  qorc_ssi_adc.begin();
  qorc_ssi_adc.setSampleRate(sensor_ssss_config.rate_hz);
```

Update the sensor\_ssss\_acquisition\_buffer\_ready function in sensor\_ssss.cpp 
to replace the accelerometer sensor reading with the following code snippet
to read Channel 3 of the ADS1015 sensor.

```
    int16_t adc_data = qorc_ssi_adc.getSingleEnded(3);
    *p_dest = adc_data;
    p_dest += 1; // advance datablock pointer to retrieve and store next sensor data    
```

Update the string value definition of json\_string\_sensor\_config in sensor\_ssss.cpp 
as described in above section.

Build and load the project to the Quickfeather.

Connect a [SparkFun Qwiic 12-bit ADC] sensor to the Quickfeather using the following pinouts

| ADS1015 module  | Quickfeather |
| --------------- | ------------ |
| SCL             | J2.7         |
| SDA             | J2.8         |
| GND             | J8.16        |
| Vcc             | J8.15        |

## SparkFun Qwiic Scale NAU7802 Example

This section describes the steps to add [SparkFun Qwiic Scale - NAU7802] sensor 
to this project.

Obtain the [SparkFun Qwiic Scale NAU7802 Arduino Library] code and add these source files 
to the qf\_ssi\_ai\_app/src folder. Update the SparkFun\_Qwiic\_Scale\_NAU7802\_Arduino\_Library.cpp 
to resolve the missing functions delay(), and millis()

Add a class instance of the ADS1015 to the source file sensor\_ssss.cpp as shown below:

```
	NAU7802 qorc_ssi_scale;
```

Update sensor\_ssss.h and sensor\_ssss.cpp as described in the above sections. 
For example, to replace the accelerometer with only the [SparkFun Qwiic Scale - NAU7802] sensor
update following macro definition for SENSOR\_SSSS\_CHANNELS\_PER\_SAMPLE in sensor\_ssss.h 
with the following code snippet:

```c++
#define SENSOR_SSSS_CHANNELS_PER_SAMPLE  ( 1)  // Number of channels
```

Update the function sensor_ssss_configure in sensor\_ssss.cpp to replace the 
accelerometer initialization and sample readings with following code snippet:

```c++
  qorc_ssi_scale.begin();
  qorc_ssi_scale.setSampleRate(sensor_ssss_config.rate_hz);
```

Update the sensor\_ssss\_acquisition\_buffer\_ready function in sensor\_ssss.cpp 
to replace the accelerometer sensor reading with the following code snippet
to read a sample from the scale. Qwiic scale outputs a 24-bit value where as the
data capture is only capable of 16-bit sensor readings. So, adjust the returned
reading to write 16-bit value into the datablock buffer as shown in the code 
snippet below.

```c++
    int16_t scale_data = qorc_ssi_scale.getReading() >> 8;
    *p_dest = scale_data;
    p_dest += 1; // advance datablock pointer to retrieve and store next sensor data    
```

Update the string value definition of json\_string\_sensor\_config in sensor\_ssss.cpp 
as described in above section.

Build and load the project to the Quickfeather.

Connect a [SparkFun Qwiic Scale - NAU7802] sensor to the Quickfeather using the following pinouts

| ADS1015 module  | Quickfeather |
| --------------- | ------------ |
| SCL             | J2.7         |
| SDA             | J2.8         |
| GND             | J8.16        |
| Vcc             | J8.15        |

Refer [Qwiic Scale Hookup Guide] for details. Quickfeather is now ready to stream data
to [Data Capture Lab]

[s3-gateware]: https://github.com/QuickLogic-Corp/s3-gateware
[SensiML QF]: https://sensiml.com/documentation/firmware/quicklogic-quickfeather/quicklogic-quickfeather.html
[SensiML Getting Started]: https://sensiml.com/documentation/guides/getting-started/index.html
[PmodAD1]: https://reference.digilentinc.com/reference/pmod/pmodad1/start
[datablock manager]: qf_vr_apps#datablock-manager
[datablock processor]: qf_vr_apps#datablock-processor
[Data Capture Lab]: https://sensiml.com/products/data-capture-lab/
[Qwiic Scale Hookup Guide]: https://learn.sparkfun.com/tutorials/qwiic-scale-hookup-guide?_ga=2.193267885.1228472612.1605042107-1202899191.1566946929
[Simple Streaming Interface]: https://sensiml.com/documentation/simple-streaming-specification/introduction.html
[sensor\_ssss.h]: inc/sensor_ssss.h
[SparkFun Qwiic 12-bit ADC]: https://www.sparkfun.com/products/15334
[SparkFun ADS1015 Arduino Library]: https://github.com/sparkfun/SparkFun_ADS1015_Arduino_Library/tree/master/src
[SparkFun Qwiic Scale - NAU7802]: https://www.sparkfun.com/products/15242
[SparkFun Qwiic Scale NAU7802 Arduino Library]: https://github.com/sparkfun/SparkFun_Qwiic_Scale_NAU7802_Arduino_Library
[Qwiic Scale Hookup Guide]: https://learn.sparkfun.com/tutorials/qwiic-scale-hookup-guide?_ga=2.267750089.614684343.1607376760-1202899191.1566946929