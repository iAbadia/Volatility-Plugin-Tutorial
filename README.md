# Volatility Plugin Tutorial

## Introduction
Developing a plugin for Volatility is way easier that it might appear. The biggest obstacle, in my opinion, is knowing how or where to start. There're plenty of sources where you can learn about Volatility ([The Art of Memory Forensics](https://www.memoryanalysis.net/amf) book, [/r/memoryforensics](https://www.reddit.com/r/memoryforensics/), [Volatility Labs](https://volatility-labs.blogspot.com) blog, etc.) but very few - almost none - where you can learn how to develop a plugin.
Also, [this tutorial](https://gist.github.com/bridgeythegeek/bf7284d4469b60b8b9b3c4bfd03d051e#file-myfirstvolatilitypluginwithunifiedoutput-md) inspired me to make a more complex one, and here we are!

I'll be working on a x64 Ubuntu 16.04 machine.

## Objectives
In this guide you will learn the following:
 - Download and run Volatility from source.
 - Get a memory dump from Oracle's VirtualBox VM.
 - Understand what exactly a Volatility plugin is.
 - Write a working Volatility plugin.

## Before we begin
Be sure to have [Oracle's VirtualBox](https://www.virtualbox.org/wiki/Downloads) installed. You won't need the Extension Pack but it's advisable to have it. Also, If you don't have a licensed Windows copy go get a [Microsoft Edge and IE test free copy](https://developer.microsoft.com/en-us/microsoft-edge/tools/vms/). Yes, for real.

## Guide

### Download Volatility
The latest Volatility stable version cat be found at [The Volatility Foundation GitHub](https://github.com/volatilityfoundation). You can just clone the repository via:

```
$ git clone https://github.com/volatilityfoundation/volatility.git
```

Now `cd` to volatility's folder and execute a couple or two... wait! We don't have a memory dump to analyze yet. Let's fix that!

### Get a memory dump

You can get a memory dump in multiple ways, we'll leverage virtualization and use VirtualBox to easily get a VM's memory dump. A more extensive version of this process can be found [here](https://andreafortuna.org/how-to-extract-a-ram-dump-from-a-running-virtualbox-machine-b803f9fd2a0), but, long story short:
 - Boot up a virtual machine in VirtualBox and note it's name.
 - `$ vboxmanage debugvm <VM-NAME> dumpvmcore --filename vm.elf`
 - That's it.

Well, this is not a memory dump per se. It's a `.elf` that also contains other data from VirtualBox we don't need. Guess what? Volatility doesn't care. If you really want a raw memory dump go check out the more extensive version.

### Analyze memory dump
Now we get to look into the memory dump. We have all we need so `cd` to volatility's folder and execute `pslist` command like this:
```
$ python vol.py -f <vm-dump> --profile=<vm-profile> pslist
```
Let's elaborate. 

 - **vm-dump**: This is the .elf - or .raw - file that contains the VM's memory dump. It's **imperative** that you provide this argument, otherwise, volatility won't run.
  - **vm-profile**: The profile tells volatility from which machine the memory dump came from. If you don't provide this information, volatility will try to guess - not a good idea btw. A list of valid profiles can be found [here](https://github.com/volatilityfoundation/volatility/wiki/Volatility-Usage#selecting-a-profile) or you can run `$ python vol.py --info`. I recommend the latter.
  - **pslist**: This is the command we invoke.

Since I extracted a Windows 7 SP1 x64 memory dump, I would run:

    $ python vol.py -f /home/user/vmcore.elf --profile=Win7SP1x64 pslist
    Volatility Foundation Volatility Framework 2.6
    Offset(V)          Name                    PID   PPID   Thds     Hnds   Sess  Wow64 Start                          Exit                          
    ------------------ -------------------- ------ ------ ------ -------- ------ ------ ------------------------------ ------------------------------
    0xffffc480b4258040 System                    4      0    107        0 ------      0 2017-07-13 18:18:39 UTC+0000                                 
    0xffffc480b45a6040 smss.exe                296      4      4        0 ------      0 2017-07-13 18:18:39 UTC+0000                                 
    0xffffc480b5b1e7c0 csrss.exe               384    368     11        0      0      0 2017-07-13 18:18:42 UTC+0000                                 
    0xffffc480b62d3080 smss.exe                448    296      0 --------      1      0 2017-07-13 18:18:42 UTC+0000   2017-07-13 18:18:42 UTC+0000  
    0xffffc480b62d6080 wininit.exe             456    368      4        0      0      0 2017-07-13 18:18:42 UTC+0000                                 
    0xffffc480b62d9580 csrss.exe               464    448     12        0      1      0 2017-07-13 18:18:42 UTC+0000                                 
    0xffffc480b630d080 winlogon.exe            516    448      5        0      1      0 2017-07-13 18:18:42 UTC+0000
    ...

`pslist` is such a trivial command, it simply lists all running processes. But don't be fooled, volatility has quite a set of useful commands which can be found [here](https://github.com/volatilityfoundation/volatility/wiki/Command-Reference).

### What's a Volatility plugin?
You want to develop a plugin for volatility, but, do you know what a plugin is? The first thing to know here is: **EVERYTHING IS A PLUGIN**. Yes, there're plugins developed by regular people - you and me - and the ones developed by The Volatility Foundation, that are included in Volatility. The so-called volatility commands are nothing more than plugins, `pslist` is a plugin. The plugin you'll create will be invoked almost the same way we used `pslist`. 

### Create the plugin
Now that we know what we're going to do, time to learn how to make it. We'll start by creating a folder for our plugin:

```
cd ~
mkdir myPlugin
cd myPlugin
```
Now we create a file for the plugin. Note that the name you give this file will be the name of the command you'll have to give volatility later to run your plugin. I'll call it `myplugin`.
```
vim myplugin.py
```

Ok, I was kidding, I'm not using vim for this. Any text editor will do: gedit, [ATOM](https://atom.io/), [VS Code](https://code.visualstudio.com/), [Sublime Text](https://www.sublimetext.com/), etc.

#### The basics
We're going to do this the right way: Write the minimum for the plugin to run and build up from it. This would be the simplest working plugin:

    import volatility.plugins.common as common
    class MyPlugin(common.AbstractWindowsCommand):
        def render_text(self, outfd, data):
            print "Hello world!"

Let me elaborate:
 - `class MyPlugin(common.AbstractWindowsCommand)`: This is the class that Volatility instantiates when you run the plugin. It **must** inherit from one of the `common.Command` subclases, I've chosen `common.AbstractWindowsCommand` because my plugin will be Windows-oriented.
 - `def render_text(self, outfd, data)`: This is the called function for presenting the results in plain text format. **outfd** is the file descriptor to which Volatility will write, by default it's `stdout` but you can give it with `--output-file`. We didn't do any work so, after (not) doing all the things my plugin does, it's called and prints `Hello world!` to `stdout`. You can tell the plugin to present the results in plain text, dot, JSON or even a formated HTML document but we'll get to that later.

Now we can run it with:

```
$ python vol.py --plugins=<myplugin-dir> -f <vm-dump> --profile=<vm-profile> myplugin
```

Note that the `--plugins` argument **must** be passed right after `vol.py`, doing it elsewhere wont' work. Also, the path provided must be absolute:

 - This is good: `python vol.py --plugins=/home/user/volpydev/myplugin -f <vmcore> myplugin`
 - This is not:  `python vol.py -f <vmcore> --plugins=/home/user/volpydev/myplugin myplugin`
 - This isn't either: `python vol.py --plugins=volpydev/myplugin -f <vmcore> myplugin`

#### The not-so-basics
Now we're going to make something with that memory dump and output something more useful than `Hello world!`. Take this code:

    import volatility.plugins.common as common
    import volatility.utils as utils
    import volatility.win32 as win32

    class MyPlugin(common.AbstractWindowsCommand):
        ''' My plugin '''

        def calculate(self):
            addr_space = utils.load_as(self._config)
            tasks = win32.tasks.pslist(addr_space)
            return tasks

        def render_text(self, outfd, data):
            for task in data:
                outfd.write("{!s}\n".format(str(task)))

Important things here:

 - `''' My plugin'''`: This docstring will be written when calling your plugin with `-h` (or `--help`)
 - `def calculate(self)`: Here is where we do the real work, the returned data will be passed as `data` to `render_text`. What we are doing is getting a list with all the running processes in the given memory dump's address space and return it. `render_text` will iterate over that list and print whatever `str(task)` returns (It happens to be the virtual offset of the process' _EPROCESS internal structure, but that's not important right now).
 - `self._config`: This is the configuration object built for a single specific run of Volatility, it identifies the given memory dump's address space. It also contains other data such as passed arguments.

Now you can run it and see what it prints:

    $ python vol.py --plugins=<myplugin-dir> -f <vmcore> --profile=<vmcore-profile> myplugin
    Volatility Foundation Volatility Framework 2.6
    2223228800
    2238952424
    2246946864
    2238545192
    2223723360
    2238307624
    2247151664
    2247164976
    ...

This looks good, but we can do better! Let's print something that we can understand, take this `render_text` function:

    def render_text(self, outfd, data):
        for task in data:
            outfd.write("{0} {1}\n".format(task.UniqueProcessId, task.ImageFileName))

This will print every process's PID next to it's name. Go check it!

    $ python vol.py --plugins=<myplugin-dir> -f <vmcore> --profile=<vmcore-profile> myplugin
    Volatility Foundation Volatility Framework 2.6
    4 System
    276 smss.exe
    352 csrss.exe
    388 wininit.exe
    396 csrss.exe
    436 winlogon.exe
    480 services.exe
    488 lsass.exe
    496 lsm.exe
    604 svchost.exe
    ...

Now we're talking! This is great but, what if you want the results in greptext for an easier processing? Or even an sqlite3 database? We'll need the **unified output** for that, let's find out how!

#### Unified Output
Here we learn how to print the right way. `render_text()` works for a quick fix but, if we want our plugin to offer the best we need `unifed_output()`. This will centralize all the presenting results stuff into one function - actually two. However, if you want to do something "special" when printing on any specific format you can always define `render_x()` - where x is any of the [available formats](https://github.com/volatilityfoundation/volatility/wiki/Unified-Output#standard-renderers) - and do it as you like. Defining `render_x()` actually overrides the default printing function. We've already done that, remember?

As I said, it's not just one function, but two: `unified_output()` and `generator()`. The latter will provide `unified_output()` a generator with the data and `unified_output()` will return a `TreeGrid` object with that data and it's structure. An example is worth a thousand words, so:

    def generator(self, data):
        for task in data:
			yield (0, [
			    int(task.UniqueProcessId),
			    str(task.ImageFileName)
			])

    def unified_output(self, data):
        tree = [
            ("PID", int),
            ("Name", str)
            ]
        return TreeGrid(tree, self.generator(data))

As you can see, `TreeGrid` takes a tuple array and the generator returned by `generator()`. This tuple contains the name and data type of each column. `generator()` will return a generator with the process's PID and name. Note that the types of the data given to `yield` in `generator()` and the types in the tuple list must coincide. Let's see the result:

    import volatility.plugins.common as common
    import volatility.utils as utils
    import volatility.win32 as win32

    from volatility.renderers import TreeGrid

    class MyPlugin(common.AbstractWindowsCommand):
        ''' My plugin '''

        def calculate(self):
            addr_space = utils.load_as(self._config)
            tasks = win32.tasks.pslist(addr_space)
            return tasks

        def generator(self, data):
            for task in data:
                yield (0, [
                    int(task.UniqueProcessId),
                    str(task.ImageFileName)
                ])

        def unified_output(self, data):
            tree = [
                ("PID", int),
                ("Name", str)
                ]
            return TreeGrid(tree, self.generator(data))

Now execute it and check that the results... are exactly the same as before. But hey! now we can get the results in other formats too. Try json for example (It'll output a minified JSON, I've beautified it):

    $ python vol.py --plugins=<myplugin-dir> -f <vmcore> --profile=<vmcore-profile> myplugin --output=json
    Volatility Foundation Volatility Framework 2.6
    {
    "rows":[
        [
            4,
            "System"
        ],
        [
            276,
            "smss.exe"
        ],
    ...
    ],
    "columns":[
        "PID",
        "Name"
    ]
    }

#### Other interesting things
Now we're doing useful work and using unified output, what else can we do? I'll teach you a couple interesting things but the best way to learn is by doing!

##### Parameters
A nice way to give your plugin some life is by taking parameters. We're gonna use the `__init__()` function for this. For example, we want to give our plugin a prefix for all processes' names. Why? Why not! Here's how we do it:

    def __init__(self, config, *args, **kwargs):
        common.AbstractWindowsCommand.__init__(self, config, *args, **kwargs)
        self._config.add_option('PREFIX', short_option = 'P', default = None, help = 'Prefix all names.', 
            action = 'store')

    def calculate(self):
        addr_space = utils.load_as(self._config)
        tasks = win32.tasks.pslist(addr_space)

        prefix = self._config.PREFIX
        if prefix:
            for task in tasks:
                task.ImageFileName = str(prefix) + task.ImageFileName
        return tasks

This how we define a new parameter:

 - `'PREFIX'`: This is the parameter name, this way we'll be able to give the prefix via `--prefix <prefix>`.
 - `short_option = 'P'`: Why write `--prefix <prefix>` when we can do it shorter: `-P <prefix>`.
 - `default = None`: If no PREFIX parameter given, self._config.PREFIX will be whatever we put here. In this case: `None`
 - `help = 'Prefix all names'`: This is what will be written about this parameter when running the plugin with `-h` - or `--help` -.
 - `action = 'store'`: This indicates that the parameter takes a value behind, the prefix in this case. Instead of `store` you can use `store_true` (giving the parameter will make it store True, or False) or `append` (multiple parameters will stack: -P hehe -P you ) too.


##### Use other plugins from inside yours
Please, don't reinvent the wheel. Volatility comes with a lot of plugins that might do part of the work your plugin needs, you simply need to call that other plugin from yours. Let's say you need a the Portable Executable that's in a process' address space, `procdump` can do it for you:

    def build_conf(self):
        # Create conf obj
        procdump_conf = conf.ConfObject()

        # Define conf
        procdump_conf.readonly = {}
        procdump_conf.PROFILE = self._config.PROFILE
        procdump_conf.LOCATION = self._config.LOCATION
        procdump_conf.DUMP_DIR = tempfile.mkdtemp()

        return procdump_conf
    
    def calculate(self):
        ...
        procdump_conf = self.build_dump_conf()
        # Run ProcDump
        p = procdump.ProcDump(procdump_conf)
        p.execute()
        # It's done, get rid of it
        del p
        ...

You need to build a `ConfObject`, just like the one that Volatility gives you in `self._config`. Give that configuration object to the plugin you want to run, execute it and remember to delete it afterwards - you no longer need it.

## Other community plugins

A nice way to learn and see what others can come up with wile developing a plugin is navigation through the [Community Plugins GitHub](https://github.com/volatilityfoundation/community).