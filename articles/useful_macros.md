# Useful Macros

## Conditional Homing

Home if not already homed. Useful to throw at the beginning of other macros.
```
[gcode_macro CG28]
gcode:
    {% if "xyz" not in printer.toolhead.homed_axes %}
        G28
    {% endif %}
```
## Beeper

Allows you to utilize your MINI12864 LCD beeper. 

This requires you to specify your beeper pin as an output pin.
- Your `pin` may be different.

Example (PWM beeper, used on MINI12864):
```
[output_pin beeper]
pin: z:EXP1_1
pwm: True
value: 0
shutdown_value: 0
cycle_time: 0.0005
```

Example (non PWM beeper, used on some other displays such as the Ender 3 stock display):
```
[output_pin beeper]
pin: P1.30
```

Example usage: `BEEP I=3` (Beep 3 times)

```
[gcode_macro BEEP]
gcode:
    # Parameters
    {% set i = params.I|default(1)|int %}
    
    {% for beep in range(i|int) %}
        SET_PIN PIN=beeper VALUE=1
        SET_PIN PIN=beeper VALUE=0
    {% endfor %}
```

## LCD RGB
This just provides an easy shortcut to change your neopixel LCD color. This may need modifying depending on your particular LCD. Mine is an MINI12864.

I have my LCD turn red during runouts, and green during filament swaps.

Example usage: `LCDRGB R=0.8 G=0 B=0`

```
[gcode_macro LCDRGB]
gcode:
    {% set r = params.R|default(1)|float %}
    {% set g = params.G|default(1)|float %}
    {% set b = params.B|default(1)|float %}

    SET_LED LED=lcd RED={r} GREEN={g} BLUE={b} INDEX=1 TRANSMIT=0
    SET_LED LED=lcd RED={r} GREEN={g} BLUE={b} INDEX=2 TRANSMIT=0
    SET_LED LED=lcd RED={r} GREEN={g} BLUE={b} INDEX=3
```

To reset the RGB / set the initial RGB. (set your default LCD colors here, and use `RESETRGB` to call set it back.)
```
[gcode_macro RESETRGB]
gcode:
    SET_LED LED=lcd RED=1 GREEN=0.45 BLUE=0.4 INDEX=1 TRANSMIT=0
    SET_LED LED=lcd RED=0.25 GREEN=0.2 BLUE=0.15 INDEX=2 TRANSMIT=0
    SET_LED LED=lcd RED=0.25 GREEN=0.2 BLUE=0.15 INDEX=3
```

To set the default colors at startup (required)
```
[delayed_gcode SETDISPLAYNEOPIXEL]
initial_duration: 1
gcode:
    RESETRGB
```

## My Pause/Resume Macros (For Runouts, Filament Swaps, and Manual Pauses)

You need `[pause_resume]` specified in your config to be able to use these.

This macro set has a few features:
- Moves the toolhead (z hops) up by 10mm, then moves the toolhead to the front for easy loading/unloading.
    - Will not z hop if this exceeds your max Z position.
- Has protections to not allow you to automatically hit pause or resume twice.
- During the pause, allows you to move toolhead around, and even run load/unload filament macros etc. It wil automatically return to its original position before resuming.
- Automatically restores your gcode state (absolute vs relative extrusion mode, etc), should it be changed during the pause by another macro.
- Primes the nozzle while traveling back to resume the print, wiping the excess along the way. This just results in one little string to pick off.
- Sets the idle timeout to 12 hours during the pause, and returns it to your configured value upon resume.
- Turns off your filament sensor during the pause, so it doesn't trip and run its runout gcode again while you're already paused.
- Turns off the hotend during the pause, and turns it back on for the resume.*
    - \* *I highly advise keeping this functionality - it's a safety feature. It's not ideal to lave your hotend cooking all night waiting for you to come and swap filament. And with a smart filament sensor, it can even sometimes catch heat creep clogs should your hotend fan fail.*

**Some things are commented out that rely on other macros.** You can uncomment them if you choose to use those other macros.

### M600 (Filament Change) Alias

This allows your pause to work natively with slicers that insert `M600` for color changes.
```
[gcode_macro M600]
gcode:
    #LCDRGB R=0 G=1 B=0  # Turn LCD green
    PAUSE
```
### Runout G-Code
This goes in your filament sensor config section.
```
runout_gcode:
    #LCDRGB R=1 G=0 B=0  # Turn LCD red
    PAUSE
    #BEEP I=12
```

