# Pregel: A System for Large-Scale Graph Processing

## Brief

A distrubuted graph computing model designed for efficient, scalable and fault-
tolerant implementation with synchronous user API.

## Note

### Common graph algorithms

1. shortest path computations
2. different flavors of clustering
3. variations on the page rank theme
4. minimum cut
5. connected components

### challenges of desgining

1. graph algorithms are lack of locality of memory access
2. very little work per vertex
3. a changing degree of parallelism over the course of execution

### Design of pregel computing model

1. pure messge passing model. For two reasons why choosing pure message passing
   model: first, better expressive; second, better perfomance and lower
   lantency comparing to shared momory
2. all user-define function are performed on vertex, edge is not the first
   level citizen in pregel
3. asynchronous semantic with superstep. In superstep S:
    1. user-defined function is performed on specific vertex
    2. user-defined functino can receive message sent by other vertex in step
       S-1
    3. user-defined function can do anything on vertex and its related out
       edges, even mutate the topology of graph
    4. user-defined function can send message to other vertex
    5. if no further task on this vertex, vertex will be marked as *Inactive*,
       and will vote to halt. pregel will halt when there are no more active
       vertex.
    6. an inactive vertex can re-active by an incoming message
4. messgae can be passed to a non-neighbor vertex, such as some well-known
   vertexes in cliques. But in common usage pattern, messages always passed to
   neightbors
5. Combiner helps combine multiple message into one. Improvment seems limited
6. (sticky)Aggregator, a global state manager or data collector built with upon
   message. It can collect message from each vertex in one(or more) superstep,
   such as counting edges in a graph, each vertex send its outgoing-edge count
   to aggregator. Q: Aggregator can be implemented by broadcast or a central
   server, but detail is not metioned in the paper.
7. Topology Mutations -- partial ordering and handlers. TODO

### System running

1. Start multiple copies of user program on a cluster of machine. One of these
   copies acts as the master. Master are not assigned any partions and is
   responsible for coordinating work activity
2. Master assign graph partions to each worker(default to Hash in Pregel). Each
   worker is given the complete set of assignments for all workers.
3. Then all workers start to read data from source. For those vertext belong to
   current worker, worker keep the vertex; for those which do not, worker send
   its meta data to the specific worker(By message).
4. Perform supperstep 0 and continue
5. halt

### Fault tolerance

Fault tolerance in Pregel is implemented by checkpoint. Failure can be detected
by 'Ping' messages between workers and master.

When Master failed, workers shutdown itself.

When one or more workers failed, master reassigns their partitions to alive
workers with latest checkpoint. Note that the checkpoint at superstep S may be
earlier than the lastest superstep S1. In this situation, system will roll back
all workers to superstep S and recalculate all partitions.

However, rollback the whole system may cause huge resource waste. Thus, Pregel
uses *Confined Recovery* to save compute resouces during recovery. In
*Confined Recovery* mode, workers not only create checkpoints, but also log all
output messages in each super step. During recovery, system only need to
recalculate the lost partitions rather than rollback the whole system. The
paper don't mention how other workers handle the new messages generated during
recovery. Pregel seems to give the choice to user's algorithm. *Confined
Recovery* requires user's algorithm to be deterministic, which means the
algorithm should not be effect by mixing saving messages and original messages.
In my opinion, this means that user-defined algorithm should handle mixing
messages from multiple supersteps correctly.
