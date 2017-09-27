---
title:  "Controlling a AdvantageAir MyPlace controller from OpenHab"
date:   2017-09-27 21:00:00
categories: blog openhab myplace smarthome
---
# Controlling the AdvantageAir MyPlace Air Conditioner control system through OpenHab

## Pre-Requisites 
MyPlace Controller
Openhab Installed (OpenHabian is a great ready to go distribution) - Testing on OpenHab 2.1 & 2.2 (Beta).
HTTP Binding (Version 1 Binding http://docs.openhab.org/addons/bindings/http1/readme.html) 

The setup is simplified if a static IP address is assigned to your MyPlace system by your router rather than a dynamic IP.

## Steps
### Binding configuration
By default the HTTP Binding provides support to include the current state or command, or the current date/time. It does this with the help of java.util.Formatter. The AdvantageAir API unfortunetly chose to use the HTTP GET method for changing state of the system and has JSON strings in the query paramater. These conflict with java.util.Formatter so we need to disable this feature.

Append to $OPENHAB_CONF/services/http.cfg
```sh
# Disable 
format = false
```

It is hoped that in a future release of the API that state changes are changed to HTTP POST

### Configure HTTP Cache Item
The Advantage Air API has a single endpoint to query all system state. It is best to cache this query every 2 seconds (2000 milliseconds) to reduce the number of requests to the controller.

```sh
myAir5SystemDataJsonCache.url=http://<MYAIRIP/Hostname>:2025/getSystemData
myAir5SystemDataJsonCache.updateInterval=2000
```

### First Item - Air Conditioner Running State
We start with a our first, and most complicated, switch to show and control the power state for the Air Conditioner.
getSystemData returns a value of "on" or "off" for the AirCon running state. OpenHab expects capatalised ON and OFF for states, we therefore use a simple Javascript transformation to convert these values to switch state.

The field we are looking at looks like this in the getSystemData output (Extra information removed)
```javascript
{
"aircons": {
  "ac1": {
   "info": {
    "state": "on" - Aircon unit state - whether the unit is "on" or "off".
...
```

Create file $OPENHAB_CONF/transformations/myairAC1state.js
```javascript
JSON.parse(input)["aircons"]["ac1"]["info"].state.toUpperCase()
```

Then create an switch item for this state in $OPENHAB_CONF/items/myair.items configuring it to load the state from the myAir5SystemDataJsonCache http cache and transform it using the transformation we created. 
```sh
Switch MyAir5_AC1_State "Ducted Air Conditioner Power" <climate> (gMyAir) {  http="<[myAir5SystemDataJsonCache:3000:JS(myairAC1state.js)] 
}
```

### Aditional Air Conditioner Information Items
We can now add the other bits of information state provided by the MyPlace controller

```javascript
"aircons": {
 "ac1": {
 "info": {
 "countDownToOff": 0, - Number of minutes before the aircon unit switches off (0 - disabled)
 "countDownToOn": 0, - Number of minutes before the aircon unit switches on (0 - disabled)
 "fan": "high", - Fan speed - can be "low", "medium" or "high". Note some aircon units also support "auto".
 "mode": "heat", - Mode - can be "heat", "cool" or "fan". Note some aircon units support "dry".
 "myZone": 0, - MyZone settings - can be set to any zone that has a temperature sensor (0 - disabled)
 "setTemp": 24.0, - Set temperature of the aircon unit - this will show the MyZone set temperature if a MyZone is set.
```

These values are either Strings or Numbers. We can extract their values using JSONPATH instead of creating a JavaScript transform.

Append to the file $OPENHAB_CONF/items/myair.items which we created for the status switch.
```sh
String MyAir5_AC1_Mode "Mode" (gMyAir) { http="<[myAir5SystemDataJsonCache:2000:JSONPATH($.aircons.ac1.info.mode)]" }
String MyAir5_AC1_FanSpeed "Fan Speed" <fan> (gMyAir) { http="<[myAir5SystemDataJsonCache:2000:JSONPATH($.aircons.ac1.info.fan)]" }
Number MyAir5_AC1_TimerOff "Timer to Off" <clock> (gMyAir) { http="<[myAir5SystemDataJsonCache:2000:JSONPATH($.aircons.ac1.info.countDownToOff)]" }
Number MyAir5_AC1_TimerOn "Timer to On" <clock-on> (gMyAir) { http="<[myAir5SystemDataJsonCache:2000:JSONPATH($.aircons.ac1.info.countDownToOn)]" }
Number MyAir5_AC1_MyZone "MyZone" (gMyAir) { http="<[myAir5SystemDataJsonCache:2000:JSONPATH($.aircons.ac1.info.myZone)]" }
Number MyAir5_AC1_SetTemp "MyZone" (gMyAir) { http="<[myAir5SystemDataJsonCache:2000:JSONPATH($.aircons.ac1.info.setTemp)]" }
```

### Zone Configuration
Zones can be configured in multiple ways. In this example we will look at two values temperature and setpoint. This assumes a MyZone setup. If you are using Return Air temperature then the setTemp value from the previous step will need to be used.

Output for Zone 1 in getSystemData
```javascript
 "zones": {
 "z01": {
 "name": "FREEGGVFUX", - Name of zone - max 12 displayed characters
 "setTemp": 25.0, - Set temperature of the zone - only valid when Zone type > 0.
 "state": "open", - State of the zone - can be "open" or "close". Note: that the
 "type": 0, - Readonly - Zone type - 0 means percentage (use value to change), any other number means it's temperature control, use setTemp.
 "value": 20 - Percentage value of zone - only valid when Zone type = 0.
 }
```

Append to the file $OPENHAB_CONF/items/myair.items for each zone installed. Changeing z01 to the correct zone number.
```sh
Number MyAir5_AC1_Zone1_RoomTemp "Zone 1 Temperature [%.1f °C]" <temperature> (gMyAir, gTemperatureSensor) { http="<[myAir5SystemDataJsonCache:2000:JSONPATH($.aircons.ac1.zones.z01.measuredTemp)]" }
Number MyAir5_AC1_Zone1_SetTemp "Zone 1 SetPoint [%.1f °C]" <temperature> (gMyAir, gACSetPoint) { http="<[myAir5SystemDataJsonCache:2000:JSONPATH($.aircons.ac1.zones.z01.setTemp)]" }
```

### Controling the system with Rules
Rules allow us to link widgets from the BasicUI to the MyPlace API. Rules are a simpler method available for our use rather than wiring up the switches directly with the HTTP bindings.

For example here is what the Air Conditioning ON/OFF switch (MyAir5_AC1_State) looks like when configured via the bindings. Due to the JSON string passed to setAircon we need to URL encode the parameter.

So json={"ac1":{"info":{"state":"on"}}} becomes json=%7B%22ac1%22%3A%7B%22info%22%3A%7B%22state%22%3A%22on%22%7D%7D%7D

```sh
Switch MyAir5_AC1_State "Ducted Heating Power" <climate> (gMyAir) { http="<[myAir5SystemDataJsonCache:2000:JS(myairAC1state.js)] >[ON:GET:http://myplaceip:2025/setAircon?json=%7B%22ac1%22%3A%7B%22info%22%3A%7B%22state%22%3A%22on%22%7D%7D%7D] >[OFF:GET:http://myplaceip:2025/setAircon?json=%7B%22ac1%22%3A%7B%22info%22%3A%7B%22state%22%3A%22off%22%7D%7D%7D]" }
```

Let's create a new file $OPENHAB_CONF/rules/myair.rules for out rules. We create a variable myAirEndpoint to allow a single point in our rules to be updated if the MyPlace controller IP changes.

```java
# MyPlace SetAircon API Endpoint
var String myAirEndpoint = "http://<MyPlaceIP>:2025/setAircon?json="
```

Rules for each component in the MyPlace system is then defined by reacting to receiving a command on one of our items and trnslating that into a call to setAircon. We use command rather than item value change since the MyPlace system values can be modified by the MyPlace UI or via the phone application.

Example rule for changing the Set Temperature for Zone 1
```java
rule "SetZone1MyAirSetPoint"
when
  Item MyAir5_AC1_Zone1_SetTemp received command
then
  var String json = '{"ac1":{"zones":{"z01":{"setTemp":"' + MyAir5_AC1_Zone1_SetTemp.state.toString + '"}}}}'
  sendHttpGetRequest(myAirEndpoint + json.encode("UTF-8"))
end
```

Set the timer to turn off the AC after a set value
```java
rule "SetMyAirTimerToOff"
when
  Item MyAir5_AC1_TimerOff received command
then
  var String json = '{"ac1":{"info":{"countDownToOff":' + MyAir5_AC1_TimerOff.state.toString + '}}}'
  sendHttpGetRequest(myAirEndpoint + json.encode("UTF-8"))
end
```

When changing the zone used by MyZone you need to ensure two commands are sent. The first is to ensure the Zone we want to enable is in an open position, changing the MyZone value doesn't automatically do this. Secondly we need to set the MyZone value.

```java
rule "SetMyAirMyZone"
when
  Item MyAir5_AC1_MyZone received command
then
  json = '{"ac1":{"zones":{"z0' +  MyAir5_AC1_MyZone.state.toString + '":{"state":"open"}}}}'
  sendHttpGetRequest(myAirEndpoint + json.encode("UTF-8"))
  var String json = '{"ac1":{"info":{"myZone":' + MyAir5_AC1_MyZone.state.toString + '}}}'
  sendHttpGetRequest(myAirEndpoint + json.encode("UTF-8"))
end
```

Note: the example above will not work for a 10th zone if you have all 10 zones installed. The json for opening the zone needs to be adjusted for this to work.

### Example Sitemap for BasicUI

$OPENHAB_CONF/sitemap/default.sitemap (Or choose your sitemap)
```sh
    Frame label="Climate" {
        Switch item=MyAir5_AC1_State
        Selection item=MyAir5_AC1_Mode label="AC Mode" mappings=[cool="Cool", heat="Heat", vent="Fan", dry="DRY"]
	Selection item=MyAir5_AC1_MyZone label="MyZone" mappings=[1="Room 1", 2="Room 2", 3="Room 3"]

        Text item=MyAir5_AC1_Zone1_RoomTemp
        Setpoint item=MyAir5_AC1_Zone1_SetTemp
    }
```

### Control away
Browse to your BasicUI sitemap you just modified and you can control the AC Power State, Mode, MyZone and Set Point for Zone 1. You can also see the current temperature recorded in Zone 1.
