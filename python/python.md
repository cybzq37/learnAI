
@staticmethod

@classmethod

鸭子类型



Python 中，一个变量的作用域总是由在代码中被赋值的地方所决定的。当 Python 遇到一个变量的话他会按照这样的顺序进行搜索：本地作用域（Local）→ 当前作用域被嵌入的本地作用域（Enclosing locals）→ 全局/模块作用域（Global）→ 内置作用域（Built-in）

线程全局锁(Global Interpreter Lock) 即Python为了保证线程安全而采取的独立线程运行的限制,说白了就是一个核只能在同一时间运行一个线程.对于io密集型任务，python的多线程起到作用，但对于cpu密集型任务，python的多线程几乎占不到任何优势，还有可能因为争夺资源而变慢。

Python里最常见的yield就是协程的思想

numpy 
pandas 
SciPy

做gis的话：
shapely  
gdal  
graphx  
