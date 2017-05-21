<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.1//EN" "http://www.w3.org/TR/xhtml11/DTD/xhtml11.dtd">

<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en">
    <head>
        <title>Linux Sockets: Which process is listening to a port?</title>
        <link rel="stylesheet" href="article.css" type="text/css" />
    </head>

    <body>

        <h1>Linux Sockets: Which process is listening to a port?</h1>

        <div id="toc"></div>


        <p>Developing software using sockets almost always involves
        starting, stopping, debugging multiple processes opening
        and closing sockets. The processes may or may not work
        properly as differing stages of development creates
        "interesting" problems. To complicate matters, multiple systems
        add to this effort.</p>

        <p>Frequently this development process encounters sockets
        left open after crashes, process not yet properly closing
        or terminating, etc. The end result has processes running
        with unwanted sockets.</p>

        <h1><code>listeningPort.py</code> - A tool to map port names to process pid</h1>

        <p><code>listeningPort.py</code> performs a mapping of a
        port number to a process id, a pid. Furthermore, the
        interface allows a range of ports as input. The output
        may take several hopefully useful forms.</p>

        <p>Additionally, since <code>listeningPort.py</code> maps
        ports to pids, it has the capability to kill a process
        using a specific port. This significantly facilitates
        socket development.</p>

        <h1>A brief demo of the problem</h1>

        <p>The demo web server, <code>demoSocket.py</code>, binds
        to a user selected port and does little else. Its purpose
        is to open a port and keep it open. <code>demoSocket.py</code>
        doesn't do much. It only listens to a socket. It serves
        its purpose by continuing to listen to a port. Its purpose
        is to illustrate the utility <code>listeningPort.py</code>
        and nothing else.</p>

        <p>To illustrate the problem, open two console windows
        and <code>cd</code> to the directory where this code lives.
        In one console, start the demo:</p>

        <pre>
./demoSocket.py
Listening on port 5570</pre>

        <p>Port 5570 is now bound to this process.<p>

        <p>In the other console window, start the <code>demoSocket.py</code>
        in an attempt to bind to the same port:</p>

        <pre>
./demoSocket.py
Listening on port 5570
Traceback (most recent call last):
  File "./demoSocket.py", line 57, in <module>
    main()
  File "./demoSocket.py", line 40, in main
    status = serversocket.bind((socket.gethostname(), port))
  File "/usr/lib64/python2.7/socket.py", line 224, in meth
    return getattr(self._sock,name)(*args)
socket.error: [Errno 98] Address already in use</pre>

        <p>The second attempt to have another process listen to the
        same port fails. Only one process at a time may listen to a specific port.
        Your traceback may be slightly different, the significant
        portion that easily stops development is: "<i>Address already in use</i>.</p>

        <h1><code>listeningPort.py</code> to sherlock who uses that port</h1>

        <p>In that same window that display that <i>Address already in use</i>
        error, determine the pid and command line of the port listener:</p>

        <pre>
./listeningPort.py 5570
Port 5570 : listening thru pid 28235 ./demoSocket.py 5570</pre>

        <p>Now we can see that port 5570 has pid 28235 and the command line
        of pid 28235 is "<code>./demoSocket.py 5570</code>".</p>

        <p>But wait, there's more! Here is the same query with different outputs.
        Your output will have different pids:</p>
        
        <pre>
./listeningPort.py --short 5570
5570 28235 ./demoSocket.py 5570
$ ./listeningPort.py --pid 5570
28235
$ ./listeningPort.py --proc 5570
python ./demoSocket.py 5570
$ ./listeningPort.py --kill 5570</pre>

        <p>The command line switch of <code>--short</code> abbreviates
        the output for ease of parsing using additional utilities such
        as <code>awk</code>, <code>perl</code>, <code>python</code>, etc.</p>

        <p>The <code>--pid</code> flag outputs only the pid of the
        process holding the port.</p>

        <p>The <code>--proc</code> supplies only the command line
        of the process holding the port. This can be useful in
        further identifying the source of that port holder.
        While finding this command line from a pid is trivial,
        encapsulating this in this utility provides yet another
        easy access to this information.</p>

        <p>The <code>--kill</code> uses the port to identify the
        pid and then attempts to kill that process. </p>

        <h1>The <code>--help</code> discussion</h1>

        <p>The <code>--help</code> flag outputs:</p>

        <pre>
./listeningPort.py --help

List the processes that are listening to a port.
Defaults to ZeroMQ port of 5570.

Use by:
  listeningPort [--help] [--short | --pid | --proc | --kill]
                &lt;port0&gt; [&lt;port1&gt; ...]
e.g.:
  listeningPort 5570             # The ZeroMQ default port
  listeningPort 5570 5571 5572   # Multiple ports may be checked
  listeningPort --short 5570     # Short form of output
  listeningPort --pid 5570       # Output pid only
  listeningPort $(seq 5570 5580) # Ports 5570 through 5580 inclusive.
  listeningPort --kill $(seq 5570 5580) # Kill all processes
                                        # listening to ports 5570 thru 5580.

