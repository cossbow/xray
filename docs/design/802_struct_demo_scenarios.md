# 802 - Struct 设计场景验证 Demo 集

基于 801 文档设计（struct + Ptr\<T\> + Arena\<T\> + ArraySlice + [N]T），
撰写一系列真实场景的 xray 伪代码 demo，用于验证：

1. **代码书写体验** — 是否直观、优雅、易读
2. **性能保证** — 是否能达到接近 C 的性能
3. **设计完备性** — 是否覆盖所有常见高性能场景

---

## Demo 1: 游戏 — ECS 粒子系统

**场景**：每帧更新 10 万个粒子的位置、速度、生命周期。要求连续内存、缓存友好。

```xray
struct Vec3 { x: float; y: float; z: float }

struct Particle {
    pos: Vec3
    vel: Vec3
    life: float
    mass: float
}

// 批量更新所有粒子（连续内存遍历，缓存友好）
fn update_particles(particles: ref Array<Particle>, dt: float, gravity: in Vec3) {
    for i in 0..particles.length() {
        if particles[i].life <= 0.0 { continue }

        // 应用重力
        particles[i].vel.x += gravity.x * dt
        particles[i].vel.y += gravity.y * dt
        particles[i].vel.z += gravity.z * dt

        // 更新位置
        particles[i].pos.x += particles[i].vel.x * dt
        particles[i].pos.y += particles[i].vel.y * dt
        particles[i].pos.z += particles[i].vel.z * dt

        // 衰减生命
        particles[i].life -= dt
    }
}

// 发射新粒子
fn emit(particles: ref Array<Particle>, pos: in Vec3, count: int) {
    for i in 0..count {
        let angle = math.random() * 6.28
        let speed = math.random() * 10.0
        particles.push(Particle {
            pos: pos,
            vel: Vec3 { x: math.cos(angle) * speed, y: speed * 2.0, z: math.sin(angle) * speed },
            life: 2.0 + math.random(),
            mass: 1.0
        })
    }
}

// 游戏主循环
let particles = Array<Particle>(100000)
let gravity = Vec3 { x: 0.0, y: -9.8, z: 0.0 }

for frame in 0..1000 {
    let dt = 0.016  // 60fps
    emit(ref particles, Vec3 { x: 0.0, y: 10.0, z: 0.0 }, 100)
    update_particles(ref particles, dt, gravity)
}
```

**性能分析**：
- `Array<Particle>` 连续内存，每个 Particle = 64B（8 个 float），缓存行友好
- `particles[i].vel.x` 编译为 `base + i*64 + 24`，一次偏移计算
- `ref Array<Particle>` 零拷贝传递
- `in Vec3` 只读引用，零拷贝
- AOT 生成的 C 代码和手写 C 完全一致

**AOT 映射**：
```c
typedef struct { double x, y, z; } Vec3;
typedef struct { Vec3 pos, vel; double life, mass; } Particle;

void update_particles(XrtArray *particles, double dt, const Vec3 *gravity) {
    Particle *data = (Particle*)particles->data;
    for (int i = 0; i < particles->length; i++) {
        if (data[i].life <= 0.0) continue;
        data[i].vel.x += gravity->x * dt;
        // ...
    }
}
```

---

## Demo 2: 游戏 — 2D AABB 碰撞检测

**场景**：检测大量游戏对象之间的碰撞（轴对齐包围盒）。

```xray
struct Vec2 { x: float; y: float }

struct AABB {
    min: Vec2
    max: Vec2
}

struct Entity {
    id: int
    bounds: AABB
    vel: Vec2
}

// AABB 碰撞检测（纯值计算，零分配）
fn aabb_overlaps(a: in AABB, b: in AABB): bool {
    return a.min.x <= b.max.x && a.max.x >= b.min.x &&
           a.min.y <= b.max.y && a.max.y >= b.min.y
}

// 移动实体并更新包围盒
fn move_entity(e: ref Entity, dt: float) {
    e.bounds.min.x += e.vel.x * dt
    e.bounds.min.y += e.vel.y * dt
    e.bounds.max.x += e.vel.x * dt
    e.bounds.max.y += e.vel.y * dt
}

// 暴力碰撞检测 O(n²)
fn detect_collisions(entities: in Array<Entity>): Array<[2]int> {
    let results = Array<[2]int>()
    for i in 0..entities.length() {
        for j in (i+1)..entities.length() {
            if aabb_overlaps(entities[i].bounds, entities[j].bounds) {
                results.push([entities[i].id, entities[j].id])
            }
        }
    }
    return results
}

// 宽相：网格空间划分
struct GridCell {
    entity_ids: [64]int    // 每格最多 64 个实体
    count: int
}

struct SpatialGrid {
    cells: [32 * 32]GridCell    // 32x32 网格
    cell_size: float
}

fn grid_insert(grid: ref SpatialGrid, entity: in Entity) {
    let cx = int(entity.bounds.min.x / grid.cell_size)
    let cy = int(entity.bounds.min.y / grid.cell_size)
    if cx >= 0 && cx < 32 && cy >= 0 && cy < 32 {
        let idx = cy * 32 + cx
        let c = grid.cells[idx].count
        if c < 64 {
            grid.cells[idx].entity_ids[c] = entity.id
            grid.cells[idx].count = c + 1
        }
    }
}

let entities = Array<Entity>(1000)
// ... 初始化 entities
let collisions = detect_collisions(entities)
print("collisions: ", collisions.length())
```

