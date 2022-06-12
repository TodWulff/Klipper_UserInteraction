# Klipper_UserInteraction
A set of macros to enable print-time user interaction with Klipper via Console and UI buttons (macros).

Video of an early implementation: https://youtu.be/pgBfhVAYsHU

Here is an example of a real world use case - optionally rebooting after an ERCF calibration event:
![https://i.imgur.com/9GNaG8N.png](https://i.imgur.com/9GNaG8N.png)

## PREREQUISITES:

Heavily relies on `[Save_Variables]` module
see: https://github.com/TodWulff/V2.2526_Config/blob/main/_persistent_variables.cfg

Makes use of **`M300`** and some custom **`M300_`** related macros I mucked with for emitting sounds, so if you're not so
interested, or haven't a beeper on your box, comment out related lines: `_ui_reminder`, `_annunciate_input_exception`,
and `_annunciate_good_input` - see: https://github.com/TodWulff/V2.2526_Config/blob/main/_m300_sounds.cfg

Makes heavy use of the **`[response]`** module - I've trapped the stock M118 (using rename_existing) with gcode such
that it uses action_notification vs. FW M118 code.  This enables emission of 'special characters' that M118's
FW code chokes on - see: https://github.com/TodWulff/V2.2526_Config/blob/main/_gcode_macros.cfg

Throughout my configs, virtually all gcode macros have been 'instrumented' - the first and last lines of each gcode block can be deleted.
I have these as I have an ability to trace macro code execution and display same in the console - quite powerful for macro development/
troubleshooting but of little/no use to others, with rare exception - 
see: https://github.com/TodWulff/V2.2526_Config/blob/main/_debug_macros.cfg

Users should consider adding a User Input macro pane, as depicted here: https://i.imgur.com/QVxLuVZ.png which calls macros herein.  Orienting it under the console pane will allow it to become intuitive with a small bit of use.  Author's UI of choice is Mainsail.  For Fluidd or other Moonraker Clients (Mooncord/Telegram Bot/...), I defer to others to adapt the macros for use therein.

## Primary Macro:  GET_USER_INPUT  Parameters and related dialog follows

Optional Parameters:

**`PROMPT`**		Quoted Text to display in console as a user input prompt.  Promts are vistually separated from balance
of console context via a dashed line being displayed before and after the prompt (see images below).

**`RCVR_MACRO`**	Macro name to run when VALID input received - UI_INPUT param, containing user input contents is 
passed to the called macro - if required, the called user macro can query svv for additional details.

**`TYPE`**		The TYPE of input needed - one of these three string/str, integer/int, float/flt

**`BOUNDS_HI`**	if TYPE is float/flt, input must be >=lo and <=hi - for integer/int, input must be >=floor(lo) and <=ceiling(hi)

**`BOUNDS_LO`**	 - if string/str TYPE, character count must be >=floor(lo) and <=ceiling(hi)

**`TO_PERIOD`**	Period in Integer seconds to wait for user input - a reminder M300 fires at rate in _ui_vars while waiting

**`EXCPT_HDLR`**	Macro name is called in the event of an input timeout or faulty input - no params passed - query svv...

For string TYPE, if **`BOUNDS_LO`** is not asserted, defaults to 1.

For string TYPE, if **`BOUNDS_HI`** is not asserted, defaults to 255.

For float TYPE, if **`BOUNDS_LO`** is not asserted, defaults to -999999999.0.

For float TYPE, if **`BOUNDS_HI`** is not asserted, defaults to 999999999.0.

For integer TYPE, if **`BOUNDS_LO`** is not asserted, defaults to -999999999.

For integer TYPE, if **`BOUNDS_HI`** is not asserted, defaults to 999999999

If additional bounds testing is desired/required, it's up to the user implementing this on their printers to craft more granular
test and validation - by way of the **`RCVR_MACRO`**.  If I've blantantly missed something, then let me know. :)

For optional parameters, read the code to understand implications of relying on defaults.

Again, ALL parameters are optional and default to something:

	PROMPT="Awaiting User Input:"		displayed on the console at macro start as a user prompt
	TYPE=STRING				'string' or 'integer' or 'float' - for buttons use string
	BOUNDS_LO=1				min string chars or min numercial value (Int/Flt -999999999)
	BOUNDS_HI=255				max string chars or max numercial value (Int/Flt  999999999)
	RCVR_MACRO=_test_show_user_input	to accept param UI_INPUT that will be an int/flt/"string" that was 
						input and passes sniff test (simple bounding checks)
	TO_PERIOD=120				in seconds
	EXCPT_HDLR=_ui_exception_handler	no params passed to this proc - use svv to get runtime specifics if needed

### Example call follows:
This call looks for a user to enter a string 1-12 chars long, with a timeout of 60 secs, that forwards (via UI_INPUT param),
the entered string to the '_test_show_user_input' macro (default if no RCVR_MACRO passed by user call)
If a timeout happens/faulty input is detected, the _ui_timeout_watchdog/_validate_user_input macros call '_ui_exception_handler' macro
(which is the default exception handler if no EXCPT_HDLR macro name is passed by the user call)

`get_user_input` `PROMPT="enter/click something:"` `TYPE=string` `BOUNDS_LO=1` `BOUNDS_HI=12` `RCVR_MACRO=_test_show_user_input` `TO_PERIOD=60` `EXCPT_HDLR=_ui_exception_handler`

## EXCEPTION HANDLER MACRO:
if a custom **`EXCPT_HDLR`** macro is to be instantiated, it may prove useful to consider the following:
 - **`GET_USER_INPUT`** does the following:
   - initializes states and then displays the user **`PROMPT`**
   - sets the timeout watchdog to fire after the asserted (120s default) **`TO_PERIOD`**
   - puts **`_await_user_input`** into a recursive loop, with a period defined in _ui_vars
 - a single **`EXCPT_HDLR`** macro services both bad input cases as well as the time-out when waiting on user input.
 - for timeouts the **default** **`EXCPT_HDLR`** macro simply recalls **`GET_USER_INPUT`** giving user another input context
   - this approach can be altered with a custom **`EXCPT_HDLR`** macro being passed to the **`GET_USER_INPUT`** call
 - When `_await_user_input` detects input from user, `_validate_user_input` tests for **`TYPE`** and **`BOUNDS_HI`**/**`BOUNDS_LO`** compliance
   - if input is NOT **`TYPE`** & **`BOUNDS_HI`**/**`BOUNDS_LO`** compliant, `_await_user_input` recalls **`GET_USER_INPUT`** so user can fix it
   - if input IS **`TYPE`** & **`BOUNDS_HI`**/**`BOUNDS_LO`** compliant (flag `_ui_bad_input` NOT set), input is sent to **`RCVR_MACRO`** via **`UI_INPUT`** param

In most conceivable use cases, the default exception handler/validation macros should be adequate.  However, author decided to give others
options, in the event something wasn't adequately considered.  Either an entirely new **`EXCPT_HDLR`** can be crafted or, as demonstrated
in _ui_test.cfg's `_ui_test_exception_handler` (custom **`EXCPT_HDLR`**) code runs and then chains to the stock `_ui_exception_handler` below

In `_ui_vars` macro, a person implementing this can selectively enable/disable Input Prompt and/or Exception hints.  Hints are
little descriptive blurbs as to what the macro is expecting as input - one can enable hints on either the input prompt,
or disable hints when an input exception is detected/raised.  It is suggested that it is likely best to have exception
hints enabled, and to have the input prompt detail what sort of input is desired, leaving input hints disabled.
The `_ui_test.cfg` file has the START_DEMO macro that iterates through some UI input event - I used it for dev testing,
it works as a demo.

## Some visual examples as it relates to the hint options that can be set in `_ui_vars`:

No Hints on Exception nor Input Prompt: https://i.imgur.com/PDPOmTJ.png (likely too terse - user should make the preamble/prompt detailed)

![https://i.imgur.com/PDPOmTJ.png](https://i.imgur.com/PDPOmTJ.png)

Hints on Exception only, not on Input Prompt:  https://i.imgur.com/D5Ih6hE.png (the author's preference)

![https://i.imgur.com/D5Ih6hE.png](https://i.imgur.com/D5Ih6hE.png)

Hints on Input Prompt but not on an Exception:  https://i.imgur.com/Deo2SNr.png (leads to some noise, which may be tolerable)

![https://i.imgur.com/Deo2SNr.png](https://i.imgur.com/Deo2SNr.png)

Hints on both Input Prompt and when an exception is raised:  https://i.imgur.com/mO7TfWW.png (likely redundant, imo)

![https://i.imgur.com/mO7TfWW.png](https://i.imgur.com/mO7TfWW.png)

## Closing comments:
This was/is a quite deep rabbit hole.  Author `MegaHurtz 🇺🇸#6544` can be reached on a number of different Discord servers:
- Voron Kit Feedback
- Voron Design
- Klipper
- Mainsail
- ...

Don't hesitate to reach out as may be needed.  Have a great day.  Happy Printing!

~MHz
