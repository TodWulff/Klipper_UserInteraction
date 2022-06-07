# Klipper_UserInteraction
A set of macros to enable print-time user interaction with Klipper via Console and using UI [macro] buttons.

## PREREQUISITES:

Heavily relies on [Save_Variables] module
see: https://github.com/TodWulff/V2.2526_Config/blob/main/_persistent_variables.cfg

Makes use of **`M300**` and some custom **`M300_**` related macros I mucked with for emitting sounds, so if you're not so
interested, or haven't a beeper on your box, comment out related lines: _ui_reminder, _annunciate_input_exception,
and _annunciate_good_input - see: https://github.com/TodWulff/V2.2526_Config/blob/main/_m300_sounds.cfg

Makes heavy use of the **`[response]**` module - I've trapped the stock M118 (using rename_existing) with gcode such
that it uses action_notification vs. FW M118 code.  This enables emission of 'special characters' that M118's
FW code chokes on - see: https://github.com/TodWulff/V2.2526_Config/blob/main/_gcode_macros.cfg

Throughout my configs, virtually all gcode macros have been 'instrumented' - the first and last lines of each gcode
block can be deleted.  I have these as I have an ability to trace macro code execution and display same in the
console - quite powerful for macro development/troubleshooting but of little/no use to others, with rare exception
see: https://github.com/TodWulff/V2.2526_Config/blob/main/_debug_macros.cfg

Users should add a User Input macro pane, as depicted here: https://i.imgur.com/QVxLuVZ.png which calls macros herein
Orienting it under the console pane will allow it to become intuitive with a small bit of use

## Primary Macro:  GET_USER_INPUT  Parameters and related dialog follows

Optional Parameters:
**PROMPT**		Text to display in console as a user input prompt
**RCVR_MACRO**	Macro name to run when VALID input received - UI_INPUT param passed to proc - if reqd, query svv for moar info
**TYPE**		The TYPE of input needed - one of these three string/str, integer/int, float/flt
**BOUNDS_HI**	if TYPE is float/flt, input must be >=lo and <=hi - for integer/int, input must be >=floor(lo) and <=ceiling(hi)
**BOUNDS_LO**	 - if string/str TYPE, character count must be >=floor(lo) and <=ceiling(hi)
**TO_PERIOD**	Period in Integer seconds to wait for user input - a reminder M300 fires at rate in _ui_vars while waiting
**EXCPT_HDLR**	Macro name is called in the event of an input timeout or faulty input - no params passed - query svv...

For string TYPE, if `BOUNDS_LO` is not asserted, defaults to 1
For string TYPE, if `BOUNDS_HI` is not asserted, defaults to 255

For float TYPE, if `BOUNDS_LO` is not asserted, defaults to -999999999.0
For float TYPE, if `BOUNDS_HI` is not asserted, defaults to 999999999.0

For integer TYPE, if `BOUNDS_LO` is not asserted, defaults to -999999999
For integer TYPE, if `BOUNDS_HI` is not asserted, defaults to 999999999

If additional bounds testing is desired/required, it's up to the user implementing this on their printers to craft more granular
test and validation - by way of the RCVR_MACRO.  If I've blantantly missed something, then let me know. :)

For optional parameters, read the code to understand implications of relying on defaults.

Again, ALL options are optional and default to something - defaults are as entered or noted
get_user_input	PROMPT="Enter or Click something:"		# displayed on the console at macro start as a user prompt
		TYPE=STRING					# 'string' or 'integer' or 'float' - for buttons use string
		BOUNDS_LO=1		(Int/Flt -999999999)	# min string chars or min numercial value (int/flt)
		BOUNDS_HI=255		(Int/Flt  999999999)	# max string chars or max numercial value (int/flt)
		RCVR_MACRO=_test_show_user_input		# to accept param UI_INPUT that will be an int/flt/"string" that was 
								# input and passes sniff (simple bounds) test
		TO_PERIOD=120					# in seconds
		EXCPT_HDLR=_ui_exception_handler		# no params passed - use svv to get runtime specifics if needed

Example call follows:
This looks for a user to enter a string 1-12 chars long, with a timeout of 60 secs, that forwards (via UI_INPUT param),
the entered string to the '_test_show_user_input' macro (default if no RCVR_MACRO passed by user call)
If a timeout happens/faulty input is detected, the _ui_timeout_watchdog/_validate_user_input macros call '_ui_exception_handler' macro
(which is the default exception handler if no EXCPT_HDLR macro name is passed by the user call)

### get_user_input PROMPT="enter/click something:" TYPE=string BOUNDS_LO=1 BOUNDS_HI=12 RCVR_MACRO=_test_show_user_input TO_PERIOD=60 EXCPT_HDLR=_ui_exception_handler

## EXCEPTION HANDLER MACRO:
if a custom EXCPT_HDLR macro is to be instantiated, it may prove useful to consider the following:
 - GET_USER_INPUT does the following:
    a. initializes states and then displays the user PROMPT
    b. sets the timeout watchdog to fire after the passed/default TO_PERIOD
    c. puts _await_user_input into a recursive loop
 - a single EXCPT_HDLR macro services both bad input cases as well as the time-out when waiting on user input.
 - for timeouts the default EXCPT_HDLR macro simply recalls GET_USER_INPUT giving user another input context
   - this approach can be altered with a custom EXCPT_HDLR macro being passed to the GET_USER_INPUT call
 - When _await_user_input senses user input, _validate_user_input tests for TYPE and BOUNDS_HI/BOUNDS_LO compliance
   - if input is NOT TYPE & BOUNDS_HI/BOUNDS_LO compliant, _await_user_input recalls GET_USER_INPUT so user can fix it
   - if input IS TYPE & BOUNDS_HI/BOUNDS_LO compliant (_ui_bad_input NOT set), input is sent to RCVR_MACRO via UI_INPUT param

in most conceivable use cases, the default exception handler/validation macros should be adequate, but want to give others
options, in the event I haven't considered something.  Either an entirely new EXCPT_HDLR can be crafted or, as demonstrated
in _ui_test.cfg's _ui_test_exception_handler (custom EXCPT_HDLR) code runs and then calls the stock _ui_exception_handler below

In _ui_vars, a person implementing this can selectively enable/disable Input Prompt and/or Exception hints.  Hints are
little descriptive blurbs as to what the code is expecting as input - one can enable hints on either the input prompt,
or disable hints when an input exception is detected/raised.  It is suggested that it is likely best to have exception
hints enabled, and to have the input prompt detail what sort of input is desired.  The _ui_test.cfg file has the START_DEMO
macro that iterates through some UI input event - I used it for dev testing, it works as a demo.
Some visual examples as it relates to the hint options that can be set in _ui_vars:

No Hints on Exception nor Input Prompt: https://i.imgur.com/PDPOmTJ.png (likely too? terse - make prompt/preamble detailed)

![https://i.imgur.com/PDPOmTJ.png](https://i.imgur.com/PDPOmTJ.png)

Hints on Exception only, not on Input Prompt:  https://i.imgur.com/D5Ih6hE.png (the author's preference)

![https://i.imgur.com/D5Ih6hE.png](https://i.imgur.com/D5Ih6hE.png)

Hints on Input Prompt but not on an Exception:  https://i.imgur.com/Deo2SNr.png (leads to some noise, which may be tolerable)

![https://i.imgur.com/Deo2SNr.png](https://i.imgur.com/Deo2SNr.png)

Hints on both Input Prompt and when an exception is raised:  https://i.imgur.com/mO7TfWW.png (likely redundant, imo)

![https://i.imgur.com/mO7TfWW.png](https://i.imgur.com/mO7TfWW.png)
