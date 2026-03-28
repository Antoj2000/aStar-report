## Introduction

The goal of this project was to build a modern C++ application capable of solving a pathfinding problem using the A* algorithm. In simple terms, the aim was to determine whether a computer could find the shortest path from point A to point B (or from S to G on a grid) while avoiding blocked cells.

The final implementation of this project developed through an iterative process. A weekly diary was kept to document the challenges, decisions, and progress made throughout the project.

---

## Initial Approach

The first step was creating a basic printed grid to visualise the problem space:

```cpp
int main() {
    vector<string> map = {
        ".....",
        ".....",
        ".....",
        ".....",
        "....."
    };
    for (int r = 0; r < map.size(); r++) {
        cout << map[r] << std::endl;
    }
    return 0;
}
```

Breaking no new ground here sure, but this helped me visualise the overall process. And also realise that strings would not be a viable solution!

My next immediate thought was directions. How would the computer know which way was up and down, left and right? If point is (0,0) on a graph, then one space right is (0,1). Left is (-1,0). But picturing this on a graph was a mistake as I found out soon enough. 

Following on my current logic, down would be (-1,0) and up would be (1,0). When I printed the co-ords of each position in my grid up and down were reversed. This is because in programming grids, row increases downward, and column to the right. So a down from point (1,1) on a grid would be (2,1). This was the first lesson of many during this project.

Now that I had this logic sorted out I created my first directions 2D array.

```cpp
int directions[4][2] = {
    {-1, 0},  // up
    { 1, 0},  // down
    { 0,-1},  // left
    { 0, 1}   // right
};
```
Then from here, when I had a good idea of the fundamentals of the grid and how these simple concepts would work, I moved onto an object-oriented structure, and finally a working A* search with closed lists, cost tracking, path reconstruction and tests. This workflow of building A* as smaller concepts that were understood one at a time suited me perfectly.

## Core Concepts

### Grid

The first component I completed was the Grid class, which I had coded in main first, as stated in the introduction. Before I could even begin to think about algorithms and pathfinding, I needed a way to represent free cells, blocked cells, and map dimensions. This made grid the natural starting point of the project.

Also as I previously mentioned, I deduced fairly quickly that strings would not be viable in the long term for this project. I quickly switched to `std::vector<std::vector<char>>> map_` as it gave me the ability to have individual editable cells.

I also was printing this initially using 

```cpp
for (int r = 0; r < map.size(); r++) {
    for (int c = 0; c < map[r].size(); c++) {
        std::cout << map[r][c];
        }
    std::cout << std::endl;
}
```
This index-based loop is more c style than c++, instead I switched to a range based for loop. 

```cpp
for (const auto& row : map_) {
    for (char cell : row) {
        std::cout << cell;
    }
    std::cout << '\n';
}
```
Range-based for loops are preferred over index-based loops when iterating over all elements of a container. Index-based loops introduce unnecessary complexity through manual bounds management, creating potential for off-by-one errors and requiring the reader to verify correctness of loop conditions. 

