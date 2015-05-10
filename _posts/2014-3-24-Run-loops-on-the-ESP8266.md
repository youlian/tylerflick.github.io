---
layout: post
title: Run loops on the ESP8266
category: ESP8266
author: Tyler Hoeflicker
icon: T
image: esp8266.jpeg
excerpt: Getting started with loop events in the esp-open-sdk.
---

Schedualing tasks on the ESP8266 isn't as simple as placing a function call in the `loop()` function of an Arduino sketch. Instead, Espressif's SDK exposes three timer functions that allow to you pass a function pointer as a callback. You only need two of these to get a loop up and running. These are `os_timer_setfn` and `os_time_arm`. All of the timing functions reside in the `osapi.h` library, but you'll need to include `os_type.h` as well to deal with types these functions expect. 

### Creating a simple loop
The timing functions can be used inside of any function, but to keep this example simple I'll placing them in `user_init(void)`, the entry point of the user's executable code.

#### Declaring Timer's data object

To begin a timer stuct needs to be declared (not initialized). 

{% highlight c%}
os_timer_t timerData;
{% endhighlight %}

This struct is passed around and holds the data for the timing event, but don't worry about the internals of this object as the SDK's methods should be the only one's mutating it.


#### Creating the User's callback function
This is where the code you want executed in a loop resides. A simple example to report back to the user's machine over a serial connections would look like this:

{% highlight c%}
void reportToUser(void* voidPtr) {
	os_printf("\r\n Timer callback function has been called. \r\n");
}
{% endhighlight %}

Technically the void pointer isn't needed in the function signature, void will do. If the function was expecting data the pointer would be necessary and then could be cast to the size of the data type expected.

{% highlight c%}
void reportToUser(void* voidPtrPointingToAnInt) {
	// Cast the void pointer to int pointer and dereference
	int callbackNumber = *(int*)voidPtrPointingToAnInt;
}
{% endhighlight %}

#### Setting the user's function as a callback
At this point we need a way to tell the SDK's timer library to call the function we just created. This is accomplished with `os_timer_setfn` and looks like this:
	
{% highlight c%}
os_timer_setfn(&timerData, (os_timer_func_t*)reportToUser, NULL);
{% endhighlight %}
	
This function expects three arguments.

1. First a pointer to the timer struct we declared earlier.
2. The second is a pointer to the callback function cast as type `os_timer_func_t`.
3. The last is a void pointer. I've set this up to pass `NULL` since I my callback function isn't expecting any data. If the function relied on external data this is where it would be passed to it (See the example above for how this works).

#### Start the timer

All that's left to do is to fire off the timer.

{% highlight c%}
os_timer_arm(&timerData, 1000, 1);
{% endhighlight %}
	
So what's going on here? Like the previous function the `os_timer_t` struct is passed. At this point the struct contains data about the user's callback. Next is the delay between callbacks in milliseconds as a 32 bit unsigned int. Finally is a boolean value letting the timer no to continue repeating. 1 being true and 0 being false.

That's all that's needed. To sum up what the complete user_main.c file might look like:

{% highlight c%}
#include "driver/uart.h"
#include "osapi.h"
#include "os_type.h"

os_timer_t timerData;

void reportToUser(void* voidPtr) {
	os_printf("\r\n Timer callback function has been called. \r\n");
}

void user_init(void) {
	// Init serial communication
	uart_init(BIT_RATE_115200, BIT_RATE_115200);

	// Set reportToUser as the callback function of the timer
	os_timer_setfn(&timerData, (os_timer_func_t*)reportToUser, NULL);
	// Fire callback every second and repeat
	os_timer_arm(&timerData, 1000, 1);
}
{% endhighlight %}
	
#### A few things to note
* In my setup I'm using the [esp-open-sdk](https://github.com/pfalcon/esp-open-sdk).
* If you need a good `MAKE` file checkout the one used in Tuan PM's [MQTT library](https://github.com/tuanpmt/esp_mqtt).
	* You'll need to change the `SDK_BASE` variable to point the `sdk` directory esp-open-sdk, or wherever your sdk reference is.
* A great starting point for ESP8266 development can be find on clonh's guest Hackaday [post](http://hackaday.com/2015/03/18/how-to-directly-program-an-inexpensive-esp8266-wifi-module/).