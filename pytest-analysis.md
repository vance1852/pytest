# pytest 源码深度分析

本文档深入分析 pytest 的三个核心机制：fixture 解析注入、assertion rewriting 和插件钩子体系。

---

## 1. Fixture 解析与注入机制

### 1.1 整体执行链路

当测试函数 `def test_foo(fixture_a, fixture_b)` 执行时，fixture 的解析和注入遵循以下完整链路：

```
pytest_runtest_setup
  └── SetupState.setup(item)
        └── item.setup()
              └── TopRequest._fillfixtures()
                    └── getfixturevalue(argname)
                          └── _get_active_fixturedef(argname)
                                └── fixturedef.execute(request=subrequest)
                                      └── pytest_fixture_setup 钩子
                                            └── pytest_fixture_setup()  [fixtures.py:1237]
                                                  └── call_fixture_func()
```

### 1.2 依赖图构建：FuncFixtureInfo

#### 1.2.1 FuncFixtureInfo 数据结构

`FuncFixtureInfo` [fixtures.py:352] 是在**收集阶段**构建的 fixture 依赖信息对象：

```python
@dataclasses.dataclass(frozen=True)
class FuncFixtureInfo:
    __slots__ = ("argnames", "initialnames", "name2fixturedefs", "names_closure")
    
    # 测试函数直接请求的 fixture 名称（函数参数）
    argnames: tuple[str, ...]
    
    # 初始需要的 fixture 名称（包含 argnames + usefixtures + autouse）
    initialnames: tuple[str, ...]
    
    # 传递闭包：所有需要的 fixture（包括被依赖的 fixture）
    names_closure: list[str]
    
    # fixture 名称到 FixtureDef 列表的映射
    name2fixturedefs: dict[str, Sequence[FixtureDef[Any]]]
```

#### 1.2.2 依赖图构建过程

构建入口在 `FixtureManager.getfixtureinfo()` [fixtures.py:1661]：

1. **解析初始 fixture 名称**：
   - `argnames`：通过 `getfuncargnames()` 从函数签名解析
   - `usefixturesnames`：从 `@pytest.mark.usefixtures` 标记获取
   - `autousenames`：从作用域链上的 autouse fixtures 获取
   - `initialnames = deduplicate_names(autousenames, usefixturesnames, argnames)`

2. **构建依赖闭包** [fixtures.py:1739]：
   ```python
   def getfixtureclosure(self, parentnode, initialnames, ignore_args):
       # 使用 traverse_fixture_closure 进行 DFS 遍历
       fixturenames_closure = sorted(
           traverse_fixture_closure(initialnames, getfixturedefs=getfixturedefs),
           key=sort_by_scope,
           reverse=True  # 按 scope 从大到小排序
       )
   ```

3. **traverse_fixture_closure 算法** [fixtures.py:303]：
   - 使用 DFS 递归遍历 fixture 依赖
   - 处理 fixture override 链（同名 fixture 可以被更内层作用域的 fixture 覆盖）
   - 每个 argname 只 yield 一次，确保去重

### 1.3 执行链路详解

#### 1.3.1 从 pytest_fixture_setup 钩子开始

`pytest_fixture_setup` 钩子定义在 [hookspec.py:833]：

```python
@hookspec(firstresult=True)
def pytest_fixture_setup(
    fixturedef: FixtureDef[Any], request: SubRequest
) -> object | None:
    """Perform fixture setup execution."""
```

#### 1.3.2 pytest_fixture_setup 默认实现

默认实现位于 [fixtures.py:1237]：

```python
def pytest_fixture_setup(fixturedef, request):
    kwargs = {}
    # 递归解析当前 fixture 依赖的其他 fixture
    for argname in fixturedef.argnames:
        kwargs[argname] = request.getfixturevalue(argname)
    
    fixturefunc = resolve_fixture_function(fixturedef, request)
    my_cache_key = fixturedef.cache_key(request)
    
    # 执行 fixture 函数
    result = call_fixture_func(fixturefunc, request, kwargs)
    
    # 缓存结果
    fixturedef.cached_result = (result, my_cache_key, None)
    return result
```

