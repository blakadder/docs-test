# Scripting
!!! failure "This feature is not included in precompiled binaries"

To use it you must [compile your build](Compile-your-build). Add the following to `user_config_override.h`:
```arduino
#ifndef USE_SCRIPT
#define USE_SCRIPT  // adds about 17k flash size, variable ram size
#endif
#ifdef USE_RULES
#undef USE_RULES
#endif  
```

Additional features are enabled by adding the following `#define` compiler directive parameters and then compiling the firmware. These parameters are explained further below in the article.

| Feature | Description |
| -- | -- |
USE_BUTTON_EVENT | enable `b` section (detect button state changes)
USE_SCRIPT_JSON_EXPORT | enable `J` section (publish JSON payload on [TelePeriod](Commands#teleperiod))
USE_SCRIPT_SUB_COMMAND | enables invoking named script subroutines via the Console or MQTT
USE_SCRIPT_HUE | enable `H` section (Alexa Hue emulation)
USE_SCRIPT_STATUS | enable `U` section (receive JSON payloads)
SCRIPT_POWER_SECTION | enable `P` section (execute on power changes)
SUPPORT_MQTT_EVENT | enables support for subscribe unsubscribe  
USE_SENDMAIL | enable `m` section and support for sending e-mail   
USE_SCRIPT_WEB_DISPLAY | enable `W` section (modify web UI)
USE_TOUCH_BUTTONS | enable virtual touch button support with touch displays
USE_WEBSEND_RESPONSE | enable receiving the response of a [`WebSend`](Commands#websend) command (received in section E)
SCRIPT_STRIP_COMMENTS | enables stripping comments when attempting to paste a script that is too large to fit
USE_ANGLE_FUNC | add sin(x),acos(x) and sqrt(x) e.g. to allow calculation of horizontal cylinder volume
USE_24C256 | enables use of 24C256 I^2^C EEPROM to expand script buffer (defaults to 4k)
USE_SCRIPT_FATFS | enables SD card support (on SPI bus). Specify the CS pin number. Also enables 4k script buffer  
USE_SCRIPT_FATFS_EXT | enables additional FS commands  
SDCARD_DIR | enables support for web UI for SD card directory upload and download  

----

!!! info "Scripting Language for Tasmota is an alternative to Tasmota [Rules](Rules)"

To enter a script, go to **Configuration - Edit script** in the Tasmota web UI menu

The maximum script size is 1535 bytes (uses rule set buffers). If the pasted script is larger than 1535 characters, comments will be stripped to attempt to  make the script fit.  

To save code space almost no error messages are provided. However it is taken care of that at least it should not crash on syntax errors.  

## Features

## Scripting Cookbook

### Switching and Dimming By Recognizing Mains Power Frequency

Switching in Tasmota is usually done by High/Low (+3.3V/GND) changes on a GPIO. However, for devices like the [Moes QS-WiFi-D01 Dimmer](https://templates.blakadder.com/qs-wifi_D01_dimmer.html), this is achieved by a pulse frequency when connected to the GPIO, and these pulses are captured by `Counter1` in Tasmota.

![pushbutton-input](https://user-images.githubusercontent.com/36734573/61955930-5d90e480-afbc-11e9-8d7e-00ac526874d3.png)

- When the **light is OFF** and there is a **short period** of pulses - then turn the light **ON** at the previous dimmer level.
- When the **light is ON** and there is a **short period** of pulses - then turn the light **OFF**.
- When there is a longer period of pulses (i.e., **HOLD**) - toggle dimming direction and then adjust the brightness level as long as the button is pressed or until the limits are reached.

[Issue 6085](https://github.com/arendst/Tasmota/issues/6085#issuecomment-512353010)

In the Data Section D at the beginning of the Script the following initialization variables may be changed:

- dim multiplier = `0..2.55` set the dimming increment value    
- dim lower limit = range for the dimmer value for push-button operation (set according to your bulb); min 0    
- dim upper limit = range for the dimmer value for push-button operation (set according to your bulb); max 100    
- start dim level = initial dimmer level after power-up or restart; max 100   


    D  
    sw=0  
    tmp=0  
    cnt=0  
    tmr=0  
    hold=0  
    powert=0  
    slider=0  
    dim=""  
    shortprl=2 ;short press lo limit  
    shortpru=10;short press up limit  
    dimdir=0   ;dim direction 0/1  
    dimstp=2   ;dim step/speed 1 to 5  
    dimmlp=2.2 ;dim multiplier  
    dimll=15   ;dim lower limit  
    dimul=95   ;dim upper limit  
    dimval=70  ;start dim level  
      
    B  
    =print "WiFi-Dimmer-Script-v0.2"  
    =Counter1 0  
    =Baudrate 9600  
    ; boot sequence  
    =#senddim(dimval)  
    delay(1000)  
    =#senddim(0)  
      
    F  
    cnt=pc[1]  
    if chg[cnt]0  
    ; sw pressed  
    then sw=1  
    else sw=0  
    ; sw not pressed  
    endif  

    ; 100ms timer  
    tmr+=1  


    ; short press  
    if sw==0  
    and tmrshortprl  
    and tmr<shortpru  
    then  
    powert^=1  

    ; change light on/off  
    if powert==1  
    then  
    =#senddim(dimval)  
    else  
    =#senddim(0)  
    endif  
    endif  


    ; long press  
    if sw0  
    then  
    if hold==0  
    then  

    ; change dim direction  
    dimdir^=1  
    endif  
    if tmrshortpru  
    then  
    hold=1  
    if powert0  

    ; dim when on & hold  
    then  
    if dimdir0  
    then  

    ; increase dim level  
    dimval+=dimstp  
    if dimvaldimul  
    then  

    ; upper limit  
    dimval=dimul  
    endif  
    =#senddim(dimval)  
    else  

    ; decrease dim level  
    dimval-=dimstp  
    if dimval<dimll  
    then  

    ; lower limit  
    dimval=dimll  
    endif  
    =#senddim(dimval)  
    endif  
    endif  
    endif  
    else  
    tmr=0  
    hold=0  
    endif  
      
    E  
    slider=Dimmer  

    ; slider change  
    if chg[slider]0  
    then  

    ; dim according slider  
    if slider0  
    then  
    dimval=slider  
    =#senddim(dimval)  
    else  
    powert=0  
    =#senddim(0)  
    endif  
    endif  

    if pwr[1]==1  
    ; on/off webui  
    then  
    powert=1  
    =#senddim(dimval)  
    else  
    powert=0  
    =#senddim(0)  
    endif  

    ; subroutine dim  
    #senddim(tmp)  
    dim="FF55"+hn(tmp*dimmlp)+"05DC0A"  
    =SerialSend5 %dim%  
    =Dimmer %tmp%  
    \#