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

- pop()从列表尾部删除一个元素，并将元素作为返回值返回。实际上也可以指定pop哪一个元素，只需要传递下标即可。

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

- 复制列表
通过创建一个包含整个列表的切片，方法是同时省略起始索引和终止索引([:])。(直接赋值，是不能赋值列表的，两个变量将指向同一列表)  

  ```python
  my_food = ['pizza', 'falafel', 'carrot cake']
  friend_foods = my_food[:]

  print("my favorite foods are:")
  print(my_food)

  friend_foods.append('candy')
  print("my friend favorite foods are:")
  print(friend_foods)
  ```  

- 元祖
列表在程序运行期间是可变的，列表是可以被修改的。然后有时候需要创建一系列不可修改的元素，元祖可以满足这种需求。Python将不能被修改的值称为不可变的，而不可变的列表被称为元祖。

- 定义元祖(Tuples)
元祖使用圆括号来标识，可使用下标来访问
```python
#元祖
dimensions = (200, 50)
print(dimensions[0])
print(dimensions[1])
#下行代码会报错，因为元祖不允许修改
#dimensions[1] = 30
```

- 修改元祖变量
虽然不能修改元祖的元素，但可以给存储元祖的变量赋值。  

  ```python
  #元祖
  dimensions = (200, 50)
  print(dimensions[0])
  print(dimensions[1])
  #下行代码会报错，因为元祖不允许修改
  # dimensions[1] = 30
  for item in dimensions:
      print(item)

  dimensions = (300, 400)
  for item in dimensions:
      print(item)
  ```  

- if语句

  ```python
  cars = ['audi', 'bmw', 'subru', 'toyota']
  for car in cars:
      if car == 'bmw':
          print(car.upper())
      else:
          print(car.title())
  ```
if后面的语句称为条件语句，如果要同时判断多个条件是否都为True，可以使用and关键词,类似的还有or

  ```python
  cars = ['audi', 'bmw', 'subru', 'toyota']

  for car in cars:
      if car == 'bmw' and len(car) <= 3:
          print(car.upper())
      else:
          print(car.title())
  ```
- 检查特定的值是否包含在列表中
可以使用in 关键词，相反的可以使用not in关键词  

  ```Python
  num = [1, 2, 3, 4, 6]

  print(1 in num)
  print(5 in num)
  print(5 not in num)
  ```

  ```python
  #输出结果
  True
  False
  True
  ```
- if-elif-else结构  
  elif可以有多个，如果不需要，最后的else也可以不写
  ```Python
    age = 12
    if age < 4:
        print("Your admission cost is $0.")
    elif age < 18:
        print("Your admission cost is $5.")
    else:
        print("Your admission cost is $10.")
  ```

> if语句中将列表名用在条件语句时，Python将在列表至少包含一个元素是返回True，并在列表为空是返回False
```Python
request_toppings = []
if request_toppings:
    for request_topping in request_toppings:
        print("Add " + request_topping + ".")
else:
    print("Are you sure you want a plain pizza?")
```  

  ```Python
  #运行结果
  Are you sure you want a plain pizza?
  ```
