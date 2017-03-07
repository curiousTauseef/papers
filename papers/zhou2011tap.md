# [TAP: Time-aware Provenance for Distributed Systems (2011)](https://scholar.google.com/scholar?cluster=7431618492252098377)
**Network provenance**---provenance, but over the network---allows network
administrators to query the *history* and *derivation* of a tuple generated by
some distributed system written in NDLog. Applying provenance to distributed
systems has a number of challenges:

1. **Transient and inconsistent state.** When a tuple is inserted or deleted
   from a declarative network, the provenance information has to be updated.
   This update is orchestrated by multiple nodes over the network, so it can
   take a while. Provenance queries that occur during the update may see
   inconsistent results.
2. **Explanations for state changes.** Traditional provenance can answer why a
   particular query exists right now, but it can't answer why the tuple has
   appeared, disappeared, or changed.
3. **Security without trusted nodes.** It would be nice if we could answer
   provenance queries even if some nodes in the system have been compromised by
   an attacker.

## Time-aware Provenance
TAP models provenance as a graph which is stored in a collection of
horizontally partitioned relations, similar to *Efficient Querying and
Maintenance of Network Provenance at Internet-Scale*. The difference is in the
vertices and edges.

### Vertices
Each TAP vertex is one of four forms:

- `Insert(n, $\tau$, t)` represents that tuple $\tau$ was created at node `n`
  at time `t`.
- `Delete(n, $\tau$, t)` represents that tuple $\tau$ was deleted at node `n`
  at time `t`.
- `Derive(n, $\tau$, R, t)` represents that tuple $\tau$ was derived using rule
  `R` at node `n` at time `t`.
- `Underive(n, $\tau$, R, t)` represents that tuple $\tau$ was underived using
  rule `R` at node `n` at time `t`.

### Edges
Edges in a TAP provenance graph represent dataflow just like they do in
traditional provenance graphs. There are also special **update edges** that
capture non-dataflow changes. For example, imagine the rule

```
foo(min<A>) :- bar(A)
```

If a new entry `bar(B)` is added which is smaller than the existing minimum
`bar(A)`, then the `bar(A)` entry is removed. This removal cannot be captured
by a traditional dataflow edge, but it can be captured by an update edge.

### Derivations
As with network provenance, NDLog programs can be rewritten to maintain
provenance information at runtime.

## Provenance Maintenance
There are three ways to physically store provenance information over time (in
addition to the naive approach of copying the provenance information in its
entirety for every timestep).

1. **Provenance deltas.** We can store provenance deltas over time. This
   reduces the storage overhead but increases the cost of querying provenance.
2. **Per-node input logs.** If our programs are deterministic, we can log the
   inputs to a program and recreate the provenance information on the fly.
3. **System input logs.** We can record the base tuples of the entire system
   and replay the entire system on the fly. We can also checkpoint to avoid
   re-doing massive amounts of work.

There are a couple of factors which influence which storage format is best:

- **Querying frequency.** If provenance queries are rare, then storing per-node
  input logs reduces the amount of stored data at the cost of more expensive
  queries. On the other hand, if provenance is queried often, provenance deltas
  is likely a good fit.
- **System runtime.** If programs run for a finite amount of time (e.g. a
  MapReduce program), then keeping system input logs is feasible.
- **Local derivations.** If programs perform a lot of computation but not a lot
  of communication, then per-node input logs can greatly reduce the amount of
  tuples that have to be stored.

## Provenance Querying
The authors are working on a query language called **TapQL** (inspired by
ProQL) that allows people to concisely query provenance. TapQL queries are
evaluated hierarchically with three layers:

1. **Macroqueries.** TapQL queries can ask for the provenance of more than a
   single tuple. A macroquery iterates over all candidate queries that match
   the TapQL query and issues microqueries for each.
2. **Microqueries.** A microquery recursively collates provenance by walking
   the provenance information relations.
3. **Vertex queries.** Vertex queries return the vertices and edges incident to
   a given vertex.

We can perform a couple of query optimizations:

- As with network provenance, we can cache.
- As with network provenance, we can return from threshold queries early.
- We can enable caching and issue microqueries serially to amplify the benefits
  of caching.
- We can perform traditional DBMS optimizations at the macroquery level to
  decide the order in which we issue microqueries.

## Secure Provenance Querying
It's future work to implement TAP in such a way that provenance can be queried
even when some nodes have been compromised.

## Commentary
- In Section 3, the paper makes the following claim. I don't understand how TAP
  remembers dependencies between tuples that existed at some point in the past.
  The paper never gave an example showing how a tuple remembers derivations
  from tuples in the past. Moreover, I don't understand how that helps prevent
  inconsistent queries that arise because of "transient state".

    > TAP also remembers dependencies between tuples that existed *at some
    > point in the past*, which enables TAP to provide consistent answers to
    > provenance queries even while the system is in a transient state.

- Figure 2 shows a traditional provenance graph and the corresponding TAP
  provenance graph. The two are pretty much identical except for the update
  edge and `Delete(c,minCost(@c,a,5),t3)` vertex. I'm confused how the new
  vertices provide anything new.
- Section 4 describes that because TAP provenance has an additional time field,
  we have to be clever about how we store provenance information in order to
  avoid storing a huge amount of data. Again, it's not clear to me why the time
  field increases the storage overhead.

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>