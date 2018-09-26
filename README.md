# BSHELL: Writing a Custom Shell

Bash as a name for a shell is already taken (it stands for Bourne
Again Shell), so in this lab we’ll develop BSH: The Bard Shell.  The
objective of this lab is to become more familiar with UNIX processes
and ever more practice with low-level C programming.

You will need to use pointers, dynamic memory allocation, `fork()`,
and `wait()`.  You must complete this lab so that it compiles and runs
correctly on your Raspberry Pi.  Beware: a program that works on a Mac
may not work on Raspbian!

The shell program you will write runs in an infinite loop, taking a
user command line, parsing it, looking for the command in the
directories specified by the environment's PATH variable, and then
spawning a child process to run the command.  Your first job will be
to parse the path and place it into an array of strings.  You will
finish the main procedure that forks processes and then go on to add
additional capabilities specified below.

Solutions will not receive full credit if they fail to compile, crash,
are incomplete, or have memory leaks.  All functions and any
significant algorithms must be fully explained by comments.

## Part 1: Fixing the helper functions

1. Read through the code (`bshell.h` and `bshell.c`). Note that the
header file, `bshell.h` specifies a number of constants you should
use.  You may need to add more.  Be warned that these constants can
cause and/or hide bugs in your code.

2. You can compile this program by typing `gcc -Wpedantic -o
bshell bshell.c`.  Run the program; it should crash with a
segmentation fault.  Rats!

3. First you need to fix the bug in `parsePath`.  Let's use a
debugger called `gdb` to help.

To use the debugger you must start by compiling with the `-g` flag.
This will retain symbol information needed for running a debugging
program.  You can then run `gdb bshell`.  To run the program, you run
it within the gdb debugger.

```
gdb bshell
(gdb) run
```

Now, when the program crashes, type `where` within the gdb program.
This will tell you where your program crashed.  Here's an example.

```
pi@raspberry$ gcc -g2 -o bshell bshell.c
pi@raspberry$ gdb bshell
...
(gdb) run
...

Program received signal SIGSEGV, Segmentation fault.
strlen () at ../sysdeps/x86_64/strlen.S:106
106	../sysdeps/x86_64/strlen.S: No such file or directory.
(gdb) where
#0  strlen () at ../sysdeps/x86_64/strlen.S:106
#1  0x00007ffff7a7c69c in _IO_puts (str=0x0) at ioputs.c:35
#2  0x0000000000400a2a in parsePath (dirs=0x7fffffffdbe0) at bshell.c:68
#3  0x0000000000400d24 in main (argc=1, argv=0x7fffffffdd48) at bshell.c:179
```


The preceding progam crashed at `bshell.c` line 68 when trying to
access protected memory.  That was within the `parsePath()` function
and seems to be withing the `dirs` variable.  You can use the debugger
to do many other things, but I won't go into that here.  Use `quit` to
exit gdb.

4. Fix the error in `parsePath` before you go on.  Note that you
should not disturb the string returned by `getenv`, so you will need
to dynamically allocate enough memory to store the entire path.

5. Now complete `parsePath`.  You need to parse `thePath`, by looping
over it (via `nextcharptr`), and pointing to each substring in the
`dirs` array that is filled.  For example, if the PATH contains
".:/bin:/usr/bin", you will fill `dirs` as follows:

```
dirs[0] contains .
dirs[1] contains /bin
dirs[2] contains /usr/bin
```

Note that you really just need each element of `dirs` to point to the
right place in `thePath`.  If these are to be interpreted as strings,
you will then replace the delimiter with the string terminator, the
value '\0'.

6.  Once `parsePath` works, you should verify this with the DEBUG loop
at the end of the function.

7. Now to write the actual shell.  This is an infinite loop in `main`
that reads a command and either executes it directly or spawns a child
process to do so.  In the latter case, the actual command is just the
one found on the PATH.

## Part 2: Finishing the shell

