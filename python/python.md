https://github.com/hantmac/Python-Interview-Customs-Collection


**题1：下面输出什么？**

```python
def g():
    print("a")
    yield 1
    print("b")
    yield 2

gen = g()
print("c")
print(next(gen))
```

答案：`c` → `a` → `1`。因为调用 `g()` 不执行函数体。

**题2：生成器表达式与列表推导的区别？**

- 列表推导立即求值，返回 list
- 生成器表达式惰性求值，返回 generator，只能迭代一次

**题3：`yield` 和 `return` 区别？**

- `return` 结束函数并返回值
- `yield` 暂停函数，保留状态，下次继续，可多次产出







### 字典底层

Python 3.7+ 保证插入顺序。3.6 起用紧凑 dict 实现，内存更省。哈希冲突用开放寻址。

---



---

## 十一、函数与高级特性



## 十二、数据结构与算法