### Pause
If you use a filament sensor, put its name in the `SET_FILAMENT_SENSOR` command. Otherwise, comment that out.
```
[gcode_macro PAUSE]
rename_existing: BASE_PAUSE
gcode:
    # Parameters
    {% set z = params.Z|default(10)|int %}                                                                                  ; z hop amount
    
    {% if printer['pause_resume'].is_paused|int == 0 %}     
        SET_GCODE_VARIABLE MACRO=RESUME VARIABLE=zhop VALUE={z}                                                             ; set z hop variable for reference in resume macro
        SET_GCODE_VARIABLE MACRO=RESUME VARIABLE=etemp VALUE={printer['extruder'].target}                                   ; set hotend temp variable for reference in resume macro
                                
        SET_FILAMENT_SENSOR SENSOR=filament_sensor ENABLE=0                                                                 ; disable filament sensor       
        SAVE_GCODE_STATE NAME=PAUSE                                                                                         ; save current print position for resume                
        BASE_PAUSE                                                                                                          ; pause print
        {% if (printer.gcode_move.position.z + z) < printer.toolhead.axis_maximum.z %}                                      ; check that zhop doesn't exceed z max
            G91                                                                                                             ; relative positioning
            G1 Z{z} F900                                                                                                    ; raise Z up by z hop amount
        {% else %}
            { action_respond_info("Pause zhop exceeds maximum Z height.") }                                                 ; if z max is exceeded, show message and set zhop value for resume to 0
            SET_GCODE_VARIABLE MACRO=RESUME VARIABLE=zhop VALUE=0
        {% endif %}
        G90                                                                                                                 ; absolute positioning
        G1 X{printer.toolhead.axis_maximum.x/2} Y{printer.toolhead.axis_minimum.y+5} F19500                                 ; park toolhead at front center
        SAVE_GCODE_STATE NAME=PAUSEPARK                                                                                     ; save parked position in case toolhead is moved during the pause (otherwise the return zhop can error) 
        M104 S0                                                                                                             ; turn off hotend
        SET_IDLE_TIMEOUT TIMEOUT=43200                                                                                      ; set timeout to 12 hours
    {% endif %}
```

### Resume
If you use a filament sensor, put its name in the `SET_FILAMENT_SENSOR` command. Otherwise, comment that out.
```
[gcode_macro RESUME]
rename_existing: BASE_RESUME
variable_zhop: 0
variable_etemp: 0
gcode:
    # Parameters
    {% set e = params.E|default(2.5)|int %}
    
    {% if printer['pause_resume'].is_paused|int == 1 %}
        SET_FILAMENT_SENSOR SENSOR=filament_sensor ENABLE=1                                                                 ; enable filament sensor
        #RESETRGB                                                                                                            ; reset LCD color
        SET_IDLE_TIMEOUT TIMEOUT={printer.configfile.settings.idle_timeout.timeout}                                         ; set timeout back to configured value
        {% if etemp > 0 %}
            M109 S{etemp|int}                                                                                               ; wait for hotend to heat back up
        {% endif %}
        RESTORE_GCODE_STATE NAME=PAUSEPARK MOVE=1 MOVE_SPEED=100                                                            ; go back to parked position in case toolhead was moved during pause (otherwise the return zhop can error)  
        G91                                                                                                                 ; relative positioning
        M83                                                                                                                 ; relative extruder positioning
        {% if printer[printer.toolhead.extruder].temperature >= printer.configfile.settings.extruder.min_extrude_temp %}                                                
            G1 Z{zhop * -1} E{e} F900                                                                                       ; prime nozzle by E, lower Z back down
        {% else %}                      
            G1 Z{zhop * -1} F900                                                                                            ; lower Z back down without priming (just in case we are testing the macro with cold hotend)
        {% endif %}                             
        RESTORE_GCODE_STATE NAME=PAUSE MOVE=1 MOVE_SPEED=100                                                                ; restore position
        BASE_RESUME                                                                                                         ; resume print
    {% endif %}
```

### Cancel Print

Clears any pause and runs my PRINT_END macro for cancelations.

