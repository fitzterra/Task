A very simple Arduino task manager
==================================

----

**Copyright, License and original work notice:**

Please note that this is **NOT** my work and I do **NOT** claim any form of
ownership hereof.
I am merely setting this library up as a GIT repo to make it easier for me to
reference it in various projects since this code is extremely useful to me.

The original work belongs to **Alan Burlison** and the text below is copied directly
from [Alan's Site]

Original text from Alan's site follows - I have tried to keep it as close as
possible to the original, with only emphasis provided, using markdown syntax,
where I think it is implied:

----

The LED chain project I'm working on requires that the AVR microcontroller
handles several different tasks:

* Read a rotary encoder and switch
* Drive a seven-segment display
* Drive a radio
* Drive four LED strips each containing twenty LEDs
* Provide logic to tie all the above together

Normally that sort of multiple-task workload might suggest the use of a [RTOS],
but the [Arduino Pro Mini] has only 32Kb of program memory (of which 2Kb is used
by the bootloader) and 2Kb of RAM, so every byte is precious. So, whatever we
come up with has to be as minimal as possible. OK, let's see if we can make some
assumptions and trade-offs that will help keep things simple and create a small
set of C++ classes we can use to implement the task management we need:

* Use cooperative multitasking rather than preemptive multitasking. Out of that
  fall a couple of constraints:
    * Each task must complete quickly when it runs. Long-running operations such
      as `delay()` are forbidden.
    * Any interrupt service routines must also complete quickly, preferably by
      recording the details of the event and scheduling a task to handle it.
* The task management system should not require the use of dynamic memory
  management (e.g. `malloc()`) so as to minimise memory requirements.
* The list of tasks will be fixed at compile-time. That's reasonable as the
  configuration of the system is fixed - we aren't going to be adding new
  hardware on the fly.
* Tasks will be scheduled in priority order to allow processing that has strict
  time constraints to be handled first, but to keep things simple the priority
  order will be fixed at compile time. Again, that's reasonable as the
  configuration of the system is known in advance.
* Tasks can communicate with each other by making standard C++ method calls but
  (as for interrupts) any such methods should simply store the details of the
  event and schedule themselves to be run to handle the event.
* Much of the processing we are doing is time-driven, e.g. sequencing the LED
  patterns, so explicit support for scheduling tasks at specific intervals
  should be provided, as well as more general _'triggered'_ tasks. 

The last constraint needs some further thought - what timer _'tick'_ interval
should we use? The _'real world'_ events we will be dealing with won't be
happening quicker that 1 millisecond apart and the CPU is clocked at 16MHz which
equates to 16000 instructions per millisecond. Scheduling time-based tasks at
millisecond resolution will allow us to run several tasks within the same
_'tick'_ and will be more than fast enough to deal with the events we have to
handle.

OK, so given all that, what would a suitable task implementation look like?

```c++
class Task {
public:
    virtual bool canRun(uint32_t now) = 0;
    virtual void run(uint32_t now) = 0;
};

class TimedTask : public Task {
public:
    inline TimedTask(uint32_t when) { runTime = when; }
    virtual bool canRun(uint32_t now);
    inline void setRunTime(uint32_t when) { runTime = when; }
    inline void incRunTime(uint32_t inc) { runTime += inc; }
    inline uint32_t getRunTime() { return runTime; }
protected:
    uint32_t runTime;
};

bool TimedTask::canRun(uint32_t now) {
    return now >= runTime;
}
```
`

Yep, that's really all there is to it. Each task can be queried to see if it can
be run via the `canRun()`, method and if it can, it will be executed via a call to
its `run()` method. We pass in the current time in milliseconds to avoid each task
having to separately determine it. The `canRun()` and `run()` methods _could_ be
merged, but having them separate allows us to provide more flexible scheduling
if we ever need to, e.g. by penalising tasks that are runnable too often.

OK, next step is to implement the actual task scheduler:

```c++
class TaskScheduler {
public:
    TaskScheduler(Task **task, uint8_t numTasks);
    void run();
private:
    Task **tasks;
    int numTasks;
};

TaskScheduler::TaskScheduler(Task **_tasks, uint8_t _numTasks) :
  tasks(_tasks),
  numTasks(_numTasks) {
}

void TaskScheduler::run() {
    while (1) {
        uint32_t now = millis();
        Task **tpp = tasks;
        for (int t = 0; t < numTasks; t++) {
            Task *tp = *tpp;
            if (tp->canRun(now)) {
                tp->run(now);
                break;
            }
            tpp++;
        }
    }
}
```
`

Again, that really is all there is to it. We create the task scheduler with a
fixed list of tasks, then call its run method. The run method iterates endlessly
over the task list, calling the `canRun()` method on each in turn to see if it
needs to be run. If it does, its `run()` method is called to execute the task.
One very important note: after running a task we break out of the iteration over
the task list and start back at the top of the list. That gives us the fixed
task priority feature - if multiple tasks are runnable the earlier tasks on the
list will always be dispatched first and the later tasks on the list will be
lower priority,

The last part is to show an example of how to actually use the scheduler. Each
task is derived from either the base `Task` class or from `TimedTask`, depending
on how it needs to be run. And example `TimedTask` that blinks the pin 13 LED
might look something like this:

```c++
// Timed task to blink a LED.
class Blinker : public TimedTask
{
public:
    // Create a new blinker for the specified pin and rate.
    Blinker(uint8_t _pin, uint32_t _rate);
    virtual void run(uint32_t now);
private:
    uint8_t pin;      // LED pin.
    uint32_t rate;    // Blink rate.
    bool on;          // Current state of the LED.
};

Blinker::Blinker(uint8_t _pin, uint32_t _rate)
: TimedTask(millis()),
  pin(_pin),
  rate(_rate),
  on(false)
{
    pinMode(pin, OUTPUT);     // Set pin for output.
}

void Blinker::run(uint32_t now)
{
    // If the LED is on, turn it off and remember the state.
    if (on) {
        digitalWrite(pin, LOW);
        on = false;
    // If the LED is off, turn it on and remember the state.
    } else {
        digitalWrite(pin, HIGH);
        on = true;
    }
    // Run again in the required number of milliseconds.
    incRunTime(rate);
}
```
`

If you compare this to the standard [Blink sketch] you'll see there are no calls
to `delay()` in the `run()` method. Instead the method simply toggles the LED
state, sets up when it is next to run and returns. If we did call `delay()` we'd
sit there waiting and no other tasks would get a chance to run - by returning as
quickly as possible we return control to the scheduler which can then find any
other tasks that are runnable and run them. The scheduler will take care of
calling the `Blinker::run()` again when the required time interval has passed.

Having defined our tasks classes, in the main body of the program we create a
list of the tasks, pass it to a `TaskScheduler` instance and then run the
scheduler, which then takes care of calling the `run()` methods of the tasks at
the right time:

```c++

    // Create the tasks.
    Blinker blinker(13, 25);
    Echoer echoer;

    // Initialise the task list and scheduler.
    Task *tasks[] = { &blinker, &echoer };
    TaskScheduler sched(tasks, NUM_TASKS(tasks));

    // Run the scheduler - never returns.
    sched.run();
```
`

That's all there is to it. With this approach it's possible to provide a
lightweight set of communicating tasks that are scheduled in priority order. The
code is both high performance and lightweight, two vital attributes considering
the constrained environment it must operate in. Providing the various `run()`
methods are reasonably short, it will run tasks within less than one millisecond
of when they are scheduled, which is perfectly adequate for our needs - I'll
give an example of using this library to perform a timing-critical process
(switch de-bouncing) in a later post.

It's simple enough to implement your own variant, but if you want the code it's
available [here][code link]. The archive also includes an example sketch that
blinks the pin13 LED whilst simultaneously reading the serial port and echoing
characters back.

Acknowledgement
---------------

I'd like to acknowledge my MSc tutor [Dr. Colin Machin] who sat down with me one
afternoon back in 1984 and outlined this approach to me, which I then used for
the Z80 robot controller that was the subject of my MSc thesis. He'd used used
the same technique for a LIDAR data acquisition system he'd written to collect
data on the wingtip vortices caused by commercial aircraft as they land. Good
ideas always stand the test of time - thanks Colin :-) 

----

[Alan's Site]: http://bleaklow.com/2010/07/20/a_very_simple_arduino_task_manager.html
[RTOS]: http://en.wikipedia.org/wiki/Real-time_operating_system 
[Arduino Pro Mini]: http://www.arduino.cc/en/Main/ArduinoBoardProMini 
[Blink sketch]: http://arduino.cc/en/Tutorial/Blink 
[code link]: http://bleaklow.com/files/2010/Task.tar.gz 
[Dr. Colin Machin]: http://www.lboro.ac.uk/departments/compsci/staff/visiting/dr-colin-machin.html 
