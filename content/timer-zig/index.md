
+++
title = "Learning Linux signal interrupts from building a timer"
date = 2026-01-28

[taxonomies]
categories = ["systems-programming"]
tags = ["ziglang", "timer"]
+++
A simple timer application built in Zig that uses Linux signal interrupts to show remaining time in terminal and notify once timer ends.

<!-- more -->

## Backstory:
I use Timers to take breaks in between from long reading and coding activities. Typically for every 45min of focused activity I take a 10 to 15 min break and use timer app in my mobile to alert me for breaks. There's one problem that I had to everytime switch my gaze to mobile and move my hands from keyboard to mobile to check remaining time before timer ends.

Then I decided to build an alarm application in zig that will simply run in my terminal and notify with a pop up and alarm bell that timer has ended.

## Requirements and Design:

![Timer Application Screenshot](assets/Screenshot_20260128_101739.png)

My requirements were the application should show remaining time in HOURS:MINUTES:SECONDS format and pop up a notification and ring the alarm bell once timer ends.

I always wanted to play with signal interrupts in my application and planned to use them in this timer application for learning and fun.

There's a main thread that accepts a value to end timer in seconds and sleeps for those many seconds before notifying that timer has ended. To show the remaining time in terminal, that keeps updating for every 1/2 second, main thread takes help of a child thread that notifies it through signal interrupt for every 1/2 second to print remaining time to terminal.


## Implementation:
### Accepting Timer value:

Application will accept timer value in seconds and parses it to an `i64` numeric datatype. Later, the parsed value is converted to `timespec` struct that will be passed as an argument to `nanosleep`, an interruptible sleep function.

```zig
const in_secs = args.next().?;
const timer_secs = try std.fmt.parseInt(i64, in_secs, 10);
var timer_spec: std.os.linux.timespec = .{ .sec = timer_secs, .nsec = 0 };
```

### Main Thread Interrupter:
Main thread cannot display remaining time if it sleeps indefinitely until timer ends. Therefore a child thread will interrupt for every half second for main thread to print remaining time and resume the timer. `pthread_kill` is used to send the interrupt signal `SIGUSR1`.

```zig
// Called in main thread
_ = try std.Thread.spawn(.{}, remaining_time, .{ init, current_thread });

// Runs in child thread
const progress_check: std.os.linux.timespec = .{ .sec = 0, .nsec = @divExact(std.time.ns_per_s, 2) };
fn remaining_time(_: std.process.Init, parent_thread: c.pthread_t) !void {
    while (true) {
        if (std.os.linux.nanosleep(&progress_check, null) != 0) {
            std.os.linux.exit(1);
        }
        _ = c.pthread_kill(parent_thread, c.SIGUSR1);
    }
}
```
### Main Thread resume:
After child thread interrupts `nanosleep` in main thread returns non-zero intimating main thread to log remaining time. Once the remaining time is logged. Main thread is again resumed by calling `nanosleep` with remaining time. Once `nanosleep` exits without any interrupt i.e, by returning zero timer application ends with alarm bell and notification.

```zig
while (true) {
    if (std.os.linux.nanosleep(&timer_spec, rem_timer_spec) != 0) {
        timer_spec = rem_timer_spec.*;
        const rem_time = TimeDelta.from_timespec(timer_spec);
        _ = try writer.write("\x1b\x5b\x32\x4b\r");
        _ = try writer.print("Remaining: {d}Hours, {d}Minutes, {d}Seconds", .{ rem_time.hours, rem_time.minutes, rem_time.seconds });
        try writer.flush();
    } else {
        _ = try writer.print("\nBeep{s}\n", .{&[_]u8{7}});
        try writer.flush();
        break;
    }
}
```





