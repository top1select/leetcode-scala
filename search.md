Searching
===

# Prerequisite
[Week 6 of Functional Programming Principles in Scala](https://www.coursera.org/learn/progfun1/home/week/6)
 talks about <tt>for</tt> expression, <tt>map</tt> and <tt>flatMap</tt>.
and [Week 2 of Functional Program Design in Scala](https://www.coursera.org/learn/progfun2/home/week/2),
which talks about lazy streams.

# A Bit of Foreword
So, depth first search (DFS) and breadth first search (BFS). The difference between them is how the next state
is selected from currently known candidates. If we use a stack, the next state to explore is the most recently
pushed state. It's depth first search. If we use a queue, the next state is the earliest state available,
which is BFS. If we instead, use a priority queue and organize the states according to a certain metric, we
wind up getting <tt>A</tt>-search or <tt>A-star</tt> search.

## Representing Queues and Stacks
In functional world, stacks are naturally represented with lists. <tt>push(x, stack)</tt> results in <tt>x::stack</tt>.
For queues, we can use 2 lists to achieve constant amortized time for enqueue and dequeue.

[LeetCode 232](https://leetcode.com/problems/implement-queue-using-stacks/description/): Implementing queue using 
stacks. The idea is to use a pair of lists: <tt>(l1, l2)</tt>. An enqueue operation conses an element to one list. At
dequeue, if the other list is not empty, we just return the head of the other list. If it's empty, we pop all elements
from l1 to l2 and dequeue:

```scala
class MyQueue() {
  var queue = (List[Int](), List[Int]())

  /** Push element x to the back of queue. */
  def push(x: Int): Unit =
    queue = (x::queue._1, queue._2)

  /** Removes the element from in front of queue and returns that element. */
  def pop(): Int = {
    var elem: Int = 0
    queue match {
      case (_, y::ys) =>
        elem = y
        queue = (queue._1, ys)
      case (xs, Nil) =>
        queue = (Nil, xs.reverse)
        elem = pop()
    }
    elem
  }

  /** Get the front element. */
  def peek(): Int =
    queue match {
      case (_, y::ys) => y
      case (xs, Nil) =>
        queue = (Nil, xs.reverse)
        queue._2.head
    }

  /** Returns whether the queue is empty. */
  def empty(): Boolean = queue == (Nil, Nil)
}
```

Note that this implementation is not idiomatic functional programming, but it's required by the prototype of
LeetCode for this question.

Scala provides <tt>scala.collections.immutable.Queue</tt>, <tt>scala.collections.mutable.Queue</tt> and
<tt>scala.collections.mutable.PriorityQueue</tt>.

# For Comprehension
## Ambiguous Coordinates
[LeetCode 816](https://leetcode.com/problems/ambiguous-coordinates/description/): We have a 2D coordinate
"(1, 3)" and then we remove all commas, spaces and decimal points. Given such string, print all the possible
coordinates subject to the following rule that original representation never had extraneous zeroes, so we never
started with numbers like "00", "0.0", "0.00", "1.0", "001", "00.01", or any other number that can be represented
with less digits.  Also, a decimal point within a number never occurs without at least one digit occuring before
it, so we never started with numbers like ".1".

This is an easy question. You'll just need to review the syntax of <tt>for</tt> comprehension and apply appropriate
filters:

```scala
def ambiguousCoordinates(S: String): List[String] = {
  val str = S.init.tail
  def allZero(s: String): Boolean = s.length > 1 && s.forall(_ == '0')
  def leadingZero(s: String): Boolean = s.length > 1 && s.startsWith("0")
  def numbers(s: String): List[String] =
    for {
      i <- (1 to s.length).toList
      (int, float) = s.splitAt(i)
      if !leadingZero(int) && !float.endsWith("0")
    } yield int + (if (float.isEmpty) "" else "." + float)
  for {
    i <- (1 until str.length).toList
    (part1, part2) = str.splitAt(i)
    if !allZero(part1) && !allZero(part2)
    n1 <- numbers(part1)
    n2 <- numbers(part2)
  } yield s"($n1, $n2)"
}
```

# Depth First Search
## Permutations
[LeetCode 46](https://leetcode.com/problems/permutations/description/): Generate
permutations of an array that contains no duplicates.

We can think recursively. Given an array A, if we take any element c out of this
array, we have an array A / c. Then we permute A / c. For each of the permutation,
we prepend c to it. This way we get the permutations of A. The code:

```scala
def permuteUnique(nums: Array[Int]): List[List[Int]] =
  nums match {
    case Array() | Array(_) => List(nums.toList)
    case xs => for {
      n <- xs.toList
      rest <- permuteUnique(nums diff Array(n))
    } yield n::rest
  }
```

[LeetCode 47](https://leetcode.com/problems/permutations-ii/description/): Generate
permutations of an array that contains duplicates.

If the array contains duplicates, all we need to pay attention to is that we cannot
blindly take an element c. We'll need to select c from the unique elements of A
to avoid duplications. The previous code only needs a minor change -- 
<tt>xs.toList --> xs.distinct.toList </tt>:

```scala
def permuteUnique(nums: Array[Int]): List[List[Int]] =
  nums match {
    case Array() | Array(_) => List(nums.toList)
    case xs => for {
      n <- xs.distinct.toList
      rest <- permuteUnique(nums diff Array(n))
    } yield n::rest
  }
```

If we fold up the <tt>for</tt> expression, the code can actually be shorten to:

```scala
def permuteUnique(nums: Array[Int]): List[List[Int]] =
  if (nums.length <= 1) List(nums.toList)
  else nums.distinct.toList.flatMap(
    n => permuteUnique(nums diff Array(n)).map(n::_))
```
But it sacrifices readability and is not recommended.

## Target Sum
[LeetCode 494](https://leetcode.com/problems/target-sum/description/): Given an
array of positive numbers, you can add either <tt>+</tt> or <tt>-</tt> in front
of each number. Given a target sum S, how many ways to assign <tt>+ or -</tt> that
makes the sum equal to S.

Given that the maximum length of the array is 20, we can enumerate through all
the possibilities. The search could be pruned with a minor optimization:
if we see that we cannot make it to the target sum even if we add all <tt>+</tt>
or <tt>-</tt> sign to the rest of numbers, we can abort our search path. The
sum of "rest of the numbers" can be represented using a postfix array which can
be obtained by <tt>nums.scanRight(0)(_+_).init</tt>.

```scala
def findTargetSumWays(nums: Array[Int], S: Int): Int = {
  val postfix = nums.scanRight(0)(_+_).init
  def constructSum(i: Int, sum: Int): Int =
    if (i == nums.length && sum == S) 1
    else if (i == nums.length || sum + postfix(i) < S || sum - postfix(i) > S) 0
    else constructSum(i + 1, sum + nums(i)) + constructSum(i + 1, sum - nums(i))
  constructSum(0, 0)
}
```

## Word Pattern II
[LeetCode 291](https://leetcode.com/problems/word-pattern-ii/description/)

```scala
def wordPatternMatch(pattern: String, str: String): Boolean = {
  def isPatternMatch(pattern: List[Char], str: List[Char],
                     context: Map[Char, List[Char]],
                     used: Set[List[Char]]): Boolean =
    (pattern, str) match {
      case (Nil, Nil) => true
      case (_, Nil) | (Nil, _) => false
      case (p::ps, ss) if context.contains(p) =>
        (ss.take(context(p).length) == context(p)) &&
          isPatternMatch(ps, ss.drop(context(p).length), context, used)
      case (p::ps, ss) =>
        ss.inits.exists(i => !used.contains(i) &&
          isPatternMatch(ps, ss.drop(i.length), context + (p -> i), used + i))
    }
  isPatternMatch(pattern.toList, str.toList, Map(), Set(List()))
}
```

## All Paths from Source to Target
[LeetCode 797](https://leetcode.com/problems/all-paths-from-source-to-target/description/):
Given a directed acyclic graph, find all paths from source to target.

This problem can be easily solved using lazy DFS. We construct all paths from source
to other nodes then filter through the paths to find the paths that reaches target:

```scala
def allPathsSourceTarget(graph: Array[Array[Int]]): List[List[Int]] = {
  def findPath(path: List[Int]): Stream[List[Int]] =
    path #:: graph(path.head).toStream.flatMap(h => findPath(h :: path))
  findPath(List(0)).filter(_.head == graph.length - 1).map(_.reverse).toList
}
```

## isAdditiveNumber
[LeetCode 306](https://leetcode.com/problems/additive-number/description/):
Tell if a string can be divided into a series of additive numbers.
Additive numbers are a list numbers whose length is at least 3. And each number
except the first 2 is a sum of previous 2 numbers.

I think the most interesting part about this problem is how we generate a series
of numbers. One way of doing this is:

```scala
def chop(s: String): Stream[Stream[Int]] = {
  if (s.length == 0) Stream(Stream.empty)
  else if (s.length == 1) Stream(Stream(s.toInt))
  else for {
    i <- (1 to s.length).toStream
    (ss, rest) <- s.splitAt(i)
    rss <- chop(rest)
  } yield ss.toInt +: rss
}
```
Then we can have a predicate <tt>isAdditive</tt> to test if the sequence is
additive sequence:
```scala
def isAdditive(nums: Stream[Long]): Boolean =
  nums.take(3).length >= 3 && nums.sliding(3).forall {
    case Stream(a, b, c) => c == a + b
  }
```

Now the whole program is trivially written as:

```scala
def isAdditiveNumber(num: String): Boolean =
  chop(num).exists(isAdditive)
```

But this solution will result in TLE in LeetCode. So what improvements could
we do? One improvement is that the sequence should be monotonically increasing
from the 3rd element since they are non-negative numbers, which implies that
the length of number <tt>num[i]</tt> is at least the length of <tt>num[i-1]</tt>
for <tt>i >= 3</tt>. The solution would look as follows. Note that I replaced
<tt>Int</tt> with <tt>Long</tt> to address potential overflow with 32-bit
integers.

```scala
def isAdditiveNumber(num: String): Boolean = {
  def chop(d: Int, minLength: Long, s: String): Stream[Stream[Long]] =
    if (s.length == 0) Stream(Stream.empty)
    else if (s.length == 1) Stream(Stream(s.toLong))
    else {
      val start = if (d < 2) 1l else minLength
      for {
        i <- (start to (s.length.toLong min 10l)).toStream
        (ss, rest) = s.splitAt(i.toInt)
        if !(ss.length > 1 && ss.startsWith("0"))
        rss <- chop(d + 1, ss.length.toLong, rest)
      } yield ss.toLong +: rss
    }
  def isAdditive(nums: Stream[Long]): Boolean =
    nums.length >= 3 && nums.sliding(3).forall {
      case Stream(a, b, c) => c == a + b
    }
  chop(0, 1, num).exists(isAdditive)
}
```

## Zuma Game
[LeetCode 488](https://leetcode.com/problems/zuma-game/description/): 
Think about Zuma Game. You have a row of balls on the table, colored red(R),
yellow(Y), blue(B), green(G), and white(W). You also have several balls in your hand.

Each time, you may choose a ball in your hand, and insert it into the row (including
the leftmost place and rightmost place). Then, if there is a group of 3 or more balls
in the same color touching, remove these balls. Keep doing this until no more balls
can be removed.

Find the minimal balls you have to insert to remove all the balls on the table. If
you cannot remove all the balls, output -1. 

This problem requires coding some utility functions. The first one is
<tt>insertionPoints</tt>, which finds all the appropriate points to insert current
ball. We browse through the balls and insert it near another ball with the same
color in the board. Or we default insert it to position 0:

```scala
def insertionPoints(board: String, ball: Char): List[Int] =
  0::board.indices.filter(i => board(i) == ball).toList
def insertBall(ball: Char, board: String): List[String] =
  for {
    p <- insertionPoints(board, ball)
    (left, right) = board.splitAt(p)
  } yield erase(left.reverse, right, ball)
```

Then <tt>insertBall</tt> is implemented by inserting the ball and simulating the
cascaded erasing effect. The erasing process is performed by looking at current ball and
its surroundings, if we find 3 or more consecutive balls, we perform the erasing. Then
we take a look at the surrounding again and recursively apply this process. 

```
    v insert next W here
GGWW GYY
GGWWWGYY now we erase all 3 Ws
 | we can cascade the process by assuming this G is inserted, and thus
 v it triggers another round of erasing
GG   GYY
YY <- no more erase to perform

```
The erasing code is as follow:
```scala
  def erase(left: String, right: String, ball: Char): String = {
    val newLeft = left.dropWhile(_ == ball)
    val newRight = right.dropWhile(_ == ball)
    if (left.length - newLeft.length + right.length - newRight.length < 2)
      left.reverse + ball + right
    else if (newLeft.nonEmpty && newRight.nonEmpty)
      erase(newLeft.drop(1), newRight, newLeft.head)
    else newLeft.reverse + newRight
  }
```

Now the whole code is just trivial enumeration through all the possibilities and
find the path that requires minimum move.

```scala
def findMinStep(board: String, hand: String): Int = {
  val NOT_FOUND = hand.length + 1
  def insertionPoints(board: String, ball: Char): List[Int] =
    0::board.indices.filter(i => board(i) == ball).toList

  def insertBall(ball: Char, board: String): List[String] =
    for {
      p <- insertionPoints(board, ball)
      (left, right) = board.splitAt(p)
    } yield erase(left.reverse, right, ball)

  def erase(left: String, right: String, ball: Char): String = {
    val newLeft = left.dropWhile(_ == ball)
    val newRight = right.dropWhile(_ == ball)
    if (left.length - newLeft.length + right.length - newRight.length < 2)
      left.reverse + ball + right
    else if (newLeft.nonEmpty && newRight.nonEmpty)
      erase(newLeft.drop(1), newRight, newLeft.head)
    else newLeft.reverse + newRight
  }

  def findMin(board: String, hand: String, steps: Int): Int =
    if (board.isEmpty) steps
    else if (hand.isEmpty) NOT_FOUND
    else {
      (for {
        ball <- hand.distinct
        rest <- insertBall(ball, board)
      } yield findMin(rest, hand diff ball.toString, steps + 1)).min
    }

  val m = findMin(board, hand, 0)
  if (m == NOT_FOUND) -1 else m
}
```

## Alien Dictionary
[LeetCode 269](https://leetcode.com/problems/alien-dictionary/description/)

This idea is to construct relative order between chars and use topological sort
to output the order.

For each 2 words, say <tt>aaabcd, aaacd</tt>, we ignore their common prefixes and
find the pair of letters that differ, say <tt>b in aaabcd</tt> and <tt>c in aaacd</tt>.
Then we have <tt>b</tt> comes before <tt>c</tt>. 

There are some preprocessing to this solution though. We'll add an additional node '1'
since char '1' does not appear in any string and connect it to every char that occurs
so that we have a single source node for topological sort. The topologic sort function
looks like:

```scala
def topoSort(node: Char, graph: Map[Char, List[Char]], visited: Set[Char],
             result: List[Char], preds: Set[Char]): T =
  graph(node).foldLeft[T](Some((result, visited))) {
    case (Some((r, vs)), n) if preds.contains(n) => None
    case (Some((r, vs)), n) if !vs.contains(n) =>
      topoSort(n, graph, vs + n, r, preds + n)
    case (t, _) => t
  }.map(r => (node::r._1, r._2))
```

It is based on the assumption that <tt>graph</tt> is a map with default value <tt>Nil</tt>.
Note that the topoSort function returns <tt>Option[(List[Char], Set[Char])]</tt> since
if we encountered a back edge that forms a circle, we just give up and return None.

The code:

```scala
def alienOrder(words: Array[String]): String = {
  type T = Option[(List[Char], Set[Char])]
  def topoSort(node: Char, graph: Map[Char, List[Char]], visited: Set[Char],
               result: List[Char], preds: Set[Char]): T =
    graph(node).foldLeft[T](Some((result, visited))) {
      case (Some((r, vs)), n) if preds.contains(n) => None
      case (Some((r, vs)), n) if !vs.contains(n) =>
        topoSort(n, graph, vs + n, r, preds + n)
      case (t, _) => t
    }.map(r => (node::r._1, r._2))
  val graph = words.foldLeft(List[String]()) {
    // This part is nasty since LeetCode has test case: ["z", "z"], for which we should
    // return "z" though I feel such case should be invalid and return "".
    case (x::xs, w) if w.equals(x) => x::xs
    case (xs, w) => w::xs
  }.reverse.sliding(2).foldLeft[Map[Char, List[Char]]](
    Map('1' -> words.flatten.distinct.toList).withDefaultValue(Nil)) {
    case (map, List(w1, w2)) =>
      w1.zip(w2).dropWhile(x => x._1 == x._2).headOption match {
        case None => map
        case Some((x, y)) => map.updated(x, y::map(x))
      }
    case (map, List(w)) => map
  }
  topoSort('1', graph, Set(), Nil, Set())
    .map(r => r._1.tail.mkString).getOrElse("")
}
```

## Redundant Connection II
[LeetCode 685](https://leetcode.com/problems/redundant-connection-ii/description/)
In this problem, a rooted tree is a directed graph such that, there is exactly one node (the root)
for which all other nodes are descendants of this node, plus every node has exactly one parent,
except for the root node which has no parents.

The given input is a directed graph that started as a rooted tree with N nodes (with distinct values
1, 2, ..., N), with one additional directed edge added. The added edge has two different vertices
chosen from 1 to N, and was not an edge that already existed.

The resulting graph is given as a 2D-array of edges. Each element of edges is a pair [u, v] that
represents a directed edge connecting nodes u and v, where u is a parent of child v.

Return an edge that can be removed so that the resulting graph is a rooted tree of N nodes. If
there are multiple answers, return the answer that occurs last in the given 2D-array. 

We just need to enumerate through all the edges in reverse order, remove it from the graph and
perform a DFS on the graph to see if it's still connected. The first edge that satisfy such 
condition is the redundant edge.

Note that we'll need to find the root of the edges. After removing an edge, the only node
with in-degree 0 is the root. If such node does not exist, this edge is not the redundant edge.

```scala
def findRedundantDirectedConnection(edges: Array[Array[Int]]): Array[Int] = {
  val count = edges.flatten.toSet.size
  def stillConnected(i: (Int, Int), g: Map[Int, Set[Int]]): Boolean = {
    def dfs(n: Int, visited: Set[Int]): Set[Int] =
      g(n).foldLeft(visited) {
        case (v, x) if !v.contains(x) => dfs(x, v + x)
        case (v, _) => v
      }
    val root = g.keySet -- g.values.flatten
    root.size == 1 && dfs(root.head, root).size == count
  }
  val graph = edges.foldLeft(Map[Int, Set[Int]]().withDefaultValue(Set.empty)) {
    case (map, Array(s, e)) => map.updated(s, map(s) + e)
  }
  edges.reverse.find {
    case Array(s, e) => stillConnected((s, e), graph.updated(s, graph(s) - e))
  }.get
}
```

# Breadth First Search
Most of the time, for enumeration purposes, you can choose between breadth first search
and depth first search freely. However, breadth first search has one nice property:
In a graph, if the edge weights are just 1, then the first found path to the target is
the shortest path. So in questions like finding the shortest solution, BFS can be very
desirable.

## Number of Islands
[LeetCode 200](https://leetcode.com/problems/number-of-islands/description/): Given a
grid of 0s and 1s, count the number of islands. An island is defined by a region that
consists of all 1s and surrounded by 0s or edges.

This is an example that a little bit impurity can help. We use a <tt>visited</tt> array
to store the islands that we have touched. Then for each 1 we encountered, we do a 
bfs from it and mark all path we've touched as visited. We let bfs return 1 and sum up
the number of times bfs has executed.

```scala
def numIslands(grid: Array[Array[Char]]): Int = {
  if (grid.isEmpty) 0
  else {
    import scala.collection.immutable.Queue
    val (rows, cols) = (grid.length, grid(0).length)
    val visited = Array.fill(rows, cols)(false)

    def bfs(queue: Queue[(Int, Int)]): Int = {
      if (queue.isEmpty) 1
      else {
        val ((row, col), q) = queue.dequeue
        val candidates = List((row - 1, col), (row + 1, col),
          (row, col - 1), (row, col + 1)).filter {
          case(r, c) =>
            r >= 0 && r < rows && c >= 0 && c < cols &&
              grid(r)(c) == '1' && !visited(r)(c)
        }
        for ((r, c) <- candidates) visited(r)(c) = true
        bfs(q ++ candidates)
      }
    }

    (for {
      r <- grid.indices
      c <- grid(0).indices
      if grid(r)(c) == '1' && !visited(r)(c)
    } yield bfs(Queue((r, c)))).sum
  }
}

```

## Walls and Gates
[LeetCode 286](https://leetcode.com/problems/walls-and-gates/description/)

The problem setting itself is impure in nature so we prefer in-place update during
BFS. For each 0, we do a BFS and updates all cells that could be reached. Note that
we could stop exploring if the cell is reachable from any previous explored gates and
assigned a distance shorter than our current path length + 1:

```scala
def wallsAndGates(rooms: Array[Array[Int]]): Unit = {
  import scala.collection.immutable.Queue
  def bfs(queue: Queue[((Int, Int), Int)]): Unit = {
    if (queue.nonEmpty) {
      val (((row, col), steps), q) = queue.dequeue
      val neighbors = List((row - 1, col), (row + 1, col), (row, col - 1),
        (row, col + 1)).filter {
        case ((r, c)) => r < rooms.length && r >= 0 && c < rooms(0).length &&
          c >= 0 && rooms(r)(c) > steps + 1
      }
      for ((r, c) <- neighbors) rooms(r)(c) = steps + 1
      bfs(q ++ neighbors.map(p => (p, steps + 1)))
    }
  }
  for (r <- rooms.indices; c <- rooms(0).indices; if rooms(r)(c) == 0)
    bfs(Queue(((r, c), 0)))
}

```

## Bus Routes
[LeetCode 815](https://leetcode.com/problems/bus-routes/description/):
We have a list of bus routes. Each routes[i] is a bus route that the i-th bus repeats forever.
For example if routes[0] = [1, 5, 7], this means that the first bus (0-th indexed) travels in
the sequence 1->5->7->1->5->7->1->... forever.

We start at bus stop S (initially not on a bus), and we want to go to bus stop T. Travelling by
buses only, what is the least number of buses we must take to reach our destination?
Return -1 if it is not possible.

For this question, if we think of individual stop as node, it may not be very obvious what their
relationship should represent. If we instead, use route as node, then we could say that
<tt>node[i][j] = 1</tt> iff we can switch from route i to route j. So our starting states are
those routes that contain our starting node. The ending states are those routes that contain
the destination.

```scala
def numBusesToDestination(routes: Array[Array[Int]], S: Int, T: Int): Int = {
  import scala.collection.immutable.Queue
  val NOT_CONNECTED = -1
  val routesSet = routes.map(_.toSet)
  val routesDist = Array.tabulate(routes.length, routes.length) {
    (r1, r2) =>
      if (r1 != r2 && routesSet(r1).intersect(routesSet(r2)).nonEmpty) 1
      else NOT_CONNECTED
  }
  val startRoutes = routesSet.indices.filter(i => routesSet(i).contains(S))
  val endRoutes = routesSet.indices.filter(i => routesSet(i).contains(T)).toSet
  def minDist(queue: Queue[(Int, Int)], visited: Set[Int]): Int = {
    if (queue.isEmpty) NOT_CONNECTED
    else {
      val ((route, dist), q) = queue.dequeue
      if (endRoutes.contains(route)) dist
      else {
        val neighbors = routesDist(route).indices.filter(
          i => routesDist(route)(i) == 1 && !visited.contains(i))
        minDist(q ++ neighbors.map(n => (n, dist + 1)), visited ++ neighbors)
      }
    }
  }
  if (S == T) 0
  else startRoutes.map(s => minDist(Queue((s, 1)), Set(s))).min
}
```