#### 1.3.3 FixtureRequest.getfixturevalue()

[fixtures.py:566] 是获取 fixture 值的核心入口：

```python
def getfixturevalue(self, argname: str) -> Any:
    fixturedef = self._get_active_fixturedef(argname)
    return fixturedef.cached_result[0]
```

#### 1.3.4 FixtureRequest._get_active_fixturedef()

[fixtures.py:609] 核心逻辑：

```python
def _get_active_fixturedef(self, argname: str):
    # 1. 特殊处理 "request" fixture
    if argname == "request":
        return RequestFixtureDef(self)
    
    # 2. 检查是否已经解析过
    fixturedef = self._fixture_defs.get(argname)
    if fixturedef is not None:
        return fixturedef
    
    # 3. 查找 FixtureDef 列表
    fixturedefs = self._arg2fixturedefs.get(argname)
    if fixturedefs is None:
        fixturedefs = self._fixturemanager.getfixturedefs(argname, self._pyfuncitem)
    
    # 4. 处理 fixture override 链
    # 计算当前在 override 链中的深度
    index = -1
    for request in self._iter_chain():
        if request.fixturename == argname:
            index -= 1
    
    fixturedef = fixturedefs[index]
    
    # 5. 处理参数化
    if callspec is not None and argname in callspec.params:
        param = callspec.params[argname]
        param_index = callspec.indices[argname]
        scope = callspec._arg2scope[argname]
    else:
        param = NOTSET
        scope = fixturedef._scope
    
    # 6. 创建 SubRequest 并执行
    subrequest = SubRequest(self, scope, param, param_index, fixturedef)
    fixturedef.execute(request=subrequest)
    
    self._fixture_defs[argname] = fixturedef
    return fixturedef
```

#### 1.3.5 FixtureDef.execute()

[fixtures.py:1114] 执行 fixture 并处理缓存：

```python
def execute(self, request: SubRequest):
    # 1. 先解析依赖的 fixture（这可能导致当前缓存失效）
    requested_fixtures_that_should_finalize_us = []
    for argname in self.argnames:
        fixturedef = request._get_active_fixturedef(argname)
        requested_fixtures_that_should_finalize_us.append(fixturedef)
    
    # 2. 检查缓存
    if self.cached_result is not None:
        request_cache_key = self.cache_key(request)
        cache_key = self.cached_result[1]
        
        # 比较 cache_key
        try:
            cache_hit = bool(request_cache_key == cache_key)
        except (ValueError, RuntimeError):
            cache_hit = request_cache_key is cache_key
        
        if cache_hit:
            # 返回缓存值或重新抛出缓存的异常
            if self.cached_result[2] is not None:
                exc, exc_tb = self.cached_result[2]
                raise exc.with_traceback(exc_tb)
            else:
                return self.cached_result[0]
        
        # cache_key 不匹配，先 tear down 旧实例
        self.finish(request)
    
    # 3. 注册 finalizer
    finalizer = functools.partial(self.finish, request=request)
    for parent_fixture in requested_fixtures_that_should_finalize_us:
        parent_fixture.addfinalizer(finalizer)
    
    # 4. 调用 pytest_fixture_setup 钩子
    ihook = request.node.ihook
    result = ihook.pytest_fixture_setup(fixturedef=self, request=request)
    
    return result
```

### 1.4 Scope 层级与缓存生命周期

#### 1.4.1 Scope 定义

Scope 枚举定义了 5 个层级 [scope.py]：

```
function < class < module < package < session
```

#### 1.4.2 Scope 节点映射

`get_scope_node()` [fixtures.py:125] 将 scope 映射到 pytest 节点：

