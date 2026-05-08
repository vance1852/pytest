# pytest 核心机制源码分析

---

## 一、Fixture 解析与注入机制

### 1.1 从 `pytest_fixture_setup` 到完整注入链路

当一个测试函数 `def test_foo(fixture_a, fixture_b)` 被执行时，pytest 的 fixture 注入遵循以下完整链路：

#### 1.1.1 入口：`FixtureDef.execute()` 触发 `pytest_fixture_setup`

整个链路从 `FixtureDef.execute()`（`src/_pytest/fixtures.py:1114`）开始。当 pytest 需要为某个测试项获取 fixture 值时，核心执行逻辑如下：

```
TopRequest._fillfixtures()
  → TopRequest.getfixturevalue(argname)        # fixtures.py:566
    → FixtureRequest._get_active_fixturedef()   # fixtures.py:609
      → FixtureDef.execute(request=subrequest)  # fixtures.py:688
        → ihook.pytest_fixture_setup()          # fixtures.py:1175
          → pytest_fixture_setup()               # fixtures.py:1237 (内置实现)
            → request.getfixturevalue(argname)   # 递归解析依赖
              → call_fixture_func()              # fixtures.py:953
```

#### 1.1.2 `FixtureRequest.getfixturevalue()`（fixtures.py:566）

```python
def getfixturevalue(self, argname: str) -> Any:
    fixturedef = self._get_active_fixturedef(argname)
    assert fixturedef.cached_result is not None
    return fixturedef.cached_result[0]
```

此方法先通过 `_get_active_fixturedef` 获取已执行（或即将执行）的 `FixtureDef`，然后直接返回其缓存结果 `cached_result[0]`。`cached_result` 是一个三元组 `(value, cache_key, exception_info)`。

#### 1.1.3 `FixtureRequest._get_active_fixturedef()`（fixtures.py:609）

这是 fixture 解析的核心方法，关键步骤如下：

1. **缓存命中检查**：首先检查 `self._fixture_defs` 字典，如果该 fixture 名已经解析过，直接返回（`fixtures.py:615-618`）。

2. **查找 FixtureDef 列表**：从 `self._arg2fixturedefs` 中获取该名称对应的所有 `FixtureDef`（可能有覆盖链），如果静态表中没有（动态调用 `getfixturevalue` 的场景），则通过 `self._fixturemanager.getfixturedefs()` 动态查找（`fixtures.py:621-628`）。

3. **处理 fixture 覆盖**：同名 fixture 可能被多层 conftest/class/module 覆盖。`fixturedefs` 列表按从远到近排序，通过 `_iter_chain()` 追踪当前请求链中同一 fixture 名出现的次数，用负索引 `index` 从最近层开始选取（`fixtures.py:643-650`）。

4. **获取参数信息**：从 `callspec` 中提取参数值（参数化场景），或使用 `NOTSET` 作为无参标记（`fixtures.py:653-666`）。

5. **Scope 检查**：调用 `self._check_scope()` 确保请求的 scope 合法（窄 scope 不能请求宽 scope 的 fixture，否则 `ScopeMismatch`）（`fixtures.py:672`）。

6. **创建 SubRequest 并执行**：构造 `SubRequest` 对象并调用 `fixturedef.execute(request=subrequest)`（`fixtures.py:673-688`）。

7. **缓存到字典**：将解析后的 `FixtureDef` 记录到 `self._fixture_defs`，后续同一测试项中对同一 fixture 的请求直接命中缓存（`fixtures.py:690`）。

#### 1.1.4 `FixtureDef.execute()`（fixtures.py:1114）

`execute()` 方法管理 fixture 的生命周期和缓存：

```python
def execute(self, request: SubRequest) -> FixtureValue:
    # 1. 先递归解析所有依赖的 fixture
    for argname in self.argnames:
        fixturedef = request._get_active_fixturedef(argname)
        requested_fixtures_that_should_finalize_us.append(fixturedef)

    # 2. 检查缓存是否命中
    if self.cached_result is not None:
        request_cache_key = self.cache_key(request)
        cache_key = self.cached_result[1]
        if cache_hit:
            return self.cached_result[0]   # 缓存命中，直接返回
        self.finish(request)               # 参数变了，先 teardown

    # 3. 注册 finalizer 到依赖 fixture
    finalizer = functools.partial(self.finish, request=request)
    for parent_fixture in requested_fixtures_that_should_finalize_us:
        parent_fixture.addfinalizer(finalizer)

    # 4. 通过 hook 调用 pytest_fixture_setup
    result = ihook.pytest_fixture_setup(fixturedef=self, request=request)

    return result
```

