<style>
  svg text {
    text-anchor: middle;
    alignment-baseline: middle;
  }
</style>

# [Eddies: Continuously Adaptive Query Processing (2000)](https://scholar.google.com/scholar?cluster=13049208738754012194)
Traditional query optimizers take in a query, generate a plan, and execute the
plan. Once the plan is generated, it is never changed. This paper focuses on
dynamic query processing which takes in a query, generates a plan, and then
executes the plan while simultaneously changing the plan. More specifically,
the paper introduces an **eddy operator** which dynamically routes tuples
through a set of other relational operators in an efficient way.

## Eddies
The eddy operator is implemented in the River query processing system. In
River, every operator of a query plan is run in its own thread, and operators
exchange tuples between each other via fixed-length message queues.

Before we insert eddy operators into a query plan, we first generate a fixed
query plan with a traditional query optimizer. We restrict the query optimizer
to choose either index joins or hash ripple joins to implement equijoins and to
choose block ripple joins to implement non-equijoins. See the "Reorderability
of Plans" section for the rationale behind this restriction.

An eddy operator encapsulates multiple inputs and multiple unary and binary
operators. It reads tuples from its inputs and feeds them in some order to the
operators. Once a tuple has been processed by all the operators, it is output
by the eddy. This looks something like this:

<svg viewBox="100 25 200 140" id="eddy1"></svg>

Here, the eddy encapsulates a single input (i.e. R) and two unary operators
(i.e. a and b). Each tuple is represented as a dot. Half the tuples are sent to
a before b; the other half are sent to b before a. After a tuple has been sent
to both operators, it is output by the eddy.

An eddy which encapsulates $n$ operators annotates each tuple with an
$n$-length Done bitstring and $n$-length Ready bitstring. Initially, a tuple
$t$'s Done bitstring is zeroed and its Ready bitstring is oned.  $t$ is then
sent to operator $i$ if the $i$th bit of $t$'s Ready bitstring is set and the
$i$th bit of $t$'s Done bitstring isn't set. Once $t$ has been processed by the
$i$th operator, the $i$th bit in Done is set. Once all the bits in Done are
set, $t$ is output.

A couple things to note:

- If there is some ordering on the operators (e.g. a tuple must be sorted
  before it can be sent to a hash join), then a tuple's Ready bitstring may
  initially contain unset bits which become set only after the tuple has been
  processed by some operators.
- The output tuple of a binary operator performs a bitwise or of its input
  tuples' Ready and Done bitstrings.
- Eddy operators can route tuples according to any fixed query plan, including
  bushy plans.

## Naive Eddy
Eddy operators need to decide (a) which tuple to route and (b) which operator
to route it to. The policy which determines (a) is pluggable, but in this
paper, we assume a simple policy which prioritizes tuples that have already
been processed by some operator.

A naive eddy implements (b) by routing a tuple equiprobably to all operators.
Consider the following query

```
SELECT *
FROM U
WHERE s1() AND s2();
```

and assume we wrap U, s1, and s2 in an eddy. Assume s1 takes 5 seconds to run
and s2 takes 10 seconds to run. Ideally, we'd pass all tuples to s1 before
passing them to s2.  The eddy will initially route half of the tuples to s1 and
half to s2. Eventually, both operators' input message queues will be saturated.
Because s1 runs twice as fast as s2, it will read a tuple from its input queue
twice as often. Thus, the eddy will deliver twice as many input tuples to s1.
In effect, a naive eddy can **estimates delays** and will end up favoring the
faster filter.

## Fast Eddy
Again consider the query above, but now assume both s1 and s2 take 5 seconds to
run. Also assume that s1 has a selectivity of 0.1 and s2 has a selectivity of
0.9. Ideally, we'd pass all tuples to s1 before s2. Because both operators take
the same amount of time to run, a naive eddy will end up passing an equal
number of tuples to s1 and s2. This is not good. A naive eddy can estimate
delays but it cannot **estimate selectivities**.