| Scope | 节点类型 | 生命周期 |
|-------|----------|----------|
| function | Item (Function) | 单个测试函数 |
| class | Class | 测试类中的所有测试 |
| module | Module | 模块中的所有测试 |
| package | Package | 包中的所有测试 |
| session | Session | 整个测试会话 |

#### 1.4.3 SetupState 管理作用域生命周期

[runner.py:439] `SetupState` 使用栈管理节点的 setup/teardown：

```
假设测试顺序: test_foo (mod1) -> test_bar (mod2)

栈变化:
1. []
2. setup(test_foo): [session, mod1, test_foo]
3. teardown_exact(test_bar): [session]  (mod1 和 test_foo 被 tear down)
4. setup(test_bar): [session, mod2, test_bar]
5. teardown_exact(None): []
```

#### 1.4.4 缓存失效时机

FixtureDef 的 `cached_result` 在以下情况被置为 `None`：

1. **scope 结束时**：通过 `finish()` 方法
2. **参数变化时**：cache_key 不匹配时调用 `finish()`
3. **依赖的 fixture 失效时**：通过 parent fixture 的 finalizer 链

### 1.5 cache_key 与参数化复用

#### 1.5.1 cache_key 的计算

[fixtures.py:1184] `FixtureDef.cache_key()`：

```python
def cache_key(self, request: SubRequest) -> object:
    return getattr(request, "param", None)
```

对于参数化的 fixture：
- `SubRequest` 持有 `self.param` 属性
- `cache_key` 就是 `request.param`

#### 1.5.2 参数化场景下的缓存复用

假设有：
```python
@pytest.fixture(params=[1, 2, 3])
def my_fixture(request):
    return request.param
```

执行流程：
1. 测试 1 用 `param=1`：`cache_key=1`，执行 fixture，缓存 `(value=1, key=1, None)`
2. 测试 2 用 `param=1`：`cache_key=1`，命中缓存，直接返回
3. 测试 3 用 `param=2`：`cache_key=2` ≠ 缓存的 `1`，先 `finish()` 旧值，再执行 fixture

#### 1.5.3 测试项重排序优化

`reorder_items()` [fixtures.py:217] 会将使用相同高 scope 参数的测试项放在一起，减少高 scope fixture 的 setup/teardown 次数：

```python
def reorder_items(items: Sequence[nodes.Item]) -> list[nodes.Item]:
    # 按 ParamArgKey 分组，相同参数的测试项连续执行
    # 这样高 scope fixture 可以被复用
```

---

## 2. Assertion Rewriting 机制

### 2.1 整体架构

```
import test_module
    │
    └── sys.meta_path 查找
          │
          └── AssertionRewritingHook.find_spec()
                │
                ├── 检查是否需要改写 (_should_rewrite)
                │
                └── 返回自定义 spec (loader=self)
                      │
                      └── AssertionRewritingHook.exec_module()
                            │
                            ├── 检查 pyc 缓存 (_read_pyc)
                            │     ├── 命中：直接执行缓存代码
                            │     └── 未命中：执行改写
                            │
                            ├── _rewrite_test()
                            │     ├── 解析 AST
                            │     ├── AssertionRewriter 改写 assert 语句
                            │     └── 编译成 code object
                            │
                            ├── 写入 pyc 缓存 (_write_pyc)
                            │
                            └── exec(code, module.__dict__)
```

### 2.2 AssertionRewritingHook

#### 2.2.1 安装时机

[assertion/__init__.py:116] `install_importhook()`：

```python
def install_importhook(config: Config):
    hook = AssertionRewritingHook(config)
    sys.meta_path.insert(0, hook)  # 插入到最前面
    return hook
```

#### 2.2.2 find_spec() 方法

[rewrite.py:102] 拦截模块导入：

