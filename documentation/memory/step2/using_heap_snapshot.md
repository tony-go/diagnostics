# Using Heap Snapshot

You can take a Heap Snapshot from your running application and load it into
Chrome Developer Tools to inspect certain variables or check retainer size. You
can also compare multiple snapshots to see differences over time.

## Warning

To create a snapshot, all other work in your main thread is stopped. Depending on the heap contents it could even take more than a minute.  
The snapshot is built in memory, so it can double the heap size, resulting in filling up entire memory and then crashing the app. 

If you're going to take a heap snapshot in production, make sure the process you're taking it from can crash without impacting your application's availability.

## How To

### Get the Heap Snapshot

1. via inspector
2. via external signal and commandline flag
3. via writeHeapSnapshot call withing the process
4. via inspector protocol

#### 1. Use memory profiling in inspector

> Works in all actively maintained versions of Node.js

Run node with `--inspect` flag. Open inspector. 
![open inspector](./tools.png)

The simplest way to get a Heap Snapshot is to connect a inspector to your process running locally and go to Memory tab, choose to take a heap snapshot.

![take a heap snapshot](./snapshot.png)

#### 2. Use `--heapsnapshot-signal` flag

> Works in v12.0.0 or later

You can start node with a commandline flag enabling reacting to a signal to create a heap snapshot.

```
$ node --heapsnapshot-signal=SIGUSR2 index.js
```

```
$ ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
node         1  5.5  6.1 787252 247004 ?       Ssl  16:43   0:02 node --heapsnapshot-signal=SIGUSR2 index.js
$ kill -USR2 1
$ ls
Heap.20190718.133405.15554.0.001.heapsnapshot
```

For details, see the latest documentation of [heapsnapshot-signal flag](https://nodejs.org/api/cli.html#--heapsnapshot-signalsignal)


#### 3. Use `writeHeapSnapshot` function

> Works in v11.13.0 or later  
> Can work in older versions with [heapdump package](https://www.npmjs.com/package/heapdump)

If you need a snapshot from a working process, like an application running on a server, you can implement getting it using:

```js
require('v8').writeHeapSnapshot()
```

Check [writeHeapSnapshot docs](https://nodejs.org/api/v8.html#v8_v8_writeheapsnapshot_filename) for file name options

You need to have a way to invoke it without stopping the process, so calling it in a http handler or as a reaction to a signal from the operating system is advised.  
Be careful not to expose the http endpoint triggering a snapshot. It should not be possible for anybody else to access it.

For versions of Node.js before v11.13.0 you can use the  [heapdump package](https://www.npmjs.com/package/heapdump)

#### 4. Trigger Heap Snapshot using inspector protocol

Inspector protocol can be used to trigger Heap Snapshot from outside of the process. 

It's not necessary to run the actual inspector from Chromium to use the API.

Here's an example snapshot trigger in bash, using `websocat` and `jq`

```bash
#!/bin/bash
set -e

kill -USR1 "$1"
rm -f fifo out
mkfifo ./fifo
websocat -B 10000000000 "$(curl -s http://localhost:9229/json | jq -r '.[0].webSocketDebuggerUrl')" < ./fifo > ./out &
exec 3>./fifo
echo '{"method": "HeapProfiler.enable", "id": 1}' > ./fifo
echo '{"method": "HeapProfiler.takeHeapSnapshot", "id": 2}' > ./fifo
while jq -e "[.id != 2, .result != {}] | all" < <(tail -n 1 ./out); do
  sleep 1s
  echo "Capturing Heap Snapshot..."
done

echo -n "" > ./out.heapsnapshot
while read -r line; do
  f="$(echo "$line" | jq -r '.params.chunk')"
  echo -n "$f" >> out.heapsnapshot
  i=$((i+1))
done < <(cat out | tail -n +2 | head -n -1)

exec 3>&-
```

Not exhaustive list of memory profiling tools usable with inspector protocol:

- [OpenProfiling for Node.js](https://github.com/vmarchaud/openprofiling-node)


## How to find a memory leak with Heap Snapshots

To find a memory leak one compares two snapshots. It's important to make sure the snapshots diff doesn't contain unnecessary information. Following steps should produce a clean diff between snapshots.

1. Let the process load all sources and finish bootstrapping. It should take a few seconds at most. 
1. Start using the functionality you suspect of leaking memory. It's likely it makes some initial allocations that are not the leaking ones.
1. Take one heap snapshot.
1. Continue using the functionality for a while, preferably without running anything else in between.
1. Take another heap snapshot. The difference between the two should mostly contain what was leaking.
1. Open Chromium/Chrome dev tools and go to *Memory* tab
1. Load the older snapshot file first, newer one second ![Load button in tools](./load-snapshot.png) 
1. Select the newer snapshot and switch mode in a dropdown at the top from *Summary* to *Comparison*. ![Comparison dropdown](./compare.png) 
1. Look for large positive deltas and explore the references that caused them in the bottom panel.


Practice capturing heap snapshots and finding memory leaks with [a heap snapshot exercise](https://github.com/naugtur/node-example-heapdump)
