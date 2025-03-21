## Debugging and Profiling

[MIT 6.NULL - Debugging and Profiling](https://missing.csail.mit.edu/2020/debugging-profiling/)

### Debugging

#### Printf Debugging and Logging
脑力劳动debug，借助打印信息思考和推断问题所在

信息除了Printf，还可以Logging，更灵活（可以输出到文件、sockets、remote servers），可复用

[Here](https://missing.csail.mit.edu/static/files/logger.py) is an example code that logs messages:

```shell
$ python logger.py
# Raw output as with just prints
python logger.py log
# Log formatted output
$ python logger.py log ERROR
# Print only ERROR levels and above
$ python logger.py color
# Color formatted output
```
利用颜色信息： [ANSI escape codes](https://en.wikipedia.org/wiki/ANSI_escape_code)

```shell
#!/usr/bin/env bash
for R in $(seq 0 20 255); do
    for G in $(seq 0 20 255); do
        for B in $(seq 0 20 255); do
            printf "\e[38;2; <img src="https://www.zhihu.com/equation?tex=%7BR%7D%3B" alt="{R};" class="ee_img tr_noresize" eeimg="1"> {G};${B}m█\e[0m";
        done
    done
done
```

#### Third party logs

* UNIX系统中第三方库的log常存在`/var/log`
  * the [NGINX](https://www.nginx.com/) webserver places its logs under `/var/log/nginx`
* `systemd`, a system daemon that controls many things in your system such as which services are enabled and running 
  * `/var/log/journal`，可用`journalctl`显示
  * macOS上`var/log/system.log`，但更多人用system log，用 [`log show`](https://www.manpagez.com/man/1/log/) 显示
  * 参考[data wrangling]()，需要对log信息处理，也可以用[lvav](http://lnav.org/)，体验更好

```shell
logger "Hello Logs"
# On macOS
log show --last 1m | grep Hello
# On Linux
journalctl --since "1m ago" | grep Hello
```

#### Debuggers

**C++**: [`gdb`](https://www.gnu.org/software/gdb/) (and its quality of life modification [`pwndbg`](https://github.com/pwndbg/pwndbg)) and [`lldb`](https://lldb.llvm.org/)

gdb：c(continue), l(ist), s(tep), n(ext), b(reak), p(rint), r(eturn), run, q(uit), watch

* 特有：start, finish, cond, disable, where
* `cond 3 this==0xXXX`
* `gdb --args sleep 20` debug带参数的binary
* bt(backtrace), frame X 进帧
* `watch -l ` 同时监视表达式本身和表达式指向的内容
* `attach $pid` debug正在运行的进程

* `ptype` 打印变量类型；打印stl使用 [python pretty print](https://lumiera.org/documentation/technical/howto/DebugGdbPretty.html)

```c++
//增加print的可读性
set print pretty on/off

//显示智能指针对象指向的变量
p ((Object*) my_ptr)->attribute //利用类型转换
p *(my_ptr._M_ptr)
  
//显示vector内部值
p *(my_vec._M_impl._M_start)@my_vec.size()  //打印大小
p (my_vec._M_impl._M_start+0).attribute
p *(my_vec._M_impl._M_start)@N //打印第N个成员
  
//pb相关
p *(std::string*)(X.rep_.elements) //repeated string, 字段X

```

[gdb的多线程调试](https://blog.csdn.net/lf_2016/article/details/59741705)

* info threads:显示当前可调试的所有线程,GDB会给每一个线程都分配一个ID。前面有*的线程是当前正在调试的线程。

* thread ID:切换当前调试的线程为指定ID的线程。

* thread apply all command:让所有被调试的线程都执行command命令。

* thread apply ID1 ID2 … command:让线程编号是ID1，ID2…等等的线程都执行command命令。

* set scheduler-locking on|off|step:在使用step或continue命令调试当前被调试线程的时候，其他线程也是同时执行的，如果我们只想要被调试的线程执行，而其他线程停止等待，那就要锁定要调试的线程，只让他运行。
  * off:不锁定任何线程，所有线程都执行。
  * on:只有当前被调试的线程会执行。
  * step:阻止其他线程在当前线程单步调试的时候抢占当前线程。只有当next、continue、util以及finish的时候，其他线程才会获得重新运行的

* show scheduler-locking：查看当前锁定线程的模式。



**Python**: [`ipdb`](https://pypi.org/project/ipdb/) is an improved `pdb` that uses the [`IPython`](https://ipython.org) REPL enabling tab completion, syntax highlighting, better tracebacks,  and better introspection while retaining the same interface as the `pdb` module.

* ipdb命令: `p locals()`, j(ump), pp([`pprint`](https://docs.python.org/3/library/pprint.html)), restart
* [pdb turorial](https://github.com/spiside/pdb-tutorial), [pdb depth tutorial](https://realpython.com/python-debugging-pdb)

[CS107 GDB and Debugging教程](https://web.stanford.edu/class/archive/cs/cs107/cs107.1202/resources/gdb)

[CS107 Software Testing Strategies](https://web.stanford.edu/class/archive/cs/cs107/cs107.1202/testing.html)

[lldb的使用](https://www.jianshu.com/p/9a71329d5c4d)

[macOS上配置VSCode的gdb调试环境](https://zhuanlan.zhihu.com/p/106935263?utm_source=wechat_session)  

```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python: Current File",
            "type": "python",
            "request": "launch",
            "program": "${file}",
            "envFile": "${workspaceFolder}/.env",
            "console": "integratedTerminal",
            "env": {
              "ENV1":"123",
              "ENV2":"abc",
            },
            "args": ["-version","4"]
        }
    ]
}
```

#### Specialized Tools

Even if what you are trying to debug is a black box binary there are tools that can help you with that. Whenever programs need to perform actions that only the kernel can, they use [System Calls](https://en.wikipedia.org/wiki/System_call). There are commands that let you trace the syscalls your program makes. In Linux there’s [`strace`](https://www.man7.org/linux/man-pages/man1/strace.1.html) and macOS and BSD have [`dtrace`](http://dtrace.org/blogs/about/). `dtrace` can be tricky to use because it uses its own `D` language, but there is a wrapper called [`dtruss`](https://www.manpagez.com/man/1/dtruss/) that provides an interface more similar to `strace` (more details [here](https://8thlight.com/blog/colin-jones/2015/11/06/dtrace-even-better-than-strace-for-osx.html)).

* [strace入门](https://blogs.oracle.com/linux/strace-the-sysadmins-microscope-v2)

```shell
# On Linux
strace git status 2>&1 >/dev/null | grep index.lock
sudo strace [-e lstat] ls -l > /dev/null

# 多线程 strace，要显示 PPID
ps -efl | grep $task_name # 显示 PPID、PID
strace -p $PID

# 一些 flag
-tt   发生时刻
-T 		持续时间
-s 1024 print输入参数的长度限制
-e write=   -e read=     -e trace=file/desc			-e recvfrom
-f 监控所有子线程   -ff


# On macOS
sudo dtruss -t lstat64_extended ls -l > /dev/null

# 与之配合的技术
readlink /proc/22067/fd/3
lsof | grep /tmp/foobar.lock
```
Under some circumstances, you may need to look at the network packets to figure out the issue in your program. Tools like [`tcpdump`](https://www.man7.org/linux/man-pages/man1/tcpdump.1.html) and [Wireshark](https://www.wireshark.org/) are network packet analyzers that let you read the contents of network packets and filter them based on different criteria.

For web development, the Chrome/Firefox developer tools are quite handy. They feature a large number of tools, including:

- Source code - Inspect the HTML/CSS/JS source code of any website.
- Live HTML, CSS, JS modification - Change the website content,  styles and behavior to test (you can see for yourself that website  screenshots are not valid proofs).
- Javascript shell - Execute commands in the JS REPL.
- Network - Analyze the requests timeline.
- Storage - Look into the Cookies and local application storage.

#### Static Analysis
* [Static Analysis介绍](https://en.wikipedia.org/wiki/Static_program_analysis)
  
  * formal methods 
  * Python: [`pyflakes`](https://pypi.org/project/pyflakes) , [`mypy`](http://mypy-lang.org/), [`shellcheck`](https://www.shellcheck.net/)
  * English也有静态分析！
  * 静态分析可以融入编辑器, vim:[`ale`](https://vimawesome.com/plugin/ale) or [`syntastic`](https://vimawesome.com/plugin/syntastic) 
* [Static Analysis仓库整理](https://github.com/analysis-tools-dev/static-analysis#go)
* [awesome linters整理](https://github.com/caramelomartins/awesome-linters#go)
  * Python:  [`pylint`](https://github.com/PyCQA/pylint) and [`pep8`](https://pypi.org/project/pep8/) 是stylistic linters，[`bandit`](https://pypi.org/project/bandit/) 可查security问题
    * `python -m autopep8 -i -r $FOLDER`
* A complementary tool to stylistic linting are code formatters such as [`black`](https://github.com/psf/black) for Python, `gofmt` for Go, `rustfmt` for Rust or [`prettier`](https://prettier.io/) for JavaScript, HTML and CSS.


### Profiling

profilers和monitoring tools的意义：[premature optimization is the root of all evil](http://wiki.c2.com/?PrematureOptimization)

[时间概念](https://stackoverflow.com/questions/556405/what-do-real-user-and-sys-mean-in-the-output-of-time1)：real/user/system time

* real time(wall time): 真实时间, user: 用户态耗时, sys: 内核态耗时, user+sys: 实际用时
* time指令

Jeff Dean推崇的back-of-the-envelope方法估算系统性能，这是一篇很好的[文章](http://highscalability.com/blog/2011/1/26/google-pro-tip-use-back-of-the-envelope-calculations-to-choo.html)，里面有各种实用数据

* scalable counter
* keep per user comment indexes when paging through comments

#### Profilers

**CPU**: [两种CPU profilers](https://jvns.ca/blog/2017/12/17/how-do-ruby---python-profilers-work-)，tracing and sampling profilers

* Python
  * cProfile: `python -m cProfile -s tottime grep.py 1000 '^(import|\s*def)[^,]*$' *.py`
  *  [`line_profiler`](https://github.com/pyutils/line_profiler) 可逐行输出，用`@prifile`decorator标注函数, `kernprof -l -v a.py`

```Python
b = [2] * (2 * 10 ** 7)
del b

kernprof -l -v sorts.py
python -m line_profiler sorts.py.lprof
```

**Event Profiling** 

[`perf`](https://www.man7.org/linux/man-pages/man1/perf.1.html) 

[perf的介绍与使用](https://www.cnblogs.com/arnoldlu/p/6241297.html)

[perf documentation](https://android.googlesource.com/kernel/msm/+/android-7.1.0_r0.2/tools/perf/Documentation)

- `perf list` - List the events that can be traced with perf
- `perf stat COMMAND ARG1 ARG2` - Gets counts of different events related a process or command
- `perf record COMMAND ARG1 ARG2` - Records the run of a command and saves the statistical data into a file called `perf.data`
- `perf report` - Formats and prints the data collected in `perf.data`

```shell
perf help
sudo perf top
perf top --call-graph graph

sudo perf record stress -c 1 # record->stat
sudo perf report

perf stat -e cache-misses,cache-references,instructions,cycles,faults,branch-instructions,branch-misses,L1-dcache-stores,L1-dcache-store-misses,L1-dcache-loads,L1-dcache-load-misses,LLC-loads,LLC-load-misses,LLC-stores,LLC-store-misses,dTLB-loads,dTLB-load-misses,iTLB-loads,iTLB-load-misses -p $pid

perf record -e cache-misses -p $pid
perf report --sort comm,dso,symbol

sudo perf kmem --alloc --caller --slab stat
sudo perf sched script
```

**内存泄露问题**

* [Valgrind](https://valgrind.org/)
* `gdb`
* [ps_mem的使用](https://linux.cn/article-8639-1.html)

**Visualization**

* [Flame Graph](http://www.brendangregg.com/flamegraphs.html)
*  [`pycallgraph`](http://pycallgraph.slowchop.com/en/master/) 

#### Resource Monitoring

- **General Monitoring** - Probably the most popular is [`htop`](https://hisham.hm/htop/index.php), which is an improved version of [`top`](https://www.man7.org/linux/man-pages/man1/top.1.html). `htop` presents various statistics for the currently running processes on the system. `htop` has a myriad of options and keybinds, some useful ones  are: `<F6>` to sort processes, `t` to show tree hierarchy and `h` to toggle threads.  See also [`glances`](https://nicolargo.github.io/glances/) for similar implementation with a great UI. For getting aggregate measures across all processes, [`dstat`](http://dag.wiee.rs/home-made/dstat/) is another nifty tool that computes real-time resource metrics for lots of different subsystems like I/O, networking, CPU utilization, context  switches, &c.
  - `dstat -nf`，n表示网络，f表示看详细信息
  - `lscpu`
  -  `cat /proc/cpuinfo` 查cpu信息，其中 flags 表示指令集支持
- **I/O operations** - [`iotop`](https://www.man7.org/linux/man-pages/man8/iotop.8.html) displays live I/O usage information and is handy to check if a process is doing heavy I/O disk operations
- **Disk Usage** - [`df`](https://www.man7.org/linux/man-pages/man1/df.1.html) displays metrics per partitions and [`du`](http://man7.org/linux/man-pages/man1/du.1.html) displays disk usage per file for the current directory. In these tools the `-h` flag tells the program to print with human readable format. A more interactive version of `du` is [`ncdu`](https://dev.yorhel.nl/ncdu) which lets you navigate folders and delete files and folders as you navigate.
- **Memory Usage** - [`free`](https://www.man7.org/linux/man-pages/man1/free.1.html) displays the total amount of free and used memory in the system. Memory is also displayed in tools like `htop`.
- **Open Files** - [`lsof`](https://www.man7.org/linux/man-pages/man8/lsof.8.html)  lists file information about files opened by processes. It can be  quite useful for checking which process has opened a specific file.
- **Network Connections and Config** - [`ss`](https://www.man7.org/linux/man-pages/man8/ss.8.html) lets you monitor incoming and outgoing network packets statistics as well as interface statistics. A common use case of `ss` is figuring out what process is using a given port in a machine. For  displaying routing, network devices and interfaces you can use [`ip`](http://man7.org/linux/man-pages/man8/ip.8.html). Note that `netstat` and `ifconfig` have been deprecated in favor of the former tools respectively.
  - ss的[使用技巧](https://www.cnblogs.com/peida/archive/2013/03/11/2953420.html)，查端口占用通常用 `-nlp`，如果出现 (Not all processes could be identified, non-owned process info will not be shown, you would have to be root to see it all.) 是正常情况，意思就是没端口没被占用

- **Network Usage** -  [`nethogs`](https://github.com/raboof/nethogs) and [`iftop`](http://www.ex-parrot.com/pdw/iftop/) are good interactive CLI tools for monitoring network usage.

If you want to test these tools you can also artificially impose loads on the machine using the [`stress`](https://linux.die.net/man/1/stress) command.

```shell
cat /etc/network/interfaces | egrep eth0 -A 1 | egrep address | awk '{print $2}' | egrep '^10'
```

##### Specialized tools

Sometimes, black box benchmarking is all you need to determine what software to use. Tools like [`hyperfine`](https://github.com/sharkdp/hyperfine) let you quickly benchmark command line programs. For instance, in the shell tools and scripting lecture we recommended `fd` over `find`. We can use `hyperfine` to compare them in tasks we run often. E.g. in the example below `fd` was 20x faster than `find` in my machine.

```
$ hyperfine --warmup 3 'fd -e jpg' 'find . -iname "*.jpg"'
Benchmark #1: fd -e jpg
  Time (mean ± σ):      51.4 ms ±   2.9 ms    [User: 121.0 ms, System: 160.5 ms]
  Range (min … max):    44.2 ms …  60.1 ms    56 runs

Benchmark #2: find . -iname "*.jpg"
  Time (mean ± σ):      1.126 s ±  0.101 s    [User: 141.1 ms, System: 956.1 ms]
  Range (min … max):    0.975 s …  1.287 s    10 runs

Summary
  'fd -e jpg' ran
   21.89 ± 2.33 times faster than 'find . -iname "*.jpg"'
```

As it was the case for debugging, browsers also come with a fantastic set of tools for profiling webpage loading, letting you figure out  where time is being spent (loading, rendering, scripting, &c). More info for [Firefox](https://developer.mozilla.org/en-US/docs/Mozilla/Performance/Profiling_with_the_Built-in_Profiler) and [Chrome](https://developers.google.com/web/tools/chrome-devtools/rendering-tools).


### Exercises

(Advanced) Read about [reversible debugging](https://undo.io/resources/reverse-debugging-whitepaper/) and get a simple example working using [`rr`](https://rr-project.org/) or [`RevPDB`](https://morepypy.blogspot.com/2016/07/reverse-debugging-for-python.html).    

rr:

```
watch -l XX
reverse-cont
```

[memory-profiler](https://pypi.org/project/memory-profiler/)

[pycallgraph](http://pycallgraph.slowchop.com/en/master/)



**限制进程资源**

taskset --cpu-list 0,2 stress -c 3

[利用cgroup控制内存等资源](https://segmentfault.com/a/1190000008125359)