```python
def find_spec(self, name, path, target):
    # 1. 快速过滤（性能优化）
    if self._early_rewrite_bailout(name, state):
        return None
    
    # 2. 使用 PathFinder 查找 spec
    spec = self._find_spec(name, path)
    
    # 3. 有效性检查
    if (spec is None
        or spec.origin is None
        or not isinstance(spec.loader, importlib.machinery.SourceFileLoader)
        or not os.path.exists(spec.origin)):
        return None
    
    fn = spec.origin
    
    # 4. 检查是否需要改写
    if not self._should_rewrite(name, fn, state):
        return None
    
    # 5. 返回自定义 spec，loader 设为 self
    return importlib.util.spec_from_file_location(
        name, fn, loader=self,
        submodule_search_locations=spec.submodule_search_locations,
    )
```

#### 2.2.3 _should_rewrite() 逻辑

[rewrite.py:229] 决定哪些模块需要改写：

```python
def _should_rewrite(self, name: str, fn: str, state: AssertionState) -> bool:
    # 1. 总是改写 conftest.py
    if os.path.basename(fn) == "conftest.py":
        return True
    
    # 2. 命令行显式指定的文件
    if self.session is not None:
        if self.session.isinitpath(absolutepath(fn)):
            return True
    
    # 3. 匹配 test 文件命名模式 (python_files 配置)
    fn_path = PurePath(fn)
    for pat in self.fnpats:  # 默认: ["test_*.py", "*_test.py"]
        if fnmatch_ex(pat, fn_path):
            return True
    
    # 4. 通过 mark_rewrite() 标记的模块
    return self._is_marked_for_rewrite(name, state)
```

**为什么只改写 conftest 和 test 模块？**
- 性能考量：改写所有模块会显著降低导入速度
- 正确性：第三方库的 assert 可能依赖特定行为，改写可能破坏它们
- 按需改写：插件可以通过 `register_assert_rewrite()` 标记需要改写的模块

#### 2.2.4 exec_module() 方法

[rewrite.py:148] 执行改写后的代码：

```python
def exec_module(self, module: types.ModuleType) -> None:
    fn = Path(module.__spec__.origin)
    self._rewritten_names[module.__name__] = fn
    
    write = not sys.dont_write_bytecode
    cache_dir = get_cache_dir(fn)
    
    # pyc 文件名包含 pytest 版本号
    cache_name = fn.name[:-3] + PYC_TAIL
    pyc = cache_dir / cache_name
    
    # 尝试读取缓存
    co = _read_pyc(fn, pyc, state.trace)
    
    if co is None:
        # 缓存未命中，执行改写
        source_stat, co = _rewrite_test(fn, self.config)
        if write:
            self._writing_pyc = True
            try:
                _write_pyc(state, co, source_stat, pyc)
            finally:
                self._writing_pyc = False
    
    # 执行代码
    exec(co, module.__dict__)
```

### 2.3 pyc 缓存机制

#### 2.3.1 PYC 文件命名

[rewrite.py:68] 特殊的文件名格式：

```python
PYTEST_TAG = f"{sys.implementation.cache_tag}-pytest-{version}"
# 例如: cpython-312-pytest-7.4.0
PYC_TAIL = "." + PYTEST_TAG + PYC_EXT
# 例如: test_foo.cpython-312-pytest-7.4.0.pyc
```

这样可以与标准 Python pyc 共存。

#### 2.3.2 _write_pyc()

[rewrite.py:318] 写入缓存：

```python
def _write_pyc(state, co, source_stat, pyc):
    # 1. 先写入临时文件
    proc_pyc = f"{pyc}.{os.getpid()}"
    with open(proc_pyc, "wb") as fp:
        _write_pyc_fp(fp, source_stat, co)
        # 格式: MAGIC_NUMBER + flags + mtime + size + marshal.dumps(co)
    
    # 2. 原子重命名（避免并发问题）
    os.replace(proc_pyc, pyc)
```

#### 2.3.3 _read_pyc()

[rewrite.py:354] 读取并验证缓存：

