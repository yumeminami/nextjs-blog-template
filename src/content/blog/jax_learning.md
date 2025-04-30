---
title: "JAX Learning"
date: "2025-04-25"
draft: false
keywords: ["robotics", "Python", "JAX"]
summary: "JAX 学习笔记， 学习使用 JAX 核心特性"
featured: true
---

## JAX 是什么

JAX 是一个用于高性能科学计算的 Python 库。它提供了自动微分、并行计算和分布式计算等功能，适用于机器学习、优化和科学计算等领域。

## JAX 的特点

- **grad** 自动微分
- **jit** 即时编译
- **pmap** 分布式计算
- **vmap** 向量化计算
- **lax** 用于构建和优化复杂的数学表达式
- **stax** 用于构建和优化神经网络

将介绍 JAX 的各个特性， 并给出示例代码(**不会详细介绍原理实现**， 而是从使用者的角度出发)

## 🔁 函数式自动微分（Autograd）

`jax.grad(fun)` 是 JAX 提供的一个核心函数转换（function transformation），它的作用是：

1. 接收一个 Python 函数 fun 作为输入。
2. 返回一个新的函数。
3. 当你调用这个新的函数时，它会计算并返回：
    - 原始函数 fun 在给定输入上的输出值。
    - 原始函数 fun 在给定输入上关于指定参数的梯度。


**grad 的参数：**

- `fun`: 要计算其值和梯度的函数。对于 grad 操作，这个函数的主要输出必须是一个标量（或者一个结构，你可以指定对其某个标量叶子进行微分）。
- `argnums`: 指定对 `fun` 的哪个或哪些输入参数计算梯度。
  - 默认为 0，即对第一个参数计算梯度。
  - 可以是一个整数（对该索引对应的参数）。
  - 可以是一个整数的元组或列表（对多个参数）。
  - 可以是一个 `pytree` 结构，其中包含你想计算梯度的参数的索引。


**grad 的返回值：**

- 返回一个元组 (grad)，其中：
  - grad 是 fun 在给定输入上关于指定参数的梯度。

**示例：**

```python
import jax
import jax.numpy as jnp
import time

# Define loss function
def loss_fn(w, b, x, y):
    y_pred = w * x + b
    loss = jnp.mean((y_pred - y) ** 2)
    return loss

key = jax.random.PRNGKey(42)

# Generate data
w = jax.random.normal(key, (1000,))
b = jax.random.normal(key, (1000,))
x = jax.random.normal(key, (1000,))

y_true = w * x + b
noise = jax.random.normal(key, (1000,)) * 0.1  # Adding some noise
y = y_true + noise

# use value_and_grad to get loss and gradient
# argnums=(0, 1) means compute gradient for w and b
# grad_fn = jax.value_and_grad(loss_fn, argnums=(0, 1), has_aux=True)
grad_fn = jax.grad(loss_fn, argnums=(0, 1))

# Compute loss and gradient
# loss, grad = grad_fn(w, b, x, y)
grad = grad_fn(w, b, x, y)

# print(loss)
print(grad)

```

- define a loss function, `loss_fn`, it returns loss
- create `grad_fn`, the `argnums=(0, 1)` means the first and second arguments are the ones to be differentiated(`w` and `b` in this case)
- call the `grad_fn`, `loss_fn` will forward compute the `loss` and `grad_fn` will BackPropagation compute the gradient `grad`. finally, it returns a tuple of (loss, grad)

## ⚡  Just-In-Time 编译（JIT）加速

`jax.jit` 是一个函数转换器，可以将 Python/JAX 代码编译成优化的、在加速器上高效运行的机器码。

被 `jax.jit` 装饰的函数需要遵循一些规则，最主要的是函数输入和内部控制流的形状必须是静态的（在编译时确定）。依赖于运行时值的动态控制流需要使用 `jax.lax` 提供的原语（如 `jax.lax.cond`, `jax.lax.while_loop`, `jax.lax.scan`）。

并且所有被装饰为 `jax.jit` 的函数， 函数中的所有 `print` 或者 `log` 语句都不会执行， 而是会在编译时被忽略。
取而代之则使用 ` jax.debug.print` 来打印信息。

🔒 注意事项

|限制|描述|
|---|---|
| not support f-string | 不支持 `"{} {}".format(x, y)`  |
| not keep order | 不能保证执行顺序 |



**示例：**

```python

import jax
import jax.numpy as jnp
import time


def multiply(x, y):
    return x * y


@jax.jit
def multiply_jit(x, y):
    return x * y

size = 1e7
x = jnp.ones(int(size))
y = jnp.ones(int(size))

print("---------No JIT---------")
start = time.time()
multiply(x, y)
end = time.time()
print(f"Time taken: {end - start} seconds")

print("---------JIT(include compile time)---------")
start = time.time()
res = multiply_jit(x, y)
res.block_until_ready()
end = time.time()
print(f"Time taken: {end - start} seconds")


print("---------JIT---------")
start = time.time()
res = multiply_jit(x, y)
res.block_until_ready()
end = time.time()
print(f"Time taken: {end - start} seconds")


# ---------No JIT---------
# Time taken: 0.02260136604309082 seconds
# ---------JIT(include compile time)---------
# Time taken: 0.022781848907470703 seconds
# ---------JIT---------
# Time taken: 0.002478361129760742 seconds
```

## 🔄 向量化函数（vmap）

使用 `jax.vmap` 自动将标量函数向量化，无需写 `for-loop`。