To estimate selectivities, we implement a fast eddy. A fast eddy performs
lottery scheduling among the operators. Each operator is given a ticket when it
is handed a tuple. Each operator is charged a ticket when it returns a tuple.
This way, the operators which hand back only a few tuples accumulate a lot of
tickets and are favored by the lottery scheduler.

## Dynamic Fluctuations
Imagine that speed of an operator varies over time. For example, an index join
that reads from an index over the network may begin fast but slow over time.
The fast eddy lottery scheduler weights all tickets equally, so it treats the
past behavior of an operator as importantly as it does the present behavior. To
overcome this, fast eddies can perform a windowed lottery scheduling in which
the scheduler only considers tickets accumulated during the last window.

## Reorderability of Plans
Recall from earlier that before we insert eddy operators into a query plan, we
first generate a fixed query plan restricted to index joins, hash ripple joins,
and block ripple joins. We make this restriction because not all join
algorithms work well (or work at all) with an eddy operator. To characterize
which join algorithms work well with an eddy, we introduce two pieces of
terminology: **synchronization barriers** and **moments of symmetry**.

Imagine a sort-merge join between two relations R and S. If reading from R is
really slow, we may spend a lot of time reading from R before we join anything
with a tuple from S. We refer to this phenomenon as a synchronization barrier.

Next, notice that some join operators are asymmetric. For example, nested loop
joins treat the outer and inner relations differently. While a nested loop join
is scanning over the inner relation, it could not swap the inner and out
relations. However, after a scan of the inner relation, it can (assuming it
records some small amount of bookkeeping). The moments during a join algorithm
in which the two relations can be swapped are known as moments of symmetry.

Eddy operators work best with join algorithms with as few synchronization
barriers and as many moments of symmetry as possible. We restrict ourselves to
index joins, hash ripple joins, and block ripple joins because they have few
synchronization barriers and lots of moments of symmetry.

<script src="https://cdnjs.cloudflare.com/ajax/libs/snap.svg/0.5.1/snap.svg.js"></script>
<script type="text/javascript">
  function main() {
    var s = Snap("#eddy1");
    var r = s.text(200, 150, "R");
    var eddy_ellipse = s.ellipse(200, 100, 50, 20);
    var eddy = s.text(200, 100, "Eddy");
    var select1_ellipse = s.ellipse(150, 50, 30, 15);
    var select1 = s.text(150, 50, "a");
    var select2_ellipse = s.ellipse(250, 50, 30, 15);
    var select2 = s.text(250, 50, "b");
    var path1 = s.path("M200 150 v-50 " +
                       "q-50 0, -50 -50 q50 0, 50 50 " +
                       "q50 0, 50 -50 q-50 0, -50 50 " +
                       "v-150");
    var path2 = s.path("M200 150 v-50 " +
                       "q50 0, 50 -50 q-50 0, -50 50 " +
                       "q-50 0, -50 -50 q50 0, 50 50 " +
                       "v-150");

    eddy_ellipse.attr({fill:"green", fillOpacity:0.5});
    select1_ellipse.attr({stroke:"black", fill:"transparent"});
    select2_ellipse.attr({stroke:"black", fill:"transparent"});
    path1.attr({fill:"transparent"});
    path2.attr({fill:"transparent"});

    var length1 = Snap.path.getTotalLength(path1);
    var length2 = Snap.path.getTotalLength(path2);
    var num_points = 25;
    var duration = 5000;

    var start_point = function(i) {
      if (i >= num_points) {
        return;
      }

      var point = s.circle(200, 150, 3);
      var path = undefined;
      if (i % 2 == 0) {
        point.attr({fill:"red"});
        path = path1;
      } else {
        point.attr({fill:"blue"});
        path = path2;
      }
      var f = function(point) {
        Snap.animate(0, length1, function(x) {
          var p = Snap.path.getPointAtLength(path, x);
          point.attr({cx:p.x, cy:p.y}, 0);
        }, 5000, mina.linear, function() {
          setTimeout(f, duration, point);
        });
      };
      f(point);

      setTimeout(start_point, duration / num_points, i + 1);
    };
    start_point(0);
  }

  window.onload = main;
</script>
<script type="text/javascript" async
  src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-MML-AM_CHTML">
</script>