```python
def _read_pyc(source, pyc, trace):
    try:
        fp = open(pyc, "rb")
    except OSError:
        return None
    
    with fp:
        # 读取 16 字节头部
        data = fp.read(16)
        
        # 验证魔数
        if data[:4] != importlib.util.MAGIC_NUMBER:
            return None
        
        # 验证 mtime 和 size
        stat_result = os.stat(source)
        mtime = int(stat_result.st_mtime)
        size = stat_result.st_size
        
        if int.from_bytes(data[8:12], "little") != mtime & 0xFFFFFFFF:
            return None  # 源文件已修改
        
        if int.from_bytes(data[12:16], "little") != size & 0xFFFFFFFF:
            return None  # 源文件大小已变
        
        # 反序列化 code object
        co = marshal.load(fp)
        return co
```

### 2.4 AST 改写详解

#### 2.4.1 _rewrite_test()

[rewrite.py:343] 执行改写：

```python
def _rewrite_test(fn: Path, config: Config):
    stat = os.stat(fn)
    source = fn.read_bytes()
    strfn = str(fn)
    
    # 1. 解析为 AST
    tree = ast.parse(source, filename=strfn)
    
    # 2. 改写 assert 语句
    rewrite_asserts(tree, source, strfn, config)
    
    # 3. 编译成 code object
    co = compile(tree, strfn, "exec", dont_inherit=True)
    return stat, co
```

#### 2.4.2 AssertionRewriter 类

[rewrite.py:606] AST NodeVisitor：

```python
class AssertionRewriter(ast.NodeVisitor):
    def run(self, mod: ast.Module):
        # 1. 插入特殊 import
        # import builtins as @py_builtins
        # import _pytest.assertion.rewrite as @pytest_ar
        imports = [
            ast.Import([ast.alias("builtins", "@py_builtins")]),
            ast.Import([ast.alias("_pytest.assertion.rewrite", "@pytest_ar")]),
        ]
        mod.body[pos:pos] = imports
        
        # 2. 遍历所有语句，查找 assert 并改写
        nodes = [mod]
        while nodes:
            node = nodes.pop()
            if isinstance(node, ast.Assert):
                # 用改写后的语句替换原 assert
                new.extend(self.visit(child))
            else:
                new.append(child)
```

#### 2.4.3 visit_Assert() 核心方法

[rewrite.py:845] 改写 assert 语句：

```python
def visit_Assert(self, assert_: ast.Assert) -> list[ast.stmt]:
    self.statements = []
    self.variables = []
    self.variable_counter = itertools.count()
    self.stack = []
    self.expl_stmts = []
    self.push_format_context()
    
    # 1. 改写断言表达式，获取条件和解释字符串
    top_condition, explanation = self.visit(assert_.test)
    
    # 2. 构建 if not condition: 分支
    negation = ast.UnaryOp(ast.Not(), top_condition)
    
    # 3. 构建错误消息
    template = ast.BinOp(assertmsg, ast.Add(), ast.Constant(explanation))
    msg = self.pop_format_context(template)
    fmt = self.helper("_format_explanation", msg)
    
    # 4. 构建 raise AssertionError
    err_name = ast.Name("AssertionError", ast.Load())
    exc = ast.Call(err_name, [fmt], [])
    raise_ = ast.Raise(exc, None)
    
    body.append(raise_)
    
    # 5. 清除临时变量
    if self.variables:
        variables = [ast.Name(name, ast.Store()) for name in self.variables]
        clear = ast.Assign(variables, ast.Constant(None))
        self.statements.append(clear)
    
    return self.statements
```

#### 2.4.4 改写示例

原始代码：
```python
def test_foo():
    a = 1
    b = 2
    assert a == b
```

