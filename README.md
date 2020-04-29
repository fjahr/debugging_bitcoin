__This document is currently optimized for MacOS. If you would like to help
me add Linux equivalent commands, please let me know.__

# Debugging Bitcoin Core

This guide is designed to give beginners of C++ development and/or people new
to the bitcoin core code base an overview of the tools available for debugging
issues as well as giving hints where issues may trip you up.

## Table of contents

* [General tips](#general-tips)
* [Debugging](#debugging)
    * [Running your own bitcoind](#running-your-own-bitcoind)
        * [Logging](#logging-from-own-bitcoind)
        * [Debugging](#debugging-from-own-bitcoind)
    * [Running unit tests](#running-unit-tests)
        * [Logging from unit tests](#logging-from-unit-tests)
        * [Logging from implementation code](#logging-from-unit-tests-code)
        * [Debugging](#debugging-from-unit-tests)
    * [Running functional tests](#running-functional-tests)
        * [Logging from functional tests](#logging-from-functional-tests)
        * [Logging from C++ code](#logging-from-functional-tests-code)
        * [Debugging](#debugging-from-functional-tests)
* [More Tools for Segfaults](#more-tools-for-segfaults)
   * [Core dumps](#core-dumps)
   * [Valgrind](#valgrind)
* [Further resources](#further-resources)





## General tips

First of all, debugging involves a lot of compiling, so you definitely want to
speed it up as much as possible. I recommend looking at the [general productivity notes](https://github.com/bitcoin/bitcoin/blob/master/doc/productivity.md#general) in
the bitcoin core docs and install `ccache` and optimize your configuration.

Also do not forget the disable optimizations using the `-O0` flag, otherwise
debugging will be impossible as symbol names will not be recognizable.

An example of configure flags: `./configure CXXFLAGS="-O0 -g" CFLAGS="-O0 -g"`

Also note this guide is using `lldb` instead of `gdb` because I am running MacOS.
For Linux users `gdb` seems to be the standard and even if you are using `gdb` the
guide should still work for you, as the tools are [very similar](https://lldb.llvm.org/use/map.html).

Are you in the right spot?
- Mainnet/Testnet/Regtest all have their own debug.log files
- Feature tests log to temp files which get cleaned up unless your test fails or
you specify `--no-cleanup`





## Debugging

### Running your own `bitcoind`

These are examples where you are interacting with the code yourself and don't rely
on tests to reproduce the error.

#### Logging from own `bitcoind`

In general you can use to your `std::out` but this will not appear in any logs.
It is rather recommended to use `LogPrintf`. Insert into your code a line similar
to the following example.

```
LogPrintf("@@@");
```

You can then grep for the result in your `debug.log` file:
```
$ cat ~/Library/Application\ Support/Bitcoin/regtest/debug.log | grep @@@
```
This example shows the path of the regtest environment `debug.log` file. Remember
to change this if you are logging from testnet or another environment.

If you would like to log from inside files that validate consensus rules
(see [`src/Makefile.am`](https://github.com/bitcoin/bitcoin/blob/master/src/Makefile.am#L407))
then you will errors from missing header files and when you have added those
the linker will complain. You can make this work, of course, but I would
recommend you use a debugger in that context instead.

#### Debugging from your own `bitcoind`

You can start your bitcoind using a debugging tool like `lldb` in order to debug
the code:
```
$ lldb src/bitcoind
```

Within the `lldb` console you can set breakpoints before actually running starting
the `bitcoind` process using the `run` command. You can then interact with the `bitcoind`
process like you normally would using your `bitcoin-cli`.

### Running unit tests

You are running unit tests which are located at `src/test/` and use the BOOST library
test framework. These are executed by a seperate executable that is also compiled with 
make. It is located at `src/test/test_bitcoin`.

So helpful tips about execution (this uses the Boost library).

Run just one test file:
`src/test/test_bitcoin --log_level=all --run_test=getarg_tests`

Run just one test:
`src/test/test_bitcoin --log_level=all --run_test=*/the_one_test`

#### Logging from unit tests

Logging from the tests you need to use the BOOST framework provided functions to
achieve seeing the logging output. Several methods are available, but simplest
is probably adding `BOOST_TEST_MESSAGE` in your code:

```
BOOST_TEST_MESSAGE("@@@");
```

#### Logging from unit tests code

To have log statements from your code appear in the unit test output you will have
to print to `stderr` directly:

```
fprintf(stderr, "from the code");
```


#### Debugging Unit Tests

You can start the `test_bitcoin` binary using `lldb` just like bitcoind. This allows you
to set breakpoints anywhere in your unit test or the code and then execute the tests
any way you want using the `run` keyword followed by the arguments you would normally
pass to `test_bitcoin` when calling it directly.

```
$ lldb src/test/test_bitcoin
(lldb) target create "src/test/test_bitcoin"
Traceback (most recent call last):
  File "<input>", line 1, in <module>
  File "/usr/local/Cellar/python@2/2.7.16/Frameworks/Python.framework/Versions/2.7/lib/python2.7/copy.py", line 52, in <module>
    import weakref
  File "/usr/local/Cellar/python@2/2.7.16/Frameworks/Python.framework/Versions/2.7/lib/python2.7/weakref.py", line 14, in <module>
    from _weakref import (
ImportError: cannot import name _remove_dead_weakref
Current executable set to 'src/test/test_bitcoin' (x86_64).
(lldb) run --log_level=all --run_test=*/lthash_tests
```


### Running functional tests

You are running tests located in `test/functional/` which are written in `python`.

#### Logging from functional tests

for log level debug run functional tests with `--loglevel=debug`.
```
self.log.info("foo")
self.log.debug("bar")
```

Use `--tracerpc` to see the log outputs from the RPCs of the different nodes running in the functional
test in std::out.

If it doesn't fail, make it fail and use this:
```
TestFramework (ERROR): Hint: Call /Users/FJ/projects/clones/bitcoin/test/functional/combine_logs.py '/var/folders/9z/n7rz_6cj3bq__11k5kcrsvvm0000gn/T/bitcoin_func_test_epkcr926' to consolidate all logs
```

You can even assert on logging messages using `with self.nodes[0].assert_debug_log(["lkagllksa"]):`.
However, don't change indentation of test lines. Just insert somewhere indented between def run_test
and the body of it.

#### Logging from functional tests code

Using `LogPrintf`, as seen before, you will be able to see the output in the combined logs.

#### Debugging from functional tests

##### 1. Compile Bitcoin for debugging

In case you have not done it yet, compile bitcoin for debugging (change other config flags as you need them).

```
$ make clean
$ ./configure CXXFLAGS="-O0 -ggdb3"
$ make -j "$(($(sysctl -n hw.physicalcpu)+1))"
```

##### 2. Ensure you have `lldb` installed

Your should see something like this:
```
$ lldb -v
lldb-1001.0.13.3
```

##### 3. Halt functional test

You could halt the test with a sleep as well but much cleaner (although a little confusing maybe) is to use another
debugger within python: `pdb`.

Add the following line before the functional test causing the error that you want to debug:
```
import pdb; pdb.set_trace()
```
Then run your test. You will need to call the test directly and not run it through the `test_runner.py` because
then you could not get access to the `pdb` console.
```
$ ./test/functional/example_test.py
```

##### 4. Attach `lldb` to running process

Start `lldb` with the running bitcoind process (might not work if you have other bitcoind processes running). Just running `lldb` instead of `PATH=/usr/bin /usr/bin/lldb` might work for you, too, but lots of people seem to run into problems when lldb tries to use their system python in that case.
```
$ PATH=/usr/bin /usr/bin/lldb -p $(pgrep bitcoind)
```

##### 5. Set your breakpoints in `lldb`

Set you breakpoint with `b`, then you have to enter `continue` since `lldb` is setting a stop to the process as well.
```
(lldb) b createwallet
Breakpoint 1: [...]
(lldb) continue
Process XXXXX resuming
```

##### 6. Let test continue

You can now let you test continue so that the process is actually running into your breakpoint.

```
(Pdb) continue
```

You should now see something like this in you `lldb` and can start debugging:
```
Process XXXXX stopped
* thread #10, name = 'bitcoin-httpworker.3', stop reason = breakpoint 1.1
    frame #0: 0x00000001038c8e43 bitcoind`createwallet(request=0x0000700004d55a10) at rpcwallet.cpp:2642:9
   2639	static UniValue createwallet(const JSONRPCRequest& request)
   2640	{
   2641	    RPCHelpMan{
-> 2642	        "createwallet",
   2643	        "\nCreates and loads a new wallet.\n",
   2644	        {
   2645	            {"wallet_name", RPCArg::Type::STR, RPCArg::Optional::NO, "The name for the new wallet. If this is a path, the wallet will be created at the path location."},
Target 0: (bitcoind) stopped.
(lldb) 
```




## More Tools for Segfaults

### Core dumps

A core dump is a full dump of the working memory of your computer. When activated it will
be created in `/cores` on your computer in case a segfault happens.

To activate core dumps on macOS you have to run
```
$ ulimit -c unlimited
```
then you have to run the process where you are observing the segfault in the same terminal.
You can then inspect them in your `/cores` directory.

You should always make sure to not generate more core dumps than you need to and clean up
your `/cores` directory when you have solved your issue. Core dumps are huge files and
can clutter up your disc space very quickly.

### `valgrind`

#### Install `valgrind`
On newer MacOS  versions this is harder than you might think because you
can only install it through homebrew up to version 10.13. If you currently
have an up-to-date MacOS Mojave installed you will have to use the
following script to make it work.

```
$ brew install --HEAD https://raw.githubusercontent.com/sowson/valgrind/master/valgrind.rb
```

#### Run `bitcoind` with `valgrind`

Using `valgrind` works similar to `lldb`. You start the executable where the segfault might
happen using `valgrind`, when an error is observed you will be able to inspect it.

For example:
```
$ sudo valgrind src/bitcoind -regtest
```

Now you can interact with your `bitcoind` normally using `bitcoin-cli`, for example, and
trigger the error by hand using the right parameters.





## Further resources

### Getting help
- [Bitcoin Stackexchange](https://bitcoin.stackexchange.com) should be your first point of contact if you are truely stuck
- [IRC channels](https://en.bitcoin.it/wiki/IRC_channels) are another possibility to find help for specific topics

### READMEs
- Bitcoin core READMEs have lots of information of course. You may find some overlaps with this guide but they should still
be very helpful.
   - https://github.com/bitcoin/bitcoin/blob/master/src/test/README.md
   - https://github.com/bitcoin/bitcoin/blob/master/test/README.md
   
### Other resources
https://medium.com/provoost-on-crypto/debugging-bitcoin-core-functional-tests-cc0aa6e7fd3e
https://bitcoin.stackexchange.com/questions/76521/debugging-bitcoin-unit-tests

### `lldb`
[Using `lldb` as a standalone debugger](https://developer.apple.com/library/archive/documentation/IDEs/Conceptual/gdb_to_lldb_transition_guide/document/lldb-terminal-workflow-tutorial.html)

[LLVM Tutorial on `lldb`](https://lldb.llvm.org/use/tutorial.html)

### BOOST
- BOOST Introduction: http://archive.is/dRBGf
- BOOST LIB: https://www.boost.org/doc/libs/1_63_0/libs/test/doc/html/boost_test/test_output/test_tools_support_for_logging/checkpoints.html
