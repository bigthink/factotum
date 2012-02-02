# Factotum #

_That being so, I employ three factotae, the first to make an announcement, the second to corroborate it, and the third to deploy the coup de grace... It is a happy arrangement, and I make it happier still by rotating the duties of the three factotae, Ned, Ned, and Ned, so none has the opportunity to relax into his role and thus have the opportunity for maleficent scheming against me._
  -- Frank Key, ["Disfigured Nuncio"][1]

**Factotum** is a Ruby-based server agent that waits for instructions and then executes them. Its intended purpose is to automate application deployment and
related tasks in semi-complex cloud scenarios. (I.e. multiple applications on a server, with multiple staging or production environments.) Secondarily, it acts as a lightweight monitoring and heartbeat system. Factotae can watch for each others' presence and take action if one does not check in within a reasonable amount of time.

Factotum's input and output are based entirely on Amazon Web Services. It makes heavy use of AWS [Simple Notification Service][2], [Simple Queue Service][3], and [SimpleDB][4]. It therefore has a sweet spot on [EC2 instances][5], but can just as easily be used in other environments as long as they can reach Amazon on the network more often than not.

## Scope, Scale and Period ##

It is expected that a server will have exactly one `factotum` instance running as a daemon. (See "Installation" below for guidance.) The instance retrieves a bare minimum of configuration from files or AWS metadata, and then pulls the rest from a specified SimpleDB domain. The actual commands it runs are defined along three dimensions.

### Scope ###

The instance is aware of three nested **scope** levels: the _organization_, the _environment_, and its own _host_. The names of these may be given by variables within a `config.yml` file, or on the command line, or (in the case of EC2 instances) drawn from instance tags.

* The _organization_ scope is shared by all factotae within a given AWS identity. Commands defined at this global level may include core system maintenance or executive reporting tasks. Unlike the deeper scopes, the organization name is mainly cosmetic and can be left blank.
* The _environment_ scope is for application clusters and may have names such as **production**, **development-bob**, **staging-20120315** and so forth. Most application deployment activity happens at this scope. If not specified, the environment name defaults to **Default**.  (We suggest not doing this.)
* The _host_ scope applies only to an individual machine. Many Factotum meta-commands (heartbeat, etc.) occur at this scope; it may also be useful for one-off maintenance or debugging. In a proper continuous delivery setup, production tasks at this level should be rare.  If not otherwise specified, the instance's hostname is used.

### Scale ###

Within each scope, a command can be executed by _all_ factotae or by a _single_ factotum.  This is known as the command's **scale**. The default scale for a command is _all._

Behind the scenes, commands with a _single_ scale are delivered to an SQS queue registered for that scope. Each Factotum instance checks three queues periodically: the _organization_ queue shared by all factotae, the _environment_ queue shared by some, and a _host_ queue checked only by itself. Delivering the command via SQS ensures that only one factotum will process it. (Unless it fails, in which it will eventually get resubmitted.)

Commands with an _all_ scale are delivered to an SNS topic for the scope. The topics are subscribed to by every _host_ queue within that scope, which will then deliver the topic to each factotum. (Using queues within topics has two advantages: no need to open and authorize a network port, and ensuring that commands will not be missed due to sporadic failures.)

The default scale for a command is _all._  Of course, within the _host_ scope there is no functional difference between _single_ and _all_.  An SNS topic is still created for each host, however, and commands scaled to _all_ are still delivered to it for consistency and in case outside subscribers wish to monitor its activity.

### Period ###

Commands that need to run on a repeating basis may be specified with a **period**. The default period is _immediate_.

* An _immediate_ command is delivered directly to the appropriate queue or topic and executed as soon as a factotum retrieves it. (Which may in practice be several seconds later, but rarely longer than a minute.)
* A _startup_ command is persisted in the SimpleDB domain, and is run exactly once by every factotum of the appropriate scope as soon as it comes online.
* A string value creates a _repeating_ command. Repeating commands are persisted in the SimpleDB domain and resubmitted to the appropriate scope on the specified schedule. Valid values are:
    * _often_: runs approximately every minute, with a few seconds' skewing
    * _N minutes_: each submission schedules the next one _N_ minutes later, with up to 30 seconds of skewing
    * _hourly_: submissions happen between 55 and 65 minutes apart
    * _N hours_: each submission schedules the next one _N_ hours later, with up to 5 minutes of skewing
    * _daily_: submissions happen between 23 and 25 hours apart
    * _N days_: each submission schedules the next one _N_ days later, with up to 1 hour of skewing

**Why the deliberate skewing?** The simple answer is to avoid the [thundering herd problem][6] and reduce the likelihood of simultaneous heavy tasks. More generally, however, it's because _Factotum already isn't exact._ Each factotum checks the repeating jobs on a regular basis, but to-the-second precision isn't built into the framework. SNS and SQS are also not real-time services and have their own delays. Rather than raise alarms when a task that repeats every 300 seconds takes 302 seconds, we set expectations based on an envelope. (Raise alarms if the task takes 450 seconds, say.)  If you need precise invocations at specific times, use cron.

## Commands ##

Commands are implemented as subclasses of **Factotum::Command** and can do almost anything. A few examples:

* Upgrade an application to a given Git revision (_environment_, _all_)
* Apply a database migration (_environment_, _single_)
* Run analytics on local log files (_environment_, _all_)
* Send a daily status report for executive review (_organization_, _single_, _repeating_)
* Upgrade a core Linux package for a security patch (_organization_, _all_)
* Add an SSH key for a new developer (_organization_, _all_)
* Open an SSH port to allow login for debugging (_host_)
* Factotum meta-commands such as heartbeat and status reporting (_host_, _repeating_)

The class must implement a `#run` method which performs the work. It may optionally provide `#before` and `#after` methods as well, for setup and teardown, and a `#condition` method which will stop the command from running if it returns a _false_ or _nil_ value. All of these have access to the `#params` hash for parameters passed in at invocation.


### Invocation ###

A command invocation is a simple JSON document:

    {
      type: "Factotum::Command",
      command: "Application::CheckStatus",
      scope: "production-apps",
      scale: "single",
      period: "hourly",
      params: {
        application: "my_finicky_app"
      },
      output: "arn:aws:sns:us-east-1:12345678910:my_output_sns",
    }

The **params** key allows arbitrary parameters to be passed to the command. Its contents are available via the `#params` hash within the command instance; what happens with them is up to your implementation.

The **output** key specifies an alternative SNS topic (or array of topics) to send any results returned by the command. By default a global topic named _FactotumOutput_ is used for the organization. 

### Output ###

Use the `#output` method in your command's run code to return results. Anything that can be rendered in JSON is a valid return value. If `#output` is called more than once, the returns from all calls will be included in an array.  (Not calling it at all is also perfectly fine.)

The output message is sent to one or more SNS topics. The document from a successful command run looks like this:

    { 
      type: "Factotum::Output",
      command: "Application::CheckStatus",
      scope: "production-apps",
      scale: "single",
      period: "hourly",
      started: "2012-03-05T19:35:42Z",
      duration: "42",
      host: "ip-10-244-125-171.ec2.internal",
      success: true,
      output: "I'm okeley-dokeley OK, Bob!"
    }
    
A failed command run will have a **success** value of _false_ and will include an **exception** key describing the exception raised.


[1]: http://hootingyard.org/archives/3230
[2]: http://aws.amazon.com/sqs
[3]: http://aws.amazon.com/sns
[4]: http://aws.amazon.com/simpledb
[5]: http://aws.amazon.com/ec2
[6]: http://en.wikipedia.org/wiki/Thundering_herd_problem