改写后（概念上）：
```python
import builtins as @py_builtins
import _pytest.assertion.rewrite as @pytest_ar

def test_foo():
    a = 1
    b = 2
    
    # 改写后的 assert
    @py_assert0 = a
    @py_assert1 = b
    @py_assert2 = @py_assert0 == @py_assert1
    
    if not @py_assert2:
        @py_format0 = "%(py0)s == %(py1)s" % {
            "py0": @pytest_ar._saferepr(@py_assert0),
            "py1": @pytest_ar._saferepr(@py_assert1),
        }
        raise AssertionError(
            @pytest_ar._format_explanation(
                "assert " + @py_format0
            )
        )
    
    @py_assert0 = @py_assert1 = @py_assert2 = None
```

#### 2.4.5 visit_Compare() 处理比较操作

[rewrite.py:1101] 详细处理比较表达式：

```python
def visit_Compare(self, comp: ast.Compare):
    self.push_format_context()
    
    left_res, left_expl = self.visit(comp.left)
    
    # 遍历每个比较器
    for i, op, next_operand in zip(range(len(comp.ops)), comp.ops, comp.comparators):
        next_res, next_expl = self.visit(next_operand)
        
        # 执行比较并保存结果
        res_expr = ast.Compare(left_res, [op], [next_res])
        self.statements.append(ast.Assign([store_names[i]], res_expr))
        
        # 生成解释字符串
        sym = BINOP_MAP[op.__class__]  # "==", "!=", etc.
        expl = f"{left_expl} {sym} {next_expl}"
        expls.append(ast.Constant(expl))
    
    # 调用 _call_reprcompare 获取格式化的比较解释
    expl_call = self.helper(
        "_call_reprcompare",
        ast.Tuple(syms, ast.Load()),
        ast.Tuple(load_names, ast.Load()),
        ast.Tuple(expls, ast.Load()),
        ast.Tuple(results, ast.Load()),
    )
    
    return res, self.explanation_param(self.pop_format_context(expl_call))
```

---

## 3. 插件钩子体系（基于 pluggy）

### 3.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                    PytestPluginManager                      │
│  (继承自 pluggy.PluginManager)                              │
│                                                             │
│  ┌─────────────────┐    ┌──────────────────────────────┐   │
│  │  hookspecs      │    │  plugins                     │   │
│  │  (钩子规范)      │    │  (插件实现)                  │   │
│  │                 │    │                              │   │
│  │ @hookspec       │    │ @hookimpl                    │   │
│  │ def pytest_xxx  │    │ def pytest_xxx              │   │
│  └─────────────────┘    └──────────────────────────────┘   │
│                                                             │
│  hook 调用: pluginmanager.hook.pytest_xxx(...)              │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Hook 声明：hookspec.py

#### 3.2.1 标记器定义

[hookspec.py:46]：

```python
hookspec = HookspecMarker("pytest")
```

`HookspecMarker` 来自 pluggy，用于标记钩子规范。

#### 3.2.2 示例：pytest_collection_modifyitems

[hookspec.py:268]：

```python
def pytest_collection_modifyitems(
    session: Session, config: Config, items: list[Item]
) -> None:
    """Called after collection has been performed. May filter or re-order
    the items in-place.
    """
```

这是一个**普通钩子**（没有 `firstresult=True`），所有实现都会被调用。

#### 3.2.3 示例：pytest_runtest_protocol

[hookspec.py:619]：

```python
@hookspec(firstresult=True)
def pytest_runtest_protocol(item: Item, nextitem: Item | None) -> object | None:
    """Perform the runtest protocol for a single test item.
    
    Stops at first non-None result, see :ref:`firstresult`.
    """
```

这是一个 **firstresult** 钩子，遇到第一个非 None 返回值就停止。

#### 3.2.4 hookspec 装饰器参数

| 参数 | 说明 |
|------|------|
| `firstresult=True` | 只取第一个非 None 结果 |
| `historic=True` | 历史钩子，新注册的插件也能收到之前的调用 |
| `warn_on_impl=Warning` | 实现时发出警告 |

### 3.3 插件注册

#### 3.3.1 PytestPluginManager

[config/__init__.py:418]：

