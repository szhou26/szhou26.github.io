# ECE 3140 Final Project: Morse Code Input System

## Overview
This project implements a real-time Morse Code input system using the FRDM-KL46Z microcontroller, allowing users to convert onboard button-press patterns into alphanumeric messages that are decoded and displayed on the computer terminal via UART. The setup provides a simple and intuitive interface inspired by historical telegraph systems.

**General System Diagram**

<img src="diagram.png" alt="General System Diagram" width="600">

## Technical Approach 
**Buttons**

Implementing the project involves configuring the microcontroller’s GPIO inputs to detect the state of pushbuttons SW1 and SW3, which are connected to pins PTC3 and PTC12, respectively. The onboard buttons were configured with internal pull-up resistors to ensure a stable logic-high state when unpressed. Pressing SW1 inputs Morse Code, and pressing SW3 resets the input and clears the terminal.

**Timing and Input Classification**

To interpret the input (whether a press is a dot or dash), the program uses the SysTick timer to precisely measure how long the button is held and how long the system remains idle between presses. This timer is configured to generate interrupts every millisecond: *SysTick_Config(SystemCoreClock/1000);*

When the timer interrupt occurs, _SysTick_Handler()_ increments a global uint32_t counter variable called tick. 

The Morse timing definitions:
 *  length of dot = 1 unit ---> 300 ms
 *  length of dash = 3 units ---> 3 x 300 ms
 *  space between letters = 3 units ---> 3 x 300 ms (button not pressed)
 *  space between words = 7 units ---> 7 x 300 ms (button not pressed)
 *  sentence/message time out = 10 units ---> 10 x 300 ms (button not pressed)

When SW1 is pressed, the press duration is calculated by subtracting the press start time from the current tick. The input is then classified as a dot or dash according to the corresponding thresholds. 

**Decoding**

The decoded character is added to a *morse* buffer at the current _morse_idx_ (an index variable), and echoed to the terminal using uart_putc(). After each symbol is added, _morse[morse_idx++] = '.' or '-'_ updates the buffer, and the string is null-terminated to maintain compatibility with decoding functions.

The system continuously monitors idle time (the time since the last button release, tracked via _tick - last_release_) to detect input boundaries:

* If idle_time ≥ 0.9 s, the system treats this as the end of a character. The function _decode_morse(morse)_ converts the accumulated sequence into an alphanumeric character. This character is then stored in the _sentence_ buffer at index _sent_idx_, which is incremented accordingly. The _morse_ buffer is then cleared by resetting morse_idx = 0 and morse[0] = '\0'.
* If idle_time ≥ 2.1 s, the system interprets this as a word boundary. It checks whether the last character in the sentence buffer is already a space (' '), and if not, inserts one. This prevents multiple spaces from being added between words.
* If idle_time ≥ 3 s, the system concludes that the sentence is complete. The full sentence buffer is printed to the terminal using _uart_puts(sentence)_, and the buffer is cleared.

The program uses a lookup table implemented as an array of _MorseEntry_ structs, where each entry contains a Morse code string and its corresponding character. The table includes mappings for all 26 letters of the alphabet and the digits 0–9. The _decode_morse()_ function takes a null-terminated Morse code string as input and iterates through the table, comparing the input to each entry using _strcmp()_. If a match is found, the function returns the corresponding character. If no match is found, it returns '?', indicating the input was invalid or unrecognized. The fully decoded message is displayed after the timeout threshold (3 seconds).

**Serial Communication**
This project uses UART0 serial communication to display both the real-time Morse code input and the decoded output on a computer terminal. UART0 is configured at the default baud rate of 115200 and communicates with the host computer via a virtual COM port. Pressing SW3 calls _uart_puts("\033[2J\033[H")_, an escape sequence to clear the screen and move the cursor to the top left.

## Testing and Debugging
Testing for this project was carried out iteratively after the initial implementation. The first step was verifying basic hardware functionality—ensuring that SW1 and SW3 inputs could be detected reliably and that UART output was properly configured to print to the terminal. A core challenge  was confirming that button press durations were measured accurately, that pauses could be interpreted correctly as letter and word boundaries, and that decoded letters appeared as expected.

After single-letter decoding was functioning, the system was expanded to support multiple words/sentences, since I did not initially account for pauses as spaces. This was refined by introducing a timeout to allow users to type multiple words before the system assumes the user is finished. Additional logic was added to prevent the insertion of multiple spaces between words and to track when a space had already been added.

Another subtle issue involved buffer clearing. At first, pressing SW3 didn't reliably clear the sentence buffer or reset the display, due to a missing flag reset. This was resolved by explicitly resetting all relevant indices and buffers in the SW3 logic block. Button press debounce was also considered during testing, particularly for SW3, to ensure that short or unintended presses wouldn't trigger multiple actions.

## Outside Resources 
 * KL46 Sub-Family Reference Manual
 * This project uses uart_example.c from ECE 3140's uploaded Serial Code Example.
 * switch_demo.c was also referenced to help with configuring buttons and polling.
 * I searched the web to learn about the SysTick Timer and how UART can clear the screen (stack overflow).
 * https://www.ti.com/lit/ml/swrp171/swrp171.pdf?ts=1747140822524
 * https://en.wikipedia.org/wiki/Morse_code