关键设计：**先解析依赖再检查缓存**（`fixtures.py:1116-1130`）。这是为了处理依赖的 parametrized fixture 参数变化导致缓存失效的场景——依赖 fixture 的 `finish()` 会把当前 fixture 的 `cached_result` 置为 `None`。

#### 1.1.5 `pytest_fixture_setup()` 钩子（fixtures.py:1237）

这是内置的 fixture 设置实现，也是 `@hookspec(firstresult=True)` 标记的 hook：

```python
def pytest_fixture_setup(fixturedef, request):
    kwargs = {}
    for argname in fixturedef.argnames:
        kwargs[argname] = request.getfixturevalue(argname)  # 递归解析依赖

    fixturefunc = resolve_fixture_function(fixturedef, request)
    my_cache_key = fixturedef.cache_key(request)

    result = call_fixture_func(fixturefunc, request, kwargs)
    fixturedef.cached_result = (result, my_cache_key, None)
    return result
```

这里 `request.getfixturevalue(argname)` 会触发递归解析，形成自底向上的依赖解析链。

### 1.2 依赖图构建：`FuncFixtureInfo`

#### 1.2.1 `FuncFixtureInfo` 数据结构（fixtures.py:351）

```python
@dataclasses.dataclass(frozen=True)
class FuncFixtureInfo:
    __slots__ = ("argnames", "initialnames", "name2fixturedefs", "names_closure")

    argnames: tuple[str, ...]          # 测试函数直接请求的 fixture 名
    initialnames: tuple[str, ...]      # argnames + usefixtures + autouse
    names_closure: list[str]           # 传递闭包（所有需要的 fixture 名）
    name2fixturedefs: dict[str, Sequence[FixtureDef]]  # 名→FixtureDef 列表
```

- `argnames`：测试函数签名中的参数名。
- `initialnames`：通过 `deduplicate_names(autousenames, usefixturesnames, argnames)` 合并所有直接需要的 fixture。
- `names_closure`：从 `initialnames` 出发，DFS 遍历所有 fixture 的依赖，得到传递闭包。
- `name2fixturedefs`：闭包中每个 fixture 名对应的 `FixtureDef` 序列（覆盖链）。

#### 1.2.2 依赖闭包构建：`FixtureManager.getfixtureclosure()`（fixtures.py:1739）

```python
def getfixtureclosure(self, parentnode, initialnames, ignore_args):
    arg2fixturedefs = {}

    def getfixturedefs(argname):
        if argname in ignore_args:      # 直接参数化的参数不算 fixture
            return None
        fixturedefs = arg2fixturedefs.get(argname)
        if not fixturedefs:
            fixturedefs = self.getfixturedefs(argname, parentnode)
            if not fixturedefs:
                return None
            arg2fixturedefs[argname] = fixturedefs
        return fixturedefs

    fixturenames_closure = sorted(
        traverse_fixture_closure(initialnames, getfixturedefs=getfixturedefs),
        key=sort_by_scope,
        reverse=True,    # scope 大的排前面 → session > package > module > class > function
    )
    return fixturenames_closure, arg2fixturedefs
```

`traverse_fixture_closure()`（fixtures.py:303）以 DFS 顺序遍历依赖图，使用 `current_indices` 字典处理覆盖链（同名 fixture 可层层覆盖）。最终 `names_closure` 按 scope 从大到小排序，确保高 scope fixture 先被 setup。

#### 1.2.3 `FuncFixtureInfo` 的创建时机

在 `FixtureManager.getfixtureinfo()`（fixtures.py:1661）中创建，收集阶段对每个测试项调用一次。创建后存储在 `Function._fixtureinfo` 属性中。

### 1.3 Scope 层级与缓存生命周期

#### 1.3.1 Scope 枚举（`src/_pytest/scope.py`）

```
Session > Package > Module > Class > Function
```

