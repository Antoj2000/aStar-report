## Introduction

The goal of this project was to build a modern C++ application capable of solving a pathfinding problem using the A* algorithm. In simple terms, the aim was to determine whether a computer could find the shortest path from point A to point B (or from S to G on a grid) while avoiding blocked cells.

The final implementation of this project developed through an iterative process. A weekly diary was maintained to document the challenges, decisions, and progress made throughout the project. :contentReference[oaicite:0]{index=0}

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


