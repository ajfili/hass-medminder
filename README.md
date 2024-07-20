# hass-medminder
A Home Assistant / ESPHome based medication reminder for tracking and reminding about medications and refills

As a person who is not particularly functional when I first wake up, as well as having ADHD, I have trouble remembering to take medications first thing in the morning. I've used the medication reminder built into iOS Health for a while with moderate success, but the number of times my phone reminds me to take it and then I have to count pills in my "initially conscious" state to see if I took it immediately when I woke up a few minutes prior is far too high. I'm a Home Automation nerd and figured leveraging HASS and ESPHome to solve a problem while making my first self-designed ESPHome project to develop my skillsets was a good use of my time.

I built a janky prototype out of a drilled 3d printed box and lots of dupont connectors and hot glue. But it didn't work half bad.

As I thought about what I wanted to improve and discussed it with a few people on some HASS discussion pages, I realized this might have some use for people who also have a hard time remembering to take their medication, or even a family tracking if someone already gave the dog/cat their medication today, etc. I wanted to release it out there so other people could benefit from what I built. So I went back to the drawing board and redesigned the whole project. Instead of using 8-10 HASS helper entities, I leveraged the Event bus more. I took logic out of the ESP and moved it to HASS and vice versa. If I was going to release it to the world, I wanted to at least release something better than v1

This is my second iteration, and while better than the first version, I'm sure this can definitely be cleaned up a bit. I apologize for any janky code, or the poor design of the casing. I merely was inspired by a design in Thingiverse and slapped some modifications onto it in TinkerCAD for a quick and dirty enclosure I could make ASAP. One of the things on my to-do list is to sit down and work on my CAD skills so I can design an enclosure from scratch, but if anyone wants to take a crack at it, I would love if you shared it with me. Any constructive feedback towards this project is welcomed, as it can only help me get better at this stuff.

In this repo are the following:

* **medminder-esp.yaml** - This file is my esphome yaml config for the D1 Mini. I have redacted the OTA and API keys, so you will have to manually enter them or simply copy and paste everything from line 30 and below into a preinstalled ESPHome config for your board
* **medminder.yaml** - This is the Home Assistant automation I'm using to drive this. It requires 5 helper entities to be created to maintain the state of things. Please see my write up below for the names I used, and if you want to change them, please change their references in the ESPHome config and in the automation as well so it can work for you.
* **MedMinder-Top.stl** - The top casing, where everything gets mounted to. It's not my finest work, but it printed perfectly fine in PLA+. I printed it in the orientation provided (bottle holder pointing up) and just did snug supports for the inner ceiling. Took about 2-3 hours on my Prusa Mk4 in PLA+
* **MedMinder-Bottom.stl** - A snapfit bottom but the tolerances might not be great. I used a few dots of hot glue to increase the friction between the pieces for now. 

# Goals

I had a few goals I wanted to accomplish with this project:

* Simple to use for regular people. Not everyone in the household can build or implement this, but placing a bottle in the holder or pushing the single button is pretty straight forward.
* Inexpensive
* Maintain most of the state/important data in HASS. The ESP is merely an input/output based on what is in Home Assistant. I know a lot of this code can be done entirely within the ESP, but I decided not to for reliability of maintaining the info during power outages and whatnot. My Home Assistant lives on a NUC in a server rack powered by a UPS. The ESP is plugged into a USB port on my nightstand, I didn't want to trust important data there.

# Requirements:

## Hardware BOM:

| Part | Qty | Link | Notes |
| --- | --- | --- | --- |
| WEMOS D1 Mini (ESP8266) | 1 | <https://a.co/d/5cXB2x8> | Went with the D1 Mini as I found an enclosure for this and the chosen PN532 module already, but you can use any ESP device. It just may require more modification to the ESPHome config. |
| PN532 RFID/NFC module (HW-147 red type) | 1 | <https://a.co/d/5j30uwH> | There are many out there, including the probably "easier to build around" Adafruit model. Again, found an enclosure that fit this one so that's what I went with. |
| 3mm Red LED | 1 |  |  |
| 3mm Green LED | 1 |  |  |
| 3mm Blue LED | 1 |  |  |
| 440Ω Resistors | 3 |  |  |
| Momentary Switch | 1 | <https://a.co/d/dofvmd4> | Any SPST Normally Open button will work here. The hole in the STL I used for the casing can take any 12mm button, but be careful on the depth. Anything deeper than \~13-15mm will be too big for the case. The button I used has an integrated LED as well, that I may integrate as a status LED or something. For an example without the LED see here: <https://a.co/d/6wV9Lmx> |
| NFC sticker | N/A | <https://a.co/d/ftuGgJh> | Any adhesive NFC sticker should work, they're cheap, and a 40 pack like this should last you over 3 years, assuming a 30 day supply each time. |



## Wiring Map:

This wiring map is only for use with a ESP8266 based Wemos D1 Mini. The number of connections will be the same, but GPIO pin mapping may vary if you use a different board.

