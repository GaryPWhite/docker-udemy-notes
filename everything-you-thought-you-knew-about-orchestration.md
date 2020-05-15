https://www.youtube.com/watch?v=Qsv-q8WbIZY
demo.consensus.group was down when I checked it, 3 years after the talk :)
https://github.com/ongardie/raftscope makes that site tho ^
bit.ly/logging-post
### Talking mostly about managers and RAFT

we want swarms to seem like one computer from the ourside
Quorum = (N/2) + 1 (round down) = how many nodes that need to be able to vote to do any work.
even numbers are bad for quorum

with multiple regions, put a manager in regions (something aws does fo you)

Raft is a distributed consensus algo, replicates logs, and elects leaders.
It's easier to understand than PAXOS/Multi-PAXOS :O for example, etcd uses raft.

Swarmkit implements Raft. You don't want to run work on manager nodes if you can help it.
using `docker node update --availabilit drain <NODE>` will make it not run work, since manager nodes will need to do work for RAFT.

the log is the source of truth for an application. Logs in this case are append-only time based record.

Debugging tip: read the raft logs, with inotifywait or directly through /var/lib/docker/swarm, whcih is a hugeeeee log for consensus monitoring etc. Not something you'll neede to do a lot, but it's available.

## Orchestrating problems

scheduling problems and HA application problems are an intersection that orchestrators care about. You can set preference with --placement-pref, instead of --constraint that only schedules on particular nodes.

to regain quorum in a disaster, you can docker swarm init --force-new-cluster (which sometimes does not work) or just bring up the managers that went down.

to restore from a backup (something blew up) use a new manager and stop docker as you're recovering.