#### 1.3.2 缓存绑定到 Node

`FixtureDef.cached_result` 存储在 `FixtureDef` 对象自身上，而非某个 Node 上。但 fixture 的 **setup/teardown 时机**由 `SetupState` 和 scope 决定：

- **function scope**：每个测试项独立 setup/teardown。
- **class scope**：同一类中所有测试方法共享，类内第一个测试 setup，最后一个测试后 teardown。
- **module scope**：同一模块所有测试共享。
- **package scope**：同一包下所有测试共享。
- **session scope**：整个测试会话共享。

#### 1.3.3 生命周期管理：`SubRequest` 的 Node 绑定

在 `_get_active_fixturedef()` 中创建 `SubRequest` 时（fixtures.py:673-675），会根据 scope 找到对应的 Node：

```python
SubRequest(self, scope, param, param_index, fixturedef, _ispytest=True)
```

`SubRequest.__init__()` 内部（fixtures.py:802-815）：

```python
if scope is Scope.Function:
    node = self._pyfuncitem
elif scope is Scope.Package:
    node = get_scope_package(self._pyfuncitem, self._fixturedef)
else:
    node = get_scope_node(self._pyfuncitem, scope)
```

`get_scope_node()`（fixtures.py:125）沿 `node.iter_parents()` 向上查找对应 scope 类型的 Node。

#### 1.3.4 Teardown 机制

`FixtureDef.finish()`（fixtures.py:1089）执行所有已注册的 finalizer（包括 yield fixture 中 yield 之后的代码）。当 `SetupState` 检测到某个 Node 即将退出活跃范围时，会调用该 Node 上绑定的所有 finalizer，从而触发 scope 对应的 fixture teardown。

具体来说，`FixtureDef.execute()` 在执行时会在依赖的 fixture 上注册 `self.finish` 作为 finalizer（fixtures.py:1158-1160），同时也会在 `request.node` 上注册（fixtures.py:1180）。这确保了当依赖的 fixture 被销毁时，当前 fixture 也会先被销毁。

### 1.4 `cache_key` 在参数化场景下的作用

#### 1.4.1 `cache_key` 的定义（fixtures.py:1184）

```python
def cache_key(self, request: SubRequest) -> object:
    return getattr(request, "param", None)
```

`cache_key` 就是 `request.param`——即参数化 fixture 当前这一组参数的值。

#### 1.4.2 缓存命中逻辑（fixtures.py:1133-1153）

```python
if self.cached_result is not None:
    request_cache_key = self.cache_key(request)
    cache_key = self.cached_result[1]
    try:
        cache_hit = bool(request_cache_key == cache_key)
    except (ValueError, RuntimeError):
        cache_hit = request_cache_key is cache_key

    if cache_hit:
        return self.cached_result[0]      # 参数相同，复用缓存
    self.finish(request)                   # 参数不同，先 teardown 再重建
```

#### 1.4.3 保证同一参数复用同一实例

当一个 session-scoped 的 fixture 被参数化时，例如：

```python
@pytest.fixture(scope="session", params=["A", "B"])
def db(request):
    return setup_db(request.param)
```

pytest 会为 `test_foo[db-A]` 和 `test_foo[db-B]` 生成两个测试项。当第一个测试项执行时，`cache_key = "A"`，`cached_result = (db_A, "A", None)`。当第二个测试项执行时，`cache_key = "B" ≠ "A"`，触发 `finish()`（teardown 旧值）然后重新执行 fixture 得到 `db_B`。

但关键点在于 **scope 控制了何时需要重新执行**。对于同 scope 内相同参数的测试项，`cache_key` 相同，直接返回缓存值，不会重新执行 fixture 函数。这就保证了同一参数在同一 scope 内只创建一个实例。

`reorder_items()`（fixtures.py:217）算法会尽量把相同参数的测试项排在一起，以最小化高 scope fixture 的 setup/teardown 次数。

---

## 二、Assertion Rewriting 机制

### 2.1 概述

pytest 的 assertion rewriting 是通过 Python 的 import 机制（PEP 302/PEP 451）拦截测试模块的加载，在 AST 层面将 `assert a == b` 改写成能输出详细比较信息的代码。

### 2.2 `AssertionRewritingHook`：Import Hook