| Physically Labeled Pin | GPIO Pin in ESPHome | Color Wire to Use | Connecting |
| --- | --- | --- | --- |
| 3v3 | N/A | Red | PN532 VCC |
| D1 | GPIO5 | Green | PN532 SCL |
| D2 | GPIO4 | Yellow | PN532 SDA |
| D4 | GPIO2 | White | Button NO terminal |
| D5 | GPIO14 | Red | RED LED + |
| D6 | GPIO12 | Green | Green LED + |
| D7 | GPIO13 | Blue | Blue LED + |
| D8 (Optional) | GPIO15 (Optional) | Blue | Button + Terminal |
| G | N/A | Black | PN532 G, - terminal of all three LEDs, - terminal of button, NO terminal on button |



## Home Assistant Helper Entities:

I've included the list of entities and what I named them below, if you want to use different ones, that's fine, but you will need to change the entity names in the ESPHome config as well as the Home Assistant automation script.

| Entity Type | Entity Name | What it does |
| --- | --- | --- |
| Dropdown | input_select.medminder_status | This is a dropdown that is the current status of the system, which can be one of four states: "Armed", "Disarmed", "Taken" and "Skipped". A full explanation of them is below. |
| Text | input_text.current_bottle_tag | This is populated by the ESP whenever you place a tagged bottle on the reader. It's mainly for comparing yesterday's tag to todays and if they change, then a 30 day cycle starts again |
| Text | input_text.last_bottle_tag | This is used to compare against the currently read tag from ESP. If they're the same, then the pill count is decremented and the status switches to taken (if status is armed). If the tags differ, then a new 30 day cycle begins, with a reminder created. |
| Number | input_number.pills_remaining | This is merely a count of the remaining pills in the 30 day cycle and has no other functionality in the app. Could be discarded if you don't need it. |
| Toggle | input_boolean.medminder_sleep_mode | This controls the sleep mode state. Sleep mode turns off the LEDs so if you have it on a nightstand the lights don't disturb you. This does not change the status of the system itself, just if the lights are on or off. Scheduling this is a possibility. |
| Local Calendar | calendar.medication_reminder | When a new tag is read, a calendar event is created 24 days from now at 10am. When the current time hits that event, a notification is triggered. |



## Explanation of operation

There are two main states we track in the system:

**Medminder Status** - This is the main control mechanism for MedMinder, and the three LEDs on the front indicate the status.

* Armed - Red LED on

  In the armed state, three things can happen.

  1- A tag is read, which means the medication was taken. MedMinder Status is set to "Taken"

  2- The button is pressed, which means the medication was skipped for the day. Status is set to "Skipped"

  3- The button is long pressed, which sets the system to "Disarmed".
* Taken - Green LED on

  Indicates the med was taken, and will stay in this state until the set time of day to "rearm" occurs
* Skipped - Blue LED on

  Indicates the med was skipped for the day, and will stay in this state until the set time of day to "rearm" occurs
* Disarmed - All LEDs off

  The system is turned off. This is useful if you need to mess with the tag or swap bottles without triggering the reset of the cycle.

**Sleep Mode** - This can turn off the LEDs without affecting the status of the system. Ideal for those who keep their AM medication on a nightstand next to their bed, by reducing light pollution in your bedroom . If the MedMinder will live somewhere else where the lights wont disturb sleep, you can ignore this and rip it out of the code if needed.



## Automation Flow Explanation:

8 events trigger the automation:

1. When a tagged bottle is placed on the reader

2. When the button is single pressed

3. When the button is long pressed

4. When the scheduled rearm time occurs

5. When the scheduled sleep mode start time occurs

6. When the scheduled sleep mode stop time occurs (currently sunrise)

7. When the time to request a refill occurs (Based off event in dedicated calendar entity)

8. When a time of day is reached but the system still shows as "armed"

Triggers 4, 5, 6 and 8 are all coded into the automation triggers and can be modified to fit your desired use case

The above triggers correspond to the following logics:

1. When bottle is placed on the reader, if the tag changes, the previous tag is updated to the new one, the pill count is reset to 30, and a calendar event is created 24 days in the future (at 10AM). Then if the system is armed, the status is changed to "Taken", and the pill count is decreased by 1.
2. When the button is pressed, and the system is armed, the status is set to Skipped.
3. If the button is long pressed, it will toggle the status between "Armed" and "Disarmed"
4. When the set time of day occurs, the status is set to "Armed"
5. When the set time of day occurs, sleep mode is turned on, which will turn off the main status lights, and turn on the LED in the button
6. When the set time of day occurs, sleep mode is turned off, which will turn the status lights back on based on the current status, and turn off the button LED.
7. When the current time is within the time for the created (25 days after the last bottle tag change) calendar event, a notification is fired off
8. When the set time of day occurs, it checks if the system is still armed and if it is, will fire a notification to remind to take it.



## Current plans for the future reversions

* Would like to consolidate the three LEDs down to one RGB LED or perhaps a NeoPixel ring around the holder. I think this would simplify wiring and make it look a bit nicer.
* Instead of having such a deep cavity for the container, perhaps a recess or something more open. Easier to print, can handle different/larger bottles, and would look nicer.
* Perhaps an i2c multiplexer and the ability to track multiple medications.
* Actionable HASS notifications, allowing to skip or logged as taken from the reminder notification in case it doesn't trigger for some reason



### Attribution:

A big thanks to MrGreen90 for his work on the case that I used for the basis of the one I used in this build. His original model can be located here: <https://www.thingiverse.com/thing:3609603>