**性能分析**：
- `aabb_overlaps` 纯值计算，`in AABB` 零拷贝，编译器可内联
- `SpatialGrid` 使用 `[32*32]GridCell` 定长数组，全部栈上（AOT）
- `GridCell` 的 `[64]int` 也是内联值，无堆分配
- 整个空间划分数据结构零堆分配

---

## Demo 3: 科学计算 — 4x4 矩阵运算

**场景**：3D 图形变换矩阵运算，要求零分配。

```xray
struct Mat4 { data: [16]float }
struct Vec4 { x: float; y: float; z: float; w: float }

// 矩阵乘法（纯计算，零分配）
fn mat4_mul(a: in Mat4, b: in Mat4): Mat4 {
    let result = Mat4{}
    for i in 0..4 {
        for j in 0..4 {
            let sum = 0.0
            for k in 0..4 {
                sum += a.data[i * 4 + k] * b.data[k * 4 + j]
            }
            result.data[i * 4 + j] = sum
        }
    }
    return result
}

// 矩阵 × 向量
fn mat4_transform(m: in Mat4, v: in Vec4): Vec4 {
    return Vec4 {
        x: m.data[0]*v.x + m.data[1]*v.y + m.data[2]*v.z + m.data[3]*v.w,
        y: m.data[4]*v.x + m.data[5]*v.y + m.data[6]*v.z + m.data[7]*v.w,
        z: m.data[8]*v.x + m.data[9]*v.y + m.data[10]*v.z + m.data[11]*v.w,
        w: m.data[12]*v.x + m.data[13]*v.y + m.data[14]*v.z + m.data[15]*v.w
    }
}

// 构建透视投影矩阵
fn mat4_perspective(fov: float, aspect: float, near: float, far: float): Mat4 {
    let f = 1.0 / math.tan(fov / 2.0)
    let range_inv = 1.0 / (near - far)
    let m = Mat4{}
    m.data[0] = f / aspect
    m.data[5] = f
    m.data[10] = (near + far) * range_inv
    m.data[11] = 2.0 * near * far * range_inv
    m.data[14] = -1.0
    return m
}

// 构建平移矩阵
fn mat4_translate(x: float, y: float, z: float): Mat4 {
    let m = Mat4{}
    m.data[0] = 1.0; m.data[5] = 1.0; m.data[10] = 1.0; m.data[15] = 1.0
    m.data[3] = x; m.data[7] = y; m.data[11] = z
    return m
}

// 批量变换顶点
fn transform_vertices(verts: ref Array<Vec4>, mvp: in Mat4) {
    for i in 0..verts.length() {
        verts[i] = mat4_transform(mvp, verts[i])
    }
}

// 使用
let proj = mat4_perspective(1.047, 1.78, 0.1, 100.0)
let view = mat4_translate(0.0, 0.0, -5.0)
let mvp = mat4_mul(proj, view)

let vertices = Array<Vec4>(10000)
// ... 初始化顶点
transform_vertices(ref vertices, mvp)
```

**性能分析**：
- `Mat4` = 128B（16 个 float），`Vec4` = 32B
- `mat4_mul` 返回值类型，AOT 用 RVO 优化，零堆分配
- `in Mat4` 传 `const Mat4*`，零拷贝
- `mat4_transform` 返回 Vec4（32B），小到可以值返回
- `transform_vertices` 直接在 `Array<Vec4>` 连续内存上操作

---

## Demo 4: 科学计算 — N-body 重力模拟

**场景**：模拟 N 个天体之间的引力相互作用。

```xray
struct Vec3 { x: float; y: float; z: float }

struct Body {
    pos: Vec3
    vel: Vec3
    mass: float
    _pad: float       // 对齐到 64B
}

fn vec3_sub(a: in Vec3, b: in Vec3): Vec3 {
    return Vec3 { x: a.x - b.x, y: a.y - b.y, z: a.z - b.z }
}

fn vec3_length_sq(v: in Vec3): float {
    return v.x*v.x + v.y*v.y + v.z*v.z
}

fn vec3_scale(v: in Vec3, s: float): Vec3 {
    return Vec3 { x: v.x * s, y: v.y * s, z: v.z * s }
}

// 计算所有天体之间的引力并更新速度
fn compute_forces(bodies: ref Array<Body>, dt: float) {
    let G = 6.674e-11
    let n = bodies.length()

    for i in 0..n {
        for j in (i+1)..n {
            let diff = vec3_sub(bodies[j].pos, bodies[i].pos)
            let dist_sq = vec3_length_sq(diff) + 1e-10  // softening
            let dist = math.sqrt(dist_sq)
            let force = G * bodies[i].mass * bodies[j].mass / dist_sq

            let acc_i = vec3_scale(diff, force / (bodies[i].mass * dist))
            let acc_j = vec3_scale(diff, -force / (bodies[j].mass * dist))

            bodies[i].vel.x += acc_i.x * dt
            bodies[i].vel.y += acc_i.y * dt
            bodies[i].vel.z += acc_i.z * dt
            bodies[j].vel.x += acc_j.x * dt
            bodies[j].vel.y += acc_j.y * dt
            bodies[j].vel.z += acc_j.z * dt
        }
    }
}

// 更新位置
fn integrate(bodies: ref Array<Body>, dt: float) {
    for i in 0..bodies.length() {
        bodies[i].pos.x += bodies[i].vel.x * dt
        bodies[i].pos.y += bodies[i].vel.y * dt
        bodies[i].pos.z += bodies[i].vel.z * dt
    }
}

// 计算系统总能量（验证守恒）
fn total_energy(bodies: in Array<Body>): float {
    let energy = 0.0
    let n = bodies.length()
    // 动能
    for i in 0..n {
        let v2 = vec3_length_sq(bodies[i].vel)
        energy += 0.5 * bodies[i].mass * v2
    }
    // 势能
    for i in 0..n {
        for j in (i+1)..n {
            let diff = vec3_sub(bodies[j].pos, bodies[i].pos)
            let dist = math.sqrt(vec3_length_sq(diff))
            energy -= 6.674e-11 * bodies[i].mass * bodies[j].mass / dist
        }
    }
    return energy
}

let bodies = Array<Body>(1000)
// ... 初始化太阳系或随机星系
let dt = 86400.0  // 1 天

for step in 0..365 {
    compute_forces(ref bodies, dt)
    integrate(ref bodies, dt)
    if step % 30 == 0 {
        print("day ", step, " energy: ", total_energy(bodies))
    }
}
```