`AssertionRewritingHook`（`src/_pytest/assertion/rewrite.py:76`）同时实现了 `importlib.abc.MetaPathFinder` 和 `importlib.abc.Loader` 接口，被注册到 `sys.meta_path` 中。

#### 2.2.1 `find_spec()`（rewrite.py:102）

当 Python 的 import 系统尝试导入任何模块时，会依次调用 `sys.meta_path` 上的 finder。`find_spec()` 的工作流程：

1. **防重入检查**：如果正在写 pyc（`self._writing_pyc`），直接返回 `None`（rewrite.py:108-109）。

2. **早期退出优化**：`_early_rewrite_bailout()`（rewrite.py:190）快速判断模块是否可能需要改写。它检查模块名的最后一段是否在 `_basenames_to_check_rewrite`（默认包含 `"conftest"`）中，或者是否匹配 `python_files` 模式（如 `test_*.py`）。如果都不匹配则直接跳过，避免昂贵的 `PathFinder.find_spec()` 调用。

3. **调用标准 PathFinder**：`self._find_spec(name, path)` 获取模块的 `ModuleSpec`（rewrite.py:116）。

4. **检查是否可改写**：排除 namespace 包（`spec.origin is None`）、非源文件（非 `SourceFileLoader`）、不存在的文件（rewrite.py:118-129）。

5. **`_should_rewrite()` 判定**（rewrite.py:229）：
   - **conftest.py 总是改写**：`os.path.basename(fn) == "conftest.py"` → `True`
   - **命令行指定的文件总是改写**：`self.session.isinitpath(absolutepath(fn))` → `True`
   - **匹配 `python_files` 模式的文件改写**：如 `test_*.py`、`*_test.py`
   - **通过 `register_assert_rewrite()` 标记的模块改写**：`_is_marked_for_rewrite()`

6. **返回新 Spec**：如果需要改写，用 `importlib.util.spec_from_file_location()` 创建新的 `ModuleSpec`，将其 `loader` 设为 `self`（即 `AssertionRewritingHook` 自身）（rewrite.py:136-141）。这确保后续 `exec_module()` 由本 hook 处理。

#### 2.2.2 `exec_module()`（rewrite.py:148）

当 import 系统确定使用此 loader 后，调用 `exec_module()` 执行模块：

```python
def exec_module(self, module):
    fn = Path(module.__spec__.origin)
    # 1. 尝试读取缓存的 pyc
    co = _read_pyc(fn, pyc, state.trace)
    if co is None:
        # 2. 无缓存，执行 AST 改写
        source_stat, co = _rewrite_test(fn, self.config)
        if write:
            # 3. 写入 pyc 缓存
            _write_pyc(state, co, source_stat, pyc)
    # 4. 在模块命名空间中执行改写后的代码
    exec(co, module.__dict__)
```

详细流程：

1. **尝试读取缓存**：调用 `_read_pyc()` 查找 `__pycache__/test_foo.cpython-XX-pytest-YY.pyc`。

2. **缓存未命中时改写**：`_rewrite_test()`（rewrite.py:343）读取源码，调用 `ast.parse()` 得到 AST，再调用 `rewrite_asserts()` → `AssertionRewriter.run()` 改写 assert 语句，最后 `compile()` 得到 code object。

3. **写入缓存**：`_write_pyc()` 原子性地写入 pyc 文件（先写临时文件再 `os.replace`，防止并发竞态）。

4. **执行代码**：`exec(co, module.__dict__)` 在模块的命名空间中执行改写后的字节码。

### 2.3 `AssertionRewriter`：AST 改写引擎

`AssertionRewriter`（rewrite.py:606）继承 `ast.NodeVisitor`，核心入口是 `run()` 方法。

#### 2.3.1 `run()`（rewrite.py:683）

1. **插入特殊 import**：在模块顶部（docstring 和 `__future__` import 之后）插入：

   ```python
   import builtins as @py_builtins
   import _pytest.assertion.rewrite as @pytest_ar
   ```

   这些别名以 `@` 开头，避免与用户代码冲突。

2. **遍历 AST**：使用栈式遍历，遇到 `ast.Assert` 节点时调用 `self.visit(child)` 进行改写（rewrite.py:747-748）。