1. Create an infinite loop.  This is where the user will remain while
interacting with the shell.  Print a prompt, for now of your choosing,
when user input is expected.  Within the loop use `fgets` to read a
command line into the cmdline variable.  You will be reading from
`stdin` (standard input).  Use `man fgets` if you need help.  You will
want to verify that fgets does not return NULL and that `cmdline[0]`
is not simply a newline.  If either of these cases obtain, the user
will just be prompted again and be allowed to enter a command.

2. If the input command is a non-empty string, call `parseCmd` to read
the command line into a Command structure.  The cmd parameter passed
to `parseCmd` must be a pointer, since that structure is changed
within `parseCmd`.  At the end of the loop you will need to loop over
all command arguments mentioned in the comment of `parseCmd`, freeing
them one by one.  If you fail to do this your program will have a
memory leak.

3. If the `cmd.argv[0]` is equivalent to "exit" then exit the shell.
To do this simply break out of the infinite loop.  Once out of the
loop don't forget to free the block of memory allocated to `dirs`.
You should certainly compile and test your program at this point.

4. Now add code to fork a child process.  If `fork` returns 0 this is
the child process.  Print a message to this effect.  If fork does not
return 0, then have the current process (the parent) wait on the child
process id (ie, `waidpid` command).  At this point your program should
prompt for an input, accept a command line, and fork a process that
simply indicates it is running.  Test your program.

5. Use `lookupPath` to get the path to the command entered by the
user.  Free the memory allocated to fullpath as appropriate.  If the
command is not found, warn the user.

6. If the command is found then fork a new process.  Within the child
process you will call `execv` to replace the copied child
process with the command typed by the user and found
by `lookupPath`.  The parent process should wait on the
child process to finish before it goes on.  At this point you will
have a rudimentary shell.

7. Note: You program should free all allocated memory that is not
needed.  You can find relevant variables by finding calls to `malloc`
and `calloc`, since these are where memory is allocated during
runtime.

8. Modify your shell so that if the last argument on the line is an
ampersand, the parent process does not wait for the child process to
complete.  The child process will continue running in the background,
sharing the CPU with its parent.

9. Add the current working directory to the prompt, so it looks like this example:

```
/home/users/cmsc226/mystuff: 
```

You can limit the entire prompt's length to 512 characters.  Use `man
-k 'working directory'` to find a suitable command.

## Part 3: Built-in Commands for Job Control


1. Add some built-in commands.  A built-in command is one that is
built into the shell: it runs directly, without creating a new
process.  Implement `jobs`, `exit` and `kill` as built-in commands.

   a. **exit** When the user types exit, you should leave the shell.

    b. **jobs** Should list, one per line, each job ID and command
       that is currently running in the background.  Try `jobs` in
       Linux to get an idea how this looks.  Assume that no more than
       10 jobs can be run in the background at one time and enforce
       this limit.  If the user backgrounds an eleventh job, just warn
       them and run it in the foreground.  Job IDs are assigned
       consecutively so that they never repeat.  Hint: see `waidpid`.
       I recommend that you put the jobs information in a structure so
       that the id, pid, and executable name are kept together.

    c. **kill JOB-ID** This command is used with the JOB-ID argument.
       It should terminate the process with the specified job ID in
       this shell (not its process ID).  Hint: see the `kill` system
       call.


## Extra

* Ctrl-C should not kill your shell. (Hint: TLPI Ch. 20-21)

* If a user runs your shell with a command line argument, interpret
  that argument as a name of a script and execute that file line by
  line rather than running interactively:

  ```bshell:lab4$ bshell test.bsh```

* Implement I/O redirection (see section 27.4 of the TLPI. )
  ```ls > listofdirectory.txt```

* Implement piping (see Chapter 44 of the TLPI.)

   ```ls | grep “areyouhere.txt”```
   
* Implement full job control (Ch. 34 of TLPI)

* Implement the wildcard (*) expansion
 ```cat *.txt```

* Implement a history function.
					  