**示例：**

```python

import jax
import jax.numpy as jnp

def f(x):
    return x + 1

# No vmap
xs = jnp.array([1.0, 2.0, 3.0])
ys = jnp.array([f(x) for x in xs])

# vmap
vmap_f = jax.vmap(f)
ys = vmap_f(xs)

# multi-dimensional vmap
def f(x, y):
    return x + y
 
xs = jnp.array([1.0, 2.0, 3.0])
ys = jnp.array([1.0, 2.0, 3.0])
vmap_f = jax.vmap(f)
ys = vmap_f(xs, ys) # output: [2.0, 4.0, 6.0]
# or
ys = vmap_f(xs, 1) # output: [2.0, 3.0, 4.0]



# vmap with jit
@jax.jit
def loss_fn(w, x, y):
    pred = jnp.dot(w, x)
    return (pred - y) ** 2, 1

x_batch = jnp.array([[1,2,3], [4,5,6], [7,8,9]])  # shape = (3, 3)
y_batch = jnp.array([1.0, 2.0, 3.0])              # shape = (3,)
w = jnp.array([0.1, 0.2, 0.3])                    # shape = (3,)
grad_fn = jax.value_and_grad(loss_fn, argnums=(0,), has_aux=True)

(loss, _), grads = jax.vmap(grad_fn, in_axes=(None, 0, 0))(w, x_batch, y_batch)
print(loss)
print(grads)

```

详细介绍一下示例代码中最后一个 `vmap` 的用法：

- `jax.vmap(grad_fn, in_axes=(None, 0, 0))` 将 `grad_fn` 向量化，`in_axes=(None, 0, 0)` 表示 `grad_fn` 的第一个参数不需要向量化，第二个和第三个参数需要向量化。
- 相当于每次调用 `grad_fn` 时，输入的 `w` 不变，`x_batch[i]` 和 `y_batch[i]` 分别作为 `grad_fn` 的第二个和第三个参数。


## 💡 lax 核心算子库

`lax` 是 JAX 提供的一个核心算子库，用于构建和优化复杂的数学表达式。它提供了许多常用的数学运算，如矩阵乘法、卷积、归一化等。

•	构建更底层、可微分的控制流（替代 Python 原生的 if、for）
•	实现高性能的 XLA 原语（跨 CPU、GPU、TPU 都高效）
•	在 jit、grad、vmap 等组合中更高效稳定

介绍一些常用的 `lax` 算子：

- `lax.cond`：条件分支
- `lax.scan`：扫描
- `lax.select/lax.switch`：分支选择

🚀 1. 条件控制流：`lax.cond`

```python

import jax
from jax import lax
import jax.numpy as jnp

def f(x):
    return lax.cond(x > 0, 
                    lambda x: x**2,    # True 分支
                    lambda x: -x,      # False 分支
                    x)

# 测试
print(f(3.0))  # 9.0
print(f(-2.0))  # -2.0

```

- 适合替代 Python 中的 if/else，保证 JIT 编译时高效。
- 只支持二元条件分支， 不支持多元条件分支。

🚀 2. 扫描：`lax.scan`

`lax.scan` 是一个高效的循环操作, 高效地在 GPU 或 TPU 上执行。


```python

import jax
from jax import lax
import jax.numpy as jnp

def step(carry, x):
    return carry + x, carry + x

xs = jnp.arange(5)
carry = 0.0

carry, ys = lax.scan(step, carry, xs)

# carry = 10.0
# ys = [0.  1.  3.  6. 10.]

```

- `lax.scan` 接受一个函数，以及函数需要的参数，以及初始值 `carry`，以及输入序列 `xs`。
- 循环次数会自动根据 `xs` 的长度确定, 如果 `xs` 不存在， 也可以可以手动指定，例如 `lax.scan(step, carry, None, length=10)`。
- 每次循环中， `carry` 都会被更新， 并作为下一次循环的输入。`xs` 则会根据当前的循环次数 `xs[i]` 传入 `step` 函数。
- `lax.scan` 会保持一个更新的状态（`carry`），返回最终结果和所有中间结果 `ys`。


🚀 3. 选择：`lax.select` 和 `lax.switch`

**lax.select**

```python

import jax
from jax import lax
import jax.numpy as jnp

x = jnp.array([1.0, 2.0, 3.0])
y = jnp.array([4.0, 5.0, 6.0])
cond = jnp.array([True, False, True])

result = lax.select(cond, x, y)
print(result)  # [1.0, 5.0, 3.0]

``` 

- 根据 `cond` 的值， 选择 `x` 或 `y` 的值。适用于需要进行元素级选择的场合。
- 注意 `cond` 的形状需要和 `x` 和 `y` 的形状一致。


**lax.switch**

```python

import jax
from jax import lax
import jax.numpy as jnp

x = jnp.array([1, 2, 3])

def case_fn_0(x): return x * 2
def case_fn_1(x): return x + 5
def case_fn_2(x): return x - 1

result = lax.switch(x[0], [case_fn_0, case_fn_1, case_fn_2])
print(result)  # 3 (因为 1 属于 case_fn_0, 1 * 2 = 3)
```

- 根据 `x[0]` 的值， 选择 `case_fn_0`, `case_fn_1`, `case_fn_2` 中的一个函数执行。
- 适用于需要根据条件选择不同函数执行的场合。
- 注意 `x[0]` 的值需要是一个整数， 且在 `[0, n)` 之间， 其中 `n` 是 `case_fn` 的数量。
