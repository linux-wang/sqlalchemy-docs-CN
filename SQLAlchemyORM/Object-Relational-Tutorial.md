## Object Relational Tutorial(关系映射入门)

SQLAlchemy对象关系映射代表了用户使用Python定义类来与数据库中的表相关联的一种方式，类的实例则对应数据表中的一行数据。SQLAlchemy包括了一套将对象中的变化同步到数据库表中的系统，这套系统被称之为工作单元（unit of work），同时也提供了使用类查询来实现数据库查询以及查询表之间关系的功能。

SQLAlchemy　ORM与SQLAlchemy表达式语言（SQLAlchemy Expression Language）是不同的，前者是在后者的基础上构建的，也就是说，是基于后者实现的（将后者封装，类似于实现一套API）。SQL表达式语言（SQL Expression Language）在 [SQL Expression Language Tutorial](SQLAlchemyCore/SQL-Expression-Language-Tutorial.md)一节有介绍，他代表了关系数据库最原始的一种架构，而SQLAlchemy ORM则代表了一种更高级也更抽象的实现，同时也是SQL表达式语言的应用。

尽管看起来ORM和表达式语言使用起来有点相似，但是其相似点比他们乍一看起来还要少（While there is overlap among the usage patterns of the ORM and the Expression Language, the similarities are more superficial than they may at first appear. 不是很确定翻译的是否正确，有问题可直接联系）。一种方式是通过透视用户定义的[领域模型](http://docs.sqlalchemy.org/en/latest/glossary.html#term-domain-model)来实现数据的内容与结构，另外一种方式通过文字模式（ literal schema）和SQL表达式来明确的操作数据库（The other approaches it from the perspective of literal schema and SQL expression representations which are explicitly composed into messages consumed individually by the database.）

一个成功的应用可能只使用对象关系映射的构造。在更复杂的情况下，可能ORM以及SQL表达式语言互相配合使用也是必不可少的。

---


## Version Check(版本检查)

使用以下代码检查SQLAlchemy版本

```
In [2]: import sqlalchemy
In [3]: sqlalchemy.__version__
Out[3]: '1.0.13'
```

PS: 安装SQLAlchemy过程，建议先安装[virtualenv](http://www.cnblogs.com/wswang/p/5391554.html)，再使用pip安装

```
pip install sqlalchemy==1.0.13  # 版本号可自选，文档对应的为1.1.0
```

---


## connecting(连接数据库)

这篇教程中我们使用sqlite数据库来演示操作，我们使用```create_engine()```来连接需要操作的数据库：

```
In [1]: from sqlalchemy import create_engine

In [2]: engine = create_engine('sqlite:///:test:', echo=True)        
```

```echo``` 参数是用来设置SQLAlchemy日志的，通过Python标准库logging模块实现。设置为True的时候我们可以看见所以的操作记录。如果你在按照本教程进行学习，那么你可以将它设置为False来减少日志的输出。

```create_engine()``` 的返回值是```Engine```的一个实例，此实例代表了操作数据库的核心接口，通过方言来处理数据库和数据库的API。在本例中，SQLite方言将被翻译成Python内置的sqlite3模块（个人理解，方言指的是每一种数据库的使用的方言，比方说mysql会有一种，sqlite又会有一种，而每种语言又会有很多在数据库的处理模块，比方说刚刚提到的Python内置的sqlite3模块）。

当第一次调用 ```Engine.execute()``` 或者 ```Engine.connect()``` 这种方法的时候，引擎（Engine）会和数据库建立一个真正的DBAPI连接，用来执行SQL语句。但是在创建了连接之后，我们很少直接使用Engine来操作数据库，更多的会使用ORM这种方式来操作数据库，这种方式在下面我们会看见具体的例子。


*PS1*:  什么是[Database Urls](http://docs.sqlalchemy.org/en/latest/core/engines.html#database-urls)

*PS2*: Lazy Connecting: 初次调用```create_engine()```的时候并不会真正的去连接数据库，只有在真正执行一条命令的时候才会去尝试建立连接。　目的是省资源，很多地方都会使用这种方式，比方说Python中的 [lazy property](https://segmentfault.com/a/1190000005818249)


-------


## Declare a Mapping


当使用ORM的时候，配置过程以描述数据库的表来开始，然后我们定义与之匹配的类。在现在的SQLAlchemy中，这两个过程一般结合在一起，通过一个称之为声明（Declarative）的系统实现。这个系统帮我们定义类以及实现与表的对应。

声明系统实现类与表的对应是通过一系列基类实现的--即声明基类（declarative base class)。我们的应用程序经常只有一个此基类的示例。使用```declarative_base()```函数，如下：

```
In [2]: from sqlalchemy.ext.declarative import declarative_base

In [3]: Base = declarative_base()
```

有了"base"之后，我们可通过他定义任何数量的映射类。我们以一个user表来开始这个教程，user表记录了用我们应用的终端用户的信息。与之对应的类称之为User。表的信息包括表名，姓名，还有列的数据类型，如下：

```
from sqlalchemy import Column, Integer, String

class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True)
    name = Column(String)
    fullname = Column(String) 
    password = Column(String)
 
    def __repr__(self):
       return "<User(name='%s', fullname='%s', password='%s')>" % (self.name, self.fullname, self.password)
```

注：
User class定义了一个 __repr__()函数，但是这并非必须的。使用它是为了在这个教程里使我们的例子（User对象）有更好看的格式。


使用Declarative定义的类最少要包括一个 __tablename__ 参数，还有一列（作为数据库primary key）。SQLAlchemy自己从不做关于类所对应的表的任何假定，包括它没有任何于名称、数据类型或者约束内置的约束。但是并不代表没有一个规范的模式，相反，它鼓励你使用帮助函数和mixin类创建自己的自动规范约束，帮助函数和mixin类在Mixin和Custom Base Classes里有详细的描述。


当构建类的时候，Declarative使用特殊的Python accessor（即[descriptors](https://docs.python.org/2/howto/descriptor.html)）代替了所有的Column对象；这个过程称之为instrumentation（中文意思：乐器）。这个"instrumented"过的类为我们的表提供了一种使用sql语句访问数据库中表的方式，同时也提供读取表中数据的方式（好特码复杂，原文：provide us with the means to refer to our table in a SQL context as well as to persist and load the values of columns from the database.）

除此之外，还有一些普通的Python类，可以让我们定义一些用于我们应用的一般的参数。

---


## Create a Schema


通过Declarative系统构建了类之后，我们也定义了被称之为表的元数据信息。被SQLAlchemy用来为某特定的表呈现这些信息的对象被称之为Table对象，Declarative系统为我们实现了这些。我们可以通过检测 __table__ 参数来查看这个对象：如下： 

```
# 代码接上面
>>User.__table__ 
Table('users', MetaData(bind=None),
            Column('id', Integer(), table=<users>, primary_key=True, nullable=False),
            Column('name', String(), table=<users>),
            Column('fullname', String(), table=<users>),
            Column('password', String(), table=<users>), schema=None)
```

PS：
经典映射-Classical Mappings
虽然在Declarative系统里十分推荐使用，但是对使用SQLAlchemy ORM来说也并非必须的。在Declarative系统外，任何一个Python类都可以通过```mapper()```函数来映射到一个数据表，但是用的相对较少，可以在[Classical Mappings](http://docs.sqlalchemy.org/en/latest/orm/mapping_styles.html#classical-mapping)看到介绍。

当声明一个类的时候，Declarative系统使用Python元类来实现，这样就可以在类被声明之后还可以执行一些额外的操作（不懂的同学可以看一下元类的左右）；在这个过程中，会根据我们的指定创建一个Table对象，然后通过构建一个Mapper对象使之与类关联。这个对象是一个幕后的对象（behind-the-scenes object），以至于我们一般不直接与他打交道（虽然在我们需要的时候它也能我们提供很多信息）

```Table``` 对象是一系列的元数据的组合。当使用Declarative的时候，这个对象可以使用我们declarative的基类的 ```.metadata``` 属性（When using Declarative, this object is available using the .metadata attribute of our declarative base class.）。

元数据是一堆包含的可以在数据库里执行的命令集。因为我们的SQLite数据库目前还没有一个users表，我们可以使用这些元数据来创建这些表。下面，我们调用```MetaData.create_all()```方法来将这些传给数据库。然后就可以看到提交这些命令之后的过程，如下：

```
>>> Base.metadata.create_all(engine)
SELECT ...
PRAGMA table_info("users")
()
CREATE TABLE users (
    id INTEGER NOT NULL, name VARCHAR,
    fullname VARCHAR,
    password VARCHAR,
    PRIMARY KEY (id)
)
()
COMMIT
```

#### Minimal Table Descriptions vs. Full Descriptions

熟悉CREATE TABLE语法的同学可能注意到了我们刚刚创建VARCHAR列的时候没有指定长度，在SQLite和Postgresql里是允许的，但是在其他地方是被禁止的。所以如果你使用了其他的数据库同时想使用SQLAlchemy来创建一个表，那么可能需要给String type添加一个i长度，格式如下：

```
Column(String(50))
```

String和其他的Integer，Numeric等待都一样，除了创建表的时候其他时候用不到。

除此之外，Firebird和Oracle需要指定一个主key，但是SQLAlchemy并不会自动生存。对于这种情况，你可以使用```Sequence```来实现：

```
from sqlalchemy import Sequence
Column(Integer, Sequence('user_id_seq'), primary_key=True)
```

使用declarative来实现一个映射的完整应用如下：

```
class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, Sequence('user_id_seq'), primary_key=True)
    name = Column(String(50))
    fullname = Column(String(50))
    password = Column(String(12))

    def __repr__(self):
        return "<User(name='%s', fullname='%s', password='%s')>" % (
                                self.name, self.fullname, self.password)
```

我们展示了这个完整的表以此对比来显示两者的区别（最小结构vs完整版），完整的可以用来完成创建某种特定的很严格的需求的表。

---


## Create an Instance of the Mapped Class¶

