---
title: Warmup mode done -- here's what's left
---

I've finished all the [estimators](https://github.com/matildah/mirage-ntp/blob/82368fdeb673e6f0b063e1f1262aa21a127ab9d7/lib/RADclock/estimators.ml#L78) for the warmup phase of the RADclock algorithm. What's left is:

* All the normal-mode estimators (the functions called in [this](https://github.com/synclab/radclock/blob/master/radclock/sync_bidir.c#L2093-L2105) block)
* Top-level window control / data expiration
* Leap second logic
* Writing the actual absolute / difference clock functions -- that take the current value of the TSC counter and multiply/add the current values of the appropriate estimators to obtain actual absolute/difference clocks


The normal-mode estimators are similar to the warmup mode estimators (except with some added complexity; there's level shift detection, for example, to robustly detect if the RTT to the server has increased). I have written [higher-order functions](https://github.com/matildah/mirage-ntp/blob/82368fdeb673e6f0b063e1f1262aa21a127ab9d7/lib/RADclock/history.ml#L111-L158) and [per-packet functions](https://github.com/matildah/mirage-ntp/blob/82368fdeb673e6f0b063e1f1262aa21a127ab9d7/lib/RADclock/estimators.ml#L18-L74) that let me [concisely write](https://github.com/matildah/mirage-ntp/blob/82368fdeb673e6f0b063e1f1262aa21a127ab9d7/lib/RADclock/estimators.ml#L114-L133) the estimators in an order of magnitude fewer lines as the [original C version](https://github.com/synclab/radclock/blob/master/radclock/sync_bidir.c#L1304-L1525) -- so writing the normal-mode estimators is not likely to pose *much* of a challenge.

I've also written a [list type](https://github.com/matildah/mirage-ntp/blob/82368fdeb673e6f0b063e1f1262aa21a127ab9d7/lib/RADclock/history.ml#L1-L30)[^1] to store historical data points that is equivalent to [RADclock's sync_hist](https://github.com/synclab/radclock/blob/master/radclock/sync_history.h#L34-L44) ring buffer along with a [tiny DSL](https://github.com/matildah/mirage-ntp/blob/82368fdeb673e6f0b063e1f1262aa21a127ab9d7/lib/RADclock/history.ml#L31-L60) to describe points (and therefore [intervals](https://github.com/matildah/mirage-ntp/blob/82368fdeb673e6f0b063e1f1262aa21a127ab9d7/lib/RADclock/history.ml#L160-L169)) in a history list. There is a terrifying quantity of arithmetic on indices and pointers in [sync_bidir.c](https://github.com/synclab/radclock/blob/master/radclock/sync_bidir.c) and my types allow me to avoid it entirely -- instead I can write [incredibly simple expressions](https://github.com/matildah/mirage-ntp/blob/master/lib/RADclock/estimators.ml#L142-L148) to specify which ranges of points are passed to my estimators. Furthermore, the functions that generate those ranges are independent from any calculations -- this lets me verify that the right points are worked on *and that all the needed data points are available* without having to run any of the actual estimating code. I need to write code to manage the top-level windows that determines which samples are kept and which are expired but that isn't difficult -- all that needs to be done is figure out closed-form expressions for window ranges from the RADclock imperative C code that manages the relevant indices.




![[Kitsu Chiri](https://en.wikipedia.org/wiki/Sayonara,_Zetsubou-Sensei) was right.](../images/leap.png)



NTP, unfortunately, uses the UTC timescale and not $(TAI, leap\,offset)$[^2] to transmit time, and to create a reliable NTP client implementation, code needs to be written to robustly deal with the horrid discontinuities and repeated time values that are caused by a leap second event -- and to not interpret those values and the subsequent 1-second offset as errors or invalid data or a change that must be corrected by the timing algorithms. RADclock has code that attempts to do this which I haven't yet analyzed.

Semi-relatedly, [Darryl Veitch](http://crin.eng.uts.edu.au/~darryl/), one of the creators of RADclock -- the superb NTP implementation that I am basing my work on, did a first-of-its-kind [analysis](http://crin.eng.uts.edu.au/~darryl/Publications/LeapSecond_camera.pdf) of public NTP Stratum 1 server replies **during a leap second event** (compared against a RADclock instance and independently measured with a GPS-synchronized precision network capture card) and demonstrated widespread[^3] erroneous/noncompliant behaviour related to the leap second event.



Finally, I need to figure out how to properly select and access a counter register under Xen -- RDTSC is [not necessarily the right choice](http://xenbits.xen.org/docs/4.3-testing/misc/tscmode.txt) and if it isn't there alternatives like HPET and the ACPI PM timer. I also need to figure out how to get a counter register on ARM systems. Right now for testing I just have an extremely simple [stub](https://github.com/matildah/mirage-ntp/blob/master/lib/tsc/rdtsc_caml.c#L11) that calls RDTSC but this is not appropriate for production use.


Also (and less importantly) I need to figure out a name for this NTP implementation -- if anyone has suggestions feel free to email me / [comment on github](https://github.com/matildah/mirage-ntp/issues/6) / tweet at me.

[^1]: I've only informally tested my list type -- but once I get around to reading [how to test monadic code with quickcheck](http://dl.acm.org/citation.cfm?id=636527) I will write an algebraic specification of my data type and quickcheck code to test my type against it.


[^2]: [GPS](http://www.navipedia.net/index.php/Time_References_in_GNSS#GPS_Time_.28GPST.29), [Galileo](http://www.navipedia.net/index.php/Time_References_in_GNSS#Galileo_System_Time_.28GST.29), [BeiDou](http://www.navipedia.net/index.php/Time_References_in_GNSS#BeiDou_Time_.28BDT.29), and PTP all use $(TAI, leap\,offset)$ as a time format -- the designers of the NTP protocol messed up and should have done so as well. [PTP](https://en.wikipedia.org/wiki/Precision_Time_Protocol) / IEEE 1588 is a protocol used to transmit time from a master source (such as a GPS receiver or an atomic clock) over Ethernet wiring and with specialized timestamping network interfaces that achieves much higher performance than NTP. It is used in industrial applications such as power grid synchronization / measurement, control/data acquisition timing in large scientific projects [(such as the LHC)](https://en.wikipedia.org/wiki/The_White_Rabbit_Project) or in test aircraft. Discontinuities in time and repeated time values, as are caused by leap second events in UTC, are *never acceptable* in such applications -- so PTP uses TAI as its timescale. To let users synthesize UTC time if needed, along with the current TAI time (which is pure and clean and discontinuity-free) PTP master clocks transmit a special field called "`currentUtcOffset`" -- the number of leap seconds so far.


[^3]: Only $61\%$ of the measured servers did not return erroneous or noncompliant data during the leap second event. This is an unacceptable state of affairs and blame should be placed entirely on the NTP standard and not the implementers. djb has the right approach here -- if a standard leaves many ways to mess up an implementation (and those traps can be removed), **[the standard](http://cr.yp.to/talks/2015.06.11/slides-djb-20150611-a4.pdf) is [broken](https://events.ccc.de/congress/2014/Fahrplan/system/attachments/2502/original/20141227-twopage.pdf)**. Modern cryptographic engineering shows that the road to trustworthy and secure systems is paved with simple standards that as are free from [deadly edge cases](https://cr.yp.to/talks/2012.08.08/slides.pdf) as possible. Every leap second event reminds us that the NTP standard is in flagrant opposition to this principle.