**性能分析**：
- `Array<Body>` 连续内存，每个 Body 64B = 1 缓存行
- `vec3_sub` 等小函数返回 Vec3（24B），编译器内联 + RVO
- `in Array<Body>` 只读引用，零拷贝
- O(n²) 是算法固有复杂度，内存访问模式最优

---

## Demo 5: 数据结构 — 单链表

**场景**：经典链表操作，验证 `Ptr<T>` 递归数据结构的代码书写体验。

```xray
struct ListNode {
    data: int
    next: Ptr<ListNode>?
}

// 头插法
fn prepend(head: Ptr<ListNode>?, val: int): Ptr<ListNode> {
    return Ptr.new(ListNode { data: val, next: head })
}

// 尾插法
fn append(head: Ptr<ListNode>?, val: int): Ptr<ListNode> {
    let new_node = Ptr.new(ListNode { data: val, next: null })
    if head == null { return new_node }

    let curr = head
    while curr.next != null {
        curr = curr.next
    }
    curr.next = new_node
    return head
}

// 查找
fn find(head: Ptr<ListNode>?, val: int): Ptr<ListNode>? {
    let curr = head
    while curr != null {
        if curr.data == val { return curr }
        curr = curr.next
    }
    return null
}

// 删除（返回新的头节点）
fn remove(head: Ptr<ListNode>?, val: int): Ptr<ListNode>? {
    if head == null { return null }
    if head.data == val { return head.next }

    let curr = head
    while curr.next != null {
        if curr.next.data == val {
            curr.next = curr.next.next
            return head
        }
        curr = curr.next
    }
    return head
}

// 反转链表
fn reverse(head: Ptr<ListNode>?): Ptr<ListNode>? {
    let prev: Ptr<ListNode>? = null
    let curr = head
    while curr != null {
        let next = curr.next
        curr.next = prev
        prev = curr
        curr = next
    }
    return prev
}

// 打印链表
fn print_list(head: Ptr<ListNode>?) {
    let curr = head
    while curr != null {
        print(curr.data, " -> ")
        curr = curr.next
    }
    print("null")
}

// 使用
let list: Ptr<ListNode>? = null
list = prepend(list, 3)
list = prepend(list, 2)
list = prepend(list, 1)
print_list(list)           // 1 -> 2 -> 3 -> null

list = append(list, 4)
print_list(list)           // 1 -> 2 -> 3 -> 4 -> null

list = remove(list, 2)
print_list(list)           // 1 -> 3 -> 4 -> null

list = reverse(list)
print_list(list)           // 4 -> 3 -> 1 -> null
```

**体验评估**：
- 代码和 C 几乎一样直观，但不需要 `malloc/free`
- `Ptr<ListNode>?` 明确表示可空，比 C 的裸 `T*` 更安全
- 自动解引用（`curr.data` 而不是 `curr->data`），更简洁
- GC/ARC 自动回收，`remove` 不需要 `free`

---

## Demo 6: 数据结构 — 二叉搜索树（BST）

**场景**：完整的 BST 实现，验证 Ptr\<T\> 的递归操作体验。

```xray
struct TreeNode {
    key: int
    value: float
    left: Ptr<TreeNode>?
    right: Ptr<TreeNode>?
}

fn new_node(key: int, value: float): Ptr<TreeNode> {
    return Ptr.new(TreeNode { key: key, value: value, left: null, right: null })
}

// 插入
fn insert(root: Ptr<TreeNode>?, key: int, value: float): Ptr<TreeNode> {
    if root == null {
        return new_node(key, value)
    }
    if key < root.key {
        root.left = insert(root.left, key, value)
    } else if key > root.key {
        root.right = insert(root.right, key, value)
    } else {
        root.value = value  // 更新已有 key
    }
    return root
}

// 查找
fn search(root: Ptr<TreeNode>?, key: int): float? {
    if root == null { return null }
    if key == root.key { return root.value }
    if key < root.key { return search(root.left, key) }
    return search(root.right, key)
}

// 找最小值节点
fn find_min(node: Ptr<TreeNode>): Ptr<TreeNode> {
    let curr = node
    while curr.left != null {
        curr = curr.left
    }
    return curr
}

// 删除
fn delete(root: Ptr<TreeNode>?, key: int): Ptr<TreeNode>? {
    if root == null { return null }

    if key < root.key {
        root.left = delete(root.left, key)
    } else if key > root.key {
        root.right = delete(root.right, key)
    } else {
        // 找到要删除的节点
        if root.left == null { return root.right }
        if root.right == null { return root.left }

        // 两个子节点：用右子树最小值替换
        let successor = find_min(root.right)
        root.key = successor.key
        root.value = successor.value
        root.right = delete(root.right, successor.key)
    }
    return root
}

// 中序遍历
fn inorder(node: Ptr<TreeNode>?) {
    if node == null { return }
    inorder(node.left)
    print(node.key, ":", node.value, " ")
    inorder(node.right)
}

// 树高度
fn height(node: Ptr<TreeNode>?): int {
    if node == null { return 0 }
    let lh = height(node.left)
    let rh = height(node.right)
    return 1 + (lh > rh ? lh : rh)
}

// 使用
let tree: Ptr<TreeNode>? = null
let keys = [50, 30, 70, 20, 40, 60, 80]
for k in keys {
    tree = insert(tree, k, float(k) * 0.1)
}

inorder(tree)              // 20:2.0 30:3.0 40:4.0 50:5.0 60:6.0 70:7.0 80:8.0
print("height: ", height(tree))  // 3

let val = search(tree, 40)
print("search 40: ", val)  // 4.0

tree = delete(tree, 50)
inorder(tree)              // 20 30 40 60 70 80
```

