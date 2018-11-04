## xsd
```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!--xsd文件本身是xml文件，第一行是xml声明-->
<xsd:schema xmlns="http://www.springframework.org/schema/beans" xmlns:xsd="http://www.w3.org/2001/XMLSchema"
targetNamespace="http://www.springframework.org/schema/beans">
<!--
   xsd作为xml文件，其根元素是schema
   属性xmlns:xsd="http://www.w3.org/2001/XMLSchema"是引入文档约束的，表示在当前文档导入"http://www.w3.org/2001/XMLSchema"中所描述的规则，并且使用里面的元素要添加xsd的前缀（和xmlns:xsd相对应，也可以指定其它前缀）
   属性targetNamespace="http://www.springframework.org/schema/beans"表示当前文档定义的规则处于命名空间"http://www.springfarmework.org/schema/beans"下面，xml文档如需要导入当前文档的规则，就可以指定这个命名空间
   属性xmlns="http://www.springframework.org/schema/beans"表示在当前文档中导入"http://www.springframework.org/schema/beans"命名空间下所描述的规则（即当前文档本身描述的规则），并且无需使用前缀，也即默认命名空间，这样，在当前文档就可以直接引用所定义的元素了
-->
</xsd:schema>
```
