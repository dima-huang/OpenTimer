# OpenTimer <img align="right" width="10%" src="image/logo.png">

[![Build Status](https://travis-ci.org/OpenTimer/OpenTimer.svg?branch=master)](https://travis-ci.org/OpenTimer/OpenTimer)
[![Standard](image/cpp17.svg)](https://en.wikipedia.org/wiki/C%2B%2B#Standardization)
[![Download](image/download.svg)](https://github.com/OpenTimer/OpenTimer/archive/master.zip)
[![Version](image/version_badge.svg)](https://github.com/OpenTimer/OpenTimer/tree/master)
[![License: MIT](./image/license_badge.svg)](./LICENSE)

A High-Performance Timing Analysis Tool for VLSI Systems

# Why OpenTimer?

<img align="right" width="30%" src="image/critical_path.jpg">

OpenTimer is a new static timing analysis (STA) tool to help IC designers
quickly verify the circuit timing.
It is developed completely from the ground up using *C++17*
to efficiently support parallel and incremental timing. 
Key features are:

+ Industry standard format (.lib, .v, .spef, .sdc) support.
+ Graph- and path-based timing analysis.
+ Parallel incremental timing for fast timing closure.
+ Award-winning tools and golden timers in ACM TAU timing analysis contests.


# Get Started with OpenTimer

The easiest way to start using OpenTimer is to use *OpenTimer shell*.
OpenTimer shell is a powerful tool for interactive analysis
and the simplest way to learn core functionalities.
[Compile OpenTimer](#compile-opentimer) and launch the shell program `ot-shell`
under the `bin` directory.

```bash
~$ ./bin/ot-shell
  ____              _______              
 / __ \___  ___ ___/_  __(_)_ _  ___ ____
/ /_/ / _ \/ -_) _ \/ / / /  ' \/ -_) __/
\____/ .__/\__/_//_/_/ /_/_/_/_/\__/_/       v2.0.0 (alpha)
    /_/                                     
MIT License: type "license" to see more details.
To see the contributor list, type "contributors".
For help, type "help".
For bug reports, issues, and manual, please see:
<https://github.com/OpenTimer/OpenTimer>.
ot> 
```

We have provided a [simple design](./example/simple)
which consists of five gates (one NAND, one NOR, two INV, one FF) and one clock. 
Move to the folder and read this simple design.

<img align="right" src="image/simple_schematic.jpg" width="55%">

```bash
ot> cd example/simple
ot> read_celllib osu018_stdcells.lib
ot> read_verilog simple.v   
ot> read_sdc simple.sdc
```

Report the timing to show the most critical path.

```bash
ot> report_timing                     # report the most critical path
Endpoint     :   f1:D
Startpoint   :   inp1
Path type    :   early
Required Time:   26.5175
Arrival Time :   2.97909
Slack        :   -23.5384
----------------------------------
   Arrival       Delay   Dir   Pin
----------------------------------
         0         n/a  fall  inp1
         0           0  fall  u1:A
   2.79669     2.79669  rise  u1:Y
   2.79669           0  rise  u4:A
   2.97909    0.182399  fall  u4:Y
   2.97909           0  fall  f1:D
----------------------------------
```

The critical path originates from the primary input `inp1` all the way 
to the data pin `f1:D` of the flip-flop `f1`.
<!--You can dump a dot format and use online tools like [GraphViz][GraphViz]
to visualize the timing graph.
The critical path is marked in red.

```bash
ot> dump_graph -o simple.dot  # dump the timing graph to the .dot format
```

![](image/simple_graph.png)

We have provided three command files, 
[simple.conf](./example/simple/simple.conf), 
[unit.conf](./example/simple/unit.conf), and 
[opt.conf](./example/simple/opt.conf)
to demonstrate more usages of OpenTimer. 
You can redirect it through stdin to OpenTimer shell.

```bash
cd example/simple
../../bin/ot-shell < opt.conf     # modify the design and perform incremental timing
```

The graph below is generated after applying gate changing operations in 
[opt.conf](./example/simple/opt.conf).
One gate (marked in cyan) is inserted to the timing graph.
By default, OpenTimer performs parallel incremental timing to maintain slack integrity.

![](image/simple_opt_graph.png) -->

# Compile OpenTimer

## System Requirements

OpenTimer is very self-contained and has very few dependencies.
To compile OpenTimer , you need:

+ A GNU [C++ Compiler G++ v7.2](https://gcc.gnu.org/gcc-7/) (or higher) with C++17 support
+ Tcl interpreter [tclsh](https://www.tcl.tk/about/language.html) 
(most Unix/Linux/OSX distributions already include tclsh)

OpenTimer has been tested to run well on Linux distributions and MAC OSX.

## Build through CMake

We use [CMake](https://cmake.org/) to manage the source and tests.
We recommend using out-of-source build.

```bash
~$ git clone https://github.com/OpenTimer/OpenTimer.git
~$ cd OpenTimer
~$ mkdir build
~$ cd build
~$ cmake ../ -DCMAKE_CXX_COMPILER=g++
~$ make 
```

After successful build, you can find binaries and libraries in the folders `bin` 
and `lib`, respectively.

## Run Tests
OpenTimer uses [Doctest](https://github.com/onqtam/doctest) for unit tests and
[TAU15][TAU15] benchmarks for integration/regression tests. These benchmarks are generated by an 
industry standard timer and are being used by many EDA researchers.

```bash
~$ make test
```

# Design Philosophy

OpenTimer has very efficient data structures and procedures to
enable parallel and incremental timing.
To make the most use of multi-threading,
each timing operation is divided into three categories,
*Builder*, *Action*, and *Accessor*.

| Type |  Description | Example | Time Complexity |
| -------  |  ----------- |  --- | -- |
| Builder  | create lazy tasks to build an analysis framework | read_celllib, insert_gate, set_slew | O(1) |
| Action   | carry out builder operations to update the timing | update_timing, report_timing, report_slack | Algorithm-dependent |
| Accessors| inspect the timer without changing any internal data structures | dump_timer, dump_slack, dump_net_load | Operation-dependent |

## Builder: OpenTimer Lineage

OpenTimer maintains a lineage graph of *builder* operations 
to create a *task execution plan* (TEP).
A TEP starts with no dependency and keeps adding tasks to the lineage graph
every time you call a builder operation.
It records what transformations need to be executed 
after an action has been called.


![](image/lineage.png)

The above figure shows an example
lineage graph of a sequence of builder operations.
The cyan path is the main lineage line 
with additional tasks attached to enable parallel execution.
OpenTimer use [Cpp-Taskflow][Cpp-Taskflow] to create dependency graphs.

## Action: Update Timing

A TEP is materialized and executed when the timer is requested to perform 
an *action* operation.
Each action operation triggers timing update
from the earliest task to the one that produces the result
of the action call.
Internally, OpenTimer creates task dependency graph to update timing in parallel,
including forward (slew, arrival time) 
and backward (required arrival time) propagations.

![](image/update_timing_simple.png)

The figure above shows the dependency graph (forward in white, backward in cyan) 
to update the timing of the 
[simple](example/simple/simple.v) design.
When an action call finishes, it cleans out the lineage graph
with all timing up-to-date.

## Accessor: Inspect OpenTimer

The *accessor* operations let you inspect the timer status and dump static information
that can be helpful for debugging and turnaround.
All accessor operations are declared as *const methods* in the timer class. 
Calling them promises not to alter any internal members.

```bash
ot> dump_slack  # dump all slack values of the present state
Slack [pins:17|time:1ns]
-----------------------------------------------------------
       E/R         E/F         L/R         L/F          Pin
-----------------------------------------------------------
       -22       -23.3        67.3          66         u1:A
       n/a       0.328         n/a       -23.3  tau2015_clk
       ...         ...         ...         ...          ...
      25.8        26.5       -15.8       -16.5          out
      26.5        25.8       -16.5       -15.8         u3:A
     -22.5       -23.3          44        49.4         u4:Y
-----------------------------------------------------------
```

# OpenTimer Shell

OpenTimer shell is a powerful command line tool to perform interactive analysis.
It is also the easiest way to get your first timing report off the ground.
The program `ot-shell` can be found in the folder `bin/` after you 
[Compile OpenTimer](#compile-opentimer).

## Commands

The table below shows a list of commonly used commands.

| Command | type | Arguments | Description | Example |
| ------- | ---- | --------- | ----------- | ------- |
| set_num_threads | builder | N | set the number of threads | set_num_threads 4 |
| read_celllib | builder | [-early \| -late] file | read the cell library for early and late splits | read_celllib -early mylib_Early.lib |
| read_verilog | builder | file | read the verilog netlist | read_verilog mynet.v |
| read_spef | builder | file | read parasitics in SPEF | read_spef myrc.spef |
| read_sdc | builder | file | read a Synopsys Design Constraint file | read_sdc myrule.sdc |
| update_timing | action | n/a | update the timing | update_timing |
| report_timing | action | n/a | report the timing | report_timing |
| report_tns | action | n/a | report the total negative slack | report_tns |
| report_wns | action | n/a | report the worst negative slack | report_wns |
| dump_graph | accessor | [-o file] | dump the present timing graph to a dot format | dump_graph -o graph.dot |
| dump_timer | accessor | [-o file] | dump the present timer details | dump_timer -o timer.txt |
| dump_slack | accessor | [-o file] | dump the present slack values of all pins | dump_slack -o slack.txt |

To see the full command list, visit [OpenTimer Wiki][OpenTimer Wiki].

# Integrate OpenTimer to your Project

There are a number of ways to develop your project on top of OpenTimer.

## Install OpenTimer through CMake

Our project [CMakeLists.txt](CMakeLists.txt) has defined the required files
to install when you hit `make install`.
The installation paths are `<prefix>/include`, `<prefix>/lib`, and `<prefix>/bin`
for the hpp files, library, and executable where `<prefix>` can be configured 
through the cmake variable [CMAKE_INSTALL_PREFIX](https://cmake.org/cmake/help/latest/variable/CMAKE_INSTALL_PREFIX.html#variable:CMAKE_INSTALL_PREFIX).
The following example installs OpenTimer to `/tmp`.

```bash
~$ cd build/
~$ cmake ../ -DCMAKE_INSTALL_PREFIX=/tmp
~$ make 
~$ make install
~$ cd /tmp    # install OpenTimer to /tmp
~$ ls
bin/  include/  lib/
```

Now create a `app.cpp` file under `/tmp` that does nothing but dumps the timer details.

```cpp
#include <ot/timer/timer.hpp>
int main(int argc, char* argv[]) {
  ot::Timer timer;
  std::cout << timer.dump_timer();
  return 0;
}
```

Compile your application together with the OpenTimer headers and library.
You will need to specify the `-std=c++1z` and `-lstdc++fs` flags
to use C++17 features and filesystem libraries.

```bash
~$ g++ app.cpp -std=c++1z -lstdc++fs -O2 -I include -L lib -lOpenTimer -o app.out
~$ ./app.out
```

# OpenTimer C++ API

OpenTimer is written in modern C++17.
The class [Timer](ot/timer/timer.hpp) is the main entry you need to 
use OpenTimer in your project. 
The table below summarizes a list of commonly used methods.

| Method | Type | Argument | Return | Description |
| ------ | ---- | -------- | ------ | ----------- |
| num_threads | builder | unsigned | self | set the number of threads |
| celllib| builder | path, split | self | read the cell library for early and late splits |
| verilog| builder | path | self | read a verilog netlist |
| spef   | builder | path | self | read parasitics in SPEF |
| sdc    | builder | path | self | read a Synopsys Design Constraint file |
| update_timing | action | n/a | void | update the timing; all timing values are up-to-date upon return |
| at | action | pin_name, split, transition | optional of float | update the timing and return the arrival time of a pin, if exists, at a give split and transition |
| slew | action | pin_name, split, transition | optional of float | update the timing and return the slew of a pin at a give split and transition, if exists |
| rat | action | pin_name, split, transition | optional of float | update the timing and return the required arrival time of a pin at a give split and transition, if exists |
| slack | action | pin_name, split, transition | optional of float | update the timing and return the slack of a pin at a give split and transition, if exists |
| tns | action | n/a | optional of float | update the timing and return the total negative slack if exists |
| wns | action | n/a | optional of float | update the timing and return the worst negative slack if exists |
| dump_graph | accessor | n/a | string | dump the present timing graph to a dot format |
| dump_timer | accessor | n/a | string | dump the present timer details |
| dump_slack | accessor | n/a | string | dump the present slack values of all pins |


*All public methods are thread-safe* as a result of OpenTimer lineage.
The example below shows an OpenTimer application and the use of builder, action, and accessor API.

```cpp
#include <ot/timer/timer.hpp>

int main(int argc, char *argv[]) {
  
  ot::Timer timer;
  
  timer.celllib("simple.lib", std::nullopt)  // read the library (builder)
       .verilog("simple.v")                  // read the verilog netlist (builder)
       .spef("simple.spef")                  // read the parasitics (builder)
       .sdc("simple.sdc")                    // read the design constraints (builder)
       .update_timing();                     // update timing (builder)

  if(auto tns = timer.tns(); tns) std::cout << "TNS: " << *tns << '\n';  // (action)
  if(auto wns = timer.wns(); wns) std::cout << "WNS: " << *wns << '\n';  // (action)

  std::cout << timer.dump_timer();  // dump the timer details (dump)
  
  return 0;
}
```

To see the full API list, visit [OpenTimer Wiki][OpenTimer Wiki].

# Examples

The folder [example](./example) contains several examples and is a great place to learn how to use OpenTimer.

| Example |  Description | How to Run? |
| ------- |  ----------- | ---- |
| [simple](./example/simple) | A simple sequential circuit design with one FF, four gates, and a clock. | ot-shell < simple.conf |
| [c17](./example/c17) | A combinational circuit design with six NAND gates, no clock.| ot-shell < c17.conf |
| [s27](./example/s27) | A C++ application using OpenTimer API to analyze the timing of a sequential circuit.| ./s27 |

The folder [benchmark](./benchmark) contains more designs but they are mainly used for internal regression 
and integration tests.



# Who is Using OpenTimer?

OpenTimer is an award-winning tools. It won ACM TAU Timing Analysis Contests multiple times
([1st place][TAU14] in 2014, [2nd place][TAU15] in 2015, and [1st place][TAU16] in 2016)
and the [special price][LibreCores] from 2016 LibreCores Design Contest.
Many industry and academic people are using OpenTimer in their projects:

- [Golden Timer][TAU19], 2019 ACM TAU Timing Analysis Contest on Timing-driven Optimization
- [Golden Timer][TAU17], 2017 ACM TAU Timing Analysis Contest on Micro Modeling
- [Golden Timer][TAU16], 2016 ACM TAU Timing Analysis Contest on Micro Modeling
- [Golden Timer][ICCAD15], 2015 ACM/IEEE ICCAD Incremental Timing-driven Placement Contest
- [VSD][VSD], VLSI System Design Corporation
- [OpenDesign Flow Database][OpenDesign], the infrastructure for VLSI design and design automation research

| [<img src="image/tau16.png" width="100px">](https://sites.google.com/site/taucontest2016/) | [<img src="image/tau17.png" width="100px">](https://sites.google.com/site/taucontest2017/) | [<img src="image/CAD-contest.png" width="100px">](http://cad-contest.el.cycu.edu.tw/problem_C/default.html) | [<img src="image/vsd.jpg" width="100px">](https://www.vlsisystemdesign.com/) |
| :---: | :---: | :---: | :---: |


Please don't hesitate to [let me know][email me] if I forgot your project! 


# Get Involved
+ Report bugs/issues by submitting a [Github issue][Github issues].
+ Submit contributions using [pull requests][Github pull requests].
+ See development status by visiting [OpenTimer Wiki][OpenTimer Wiki].
+ Read and cite our ACM/IEEE [ICCAD paper][OpenTimerPaper].


# Contributors & Acknowledgment
OpenTimer is being actively developed and contributed by the following people:
- [Tsung-Wei Huang][Tsung-Wei Huang] created the OpenTimer project is now the chief architect.
- [Martin Wong][Martin Wong] supported the OpenTimer project through NSF and DARPA funding.
- [Chun-Xun Lin][Chun-Xun Lin] implemented the prompt interface of OpenTimer shell.
- [Kunal Ghosh][Kunal Ghosh] provided a list of practical features to include in OpenTimer.
- [Pei-Yu Lee][Pei-Yu Lee] provided useful incremental timing discussion and helped fix bugs.
- [Tin-Yin Lai][Tin-Yin Lai] discussed micro modeling algorithms and helped fix bugs.
- [Jin Hu][Jin Hu] helped define the timing API and produced the golden benchmarks for integration tests.
- [Myung-Chul Kim][Myung-Chul Kim] helped test OpenTimer through ICCAD CAD contest.
- [George Chen][George Chen] helped defined path report formats and test OpenTimer through TAU contest.
- [Pao-I Chen][Pao-I Chen] helped design the logo of OpenTimer.
- [Leslie Hwang][Leslie Hwang] reviewed the documentation and README.

<!--

| [<img src="image/twhuang.jpg" width="100px">][Tsung-Wei Huang] | [<img src="image/mdfwong.jpg" width="100px">][Martin Wong] | [<img src="image/cxlin.jpg" width="100px">][Chun-Xun Lin] | [<img src="image/kunal_ghosh.jpg" width="100px">][Kunal Ghosh] | [<img src="image/pei_yu_lee.jpg" width="100px">][Pei-Yu Lee] | [<img src="image/jin_hu.jpg" width="100px">][Jin Hu] | [<img src="image/myung_chul_kim.jpg" width="100px">][Myung-Chul Kim] |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| [<img src="image/george_chen.jpg" width="100px">][George Chen]| [<img src="image/poyi.jpg" width="100px">][Pao-I Chen] |
-->

Please don't hesitate to [let me know][email me] if I forgot someone! 
Meanwhile, we appreciate the funding support from our sponsors to continue our development of OpenTimer.

| [<img src="image/nsf.png" width="100px">](https://www.nsf.gov/) | [<img src="image/darpa.png" width="100px">](https://www.darpa.mil/work-with-us/electronics-resurgence-initiative) |
| :---: | :---: |



# License

<img align="right" src="http://opensource.org/trademarks/opensource/OSI-Approved-License-100x137.png" height="80px">
<img align="right" src="image/uiuc.png" height="80px">

OpenTimer is licensed under the [MIT License](./LICENSE):

Copyright &copy; 2018 [Dr. Tsung-Wei Huang][Tsung-Wei Huang] and [Dr. Martin Wong][Martin Wong]

The University of Illinois at Urbana-Champaign, IL, USA

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the “Software”), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.


* * *

[Tsung-Wei Huang]:       http://web.engr.illinois.edu/~thuang19/
[Chun-Xun Lin]:          https://github.com/clin99
[Martin Wong]:           https://ece.illinois.edu/directory/profile/mdfwong
[Kunal Ghosh]:           mailto:kunalpghosh@gmail.com
[Jin Hu]:                mailto:jinhu@umich.edu
[Myung-Chul Kim]:        mailto:mckima@us.ibm.com
[Tin-Yin Lai]:           mailto:tinyinlai@gmail.com
[Pei-Yu Lee]:            mailto:palacedeforsaken@gmail.com
[George Chen]:           mailto:george@geochrist.com
[Jignesh Shah]:          mailto:jignesh.shah@intel.com
[Pao-I Chen]:            mailto:poyipenny@gmail.com
[Leslie Hwang]:          mailto:tkdlezz@gmail.com 
[Github releases]:       https://github.com/OpenTimer/OpenTimer/releases
[Github issues]:         https://github.com/OpenTimer/OpenTimer/issues
[Github pull requests]:  https://github.com/OpenTimer/OpenTimer/pulls
[GraphViz]:              https://dreampuf.github.io/GraphvizOnline/
[OpenTimer Wiki]:        https://github.com/OpenTimer/OpenTimer/wiki
[OpenTimer-1.0]:         https://web.engr.illinois.edu/~thuang19/software/timer/OpenTimer.html
[OpenTimerCitation]:     https://scholar.google.com/scholar?oi=bibs&hl=en&cites=142282068238605079
[OpenTimerPaper]:        https://github.com/OpenTimer/OpenTimer/blob/master/doc/iccad15.pdf
[UI-TimerPaper]:         https://github.com/OpenTimer/OpenTimer/blob/master/doc/iccad14.pdf
[email me]:              mailto:twh760812@gmail.com
[Cpp-Taskflow]:          https://github.com/cpp-taskflow/cpp-taskflow
[VSD]:                   https://www.vlsisystemdesign.com/
[ICCAD15]:               http://cad-contest.el.cycu.edu.tw/problem_C/default.html
[TAU14]:                 https://sites.google.com/site/taucontest2014/
[TAU15]:                 https://sites.google.com/site/taucontest2015/
[TAU16]:                 https://sites.google.com/site/taucontest2016/
[TAU17]:                 https://sites.google.com/site/taucontest2017/
[TAU19]:                 https://sites.google.com/view/tau-contest-2019/home
[LibreCores]:            https://fossi-foundation.org/2016/10/13/designcontest
[OpenDesign]:            https://github.com/jinwookjungs/open_design_flow
[DARPA IDEA]:            https://www.darpa.mil/news-events/2017-09-13
  