**体验评估**：
- 和 C 的 BST 实现几乎一模一样，但无需 `malloc/free`
- 递归操作非常自然（`insert(root.left, key, value)`）
- `Ptr<TreeNode>?` + `null` 检查替代 C 的 `NULL` 指针检查

---

## Demo 7: 数据结构 — 图（邻接表）

**场景**：有向加权图 + BFS/DFS + Dijkstra 最短路径。

```xray
struct Edge {
    to: int
    weight: float
}

struct Graph {
    adj: Array<Array<Edge>>    // 邻接表：adj[v] = 从 v 出发的边列表
    n: int                     // 顶点数
}

fn new_graph(n: int): Graph {
    let g = Graph { adj: Array<Array<Edge>>(n), n: n }
    for i in 0..n {
        g.adj.push(Array<Edge>())
    }
    return g
}

fn add_edge(g: ref Graph, from: int, to: int, weight: float) {
    g.adj[from].push(Edge { to: to, weight: weight })
}

// DFS（递归）
fn dfs(g: in Graph, v: int, visited: ref Array<bool>) {
    if visited[v] { return }
    visited[v] = true
    print(v, " ")

    let edges = g.adj[v]
    for i in 0..edges.length() {
        dfs(g, edges[i].to, ref visited)
    }
}

// BFS
fn bfs(g: in Graph, start: int) {
    let visited = Array<bool>(g.n)
    for i in 0..g.n { visited.push(false) }
    let queue = Array<int>()

    visited[start] = true
    queue.push(start)

    while queue.length() > 0 {
        let v = queue.shift()
        print(v, " ")

        let edges = g.adj[v]
        for i in 0..edges.length() {
            let u = edges[i].to
            if !visited[u] {
                visited[u] = true
                queue.push(u)
            }
        }
    }
}

// Dijkstra 最短路径（简化版，无优先队列）
struct DijkResult {
    dist: Array<float>
    prev: Array<int>
}

fn dijkstra(g: in Graph, source: int): DijkResult {
    let dist = Array<float>(g.n)
    let prev = Array<int>(g.n)
    let visited = Array<bool>(g.n)

    for i in 0..g.n {
        dist.push(1e18)
        prev.push(-1)
        visited.push(false)
    }
    dist[source] = 0.0

    for _ in 0..g.n {
        // 找未访问的最小距离节点
        let u = -1
        let min_dist = 1e18
        for v in 0..g.n {
            if !visited[v] && dist[v] < min_dist {
                min_dist = dist[v]
                u = v
            }
        }
        if u == -1 { break }
        visited[u] = true

        // 松弛邻居
        let edges = g.adj[u]
        for i in 0..edges.length() {
            let e = edges[i]
            let new_dist = dist[u] + e.weight
            if new_dist < dist[e.to] {
                dist[e.to] = new_dist
                prev[e.to] = u
            }
        }
    }

    return DijkResult { dist: dist, prev: prev }
}

// 使用
let g = new_graph(6)
add_edge(ref g, 0, 1, 4.0)
add_edge(ref g, 0, 2, 2.0)
add_edge(ref g, 1, 3, 5.0)
add_edge(ref g, 2, 1, 1.0)
add_edge(ref g, 2, 3, 8.0)
add_edge(ref g, 3, 4, 2.0)
add_edge(ref g, 4, 5, 1.0)

let result = dijkstra(g, 0)
for i in 0..6 {
    print("dist[", i, "] = ", result.dist[i])
}
// dist[0] = 0.0, dist[1] = 3.0, dist[2] = 2.0, dist[3] = 8.0, dist[4] = 10.0, dist[5] = 11.0
```

**注意**：
- `Graph` 包含 `Array<Array<Edge>>`，这里 `Array` 是引用类型（非 struct 字段约束下的嵌套引用类型）
- 但 `Edge` 本身是纯值 struct，`Array<Edge>` 中 Edge 连续存储
- `DijkResult` 包含 `Array<float>`，同理是引用类型字段

**问题发现**：`Graph` 和 `DijkResult` 包含 `Array` 引用类型字段。根据 801 设计，struct 不能包含引用类型。这里应该用 Json 或 class 代替？或者这些应该是普通变量而非 struct？

**修正方案**：

```xray
// 方案 A：Graph 不定义为 struct，而是用 class
class Graph {
    let adj: Array<Array<Edge>>
    let n: int

    fn init(n: int) {
        this.n = n
        this.adj = Array<Array<Edge>>(n)
        for i in 0..n { this.adj.push(Array<Edge>()) }
    }
}

// 方案 B：只把纯值部分定义为 struct，容器是独立变量
// Edge 保持为 struct（纯值，连续存储）
struct Edge { to: int; weight: float }

// Graph 的数据用函数参数传递，而不包装成一个类型
fn dijkstra(adj: Array<Array<Edge>>, n: int, source: int): Array<float> {
    // ...
}
```

