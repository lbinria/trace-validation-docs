# Works

We make 3 works around trace validation:

- Development of a library that enable to log events and changes
happening on variables
- Development of a template for trace specification (this is the
spec that is used for the validation of a trace against a
particuliar specification)
- Development of a “method” in order to log implementation
properly

3 implementations:

- Two phase protocol (distributed)
- Key value store
- Raft (distributed)

# Trace specification template

Here an example of a base specification (Raft):

```
\* Defines how the variables may transition.
Next == /\ \/ \E i \in Server : Restart(i)
           \/ \E i \in Server : Timeout(i)
           \/ \E i \in Server : BecomeLeader(i)
...
...

Spec == Init /\ [][Next]_vars
```

# Trace specification template - accept trace

SHEMA

# Trace specification template - accept trace

 - At least one path of state space graph must lead to the complete reading of trace file
 - Use a `POSTCONDITION` (hyperproperty)
 - Allow TLC to have non-deterministric behavior

```
TraceAccepted ==
    (* Diameter equal to trace length => *)
    (* Trace file has been read completely at least one time *)
    LET d == TLCGet("stats").diameter IN
    IF d - 1 = Len(Trace) THEN TRUE
    ELSE Print(<<"Failed matching the trace to (a prefix of
    "TLA+ debugger breakpoint hit count " \

POSTCONDITION
    TraceAccepted
```

# Trace specification template - spec refinement

We have to write a trace spec that is a refinement of a base spec (here Raft).

```
(* Temporal formula for trace spec *)
TraceSpec == TraceInit /\ [][TraceNext]_<<l, vars>>

(* Instanciate raft *)
BASE == INSTANCE raft
BaseSpec == BASE!Init /\ [][BASE!Next \/ ComposedNext]_BASE!vars
```

```
SPECIFICATION
    TraceSpec
PROPERTIES
    (* Refine raft *)
    BaseSpec
```

# Trace specification template - read lines

 - Read trace line after line
 - Each line is an event
 - Apply all operations, on all variables found in each events

```
logline == Trace[l]

ReadNext ==
    /\ l' = l + 1 (* depth: line number *)
    /\ MapVariables(logline) 
    /\ BaseSpec::Next (* Advance base spec *)
```

# Trace specification template

For each action of base spec we define an action (a predicate):

```
IsBecomeLeader ==
    /\ IsEvent("BecomeLeader")
    /\ \E i \in Server : BecomeLeader(i)

IsTimeout ==
    /\ IsEvent("Timeout")
    /\ \E i \in Server : Timeout(i)
...
...

TraceNext ==
        \/ IsRestart
        \/ IsTimeout
        \/ IsBecomeLeader
...
```

# Instrumentation - purpose

- Instrumentation aims to generate a trace by logging some
events
- An event can be seen as an atomic TLA+ action
- Each event is compound by one or many variable updates
- A trace can be seen as a behavior of a system

Trace example:

```json
{
    "clock": 1,
    "state": [{"op": "Replace", "path": ["node2"], "args":
    "commitIndex": [{"op": "Replace", "path": ["node2"], "a
    "desc": "Restart"
}
...
```

# Instrumentation - How to log

1. We have to log events: log all commits is necessary because TLC cannot fill holes in
events
2. We have to log variable changes: log of all variables isn’t necessary, but more variables we log,
more the statespace reduce, and more we are confident in the
implementation

# Instrumentation - log event

Example of log “Timeout” event in Raft:

```java
public void timeout() {
    assert state == NodeState.Follower;
    ...
    spec.commitChanges("Timeout");
}
```

# Instrumentation - log variables

The idea is to log variable updates like you manipulate directly the
specification’s variables.

Declare spec variable example:

```java
this.spec = new TraceInstrumentation(nodeInfo.name() + ".ndjson", clock);
// Binding to variable state at path nodeName (state[nodeName ])
this.specState = spec.getVariable("state").getField(nodeInfo.name());
```

# Instrumentation - log variables

Log variable changes example:

```java
private void setState(NodeState state) {
    this.state = state;
    // this.spec.notify(specState, SET, state.toString());
    this.specState.set(state.toString());
}
...
if (m.isGranted()) {
    // Add node that granted a vote to me
    candidateState.getGranted().add(m.getFrom());
    specVotesGranted.add(m.getFrom());
}
```

# Instrumentation - clocks

We can use two way to sync clock between distributed processes:

 - Lamport clock, we send clock in the message and we call 
explicitly sync method on logging framework
 - Shared clock, if all the system is executed on the same physical
machine, all process can share a clock in a memory mapped file: `SharedClock.get(clockName);`


# Execution pipeline

In all our tests we make a script execution pipeline that do the
following:

- Execute implementation (which create a trace file by logging
events and variable updates)
- Merge trace files that was produced by different processes
- Execute TLC on the trace spec for a given trace file in order to
make validation