```python
class PytestPluginManager(PluginManager):
    def __init__(self):
        super().__init__("pytest")  # "pytest" 是插件命名空间
        
        # 添加 pytest 内置的 hookspec
        self.add_hookspecs(_pytest.hookspec)
        
        # 注册自己
        self.register(self)
```

#### 3.3.2 插件注册流程

插件可以通过多种方式注册：

1. **命令行**：`-p plugin_name`
2. **环境变量**：`PYTEST_PLUGINS=plugin1,plugin2`
3. **conftest.py 中的 pytest_plugins**
4. **entry_points**：`pytest11` group

#### 3.3.3 register() 方法

（来自 pluggy.PluginManager）

```python
def register(self, plugin, name=None):
    """注册一个插件
    
    1. 扫描插件中的所有 @hookimpl 装饰的函数
    2. 将它们与对应的 hookspec 关联
    3. 存储在内部字典中
    """
```

### 3.4 多个 Hook 实现的执行顺序

#### 3.4.1 hookimpl 装饰器参数

```python
@hookimpl(tryfirst=True)    # 优先执行
@hookimpl(trylast=True)     # 最后执行  
@hookimpl(hookwrapper=True) # 包装器，包裹其他实现
@hookimpl(optionalhook=True) # 可选，没有对应 hookspec 也不报错
```

#### 3.4.2 执行顺序示例

假设有 5 个实现：

```
A: @hookimpl(tryfirst=True)
B: @hookimpl
C: @hookimpl(hookwrapper=True)
D: @hookimpl
E: @hookimpl(trylast=True)
```

调用顺序（对于非 wrapper）：
```
tryfirst 组 → 普通组 → trylast 组
    A          B, D        E
```

### 3.5 HookWrapper 机制

#### 3.5.1 示例：pytest_runtest_protocol 的 wrapper

[assertion/__init__.py:142]：

```python
@hookimpl(wrapper=True, tryfirst=True)
def pytest_runtest_protocol(item: Item) -> Generator[None, object, object]:
    """Setup the pytest_assertrepr_compare and pytest_assertion_pass hooks."""
    
    # === pre-yield: 前置处理 ===
    ihook = item.ihook
    
    # 设置 util._reprcompare 以便改写的 assert 可以调用自定义比较
    def callbinrepr(op, left, right):
        hook_result = ihook.pytest_assertrepr_compare(
            config=item.config, op=op, left=left, right=right
        )
        # ... 处理结果
        return None
    
    saved_assert_hooks = util._reprcompare, util._assertion_pass
    util._reprcompare = callbinrepr
    util._config = item.config
    
    # === yield: 让下游 hook 执行 ===
    # yield 接收的是下游所有 hook 的结果
    try:
        return (yield)  # 返回下游的结果
    finally:
        # === post-yield: 后置处理 ===
        util._reprcompare, util._assertion_pass = saved_assert_hooks
        util._config = None
```

#### 3.5.2 HookWrapper 执行模型

```
hookwrapper 执行流程:

@hookimpl(hookwrapper=True)
def my_wrapper():
    # 1. pre-yield (setup)
    print("before")
    
    # 2. yield 暂停，让其他 hook 执行
    #    outcome 包含:
    #      - outcome.get_result(): 下游结果
    #      - outcome.excinfo: 异常信息
    outcome = yield
    
    # 3. post-yield (teardown)
    print("after")
    return outcome.get_result()  # 可以修改结果
```

#### 3.5.3 多个 wrapper 的嵌套

假设有两个 wrapper：

```
Wrapper A:
  print("A before")
  result = yield
  print("A after")
  return result

Wrapper B:
  print("B before")  
  result = yield
  print("B after")
  return result
```

执行顺序（洋葱模型）：

```
A before
  B before
    实际 hook 执行
  B after
A after
```

#### 3.5.4 与 tryfirst/trylast 结合

```
@hookimpl(hookwrapper=True, tryfirst=True)  # 最外层 wrapper
@hookimpl(hookwrapper=True)                # 中间 wrapper
@hookimpl(hookwrapper=True, trylast=True)  # 最内层 wrapper
```