**这暴露了一个设计权衡**：纯值 struct 约束在"容器+数据混合"场景下，有时不如直接用 class/Json 方便。但 Edge 作为 struct 仍然有优势（连续存储）。

---

## Demo 8: 数据结构 — Arena 版 BST（高性能场景）

**场景**：用 Arena 分配所有节点，连续内存，批量释放。适合编译器 IR、ECS 等场景。

```xray
struct TreeNode {
    key: int
    value: float
    left: int       // Handle（Arena 索引），-1 = null
    right: int      // Handle（Arena 索引），-1 = null
}

// Arena-based BST
fn arena_insert(arena: ref Arena<TreeNode>, root: int, key: int, value: float): int {
    if root == -1 {
        return arena.alloc(TreeNode { key: key, value: value, left: -1, right: -1 })
    }

    if key < arena[root].key {
        arena[root].left = arena_insert(ref arena, arena[root].left, key, value)
    } else if key > arena[root].key {
        arena[root].right = arena_insert(ref arena, arena[root].right, key, value)
    } else {
        arena[root].value = value
    }
    return root
}

fn arena_search(arena: in Arena<TreeNode>, root: int, key: int): float? {
    if root == -1 { return null }
    if key == arena[root].key { return arena[root].value }
    if key < arena[root].key { return arena_search(arena, arena[root].left, key) }
    return arena_search(arena, arena[root].right, key)
}

fn arena_inorder(arena: in Arena<TreeNode>, root: int) {
    if root == -1 { return }
    arena_inorder(arena, arena[root].left)
    print(arena[root].key, " ")
    arena_inorder(arena, arena[root].right)
}

// 使用
let arena = Arena<TreeNode>(10000)
let root = -1

for i in 0..10000 {
    let key = int(math.random() * 100000)
    root = arena_insert(ref arena, root, key, float(i))
}

arena_inorder(arena, root)
print("size: ", arena.count())

// Arena 析构时自动释放所有节点 — 一次 free
```

**性能分析**：
- 所有 TreeNode 在 Arena 连续内存中，缓存友好
- `arena[root].key` = `base + root * sizeof(TreeNode) + offset`，一次偏移
- 无 GC 开销（Arena 整块分配/释放）
- 索引 `int` 代替 `Ptr<T>`，4B vs 8B，内存更紧凑
- **缺点**：代码不如 Ptr 版直观（`arena[root].left` vs `root.left`）

---

## Demo 9: 编译器 IR — SSA 指令表示

**场景**：编译器内部的 IR 表示，大量小对象，Arena 批量分配。

```xray
struct IRInst {
    op: int              // 操作码
    dst: int             // 目标寄存器
    src1: int            // 源操作数 1
    src2: int            // 源操作数 2
    type: int            // 类型标记
    next: int            // Arena 索引，下一条指令
}

struct IRBlock {
    id: int
    first_inst: int      // Arena 索引，第一条指令
    last_inst: int       // Arena 索引，最后一条指令
    inst_count: int
    succs: [4]int        // 最多 4 个后继块
    succ_count: int
    preds: [8]int        // 最多 8 个前驱块
    pred_count: int
}

struct IRFunction {
    blocks: Array<IRBlock>
    inst_arena: Arena<IRInst>
    vreg_count: int
    name_id: int
}

// 添加指令到基本块
fn emit_inst(func: ref IRFunction, block_id: int, op: int, dst: int, s1: int, s2: int): int {
    let idx = func.inst_arena.alloc(IRInst {
        op: op, dst: dst, src1: s1, src2: s2, type: 0, next: -1
    })

    let blk = func.blocks[block_id]
    if blk.first_inst == -1 {
        func.blocks[block_id].first_inst = idx
    } else {
        func.inst_arena[blk.last_inst].next = idx
    }
    func.blocks[block_id].last_inst = idx
    func.blocks[block_id].inst_count += 1
    return idx
}

// 遍历基本块的所有指令
fn for_each_inst(func: in IRFunction, block_id: int, callback: fn(in IRInst)) {
    let idx = func.blocks[block_id].first_inst
    while idx != -1 {
        callback(func.inst_arena[idx])
        idx = func.inst_arena[idx].next
    }
}

// 使用
let func = IRFunction {
    blocks: Array<IRBlock>(),
    inst_arena: Arena<IRInst>(4096),
    vreg_count: 0,
    name_id: 0
}

// 创建基本块
func.blocks.push(IRBlock {
    id: 0, first_inst: -1, last_inst: -1, inst_count: 0,
    succs: [0,0,0,0], succ_count: 0,
    preds: [0,0,0,0,0,0,0,0], pred_count: 0
})

// 生成指令: v0 = add v1, v2
emit_inst(ref func, 0, 1, 0, 1, 2)  // OP_ADD=1
emit_inst(ref func, 0, 2, 3, 0, 4)  // OP_MUL=2, v3 = mul v0, v4
```

**注意**：`IRFunction` 包含 `Array` 和 `Arena`（引用类型），不能是 struct。应改为 class 或直接使用独立变量。

---

## Demo 10: 网络 — 零拷贝协议解析

**场景**：解析二进制网络协议（如 Redis RESP），使用 ArraySlice 零拷贝。

