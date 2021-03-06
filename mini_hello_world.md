---
---
# A Minimal *Hello, World* on EdgeNet
In this tutorial, we're going to show you how to deploy a minimal
experiment across EdgeNet and use it.  This tutorial assumes that you're
using a unixoid system, such as Linux, macOS, or a Linux-based
VM on Windows. (It's likely a Cygwin environment will work on Windows as
well, but we haven't tested it.)

## What You Will Do
In **Part 1** of this tutorial,
you will deploy a pre-built Docker image to EdgeNet nodes around the
world. The image contains a simple web echo server that
provides the following interaction: navigating to it with the query
string `?hostname=foo` will return a page with the text `Hello
dyn1234567.yourisp.com from foo!` (using the name or IP address
associated with your current uplink).

In **Part 2** of this tutorial,
a script on your own machine will navigate to each of
the web servers and collect the responses.


## Preparations

First of all, create an account at EdgeNet (see [Using EdgeNet](https://edge-net.org/using_EdgeNet.html)),
making a note of your *namespace*.

For Part 1 of the tutorial, all you need is a web browser.

Part 2 requires a working installation of [Python 2.7](https://www.python.org/downloads/).
We also recommend the [Kubernetes](https://kubernetes.io/) control tool,
[`kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl/),
to control your Kubernetes cluster from the command line.


## Download and test config
1. Download your config file from the [EdgeNet portal](https://sundewcluster.appspot.com/)
   to `$HOME/.kube/config`. See [Using EdgeNet](https://edge-net.org/using_EdgeNet.html)
   for details of the process.
2. If you already installed `kubectl`, run it to make sure that you're talking to the
  right cluster:  `$ kubectl get nodes`. The resulting list should show many machines
  under the `edge-net.io` domain.


# Part 1: Deploy a Service on EdgeNet
Log in to the [EdgeNet head node](https://headnode.edge-net.org/)
following the directions in [Using EdgeNet](https://edge-net.org/using_EdgeNet.html).
Once you are logged in and have chosen your namespace, you should
see this screen:

![Create Button](assets/images/createButton.png)

Click the Create Button in the top right.  You should see this
screen:

![Create](assets/images/create.png)

Enter the following YAML code into the text box:

```yaml
apiVersion: extensions/v1beta1
kind: ReplicaSet
metadata:
  name: hello-world
spec:
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: hello-world
          image: edgenetproject/helloworld
          ports:
          - containerPort: 8000
            hostPort: <YOUR PORT>
```

Substitute `<YOUR PORT>` with a TCP port number between 40000 and 65000.
<span id="SubstituteRandomNumberHere"></span>

{% raw %}
<script>
/*
 * Replace the suggestion to use "a random port" with
 * an actual random port number in the range min to max.
 * Random int snippet from
 * https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/random
 * Thanks!
 */
min = 40000;
max = 65000;
myPort = Math.floor(Math.random() * (max - min + 1)) + min;
mySpan = document.getElementById("SubstituteRandomNumberHere");
mySpan.innerHTML = "For example, try port " + myPort + " (but don't worry if it is already used by another experimenter &mdash; in that case just try another).";
</script>
{% endraw %}

A `ReplicaSet` [defines a group of Pods placed in the cluster](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/);
in this case, we have one single pod, and since we didn't specify where this
should be placed, Kubernetes will choose a node for us.
The `image` line request the `helloworld` Docker image from our
account on Docker Hub.
The `ports` section of the config tells Kubernetes to map port 8000/TCP
on the container (where the echo server in our Docker image listens)
to the port number
you choose, allowing the echo server to accept incoming connections
on your chosen port.

Next, hit `Upload`. You should now see this:

![Deployed](assets/images/replica_set_deployed.png).

Supposing the node is `toronto.edge-net.io` as shown above, you can now
test with any browser by navigating to
`http://toronto.edge-net.io:<YOUR-PORT>/hello?hostname=Toronto` or
with

```bash
$ curl http://toronto.edge-net.io:<YOUR-PORT>/hello?hostname=Toronto
```

And get "Hello dyn12345678.yourisp.com from host Toronto!" (or similar)
in reply.

Clicking on the links and menus of the Dashboard will give you various
views into your
ReplicaSet.  Play around with them and see what you can find out.  When
you're done, choose `Delete` from the right-hand menu in ReplicaSets.

![Delete](assets/images/delete.png)

It may take a few minutes to delete.

## Deploy A DaemonSet
In this last section we're going to make `hello-world` run on _every_
node in EdgeNet.  And it's just as easy as it was to run on a single
node.

Once again, go to the EdgeNet dashboard and click the `Create` button in
the top right.  This time, when the wizard comes up, enter this YAML
code into the text box:

```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: hello-world
spec:
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: hello-world
          image: edgenetproject/helloworld
          ports:
          - containerPort: 8000
            hostPort: <YOUR PORT>
```

(Remember to substitute `<YOUR PORT>` again.)

Notice that the change from our previous YAML is _one word_: DaemonSet
replaces ReplicaSet.  But this gives a dramatic change in result, as
we'll see.  Click `Upload`.  You will now see this:
![DaemonSet](assets/images/daemon_set.png).

_24 pods running, one on every active EdgeNet node!_. This is precisely
what `DaemonSet`s  are [supposed to do](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/):
Running a copy of a Pod on every node.

--------------

# Part 2: Experiments With Remote Pods

In Part 1 of the tutorial, you deployed a pre-built Docker image of
a simple web echo server to EdgeNet nodes around the world. Now,
we will construct a simple experiment based on this deployment.

As said above, `curl` or your browser essentially suffice to test the
functionality of our deployed web echo service. However, it gets tedious
quickly to check *every* pod, so we will script the interaction and
also gather measurements.

First of all, we need the names of all nodes that we run on. Assuming
you have [`kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
installed already, open a terminal window and type

```bash
$ kubectl get pods -o wide
```

This instructs `kubectl` to query the EdgeNet cluster for all pods
belonging to you, and show the output in "wide" format.
You'll get an output like this (though many more lines):

```bash
NAME                READY     STATUS    RESTARTS   AGE       IP                                      NODE
hello-world-2l6t7   1/1       Running   0          4m        192.1.242.153                           gpo.edge-net.io
hello-world-57sl4   1/1       Running   0          4m        72.36.65.80                             illinois.edge-net.io
hello-world-6qn5z   1/1       Running   0          4m        10.103.0.13                             ufl.edge-net.io
# etc. etc.
```


`kubectl` is an extremely flexible and powerful tool to query and manage
your deployments and interaction with EdgeNet.  We can simply pipe the above
into a file and do some editing, but fortunately `kubectl` will do a lot
of the work for us:

```bash
$ kubectl get pods -o=custom-columns=node:.spec.nodeName --no-headers
illinois.edge-net.io
ufl.edge-net.io
waynestate.edge-net.io
# etc. etc.
```

Just the node names!  That's what we need.  Now let's put them in a file:

```bash
$ kubectl get pods -o=custom-columns=node:.spec.nodeName --no-headers > data.txt
```

We can then use the node list in `data.txt` with some reporting code.

```python
#!/usr/bin/python2.7
import urllib2
import sys
from data import nodes, port
import time

def get_response(node_tuple):
  try:
    query = node_tuple[0]
    return (query, node_tuple[1], urllib2.urlopen(query).read().rstrip())
  except urllib2.URLError:
    return (node_tuple[1], 'Error')


pairs = [(node, node.split('.')[0]) for node in nodes]


#
# build the queries
#
queries = [('http://%s:%d/hello?hostname=%s' % (pair[0], port, pair[1]), pair[0]) for pair in pairs]

#
# get the results and split into error and non-error
#

results = [get_response(query) for query in queries]
errors = [result for result in results if result[1] == 'Error']
results = [result for result in results if result[1] != 'Error']

#
# Print the unreachable nodes
#
if (len(errors) > 0): 
  print '| Unreachable |'
  print '|-------------|'
  for e in errors: print '|'  + e[0] + ' |'
if (len(results) > 0):
  # get   the times for each result, and set up records
  # for printing (node, greeting, time)
  final = []
  for r in results:
    start = time.time()
    get_response(r)
    end = time.time()
    final.append((r[1], r[2], (end - start) * 1000))
  #
  # print the results
  #
  
  print '| Node | Greeting | Time in ms |'
  print '|------|:--------:|-----------:|'
 
  for f in final:
    print '%s | %s | %d' % f

```

This will take awhile, and we may find that some nodes aren't as healthy
as we think.  Those are all the errors.  When the code runs, this is
what we see:


| Unreachable |
|-------------|
|toronto-core.edge-net.io |
|toronto.edge-net.io |
|node-0 |
|france.edge-net.io |
|clemson.edge-net.io |


| Node | Greeting | Time in ms |
|------|:--------:|-----------:|
illinois.edge-net.io | Hello, World, from illinois! | 134
ufl.edge-net.io | Hello, World, from ufl! | 156
waynestate.edge-net.io | Hello, World, from waynestate! | 147
osf.edge-net.io | Hello, World, from osf! | 17
wv.edge-net.io | Hello, World, from wv! | 153
ucsd.edge-net.io | Hello, World, from ucsd! | 35
nysernet.edge-net.io | Hello, World, from nysernet! | 160
uh.edge-net.io | Hello, World, from uh! | 127
ohio.edge-net.io | Hello, World, from ohio! | 148
indiana.edge-net.io | Hello, World, from indiana! | 134
cenic.edge-net.io | Hello, World, from cenic! | 29
louisiana.edge-net.io | Hello, World, from louisiana! | 117
iminds.edge-net.io | Hello, World, from iminds! | 491
nps.edge-net.io | Hello, World, from nps! | 34
umich.edge-net.io | Hello, World, from umich! | 189
nyu.edge-net.io | Hello, World, from nyu! | 188
northwestern.edge-net.io | Hello, World, from northwestern! | 147
hawaii.edge-net.io | Hello, World, from hawaii! | 132

## Be Sure to Clean Up!

When you're done, choose `Delete` from the right-hand menu in
ReplicaSets:

![Delete](assets/images/delete.png)


## Suggested Future Reading
Here are some starting points for you to further explore the
technologies used in this tutorial, and related technologies:
1. [Using EdgeNet](https://edge-net.org/using_EdgeNet.html)
2. [Docker Tutorial](https://docs.docker.com/get-started/)
3. [Hello, World in Kubernetes](https://kubernetes-v1-4.github.io/docs/hellonode/)
4. [Hello, Minikube](https://kubernetes.io/docs/tutorials/hello-minikube/);
  Minikube allows you to run Kubernetes on your local machine.

