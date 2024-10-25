**Reading Kaifa MB310H4BNL**  
My house has a Kaifa MB310H4BNL digial power meter but although it's digital it lacks a P1 port because it's not a "smart" meter.

My net provider said it wasn definitely impossible to read out this meter but impossible is one of my trigger words :D

I'll fill in some more details later the summary is that the meter exposes something called an S0 port.
Basically this is an infrared port and it has an iron ring in the back so you can connect a magnet to it.

In countries such as Germany and Austria it's quite common to read out the S0 port.  
Good resources to learn more are the [volkzaehler wiki](https://wiki.volkszaehler.org/) and an article at [heise](https://www.heise.de/tests/Ausprobiert-Guenstiger-IR-Lesekopf-fuer-Smart-Meter-mit-Tastmota-Firmware-7065559.html).

There's 2 choices of reading heads, infrared to USB but very there's also versions with WiFi which can connect with Home Assistant using MQTT.
The latter was most interesting to me as I want the data to be available in Home Assistant.

In the reading head consists of a lower board with an IR LED and an IR phototransistor. Their signal is processed by a Schmitt trigger (74HC14D) and sent to the upper board via an 8-pin connector. 
The lower board also has a micro-USB port to supply power to the read head and a 3.3V voltage regulator. The upper board houses an ESP8266, which handles the interpretation of the meter signals and the Wi-Fi connection.  
  
![Reading head internals](https://heise.cloudimg.io/v7/_www-heise-de_/imgs/18/3/5/2/9/2/6/7/IMG_7404B-0f3de2dc5c9a8327.jpg?force_format=avif%2Cwebp%2Cjpeg&org_if_sml=1&q=85&width=610)

The device is often called a "Hichi" reader or simply "WiFi IR Lesekopf" and they use Tasmota Smart Meter Interface (see [here](https://tasmota.github.io/docs/Smart-Meter-Interface/#kaifa-mb310h4bde)).

I purchased this one from [Amazon](https://www.amazon.de/dp/B0D28ZQ28F), there's also a few to be found on ebay.

It took a lot of fiddling, testing and retrying I finally found the recipe:
To start communication with the read we must send the following string:
`/?!` followed by `<CR><LF>` or in HEX `2F3F210D0A` at 300 baud using 7E1 parity.

The meter then responds with an identification string, in my case:
`/KFM5\2Kaifa MA309M`

Per protocol, it start with / then 3 digits for Manufacturer, followed by a digit to indicate the baudrate, \, then mode and the rest identifies the meter type.
Interestingly the meter identifies as MA309M and not MB310.

THe baud rate is as follows:
```
0 -> 300
1 -> 600
2 -> 1200
3 -> 2400
4 -> 4800
5 -> 9600
6 -> 19200
```

Here's a small script to form the right hex string:
```
$data = '/KFM5\2Kaifa MA309M'

$manufacturersIdentification = $data.Substring(1, 3)
$baudrateIdentification = $data.Substring(4, 1)
$modeIdentification = $data.Substring(6, 1)
$identification = $data.Substring(7)

switch ($baudrateIdentification)
{
	"1" { $baudRate = 600 ; Break }
	"2" { $baudRate = 1200 ; Break }
	"3" { $baudRate = 2400 ; Break }
	"4" { $baudRate = 4800 ; Break }
	"5" { $baudRate = 9600 ; Break }
	"6" { $baudRate = 19200 ; Break }
}
$acknowledge = [string]([char]0x6)+ "0" + $baudrateIdentification + $modeIdentification + "`r`n"
$hexString = -join ($acknowledge.ToCharArray() | ForEach-Object { [Convert]::ToString([byte][char]$_, 16).ToUpper().PadLeft(2, '0') })

"Data          : $data"
"Manufacturer  : $manufacturersIdentification"
"Baud rate     : $baudRate"
"Mode          : $modeIdentification"
"Identification: $identification"
"Hex string    : $hexString"
```

Output:

```
Data          : /KFM5\2Kaifa MA309M
Manufacturer  : KFM
Baud rate     : 9600
Mode          : 2
Identification: Kaifa MA309M
Hex string    : 063035320D0A
```

As you can see, mode is 02, which I understand means send the initialisation string once and it will continue outputting the meter data continously.
As that's not how the Tasmota smart metering is designed I used the hex string `063035300D0A` instead.

Before using the script (see repo), go to the Tasmota console and give the following command: `SerialConfig 8E1` to set property parity and stop bits.

And finally we get output!
```
/KFM5\2Kaifa MA309M
<STX>F.F(00000000)
0.0.0(000000xxxx) (meter id redacted)
0.0.1(00xxxxxx) (value redacted)
1.8.0(026348.8*kWh)
1.8.1(000000.0*kWh)
1.8.2(026348.8*kWh)
2.8.0(009281.3*kWh)
2.8.1(000000.0*kWh)
2.8.2(009281.3*kWh)
0.2.0(01.03-21)
C.90.2(239b1249)
0.2.1(01.02-19)
C.91.2(7bed5b2f)
<ETX>
```