```xray
struct ParseResult {
    type: int           // 0=string, 1=int, 2=array, 3=error
    int_val: int
    str_start: int      // slice 偏移
    str_len: int
    arr_count: int
}

// 读取一行（找 \r\n），返回 slice
fn read_line(buf: Array<uint8>, pos: ref int): Array<uint8> {
    let start = pos
    while pos < buf.length() - 1 {
        if buf[pos] == 13 && buf[pos + 1] == 10 {  // \r\n
            let line = buf[start:pos]  // 零拷贝 slice！
            pos = pos + 2
            return line
        }
        pos = pos + 1
    }
    return buf[start:start]  // 空 slice
}

// 解析整数（从 slice）
fn parse_int_from_slice(slice: Array<uint8>): int {
    let result = 0
    let neg = false
    let start = 0
    if slice.length() > 0 && slice[0] == 45 {  // '-'
        neg = true
        start = 1
    }
    for i in start..slice.length() {
        result = result * 10 + (int(slice[i]) - 48)
    }
    return neg ? -result : result
}

// RESP 协议简单解析器
fn parse_resp(buf: Array<uint8>, pos: ref int): ParseResult {
    let type_byte = buf[pos]
    pos = pos + 1

    if type_byte == 43 {  // '+' Simple String
        let line = read_line(buf, ref pos)
        return ParseResult {
            type: 0, int_val: 0,
            str_start: pos - line.length() - 2, str_len: line.length(),
            arr_count: 0
        }
    }

    if type_byte == 58 {  // ':' Integer
        let line = read_line(buf, ref pos)
        return ParseResult {
            type: 1, int_val: parse_int_from_slice(line),
            str_start: 0, str_len: 0, arr_count: 0
        }
    }

    if type_byte == 36 {  // '$' Bulk String
        let len_line = read_line(buf, ref pos)
        let len = parse_int_from_slice(len_line)
        if len == -1 {
            return ParseResult { type: 3, int_val: 0, str_start: 0, str_len: 0, arr_count: 0 }
        }
        let str_start = pos
        pos = pos + len + 2  // skip data + \r\n
        return ParseResult {
            type: 0, int_val: 0,
            str_start: str_start, str_len: len, arr_count: 0
        }
    }

    if type_byte == 42 {  // '*' Array
        let count_line = read_line(buf, ref pos)
        let count = parse_int_from_slice(count_line)
        return ParseResult {
            type: 2, int_val: 0, str_start: 0, str_len: 0, arr_count: count
        }
    }

    return ParseResult { type: 3, int_val: 0, str_start: 0, str_len: 0, arr_count: 0 }
}

// 使用
let buf: Array<uint8> = receive_data()  // 网络接收的原始字节
let pos = 0
let result = parse_resp(buf, ref pos)
// 零拷贝：ParseResult 只记录偏移量，不拷贝数据
```

**性能分析**：
- `ParseResult` 是纯值 struct，栈分配
- `buf[start:pos]` 零拷贝 ArraySlice
- `parse_int_from_slice` 直接操作 slice 数据，无字符串转换
- 整个解析过程零堆分配（除了 ArraySlice 本身）

---

## Demo 11: 音频 — DSP 信号处理

**场景**：实时音频处理，定长缓冲区，IIR 滤波器。

```xray
struct AudioBuffer {
    samples: [512]float    // 512 采样点，@44.1kHz ≈ 11.6ms
}

// 二阶 IIR 滤波器状态
struct BiquadState {
    x1: float; x2: float   // 输入历史
    y1: float; y2: float   // 输出历史
}

// 二阶 IIR 滤波器系数（低通/高通/带通）
struct BiquadCoeffs {
    b0: float; b1: float; b2: float   // 前向系数
    a1: float; a2: float               // 反馈系数
}

// 应用 biquad 滤波器（原地处理）
fn biquad_process(buf: ref AudioBuffer, state: ref BiquadState, c: in BiquadCoeffs) {
    for i in 0..512 {
        let x0 = buf.samples[i]
        let y0 = c.b0*x0 + c.b1*state.x1 + c.b2*state.x2 - c.a1*state.y1 - c.a2*state.y2
        state.x2 = state.x1
        state.x1 = x0
        state.y2 = state.y1
        state.y1 = y0
        buf.samples[i] = y0
    }
}

// 设计低通滤波器系数
fn lowpass_coeffs(sample_rate: float, cutoff: float, q: float): BiquadCoeffs {
    let w0 = 2.0 * 3.14159265 * cutoff / sample_rate
    let alpha = math.sin(w0) / (2.0 * q)
    let cos_w0 = math.cos(w0)
    let a0 = 1.0 + alpha
    return BiquadCoeffs {
        b0: (1.0 - cos_w0) / 2.0 / a0,
        b1: (1.0 - cos_w0) / a0,
        b2: (1.0 - cos_w0) / 2.0 / a0,
        a1: -2.0 * cos_w0 / a0,
        a2: (1.0 - alpha) / a0
    }
}

// 混音：两个缓冲区叠加
fn mix(dst: ref AudioBuffer, src: in AudioBuffer, gain: float) {
    for i in 0..512 {
        dst.samples[i] += src.samples[i] * gain
    }
}

// 计算 RMS 音量
fn rms(buf: in AudioBuffer): float {
    let sum = 0.0
    for i in 0..512 {
        sum += buf.samples[i] * buf.samples[i]
    }
    return math.sqrt(sum / 512.0)
}

// 使用
let lp = lowpass_coeffs(44100.0, 1000.0, 0.707)
let state = BiquadState { x1: 0.0, x2: 0.0, y1: 0.0, y2: 0.0 }
let buf = AudioBuffer { samples: /* from audio input */ }

biquad_process(ref buf, ref state, lp)
print("output RMS: ", rms(buf))
```