```
[gcode_macro CANCEL_PRINT]
rename_existing: BASE_CANCEL_PRINT
gcode:
    SET_IDLE_TIMEOUT TIMEOUT={printer.configfile.settings.idle_timeout.timeout} ; set timeout back to configured value
    CLEAR_PAUSE
    SDCARD_RESET_FILE
    PRINT_END
    BASE_CANCEL_PRINT
```

### Octoprint
If you use Octoprint, put these in your "GCODE Script" section to enable the UI buttons to work properly.

- ![](/images/Octoprint-Gcode-Scripts.png)

## Filament Sensor Management
Disables the filament sensor 1 second after startup. This prevents it from tripping while you're just loading filament, doing testing or maintenance, etc.

Put your filament sensor's name after `SENSOR=`.

```
[delayed_gcode DISABLEFILAMENTSENSOR]   
initial_duration: 1
gcode:
    SET_FILAMENT_SENSOR SENSOR=filament_sensor ENABLE=0
```

Then:
- Put `SET_FILAMENT_SENSOR SENSOR=filament_sensor ENABLE=1` in your `PRINT_START`/resume macros.
- Put `SET_FILAMENT_SENSOR SENSOR=filament_sensor ENABLE=0` in your `PRINT_END`/pause/cancel macros. 

The above pause/resume/cancel macros have this already. Just update the sensor name.

## Parking

Park the toolhead at different places. Automatically determined based on your printer's configured size.

```
# Park front center
[gcode_macro PARKFRONT]
gcode:
    CG28                                                                                                                        ; home if not already homed
    SAVE_GCODE_STATE NAME=PARKFRONT
    G90                                                                                                                         ; absolute positioning
    G0 X{printer.toolhead.axis_maximum.x/2} Y{printer.toolhead.axis_minimum.y+5} Z{printer.toolhead.axis_maximum.z/2} F19500        
    RESTORE_GCODE_STATE NAME=PARKFRONT
```
```
# Park front center, but low down
[gcode_macro PARKFRONTLOW]
gcode:
    CG28                                                                                                                        ; home if not already homed
    SAVE_GCODE_STATE NAME=PARKFRONT
    G90                                                                                                                         ; absolute positioning
    G0 X{printer.toolhead.axis_maximum.x/2} Y{printer.toolhead.axis_minimum.y+5} Z20 F19500                                     
    RESTORE_GCODE_STATE NAME=PARKFRONT
```
```
# Park top rear left
[gcode_macro PARKREAR]
gcode:
    CG28                                                                                                                        ; home if not already homed
    SAVE_GCODE_STATE NAME=PARKREAR
    G90                                                                                                                         ; absolute positioning
    G0 X{printer.toolhead.axis_minimum.x+10} Y{printer.toolhead.axis_maximum.y-10} Z{printer.toolhead.axis_maximum.z-50} F19500     
    RESTORE_GCODE_STATE NAME=PARKREAR
```
```
# Park center of build volume
[gcode_macro PARKCENTER]
gcode:
    CG28                                                                                                                        ; home if not already homed
    SAVE_GCODE_STATE NAME=PARKCENTER
    G90                                                                                                                         ; absolute positioning
    G0 X{printer.toolhead.axis_maximum.x/2} Y{printer.toolhead.axis_maximum.y/2} Z{printer.toolhead.axis_maximum.z/2} F19500    
    RESTORE_GCODE_STATE NAME=PARKCENTER
```
```
# Park 15mm above center of bed
[gcode_macro PARKBED]
gcode:
    CG28                                                                                                                        ; home if not already homed
    SAVE_GCODE_STATE NAME=PARKBED
    G90                                                                                                                         ; absolute positioning
    G0 X{printer.toolhead.axis_maximum.x/2} Y{printer.toolhead.axis_maximum.y/2} Z15 F19500                                     
    RESTORE_GCODE_STATE NAME=PARKBED
```

# Dump Parameters
This dumps all Klipper parameters to the g-code terminal. This helps to find Klipper system variables for use in macros.
```
[gcode_macro DUMP_PARAMETERS]
gcode:
   {% for name1 in printer %}
      {% for name2 in printer[name1] %}
         { action_respond_info("printer['%s'].%s = %s" % (name1, name2, printer[name1][name2])) }
      {% else %}
         { action_respond_info("printer['%s'] = %s" % (name1, printer[name1])) }
      {% endfor %}
   {% endfor %}-
```