### 3.6 示例：pytest_collection_modifyitems 调用

#### 3.6.1 内置实现

[fixtures.py:1818] `FixtureManager.pytest_collection_modifyitems()`：

```python
def pytest_collection_modifyitems(self, items: list[nodes.Item]):
    # 重排序测试项，优化高 scope fixture 复用
    items[:] = reorder_items(items)
```

#### 3.6.2 用户自定义实现

```python
# conftest.py
def pytest_collection_modifyitems(items):
    # 按测试名称排序
    items.sort(key=lambda x: x.name)
```

#### 3.6.3 调用时机

在 `main.py` 的收集流程中：

```
pytest_collection()
  └── ... 收集所有 items ...
        └── pytest_collection_modifyitems(session, config, items)
              └── 所有实现按顺序执行，都可以修改 items
```

### 3.7 示例：pytest_runtest_protocol 调用

#### 3.7.1 内置实现

[runner.py:115]：

```python
def pytest_runtest_protocol(item: Item, nextitem: Item | None) -> bool:
    ihook = item.ihook
    
    # 1. 日志开始
    ihook.pytest_runtest_logstart(nodeid=item.nodeid, location=item.location)
    
    # 2. 执行 runtest 协议 (setup → call → teardown)
    runtestprotocol(item, nextitem=nextitem)
    
    # 3. 日志结束
    ihook.pytest_runtest_logfinish(nodeid=item.nodeid, location=item.location)
    
    return True  # firstresult=True，返回非 None 停止
```

#### 3.7.2 调用流程

```
pytest_runtestloop(session)
  └── for item in session.items:
        └── pytest_runtest_protocol(item=item, nextitem=nextitem)
              │
              └── 因为是 firstresult：
                    1. 先调用 tryfirst 组
                    2. 再调用普通组  
                    3. 直到第一个返回非 None 的实现
                    4. 返回该结果
```

### 3.8 Pluggy 核心概念总结

| 概念 | 说明 |
|------|------|
| `HookspecMarker` | 定义钩子规范（接口） |
| `HookimplMarker` | 标记钩子实现 |
| `PluginManager` | 管理插件和钩子调用 |
| `firstresult` | 只取第一个非 None 结果 |
| `hookwrapper` | 用生成器包裹其他实现 |
| `tryfirst/trylast` | 控制执行顺序 |
| `historic` | 历史钩子，支持晚注册 |

---

## 总结

### Fixture 机制
- **收集阶段**：构建 `FuncFixtureInfo`，DFS 遍历生成依赖闭包，按 scope 排序
- **执行阶段**：`getfixturevalue()` → `_get_active_fixturedef()` → `execute()` → `pytest_fixture_setup`
- **缓存**：`cached_result` 存储 `(value, cache_key, exception)`，参数化时通过 `cache_key` 比较实现复用
- **Scope**：`SetupState` 用栈管理节点生命周期，scope 结束时通过 `finish()` 清理缓存

### Assertion Rewriting
- **拦截导入**：`AssertionRewritingHook` 作为 `MetaPathFinder` 插入 `sys.meta_path`
- **选择性改写**：`_should_rewrite()` 只处理 conftest、test 文件和显式标记的模块
- **AST 改写**：`AssertionRewriter.visit_Assert()` 将 `assert a == b` 改写为条件判断 + 详细错误信息
- **缓存**：特殊命名的 pyc 文件（包含 pytest 版本），通过 mtime 和 size 验证有效性

### 插件体系
- **声明**：`@hookspec` 在 `hookspec.py` 中定义接口
- **注册**：`PytestPluginManager.register()` 扫描 `@hookimpl` 实现
- **顺序**：`tryfirst` → 普通 → `trylast`
- **HookWrapper**：通过 yield 实现洋葱模型，可在下游执行前后插入逻辑，通过 `outcome` 获取结果
