Perseus is a set of scripts to test how Etcd behaves when a leader is separated from the peer but maintain connection to the clients. It consists of scripts:

  * to download Etcd
  * to run it
  * to generate load and measure a number of successful operations per second per node and per whole cluster

The load generating script is pretty straightforward, it open connections to each of the nodes in the cluster and execute the following loop:

 1. read a value by a key
 2. if the wasn't set then set it to 0
 3. increment the value
 4. write it back
 5. increment a number of successful iterations 
 6. repeat the loop

Each connection uses its own key to avoid collision. If there is an error during the loop then it closes the current connection, opens a new one and begins the next iteration.

Once in a second the script dumps the number of successful iterations for the last second per cluster and per each node (metrics).

A user is expected to mess with the cluster and observe its effect of the metrics.

## Example of an output

The first column is the number of second since the begining of the experiment, the second column is the number of successful iterations per cluster, the last three columns represent the number of successful iterations per each node of the cluster.

<pre>
1	330	140	115	75
2	424	181	149	94
3	433	187	152	94
4	429	183	150	96
5	445	191	157	97
6	429	185	150	94
7	430	187	150	93</pre>

When I killed a leader the whole cluster became unavailable for 2 seconds more a less:

<pre>
117	442	157	184	101
118	454	158	191	105
119	373	131	155	87 #kill -9
120	0	0	0	0
121	33	21	0	12
122	230	149	0	81
123	247	159	0	88</pre>

Since I didn't restart the second node (the former leader) it continued generating zero metrics. Let's repeat experiment with 10x precision (each tick represents 100 ms now):

<pre>
265	49	15	16	18
266	39	13	12	14  #kill -9
267	4	1	1	2
268	0	0	0	0
...
286	0	0	0	0
287	23	13	10	0
288	28	15	13	0</pre>

As you can see the unavailability interval is indeed 2 seconds (20 * 100 ms).

Let's partition the leader with the iptables rules. The unavailability window is around 2 seconds too:

<pre>
22	394	169	138	87 #iptables
23	0	0	0	0
24	5	0	3	2
25	253	0	161	92</pre>

## How to reproduce the test

Prerequisites: [jq](https://stedolan.github.io/jq/) and [node-nightly](https://www.npmjs.com/package/node-nightly).

1. clone this repo
2. update etc/etcd-cluster.json with the ip addresses of your set of nodes
3. commit 'n' push
4. on each of the nodes:
  1. clone your repo
  2. execute `./bin/get-etcd-linux.sh` to download Etcd
  3. execute `./bin/run-etcd.sh ectd1` (if you're on the ectd1 node, use use ectd2 or ectd3 on others)
5. clone your repo on your client node 'n' execute `./bin/test.sh` to start the test

### How to kill a node

1. ctrl-c + ctrl-c
2. `kill -9 ...`
3. Run `sudo iptables -A INPUT -s 10.0.0.5 -j DROP; sudo iptables -A INPUT -s 10.0.0.6 -j DROP; sudo iptables -A OUTPUT -d 10.0.0.5 -j DROP; sudo iptables -A OUTPUT -d 10.0.0.6 -j DROP` to separate current node from the rest of the cluster (10.0.0.5 and 10.0.0.6 in my case); run `sudo iptables -D INPUT -s 10.0.0.5 -j DROP; sudo iptables -D INPUT -s 10.0.0.6 -j DROP; sudo iptables -D OUTPUT -d 10.0.0.5 -j DROP; sudo iptables -D OUTPUT -d 10.0.0.6 -j DROP` to undo the effect