3. **scope 跟踪**：通过 `self.scope` 元组跟踪当前所在的类/函数作用域，处理 walrus operator (`:=`) 的变量覆盖。

#### 2.3.2 `visit_Assert()`（rewrite.py:845）

这是改写的核心。将 `assert a == b` 改写为类似如下结构：

```python
# 原始: assert a == b

# 改写后:
@py_assert0 = a
@py_assert1 = b
@py_assert2 = @py_assert0 == @py_assert1
if not @py_assert2:
    @py_format3 = @pytest_ar._call_reprcompare(
        ('==',), (@py_assert2,), ('%@py_assert4 == %@py_assert5',),
        (@py_assert0, @py_assert1)
    )
    @py_format5 = @pytest_ar._format_explanation("" + @py_format3)
    raise AssertionError(@py_format5)
@py_assert0 = @py_assert1 = @py_assert2 = None  # 清理临时变量
```

关键步骤：

1. **递归 visit test 表达式**：`self.visit(assert_.test)` 返回 `(top_condition, explanation)`，其中 `top_condition` 是包含中间变量的条件表达式，`explanation` 是格式化字符串。

2. **生成 if 语句**：`if not top_condition: <error handling>` 或包含 `pytest_assertion_pass` hook 调用的双向分支。

3. **格式化错误信息**：使用 `%-formatting` 机制（`push_format_context` / `pop_format_context` / `explanation_param`），在断言失败时将中间变量值填充到解释字符串中。

4. **清理临时变量**：将所有 `@py_assert*` 变量设为 `None`，避免内存泄漏。

#### 2.3.3 各种 AST 节点的改写

- **`visit_Compare`**（rewrite.py:1101）：处理链式比较如 `a < b < c`，每个比较运算生成独立变量，最终用 `_call_reprcompare` 生成详细对比信息。
- **`visit_BoolOp`**（rewrite.py:988）：处理 `and`/`or`，使用短路求值逻辑，逐步记录每个操作数的解释。
- **`visit_Call`**（rewrite.py:1050）：记录函数调用结果和参数的解释。
- **`visit_BinOp`**（rewrite.py:1040）：记录二元操作的左右操作数。
- **`visit_Name`**（rewrite.py:978）：对局部变量显示其 `repr()`，对全局变量则只显示变量名。

### 2.4 pyc 缓存机制

#### 2.4.1 缓存文件命名

```
__pycache__/test_foo.cpython-311-pytest-8.3.4.pyc
```

格式：`{module_name}.{cache_tag}-pytest-{pytest_version}.pyc`（rewrite.py:68-70）。

#### 2.4.2 `_write_pyc()`（rewrite.py:318）

```python
def _write_pyc(state, co, source_stat, pyc):
    proc_pyc = f"{pyc}.{os.getpid()}"
    with open(proc_pyc, "wb") as fp:
        _write_pyc_fp(fp, source_stat, co)
    os.replace(proc_pyc, pyc)    # 原子替换
```

写入内容包括：Python 魔数 + flags + 源文件 mtime/size + marshal 序列化的 code object。使用进程 ID 后缀的临时文件 + `os.replace()` 保证写入的原子性，避免并发 pytest 进程读到不完整的 pyc。

#### 2.4.3 `_read_pyc()`（rewrite.py:354）

读取 pyc 时的校验：

1. 检查魔数是否匹配当前 Python 版本。
2. 检查 flags 是否为 `0x00000000`。
3. 检查 mtime 是否与源文件一致。
4. 检查 size 是否与源文件一致。
5. `marshal.load()` 反序列化得到 code object。

任何一项不匹配就返回 `None`，触发重新改写。

#### 2.4.4 缓存的作用

- **加速启动**：已改写的模块无需每次重新解析 AST 和改写，直接从 pyc 加载。
- **正确性保证**：源文件修改后 mtime/size 变化，缓存自动失效。
- **版本隔离**：pyc 文件名包含 pytest 版本号，不同版本不冲突。

### 2.5 为什么只有 conftest 和 test 模块被改写

`_should_rewrite()` 的逻辑（rewrite.py:229）：