**性能分析**：
- `AudioBuffer` = `[512]float` = 4096B，完全栈分配（AOT）
- `BiquadState` = 32B，`BiquadCoeffs` = 40B，全部栈上
- 内层循环只有浮点运算 + 数组访问，无任何分配
- `in BiquadCoeffs` 编译为 `const BiquadCoeffs*`，零拷贝
- 这段代码 AOT 后和手写 C 完全一致

---

## Demo 12: A* 寻路算法

**场景**：2D 网格上的 A* 路径查找，Arena 分配节点。

```xray
struct Vec2i { x: int; y: int }

struct AStarNode {
    pos: Vec2i
    g_cost: float      // 从起点到当前的代价
    h_cost: float      // 启发式（到终点的估算代价）
    parent: int         // Arena 索引，-1 = 无
}

fn f_cost(node: in AStarNode): float {
    return node.g_cost + node.h_cost
}

fn heuristic(a: in Vec2i, b: in Vec2i): float {
    // Manhattan distance
    let dx = a.x - b.x
    let dy = a.y - b.y
    if dx < 0 { dx = -dx }
    if dy < 0 { dy = -dy }
    return float(dx + dy)
}

fn astar(grid: Array<Array<int>>, start: Vec2i, goal: Vec2i): Array<Vec2i> {
    let rows = grid.length()
    let cols = grid[0].length()

    let arena = Arena<AStarNode>(rows * cols)
    let open_list = Array<int>()           // Arena 索引
    let closed: Array<Array<bool>> = /* rows x cols grid */

    // 起始节点
    let start_idx = arena.alloc(AStarNode {
        pos: start, g_cost: 0.0,
        h_cost: heuristic(start, goal), parent: -1
    })
    open_list.push(start_idx)

    let dirs: [4]Vec2i = [
        Vec2i{x:0, y:-1}, Vec2i{x:0, y:1},
        Vec2i{x:-1, y:0}, Vec2i{x:1, y:0}
    ]

    while open_list.length() > 0 {
        // 找 f_cost 最小的节点（简化版，无优先队列）
        let best = 0
        for i in 1..open_list.length() {
            if f_cost(arena[open_list[i]]) < f_cost(arena[open_list[best]]) {
                best = i
            }
        }
        let current_idx = open_list[best]
        let current = arena[current_idx]

        // 检查到达目标
        if current.pos.x == goal.x && current.pos.y == goal.y {
            // 回溯路径
            let path = Array<Vec2i>()
            let trace = current_idx
            while trace != -1 {
                path.push(arena[trace].pos)
                trace = arena[trace].parent
            }
            path.reverse()
            return path
        }

        // 从 open_list 移除
        open_list[best] = open_list[open_list.length() - 1]
        open_list.pop()
        closed[current.pos.y][current.pos.x] = true

        // 展开邻居
        for d in 0..4 {
            let nx = current.pos.x + dirs[d].x
            let ny = current.pos.y + dirs[d].y

            if nx < 0 || nx >= cols || ny < 0 || ny >= rows { continue }
            if grid[ny][nx] == 1 { continue }  // 障碍物
            if closed[ny][nx] { continue }

            let new_g = current.g_cost + 1.0
            let neighbor_pos = Vec2i { x: nx, y: ny }
            let idx = arena.alloc(AStarNode {
                pos: neighbor_pos,
                g_cost: new_g,
                h_cost: heuristic(neighbor_pos, goal),
                parent: current_idx
            })
            open_list.push(idx)
        }
    }

    return Array<Vec2i>()  // 无路径
}

// 使用
let grid = [
    [0, 0, 0, 0, 1, 0],
    [0, 1, 1, 0, 1, 0],
    [0, 0, 0, 0, 0, 0],
    [0, 1, 0, 1, 1, 0],
    [0, 0, 0, 0, 0, 0]
]
let path = astar(grid, Vec2i{x:0, y:0}, Vec2i{x:5, y:4})
for p in path {
    print("(", p.x, ",", p.y, ") ")
}
```

**性能分析**：
- `AStarNode` = 36B 纯值 struct，Arena 连续分配
- `Vec2i` = 16B，值传递零开销
- `dirs: [4]Vec2i` 栈上定长数组，零堆分配
- Arena 退出作用域后一次性释放所有节点

---

## Demo 13: 双向链表 + weak 引用

**场景**：LRU 缓存需要双向链表，验证循环引用处理。