For the case of a free port, the output is similar to:
  Port 5571 : Nobody listening

--help = this message

Only one of the following can be supplied:
--short = Output consists of only three space separated fields:
    &lt;port&gt; &gt;pid of listener&gt; &lt;process name of listener&gt;
    Ports with nobody listening gets ignored for output.
--pid  = Output consists only of a pid
--proc = Output consists only of process names
--kill = Any ports with a listener will be killed with "kill -9 <pid>"
         A successful kill has a return code of non-zero.
         A non-zero output communicates the number of processes
         killed. The --kill may kill multiple processes when a
         sequence of ports gets used.
         An unsuccessful kill returns 0.
         A "sudo" may be required to kill some processes.

Return codes:
   255  == Invalid command line.
    0   == Nobody listening to <port>
  &gt; 0   == The number of ports someone is listening to.
           For a series of port, this value is the number
           of ports with a listener.
           For a single port, this will be 1 is someone
           is listening.


***NOTICE***: This routine does NOT work on OSX!
Replace this with:
    lsof -i&lt;port&gt; | awk '{ print $2; }' | head -2
    PID
    18101
This prints only the pid of the process using this port.
Now use "ps" to find the process:
    ps ax | grep 18191 | grep -v grep
    10191 s001  S+    0:00.00 /usr/bin/python /usr/local/bin/logCollector
    </pre>        

    <h1>Querying multiple ports</h1>

    <p>In socket development a common scenarios uses a range of ports.
    The linux utility "<code>seq</code>" easily supplies these value
    to <code>listeningPort.py</code>. Thus, for our examples that use
    ports 5570 through 5580, <code>seq</code> would display:</p>

    <pre>
seq -s ' ' 5570 5580
5570 5571 5572 5573 5574 5575 5576 5577 5578 5579 5580</pre>

    <p>The -s flag says use a space to separate each sequence number.
    <code>seq</code> has other useful flags. See the man page if interested.</p>

    <p>To illustrate multiple ports, in the console window running
    <code>demoSocket.py</code>, start multiple instance with different
    sockets. Throw each invocation in the background:</p>

    <pre>
./demoSocket.py 5570 &amp;
[1] 29385
$ Listening on port 5570
./demoSocket.py 5573 &amp;
[2] 29487
$ Listening on port 5573
./demoSocket.py 5574 &amp;
[3] 29492
$ Listening on port 5574
./demoSocket.py 5579 &amp;
[4] 29497
$ Listening on port 5579
jobs
[1]   Running                 ./demoSocket.py 5570 &amp;
[2]   Running                 ./demoSocket.py 5573 &amp;
[3]-  Running                 ./demoSocket.py 5574 &amp;
[4]+  Running                 ./demoSocket.py 5579 &amp;
    </pre>

    <p>Now find the pids of these given port numbers:</p>

    <pre>
./listeningPort.py $(seq 5570 5580)
Port 5570 : listening thru pid 29385 ./demoSocket.py 5570
Port 5571 : Nobody listening
Port 5572 : Nobody listening
Port 5573 : listening thru pid 29487 ./demoSocket.py 5573
Port 5574 : listening thru pid 29492 ./demoSocket.py 5574
Port 5575 : Nobody listening
Port 5576 : Nobody listening
Port 5577 : Nobody listening
Port 5578 : Nobody listening
Port 5579 : listening thru pid 29497 ./demoSocket.py 5579
Port 5580 : Nobody listening
    </pre>

    <p>Get a more succinct list:</p>

    <pre>
./listeningPort.py --short $(seq 5570 5580)
5570 29385 ./demoSocket.py 5570
5573 29487 ./demoSocket.py 5573
5574 29492 ./demoSocket.py 5574
5579 29497 ./demoSocket.py 5579
    </pre>

    <p>You have the idea. Now let's kill all those background processes.</p>

    <pre>
./listeningPort.py --kill $(seq 5570 5580)
Port 5571 : Nobody listening
Port 5572 : Nobody listening
Port 5575 : Nobody listening
Port 5576 : Nobody listening
Port 5577 : Nobody listening
Port 5578 : Nobody listening
Port 5580 : Nobody listening
    </pre>

    <p>The processes killed do not report anything, only the ports
    with nobody listening gets reported. Perhaps this could be
    improved?</p>

    <p>In the console starting the <code>demoSocket.py</code>
    processes, press the return key:</p>

    <pre>
[1]   Terminated              ./demoSocket.py 5570
[2]   Terminated              ./demoSocket.py 5573
[3]-  Terminated              ./demoSocket.py 5574
[4]+  Terminated              ./demoSocket.py 5579
    </pre>

    </body>

</html>