1. **conftest.py 总是改写**：conftest 中可能包含 `assert` 语句用于验证 fixture 设置。
2. **命令行指定的文件总是改写**：用户明确指定要运行的测试文件。
3. **匹配 `python_files` 模式的文件改写**：默认 `test_*.py` 和 `*_test.py`。
4. **`register_assert_rewrite()` 标记的模块改写**：用户可以通过 `pytest_register_assert_rewrite()` 钩子请求改写特定模块。
5. **其他模块不改写**：生产代码、第三方库等不应被改写，原因：
   - 改写会增加 import 时间。
   - 可能破坏非测试代码的 assert 行为。
   - 改写后的代码引入了额外的临时变量和函数调用，不应影响生产代码的性能。
   - `_early_rewrite_bailout()`（rewrite.py:190）在更早阶段就排除了明显不需要改写的模块（模块名不匹配任何 test 模式），避免调用昂贵的 `PathFinder.find_spec()`。

模块的 docstring 中包含 `PYTEST_DONT_REWRITE` 可以显式禁止改写（`is_rewrite_disabled()`，rewrite.py:763）。

---

## 三、插件钩子体系（基于 pluggy）

### 3.1 Hook 声明：`@hookspec`

#### 3.1.1 声明方式

在 `src/_pytest/hookspec.py` 中，所有 hook 规范通过 `@hookspec` 装饰器声明：

```python
from pluggy import HookspecMarker
hookspec = HookspecMarker("pytest")

@hookspec(firstresult=True)
def pytest_fixture_setup(fixturedef, request): ...

def pytest_collection_modifyitems(session, config, items): ...

@hookspec(firstresult=True)
def pytest_runtest_protocol(item, nextitem): ...
```

`HookspecMarker("pytest")` 创建与项目名 `"pytest"` 绑定的装饰器。当 `PytestPluginManager.add_hookspecs()` 被调用时，pluggy 会扫描被装饰函数的 `pytest_spec` 属性来注册 hook 规范。

#### 3.1.2 hookspec 参数

- **`firstresult=True`**：hook 调用时，第一个返回非 `None` 结果的实现之后停止，不再调用后续实现。例如 `pytest_fixture_setup`、`pytest_runtest_protocol` 都有此标记。
- **`historic=True`**：hook 调用会被记录，后续注册的插件会收到历史调用。例如 `pytest_addhooks`、`pytest_configure`。
- 无参数：所有实现都会被调用，结果收集到列表中。例如 `pytest_collection_modifyitems`。

### 3.2 插件注册：`PytestPluginManager`

#### 3.2.1 继承关系

`PytestPluginManager`（`src/_pytest/config/__init__.py:418`）继承自 `pluggy.PluginManager`，增加了 pytest 特有的功能：

```python
class PytestPluginManager(PluginManager):
    def __init__(self):
        super().__init__("pytest")
        self.add_hookspecs(_pytest.hookspec)   # 注册所有 hook 规范
        self.register(self)                     # 注册自身
```

#### 3.2.2 注册流程

1. **`PluginManager.register(plugin, name)`**：扫描插件对象的所有方法，通过 `parse_hookimpl_opts()` 识别以 `pytest_` 开头的方法作为 hook 实现。

2. **`PytestPluginManager.parse_hookimpl_opts()`**（config/__init__.py:479）：
   ```python
   def parse_hookimpl_opts(self, plugin, name):
       if not name.startswith("pytest_"):
           return None
       if name == "pytest_plugins":
           return None
       opts = super().parse_hookimpl_opts(plugin, name)  # 检查 @hookimpl 装饰器
       if opts is not None:
           return opts
       # 没有 @hookimpl 装饰器，但以 pytest_ 开头的函数也作为 hook
       method = getattr(plugin, name)
       if not inspect.isroutine(method):
           return None
       legacy = _get_legacy_hook_marks(method, "impl",
                   ("tryfirst", "trylast", "optionalhook", "hookwrapper"))
       return legacy
   ```

   这意味着 pytest 插件有两种注册 hook 的方式：
   - **显式**：`@pytest.hookimpl(tryfirst=True)` 装饰器。
   - **隐式**：只要方法名以 `pytest_` 开头，就自动被识别为 hook 实现。

3. **注册后回调**：注册成功后触发 `pytest_plugin_registered` hook（historic），如果是模块类型插件还会调用 `consider_module()` 处理 `pytest_plugins` 变量。