```xray
struct DListNode {
    key: int
    value: int
    prev: weak Ptr<DListNode>?    // 弱引用，不增加引用计数
    next: Ptr<DListNode>?
}

// 双向链表（头尾哨兵）
fn dlist_new(): Ptr<DListNode> {
    let head = Ptr.new(DListNode { key: -1, value: -1, prev: null, next: null })
    let tail = Ptr.new(DListNode { key: -1, value: -1, prev: head, next: null })
    head.next = tail
    return head
}

// 在头部之后插入
fn dlist_push_front(head: Ptr<DListNode>, node: Ptr<DListNode>) {
    node.next = head.next
    node.prev = head
    head.next.prev = node
    head.next = node
}

// 移除指定节点
fn dlist_remove(node: Ptr<DListNode>) {
    node.prev.next = node.next
    node.next.prev = node.prev
    node.prev = null
    node.next = null
}

// 移动到头部
fn dlist_move_to_front(head: Ptr<DListNode>, node: Ptr<DListNode>) {
    dlist_remove(node)
    dlist_push_front(head, node)
}

// LRU 缓存
class LRUCache {
    let capacity: int
    let map: Map<int, Ptr<DListNode>>
    let head: Ptr<DListNode>
    let tail: Ptr<DListNode>

    fn init(cap: int) {
        this.capacity = cap
        this.map = {}
        this.head = Ptr.new(DListNode { key: -1, value: -1, prev: null, next: null })
        this.tail = Ptr.new(DListNode { key: -1, value: -1, prev: this.head, next: null })
        this.head.next = this.tail
    }

    fn get(key: int): int {
        if !this.map.has(key) { return -1 }
        let node = this.map[key]
        dlist_move_to_front(this.head, node)
        return node.value
    }

    fn put(key: int, value: int) {
        if this.map.has(key) {
            let node = this.map[key]
            node.value = value
            dlist_move_to_front(this.head, node)
            return
        }

        let node = Ptr.new(DListNode { key: key, value: value, prev: null, next: null })
        this.map[key] = node
        dlist_push_front(this.head, node)

        if this.map.length() > this.capacity {
            // 删除尾部（最久未使用）
            let lru = this.tail.prev
            dlist_remove(lru)
            this.map.delete(lru.key)
        }
    }
}

// 使用
let cache = new LRUCache(3)
cache.put(1, 10)
cache.put(2, 20)
cache.put(3, 30)
print(cache.get(1))     // 10（移到头部）
cache.put(4, 40)         // 淘汰 key=2
print(cache.get(2))     // -1（已淘汰）
```

**注意**：
- `weak Ptr<DListNode>?` 用于 `prev` 字段，防止双向引用导致的 ARC 循环
- GC 模式下 `weak` 不是必须的（GC 可处理循环），但 AOT 的 ARC 模式需要
- 编译器可以检测双向 Ptr 引用并建议使用 `weak`

---

## 设计完备性总结

### 场景覆盖矩阵

| 场景 | struct | Ptr\<T\> | Arena\<T\> | ArraySlice | [N]T | ref/in | 体验评价 |
|------|--------|---------|-----------|------------|------|--------|---------|
| 粒子系统 | ✅ | — | — | — | — | ✅ | ⭐⭐⭐⭐⭐ 优秀 |
| AABB 碰撞 | ✅ | — | — | — | ✅ | ✅ | ⭐⭐⭐⭐⭐ 优秀 |
| 矩阵运算 | ✅ | — | — | — | ✅ | ✅ | ⭐⭐⭐⭐⭐ 优秀 |
| N-body | ✅ | — | — | — | — | ✅ | ⭐⭐⭐⭐⭐ 优秀 |
| 单链表 | ✅ | ✅ | — | — | — | — | ⭐⭐⭐⭐⭐ 优秀 |
| BST | ✅ | ✅ | — | — | — | — | ⭐⭐⭐⭐⭐ 优秀 |
| 图+Dijkstra | ✅ | — | — | — | — | ✅ | ⭐⭐⭐⭐ 良好 |
| Arena BST | ✅ | — | ✅ | — | — | ✅ | ⭐⭐⭐⭐ 良好 |
| 编译器 IR | ✅ | — | ✅ | — | ✅ | ✅ | ⭐⭐⭐⭐ 良好 |
| 协议解析 | ✅ | — | — | ✅ | — | ✅ | ⭐⭐⭐⭐⭐ 优秀 |
| 音频 DSP | ✅ | — | — | — | ✅ | ✅ | ⭐⭐⭐⭐⭐ 优秀 |
| A* 寻路 | ✅ | — | ✅ | — | ✅ | — | ⭐⭐⭐⭐ 良好 |
| LRU 缓存 | ✅ | ✅ | — | — | — | — | ⭐⭐⭐⭐ 良好 |

### 发现的设计问题

**问题 1：struct 不能包含引用类型字段的限制**

在 Demo 7（Graph）和 Demo 9（编译器 IR）中，包含 `Array` 字段的复合类型不能定义为 struct。

- **影响**：需要用 class 包装，或将容器作为独立参数传递
- **评估**：可接受。纯数据部分（Edge, IRInst）仍然是 struct，获得连续存储优势。复合管理类型用 class 是合理的分层
- **建议**：不需要改变设计。struct = 纯数据，class = 行为+引用，分工明确

**问题 2：Arena 的索引操作不如 Ptr 直观**

`arena[root].left` vs `root.left` — Arena 版代码需要显式传 arena 参数。

- **影响**：Arena 版代码更冗余（多一个参数），但更快（连续内存）
- **评估**：这是性能 vs 便利性的合理权衡
- **建议**：Arena 推荐用于 ECS/编译器等批量场景，通用场景用 Ptr

**问题 3：`DijkResult` 等返回多个容器的需求**

801 设计中 struct 不能包含 `Array`，但算法经常需要返回多个容器。

- **影响**：需要用 class/Json 包装返回值，或用多返回值语法
- **建议**：考虑支持多返回值 `fn f(): (Array<int>, Array<int>)` 或 tuple 解构

**问题 4：`Array<Struct>` 的 slice**

`particles[10:20]` 应该返回什么？如果是 ArraySlice，底层已支持（`elem_size` 正确偏移）。

- **评估**：现有 ArraySlice 机制天然支持 `Array<Struct>` 的 slice（零拷贝视图）
- **建议**：确认编译器能正确处理 struct elem_size

### 性能等级评估

| 等级 | 描述 | 场景 |
|------|------|------|
| **S（C 同等）** | AOT 后和手写 C 完全一致 | 矩阵运算、音频DSP、粒子系统、碰撞检测 |
| **A（接近 C）** | 仅有 GC header / ARC 微小开销 | BST、链表、A*、N-body |
| **B（优于脚本）** | 比 JS/Python 快 10-100x | 图算法、编译器 IR、LRU |