[Range based for loops](https://en.cppreference.com/w/cpp/language/range-for.html)

#### Purpose

The main role of the Grid class is to separate map storage from pathfinding logic. The grid is responsible for answering environmental questions such as

- Is the coordinate inside the map?
- Is a tile walkable?
- What symbol should be stored in a cell?

I defined these core questions my grid class needed to answer, and built those functions first. This gave me a stable foundation to work from. Path printing was added later, but the essential structure was already in place. 

So I got to work on the grid class, and declared my attributes and methods.

```cpp
class Grid {
public:
	Grid(int rows = 5, int cols = 5, char fill = '.');

	void SetCell(int r, int c, char value);
	char GetCell(int r, int c) const;

	bool InBounds(int r, int c) const;
	bool IsWalkable(int r, int c) const;

	int Rows() const;
	int Cols() const;

	void Print() const;
	void PrintPath(
		const std::vector<Node>& path,
		int startRow, int startCol,
		int goalRow, int goalCol
	) const;

private:
	std::vector<std::vector<char>> map_;
};
```
The Grid class is intentionally simple, but its design decisions matter because all pathfinding behaviour sits on top of it.

- SetCell, which lets me set the start and goal
- GetCell, which lets A* read whats in that cell
- InBounds, which is a bounds check
- IsWalkable, which checks if a cell is walkable or not
- Rows and Cols, which return the number of rows and cols
- Print and PrintPath whcih will print the starting grid and final grid respectively

The constructor allows the grid size and fill character to be chosen dynamically
```cpp
Grid(int rows = 5, int cols = 5, char fill = '.');
```
In GetCell, I treat out of bounds co-ordinates as blocked cells '#'. It means that I don't have to write in a wall around every grid, and instead these invalid positions can be treated as walls, which simplifies logic later on when I am expanding the search.
```cpp
char Grid::GetCell(int r, int c) const {
    if (!InBounds(r, c)) return '#'; 
    return map_[r][c];
}
```
When I had this code up and running along with tests, I asked ChatGPT if I could modernize it further. It suggested that instead of using map.size() like below 
```cpp
bool Grid::InBounds(int r, int c) const {
    return r >= 0 && r < (int)map_.size()
        && c >= 0 && c < (int)map_[0].size();
}
```
I should switch to using std::ssize to get a signed size directly. 
```cpp
bool Grid::InBounds(int r, int c) const {
    return r >= 0 && r < std::ssize(map_)
        && c >= 0 && c < std::ssize(map_[0]);
}
```
The reason for this is because `map.size()` returns an unsigned value, which can cause awkward comparisons when checking negative indices, whereas `std:ssize` returns a signed value and makes bound checking safer and more consistent.

#### Grid Tests

## Separating Concerns

Another important decision was made at this point which paralysed my progress for awhile. I needed to determine the overall architecture of the entire project. I knew that I would have an AStar class and a Node class but what should I put in which? 

Similar to the questions I asked myself at the start of the grid class I defined roles for each class.

- Grid is the environment
- Node is the search state
- AStar is the algorithm

The Grid class only stores map data and handles environment-related operations such as bounds, walkability, and printing. The Node struct stores the state of a visited or unvisited position in the search, including costs and parent information. The AStar class is responsible for movement rules, heuristic calculation, open and closed set handling, and path reconstruction.

This confusion came about whilst trying to assign the movement directions to a class. My final decision was that because directions would be an algorithm behavior, they belong to AStar.

## Node

The node struct represents an individual search state. It stores
- row and col 
- gCost, the known cost from the start 
- hCost, the heuristic estimate to the goal 
- parentRow and parentCol, which store where the node came from

```cpp
struct Node {
    int row{};
    int col{};

    int gCost{};
    int hCost{};

    int parentRow{ -1 };
    int parentCol{ -1 };

    Node() = default;
    Node(int row, int col, int gCost, int hCost, int parentRow = -1, int parentCol = -1);

    [[nodiscard]] int FCost() const {
        return gCost + hCost;
    }
};
```

Here the choice to use a struct rather than a fully fleged class was made because Node is essentially a compact data carrier. The only behavior it includes is FCost(), which returns the sum of the path cost so far and the heuristic estimate.

This is the main value used to prioritise nodes in A*.

Initially Node was a bare struct, and was being constructed like this. 

```cpp
Node startNode;
startNode.row = startRow;
startNode.col = startCol;
startNode.gCost = 0;
startNode.hCost = Heuristic(startRow, startCol, goalRow, goalCol);
startNode.parentRow = -1;
startNode.parentCol = -1;
```
This way of constructing was also being use to create the neighbor nodes, when expanding neighbors. As you can see it's not too pretty and very repetitive. So I asked ChatGPT how I could reduce repetition when constructing these nodes. It first suggested adding helper functions within AStar
```cpp
Node AStar::CreateStartNode(
    int startRow,
    int startCol,
    int goalRow,
    int goalCol
) const {
    return Node{
        startRow,
        startCol,
        0,
        Heuristic(startRow, startCol, goalRow, goalCol),
        -1,
        -1
    };
}

Node AStar::CreateNeighbourNode(
    int neighbourRow,
    int neighbourCol,
    int proposedGCost,
    int goalRow,
    int goalCol,
    const Node& current
) const {
    return Node{
        neighbourRow,
        neighbourCol,
        proposedGCost,
        Heuristic(neighbourRow, neighbourCol, goalRow, goalCol),
        current.row,
        current.col
    };
}
```
But this was not ideal, the whole point was to reduce the clutter in AStar. So I suggested moving Node construction into its own cpp class, which made the most sense. Up to this point Node had no cpp and was only a header file containing the struct. So I added the default and parameterized constructor:
```cpp
Node() = default;
Node(int row, int col, int gCost, int hCost, int parentRow = -1, int parentCol = -1);
```
A default constructor is needed as when I create a vector of Nodes `std::vector<Node>`, which you will see in my AStar class, requires it when allocating and resizing internal storage. Basically I need a blank node before I start filling in values.

And then in Node.cpp I added the intialiser method
```cpp
Node::Node(int row, int col, int gCost, int hCost, int parentRow, int parentCol)
    : row(row), col(col), 
      gCost(gCost), hCost(hCost), 
      parentRow(parentRow), parentCol(parentCol)
{}
```
This method does not include the parentRow and parentCols as this will be the same for every Node. And we can use:
```cpp
Node startNode(startRow, startCol, 0, Heuristic(startRow, startCol, goalRow, goalCol));
```
To construct the nodes each time now. A significant reduction in clutter and improvement in readability as nodes can be created in a single line rather than being declared and populated field by field.

## AStar

Now that I had a data structure, and a way to print my grid, it was time to start working on the AStar class.

The AStar class is where the main algorithm is implemented. First I defined what responsibilities AStar has.

It needs to: 
- Take a grid
- Take a start and goal position
- Explore valid neighbors
- Use a heuristic to guide the search
- Return a final path

So my initial methods were
- Heuristic
- FindPath
- ExpandNeighbours

I also defined my neighbors here

```cpp
namespace {
    constexpr std::array<std::pair<int, int>, 4> Directions{
        std::pair{-1, 0},  // up
        std::pair{1, 0},   // down
        std::pair{0, -1},  // left
        std::pair{0, 1}    // right
    };
}
```
A few changes here to note from my original directions array. As I coded I constantly asked ChatGPT if any improvements could be made, or if my code could be made safer.

We use `namespace` giving directions internal linkage, so it is invisible outside this cpp file. This is the modern alternative to static. Since Directions is an implementation detail of the A* algorithm, nothing outside this file should know it exists.

`constexpr` tells the compiler that Directions is a compile-time constant, so the value is known before the program runs. 

This means
- Zero runtime cost to initialise it
- The compiler can optimize loops over it aggressively
- Enforces nobody accidentally mutates it at runtime.

`std::pair<int, int>` is used because each direction is naturally a pair of two related integers `(rowOffset, colOffset)`. `std::pair` is a lightweight, standard way to group them without defining a whole struct. It also enables a clean structured binding in the loop `for (const auto& [rowOffset, colOffset] : Directions)` which is used in ExpandNeighbours.

So this approach is basically zero-cost at runtime whilst also being safer.

### Heuristic

A heuristic is an estimate of how close a given state is to a goal, in this case it will be used to guide AStar. I have two options here.

#### Manhattan

Manhattan heuristic is the sum of the horizontal and vertical distances between two points, start and goal. A nice way to visualise it is thinking of city blocks in America, hence "Manhattan".
The Manhattan distance between two points $(x_1, y_1)$ and $(x_2, y_2)$ is defined as:

$$d = |x_1 - x_2| + |y_1 - y_2|$$

This generalises to $n$ dimensions as:

$$d = \sum_{i=1}^{n} |p_i - q_i|$$

This would be suitable for 4-directional movement

#### Euclidean

The Euclidean heuristic is the straight-line distance between two points, start and goal.
The Euclidean distance between two points $(x_1, y_1)$ and $(x_2, y_2)$ is defined as:

$$d = \sqrt{(x_1 - x_2)^2 + (y_1 - y_2)^2}$$

This generalises to $n$ dimensions as:

$$d = \sqrt{\sum_{i=1}^{n} (p_i - q_i)^2}$$

This would be suitable for 8-directional or free movement.

### Heuristic Choice 

For my project the realistic choice was manhattan as I planned on using 4 directional movement, but if I wanted to expand to 8-directional or diagonal, I would just add 4 co-ordinate sets to my to my directional array, and replace the manhattan calculation with the euclidean.

So now my choice was made I defined my method. 

```cpp
int AStar::Heuristic(int row, int col, int goalRow, int goalCol) const {
    return std::abs(row - goalRow) + std::abs(col - goalCol);
}
```
Here we used the given co-ordinates along with the goal co-ordinates to calculate the hCost of the Node. We use `std::abs` to ensure the return is never negative.














