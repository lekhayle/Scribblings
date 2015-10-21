# Problem / Requirements

Running small web services from hardware you own, from your private internet connection, is increasingly feasible. Average uplink speeds are increasing, and with the rising popularity of hardware like the raspberry pi, having low powered, cheap, hardware which can host such content is more widely available.

For example, if you are hosting a blog discussing controversial issues, then hosting the blog yourself could result in several real benifits. It very much restricts the ability of third parties to censor your content without directly involving you. Tumblr, blogger, twitter, etc, all of these services are able to unilaterally terminate your service as a result of external pressures - whether that be governmental or business. The only way to really "take down" your content would be to terminate your internet service - which is definitely a more difficult task.

But there are several important issues that prevent this from being a viable model currently. One of them is being able to address your services. The IP of your home internet connection can change at the whim of your ISP, so maintaining DNS rules that point to your IP is non-trivial. This problem is one of usability. With this problem your content is hard to find and address. There have been several attempts at solving this problem in the past - opendns, dyndns, no-ip.

The other big problem is your IP being public. This exposes too much of your identity and location to the web, importantly your physical location. This exposes you up to real world dangers very directly. This problem is one of protecting important personal information. Tor is the big player at the moment in addressing this issue.

A solution to this use case, whatever form it takes, has to meet the following requirements:

1. A device should be able to be connected to home internet connections to host services.
2. Those services should be addressable by anyone on the internet without requiring them to install bespoke services.
3. It shouldn't be possible to use the addressing system of the service to find the host's IP address.
4. The system should run on cheap-low powered devices (rpi2 as a first pass)
5. It is sufficient if the only supported traffic is HTTP or HTTPS. The big benifit here is that most compliant HTTP clients are capable of relatively complicated redirection logic, and the protocol is neatly proxyable.
6. The purpose of the system is **NOT** to hide the fact from thrid parties that the service is receiving requests, nor is it to protect the identities of those making the requests. It is merely to make it safe and easy to host these services. Users of these services can access them through Tor to attain private anonymized reading.
7. As far as possible the system should not require significant investment of third parties to sustain itself, the CPU and bandwidth of the service-running devices connected to the network should be providing the majority of the horse power.

# Proposal

At a very high level the proposal is a mixture of the offerings of standard dynamic dns solutions and the protocols driving Tor hidden services.

## Example

As an example, take a service called `wordcounter`, which hosts a simple service which tells you how many words are in a piece of text. `wordcounter` runs on an rpi connected to Alice's internet connection. Alice connects her device to the `gananam.net` anonymising network. Her service is now addressable at `wordcounter.gananam.net`. There are many other similar services connected to the network.

When Bob requests `wordcounter.gananam.net` the first step is a DNS request to resolve the host. This request returns the host of any of the many services on the `gananam.net` network. Lets say that it resolves to `carolsblog`. The connection is made directly to the IP of the host running `carolsblog.gananam.net`. This is a HTTP connection, with `wordcounter.gananam.net` in the host header. With that information the node running `carolsblog` uses a system very similar to Tor's routing mechanism to channel the request and the response to `wordcounter.gananam.net`.

The result is that Bob is given an IP to communicate with. The only thing that he can determine about this IP is that it hosts a gananam node. Nodes in the network know that there is a `wordcount` on the network, but do not know the IP of the host serving it. 

One big difference between this system and other similar systems, is it does not aim to provide a single global network, but rather many smaller overlapping networks. Because there is a desire to make each service node an exit node (to borrow Tor terminology), the current issues around personal liability for network traffic is particularly acute. To resolve the issues having networks which have the ability to restrict membership allows the members of the network have some control over the traffic that exits across their personal internet connection without allowing for any given agenda unilaterally censor the entire network.

## Details

There are two classes of node in the system:
* Management Nodes
  * Manages the DNS records. Each network has a single root domain name, in the example above it was `gananam.net`.
  * Keeps track of what service nodes are available and alive. Does this purely based on IP. The management node never gets told what services are registered to what IPs.
  * Validates that certain service names are allowed on the network.
  * Acts as a directory server for the routing logic.
* Service Nodes
  * Provides a named service, or set of named services.
  * Provides a public endpoint to check whether the host is accessible to the web.
  * Manages routing traffic in the network.
  * Checks against the service validation lists on the management nodes before accepting traffic for a service.
  * Can connect to multiple management domains. As per the above example, the node could exist happily in `gananam.net` and `floop.net` and as many other domains as it wishes.

### DNS Management

The management node acts as a DNS nameserver. When issued requests for domains it returns back random sets of IPs from among its list of currently live hosts. The node uses a liveness check (described below) to maintain a list of IPs that it deems currently alive.

The DNS protocol makes this slightly tricky, especially if there are many nodes, and made worse by trying to manage TTLs well.

My current thinking is that the response for `wordcount.gananam.net` results in a `CNAME` record pointing to `SOME-HASH.group.gananam.net`. That `SOME-HASH.group.gananam.net` returns a set of `A` records - on the order of about a dozen nodes. The CNAME record will expire, so periodically the set of nodes hosting a given service will rotate through the network. Each service will be `CNAME`d to a different subgroup, and the subgroup will change every few seconds. This will put significant load on the management node, but this should be horizontally scalable, with each node registered as a name server for the root domain.

### Liveness Check

Service nodes, on launch, sends a request to their management servers. This informs the management servers that they are alive and the management server can tell the service how often to send heartbeat messages. When the management node receives a heartbeat it attempts to issue an HTTP request to the service node. This should hopefully determine whether the node is accessible from the internet. If the request is successful the IP is kept in the list of live servers, and if not, it is discarded. The list is also pruned periodically of dead IPs.
