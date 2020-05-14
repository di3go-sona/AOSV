### KERNEL MESSAGING - (15-04-2020)

Useful for printing information related to events when the system is running in kernel mode. This procedure is called "debugging by printing".
The kernel level function to produce output messages is `printk()`, defined in `kernel/printk/printk.c`. It works very similarly to `printf()`, but formatting floating point values is not allowed, since it's too costly.

#### printk

`printk` takes care of the following tasks:

- Printing on the "console" device, which is a **synchronous** operation, i.e. it cannot be parallalelized and is slow
- Logging the message onto a **circular buffer** located within the kernel level virtual addresses. Messages can be retrieved **asynchronously** from this buffer.

The initial characters of a printk format string represent the "importance" of the message to be printed.

From `include/linux/kern_levels.h`, kernel version 5.6.11:

```C
#define KERN_SOH	"\001"		/* ASCII Start Of Header */
#define KERN_SOH_ASCII	'\001'

#define KERN_EMERG	KERN_SOH "0"	/* system is unusable */
#define KERN_ALERT	KERN_SOH "1"	/* action must be taken immediately */
#define KERN_CRIT	KERN_SOH "2"	/* critical conditions */
#define KERN_ERR	KERN_SOH "3"	/* error conditions */
#define KERN_WARNING	KERN_SOH "4"	/* warning conditions */
#define KERN_NOTICE	KERN_SOH "5"	/* normal but significant condition */
#define KERN_INFO	KERN_SOH "6"	/* informational */
#define KERN_DEBUG	KERN_SOH "7"	/* debug-level messages */
```

There are 4 configurable parameters that determine how kernel messages are managed.  
These are:

- `console_loglevel`: priority below which the messages are printed to the console device
- `default_message_loglevel`: default priority given to messages that haven't specified a priority
- `minimum_console_loglevel`: minimum priority allowed to appear on the console
- `default_console_loglevel`: default priority for messages destined to the console device at boot time

These variables are stored in the `/proc/sys/kernel/printk` file as four integers.

#### Circular Buffer

```C
int syslog(int type, char *bufp, int len);
```

Syslog is the system call used to retrieve the messages contained in the circular buffer.

Syslog can be used with the following commands (from include/linux/syslog.h) in the `type` argument:

```C
/* Close the log.  Currently a NOP. */
#define SYSLOG_ACTION_CLOSE          0
/* Open the log. Currently a NOP. */
#define SYSLOG_ACTION_OPEN           1
/* Read from the log. */
#define SYSLOG_ACTION_READ           2
/* Read all messages remaining in the ring buffer. */
#define SYSLOG_ACTION_READ_ALL       3
/* Read and clear all messages remaining in the ring buffer */
#define SYSLOG_ACTION_READ_CLEAR     4
/* Clear ring buffer. */
#define SYSLOG_ACTION_CLEAR          5
/* Disable printk's to console */
#define SYSLOG_ACTION_CONSOLE_OFF    6
/* Enable printk's to console */
#define SYSLOG_ACTION_CONSOLE_ON     7
/* Set level of messages printed to console */
#define SYSLOG_ACTION_CONSOLE_LEVEL  8
/* Return number of unread characters in the log buffer */
#define SYSLOG_ACTION_SIZE_UNREAD    9
/* Return size of the log buffer */
#define SYSLOG_ACTION_SIZE_BUFFER   10
```

`klogd` is a system daemon which intercepts and logs Linux kernel messages.

The circular buffer has size `LOG_BUF_LEN`. Originally it was 4096 bytes, since 2.1.113 it's been moved to 16384 bytes and today it can be defined at compile time.

A unique buffer is used for any message, independently of its priority level.

The buffer content can be accessed also by using `dmesg`.

Standard library functions use an "at-most-once" semantic, due to asyncronous management. In order for messages to be delivered with an "exactly-once" semantic, `printk` needs to be syncronous. This means that the driver does not return control until the message is sent to the physical console device, slowing the whole system.

#### Panic

`panic` is a function that needs to be invoked when the condition of the kernel reaches such a critical level that the machine needs to be halted.

There are two macros used today in the kernel, `BUG()` and `BUGON()` which are basically `assert`s that will call `panic()` if they fail.