#### 3.2.3 内置插件注册

在 `get_config()`（config/__init__.py:313）中：

```python
for spec in default_plugins:
    pluginmanager.import_plugin(spec)
```

所有内置插件（mark、main、runner、fixtures、python、terminal 等）在启动时自动加载。

### 3.3 Hook 调用时多个实现的执行顺序

#### 3.3.1 HookImpl 排序规则

`HookCaller._add_hookimpl()`（pluggy/_hooks.py:451）决定了 hook 实现的存储顺序。`_hookimpls` 列表的格式为（按索引从小到大）：

```
1. trylast 非wrapper
2. 普通 非wrapper
3. tryfirst 非wrapper
4. trylast wrapper
5. 普通 wrapper
6. tryfirst wrapper
```

**调用时逆序遍历**，所以实际执行顺序为：

```
非wrapper (tryfirst → 普通 → trylast) → wrapper (tryfirst → 普通 → trylast)
```

即：
- `tryfirst` 的非 wrapper 最先执行
- `trylast` 的非 wrapper 最后执行
- wrapper 在所有非 wrapper 之后执行
- wrapper 内部也按 `tryfirst → 普通 → trylast` 排序

#### 3.3.2 `_multicall()` 执行逻辑（pluggy/_callers.py:76）

```python
def _multicall(hook_name, hook_impls, caller_kwargs, firstresult):
    results = []
    exception = None
    try:
        teardowns = []
        try:
            for hook_impl in reversed(hook_impls):  # 逆序遍历
                args = [caller_kwargs[argname] for argname in hook_impl.argnames]

                if hook_impl.hookwrapper:            # 旧式 hookwrapper
                    function_gen = run_old_style_hookwrapper(...)
                    next(function_gen)               # 执行到 yield
                    teardowns.append(function_gen)

                elif hook_impl.wrapper:              # 新式 wrapper
                    res = hook_impl.function(*args)
                    next(res)                        # 执行到 yield
                    teardowns.append(res)

                else:                                # 普通 hook 实现
                    res = hook_impl.function(*args)
                    if res is not None:
                        results.append(res)
                        if firstresult:
                            break
        except BaseException as exc:
            exception = exc
    finally:
        # 收集结果
        result = results[0] if firstresult else results

        # 反向执行所有 wrapper 的 yield 后代码
        for teardown in reversed(teardowns):
            if exception is not None:
                teardown.throw(exception)
            else:
                teardown.send(result)
    ...
```

执行分为两个阶段：

1. **Setup 阶段**：逆序遍历所有 hook 实现。普通实现直接执行并收集结果；wrapper/hookwrapper 执行到 `yield` 暂停，保存生成器。
2. **Teardown 阶段**：反向遍历保存的生成器，将结果（或异常）发送给它们执行 yield 后的代码。

### 3.4 `hookwrapper` 装饰的 hook 如何工作

#### 3.4.1 旧式 `hookwrapper`

```python
@pytest.hookimpl(hookwrapper=True)
def pytest_collection_modifyitems(session, config, items):
    print("before collection modify")      # 在所有非 wrapper 实现之前执行
    result = yield                         # 暂停，让其他 hook 实现执行
    print("after collection modify")       # 所有非 wrapper 实现之后执行
    # result 是 pluggy.Result 对象
    # result.get_result() 获取返回值
    # 如果有异常，result.excinfo 不为 None
```

旧式 hookwrapper 通过 `run_old_style_hookwrapper()` 包装（_callers.py:25）。`yield` 接收到的是 `Result` 对象（包含 `.result` 和 `.excinfo`）。

#### 3.4.2 新式 `wrapper`

```python
@pytest.hookimpl(wrapper=True)
def pytest_runtest_protocol(item, nextitem):
    print("before runtest protocol")
    result = yield                         # 暂停，让其他 hook 实现执行
    print("after runtest protocol")
    # result 直接是返回值
    # 如果有异常，会通过 throw 抛出
    return result                          # 返回值成为 hook 的最终结果
```

新式 wrapper 的 `yield` 直接接收返回值（而非 `Result` 对象），异常通过 `throw` 抛出，更 Pythonic。

#### 3.4.3 执行示例：`pytest_collection_modifyitems`

