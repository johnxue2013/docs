- python使用两个**表示乘方运算
- 当数字和字符串拼接时，需要显式调用str(数字变量/数字)告诉python将非字符串值表示为字符串
```python
age = 23
message = "hello" + str(age)
print(message)
```
- python注释使用#标识
- []表示列表，并用逗号分隔其中的元素,可通过下标访问列表里的元素,可以通过-1下标访问列表最后一个元素。以此类推，可以通过-2访问倒数第二个元素。**列表并不要求所有元素是同一类型**。列表的元素是可以动态变化的
```python
#python中的列表
bicycles = ['trek', 'cannonde', 'redline',1]
print(bicycles)
print(bicycles[-1])
```

- 通过append函数将元素添加至列表末尾  

```python
#python中的列表
bicycles = ['trek', 'cannonde', 'redline',1]
print(bicycles)
print(bicycles[-1])

bicycles.append('jieante')

print(bicycles)
```

- insert在列表中插入元素  

```python
bicycles = ['trek', 'cannonde', 'redline',1]
print(bicycles)
bicycles.insert(0, 'jieante')
print(bicycles)
```
插入后将导致列表中响应位置元素向右移动一个位置

- 使用del语句根据值或者位置来删除元素
```python
motobicycle = ['honda', 'yamaha', 'suzuki']
print(motobicycle)
del motobicycle[0]
print(motobicycle)
```

- pop()从列表尾部删除一个元素，并将元素作为返回值返回。十几张也可以指定pop哪一个元素，只需要传递下标即可。

- remove()可以根据元素值删除元素。（只是删除第一次出现的元素，如果一个元素多次出现，则要循环删除）

- 可以使用sort()方法对列表进行永久性排序，参数sort(reverse=true)可以反序。相应的sorted()可以对列表进行临时性排序。

- for循环
```python
magicians = ['alice', 'david', 'carolina']
for magician in magicians:
    print(magician)
```

- range()函数可以轻松生成一系列数字。
```python
for value in range(1, 5):
    print(value)
```

- 函数list()将range()作为list的参数，输出将为一个数字列表。使用range时还可指定步长
```python
numbers = list(range(1, 5))
print(numbers)

#指定步长
numbers = list(range(1,5, 2))
print(numbers)
```

- 切片，可以使用列表的部分元素
```python
squares = []
for value in range(1, 11):
    square = value ** 2
    squares.append(square)

print(squares)
#如果没有指定起始位置，切片将从0开始，如果没有指定结尾，则切片到最后一个元素
part_square = squares[0:2]
print(part_square)
```
