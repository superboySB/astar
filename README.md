# astar

[![PkgGoDev](https://pkg.go.dev/badge/github.com/fzipp/astar)](https://pkg.go.dev/github.com/fzipp/astar)
![Build Status](https://github.com/fzipp/astar/workflows/build/badge.svg)
[![Go Report Card](https://goreportcard.com/badge/github.com/fzipp/astar)](https://goreportcard.com/report/github.com/fzipp/astar)

Package astar implements the
[A* search algorithm](https://en.wikipedia.org/wiki/A*_search_algorithm)
for finding least-cost paths.

## Examples

In order to use the `astar.FindPath` function to find the least-cost path
between two nodes of a graph you need a graph data structure that implements
the `Neighbours` method to satisfy the `astar.Graph[Node]` interface and a
cost function. It is up to you how the graph is internally implemented.

### A maze

In this example the graph is represented by a slice of strings, each character
representing a cell of a floor plan. Graph nodes are cell positions
as `image.Point` values, with (0, 0) in the upper left corner. 
Spaces represent free cells available for walking, other characters like
`#` represent walls.
The `Neighbours` method returns the positions of the adjacent free cells
to the north, east, south, and west of a given position (diagonal movement
is not allowed in this example).

The cost function `nodeDist` simply calculates the Euclidean distance
between two cell positions.

```go
package main

import (
	"fmt"
	"image"
	"math"

	"github.com/superboySB/astar"
)

func main() {
	maze := graph{
		"###############",
		"#   # #     # #",
		"# ### ### ### #",
		"#   # # #   # #",
		"### # # # ### #",
		"# # #         #",
		"# # ### ### ###",
		"#   # # # #   #",
		"### # # # # ###",
		"# #       # # #",
		"# # ######### #",
		"#         #   #",
		"# ### # # ### #",
		"#   # # #     #",
		"###############",
	}
	start := image.Pt(1, 13) // Bottom left corner
	dest := image.Pt(13, 1)  // Top right corner

	// Find the shortest path
	path := astar.FindPath[image.Point](maze, start, dest, nodeDist, nodeDist)

	// Mark the path with dots before printing
	for _, p := range path {
		maze.put(p, '.')
	}
	maze.print()
}

// nodeDist is our cost function. We use points as nodes, so we
// calculate their Euclidean distance.
func nodeDist(p, q image.Point) float64 {
	d := q.Sub(p)
	return math.Sqrt(float64(d.X*d.X + d.Y*d.Y))
}

type graph []string

// Neighbours implements the astar.Graph[Node] interface (with Node = image.Point).
func (g graph) Neighbours(p image.Point) []image.Point {
	offsets := []image.Point{
		image.Pt(0, -1), // North
		image.Pt(1, 0),  // East
		image.Pt(0, 1),  // South
		image.Pt(-1, 0), // West
	}
	res := make([]image.Point, 0, 4)
	for _, off := range offsets {
		q := p.Add(off)
		if g.isFreeAt(q) {
			res = append(res, q)
		}
	}
	return res
}

func (g graph) isFreeAt(p image.Point) bool {
	return g.isInBounds(p) && g[p.Y][p.X] == ' '
}

func (g graph) isInBounds(p image.Point) bool {
	return (0 <= p.X && p.X < len(g[p.Y])) && (0 <= p.Y && p.Y < len(g))
}

func (g graph) put(p image.Point, c rune) {
	g[p.Y] = g[p.Y][:p.X] + string(c) + g[p.Y][p.X+1:]
}

func (g graph) print() {
	for _, row := range g {
		fmt.Println(row)
	}
}
```

Output:

```
###############
#   # #     #.#
# ### ### ###.#
#   # # #   #.#
### # # # ###.#
# # #  .......#
# # ###.### ###
#   # #.# #   #
### # #.# # ###
# #.....  # # #
# #.######### #
#...      #   #
#.### # # ### #
#.  # # #     #
###############
```

### 2D points as nodes

In this example the graph is represented by an adjacency list. Nodes are
2D points in Euclidean space as `image.Point` values. The `link` function
creates a bi-directed edge between a pair of nodes.

The cost function `nodeDist` calculates the Euclidean distance
between two points (nodes).

![Example graph with shortst path](doc/example1.png?raw=true)

```go
package main

import (
	"fmt"
	"image"
	"math"

	"github.com/fzipp/astar"
)

func main() { 
	// Create a graph with 2D points as nodes
	p1 := image.Pt(3, 1)
	p2 := image.Pt(1, 2)
	p3 := image.Pt(2, 4)
	p4 := image.Pt(4, 5)
	p5 := image.Pt(4, 3)
	p6 := image.Pt(5, 1)
	p7 := image.Pt(8, 4)
	p8 := image.Pt(8, 3)
	p9 := image.Pt(6, 3)
	g := newGraph[image.Point]().
		link(p1, p2).link(p1, p3).
		link(p2, p3).link(p2, p4).
		link(p3, p4).link(p3, p5).
		link(p4, p6).link(p4, p7).
		link(p5, p7).
		link(p6, p9).
		link(p7, p8).
		link(p8, p9)

	// Find the shortest path from p1 to p9
	p := astar.FindPath[image.Point](g, p1, p9, nodeDist, nodeDist)

	// Output the result
	if p == nil {
		fmt.Println("No path found.")
		return
	}
	for i, n := range p {
		fmt.Printf("%d: %s\n", i, n)
	}
}

// nodeDist is our cost function. We use points as nodes, so we
// calculate their Euclidean distance.
func nodeDist(p, q image.Point) float64 {
	d := q.Sub(p)
	return math.Sqrt(float64(d.X*d.X + d.Y*d.Y))
}

// graph is represented by an adjacency list.
type graph[Node comparable] map[Node][]Node

func newGraph[Node comparable]() graph[Node] {
	return make(map[Node][]Node)
}

// link creates a bi-directed edge between nodes a and b.
func (g graph[Node]) link(a, b Node) graph[Node] {
	g[a] = append(g[a], b)
	g[b] = append(g[b], a)
	return g
}

// Neighbours returns the neighbour nodes of node n in the graph.
func (g graph[Node]) Neighbours(n Node) []Node {
	return g[n]
}
```

Output:

```
0: (3,1)
1: (2,4)
2: (4,5)
3: (5,1)
4: (6,3)
```

## License

This project is free and open source software licensed under the
[BSD 3-Clause License](LICENSE).


## 讨论
用如下方法测试效果
```sh
go test -v
```
根据 astar.go 和 pqueue.go 文件的实现，我不建议在这里设计并行处理。A* 算法的核心是维护一个全局的优先队列，用于确定下一个要扩展的节点。由于这个过程需要精确的顺序来保证找到最短路径，它本质上是串行的。尽管有些独立计算可以并行执行，比如节点的启发式评估，但是优先队列的操作（如插入和删除）很难并行化，因为它们需要对数据结构进行精确的控制以保持堆的性质。如果你想要提高性能，可以考虑其他方法，如优化启发式函数或是使用其他策略来减少搜索空间。并行化 A* 算法可能会涉及复杂的同步机制，这可能会引入新的问题，而不是提高效率。