假设有以下实现：

```python
# 内置 FixtureManager
@pytest.hookimpl(trylast=True)
def pytest_collection_modifyitems(self, items):
    items[:] = reorder_items(items)

# 用户插件 A
def pytest_collection_modifyitems(session, config, items):
    items[:] = [i for i in items if not i.keywords.get("skip_me")]

# 用户插件 B (hookwrapper)
@pytest.hookimpl(hookwrapper=True)
def pytest_collection_modifyitems(session, config, items):
    start = len(items)
    yield                                    # ← 暂停
    end = len(items)
    print(f"Filtered {start - end} items")   # ← 后处理
```

执行顺序：
1. **非 wrapper**（按 tryfirst → 普通 → trylast）：用户插件 A → 内置 FixtureManager
2. **wrapper**：用户插件 B 的 yield 前代码 → yield → yield 后代码

实际上因为 `reversed(hook_impls)` 的顺序，非 wrapper 的 tryfirst 先执行，所以完整顺序为：
- 如果 A 是 `tryfirst`：A → FixtureManager(trylast) → B(yield 前) → B(yield 后)
- 如果 A 是普通：取决于注册顺序

#### 3.4.4 执行示例：`pytest_runtest_protocol`

`pytest_runtest_protocol` 是 `firstresult=True` 的 hook，默认实现是 `runner.runtestprotocol()`。

```python
@hookspec(firstresult=True)
def pytest_runtest_protocol(item, nextitem): ...
```

默认实现（`src/_pytest/runner.py:123`）：

```python
def runtestprotocol(item, log=True, nextitem=None):
    rep = call_and_report(item, "setup", log)
    reports = [rep]
    if rep.passed:
        reports.append(call_and_report(item, "call", log))
    reports.append(call_and_report(item, "teardown", log, nextitem=nextitem))
    return reports
```

如果有 wrapper 插件：

```python
@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_protocol(item, nextitem):
    start_time = time.time()
    yield
    duration = time.time() - start_time
    print(f"Test {item.nodeid} took {duration}s")
```

执行流程：
1. wrapper 的 yield 前代码（记录开始时间）
2. `runtestprotocol()` 执行（setup → call → teardown）
3. wrapper 的 yield 后代码（计算并打印耗时）

由于 `firstresult=True`，`_multicall` 会在第一个非 wrapper 返回非 `None` 值后停止调用其他非 wrapper 实现。但 wrapper 不受此影响——所有 wrapper 都会执行。

### 3.5 `PytestPluginManager` 的额外功能

除了标准 `PluginManager` 功能外，`PytestPluginManager` 还提供：

1. **conftest 加载**：`_loadconftestmodules()` / `_importconftest()` 按 conftest 在目录树中的位置加载，子目录的 conftest 可以覆盖父目录的 hook 实现。
2. **`pytest_plugins` 变量支持**：`consider_module()` 读取模块中的 `pytest_plugins` 变量，自动加载引用的插件。
3. **`-p no:plugin` 禁用插件**：`consider_preparse()` 处理命令行 `-p` 参数。
4. **Legacy mark 兼容**：`_get_legacy_hook_marks()` 支持用 `@pytest.mark.tryfirst` 等标记替代 `@hookimpl(tryfirst=True)`（已弃用但兼容）。

---

## 总结

三大机制的关系：

- **Fixture 机制**：通过 `FuncFixtureInfo` 在收集阶段静态构建依赖图，在执行阶段通过 `FixtureRequest._get_active_fixturedef()` → `FixtureDef.execute()` → `pytest_fixture_setup` 递归解析和注入。`cache_key` 确保参数化场景下同一参数复用同一实例，scope 控制缓存的生命周期。
- **Assertion Rewriting**：通过 `AssertionRewritingHook` 拦截 Python import，在 `find_spec()` 中判断是否需要改写，在 `exec_module()` 中通过 `AssertionRewriter.visit_Assert()` 改写 AST。pyc 缓存加速重复加载。
- **Hook 体系**：基于 pluggy，`@hookspec` 声明规范，`@hookimpl`（或隐式 `pytest_` 前缀）注册实现。`_multicall()` 按 tryfirst/trylast/wrapper 分层排序执行，hookwrapper 通过 yield 机制实现后处理。
