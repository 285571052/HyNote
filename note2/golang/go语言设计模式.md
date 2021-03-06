https://golangbyexample.com/all-design-patterns-golang/
https://github.com/aierui/design-pattern-in-go

# 创建型模式(Creational Design Patterns)


## 工厂方法模式(Factory Method)

工厂模式 定义一个用于创建对象的接口，让子类决定实例化哪一个类
```
package main

//定义单个产品
type iGun interface {
    setName(name string)
    setPower(power int)
    getName() string
    getPower() int
}

//客户端自己选择使用哪个产品
func getGun(gunType string) (iGun, error) {
    if gunType == "ak47" {
        return newAk47(), nil
    }
    if gunType == "maverick" {
        return newMaverick(), nil
    }
    return nil, fmt.Errorf("Wrong gun type passed")
}
```

工厂vs抽象工厂: 主要取决于抽象的层级, 单层为工厂模式, 多层为抽象工厂模式


## 抽象工厂模式(Abstract Factory)

抽象工厂模式 为创建一组相关或相互依赖的对象提供一个接口，而且无需指定他们的具体类

```
//一个抽象工厂包含对应的不同的多组产品
type iSportsFactory interface {
    makeShoe() iShoe	//产品1
    makeShort() iShort	//产品2
}

//指定支持不同产品的工厂
func getSportsFactory(brand string) (iSportsFactory, error) {
    if brand == "adidas" {
        return &adidas{}, nil
    }
    if brand == "nike" {
        return &nike{}, nil
    }
    return nil, fmt.Errorf("Wrong brand type passed")
}
```

## 生成器模式(Builder)

对象里边有builer对象, builder是接口类型
客户端不必知道产品内部组成的细节，将产品本身与产品的创建过程解耦，使得相同的创建过程可以创建不同的产品对象

使用时机:
1 组成成分类似或存在多个可选版本


```
//包含builder
type director struct {
    builder iBuilder
}

func newDirector(b iBuilder) *director {
    return &director{
        builder: b,
    }
}

func (d *director) setBuilder(b iBuilder) {
    d.builder = b
}

func (d *director) buildHouse() house {
...
}

type iBuilder interface {
...
}

// 创建不同的builder
func getBuilder(builderType string) iBuilder {
    if builderType == "normal" {
        return &normalBuilder{}
    }
    if builderType == "igloo" {
        return &iglooBuilder{}
    }
    return nil
}
```

## 对象池模式(Object Pool)

创建对象放到池中, 使用时取出, 不用时放回
使用时机: 当对象创建的成本特别高并且使用时数量需求不多的场景(比如数据库连接池)

```
type iPoolObject interface {
    getID() string //This is any id which can be used to compare two different pool objects
}

type pool struct {
    idle   []iPoolObject
    active []iPoolObject
    capacity int
    mulock   *sync.Mutex
}

func (p *pool) receive(target iPoolObject) error {
    p.mulock.Lock()
    defer p.mulock.Unlock()
    err := p.remove(target)	//这里不是直接移出, 需要校验
    if err != nil {
        return err
    }
    p.idle = append(p.idle, target)
    fmt.Printf("Return Pool Object with ID: %s\n", target.getID())
    return nil
}

func (p *pool) remove(target iPoolObject) error {	//receive中调用
    currentActiveLength := len(p.active)
    for i, obj := range p.active {
        if obj.getID() == target.getID() {	//belong to the pool
            p.active[currentActiveLength-1], p.active[i] = p.active[i], p.active[currentActiveLength-1]
            p.active = p.active[:currentActiveLength-1]
            return nil
        }
    }
    return fmt.Errorf("Targe pool object doesn't belong to the pool")
}
```

样例的这段代码的方式都是小写的, 都在main包, 其实可以利用大小写区分内部方法, 比如remove方法小写, 其他方法大写

## 原型模式(Prototype)

实现Clone或Deepcopy接口来复制对象
原型模式实际上就是从一个对象再创建另外一个可定制的对象，而且不需要知道任何创建的细节。
1. 简化对象内部属性的复制实现逻辑(直接调用`clone`接口即可)
2. 可以根据运行时的状态实时复制内部的属性(比如文件系统目录的嵌套结构是不确定的)

```
type inode interface {
    print(string)
    clone() inode
}

type file struct {
    name string
}

func (f *file) clone() inode {	//单文件
    return &file{name: f.name+ "_clone"}
}

type folder struct {
    childrens []inode
    name      string
}

func (f *folder) clone() inode {
    cloneFolder := &folder{name: f.name+ "_clone"}
    var tempChildrens []inode
    for _, i := range f.childrens {
        copy := i.clone()			//递归地按运行是状态进行, 可能是file也可能是folder, 有clone方法, folder不需要再循环展开
        tempChildrens = append(tempChildrens, copy)
    }
    cloneFolder.childrens = tempChildrens
    return cloneFolder
}
```

## 单例模式(Singleton)

单例模式: 全局只需要1个实例, 比如db等
```
var lock = &sync.Mutex{}

type single struct {
}

var singleInstance *single

func getInstance() *single {
    if singleInstance == nil {
        lock.Lock()
        defer lock.Unlock()
        if singleInstance == nil {	//double check
            fmt.Println("Creting Single Instance Now")
            singleInstance = &single{}
        } else {
            fmt.Println("Single Instance already created-1")
        }
    } else {
        fmt.Println("Single Instance already created-2")
    }
    return singleInstance
}
```

```
var once sync.Once

type single struct {
}

var singleInstance *single

func getInstance() *single {
    if singleInstance == nil {
        once.Do(	//用once来保证只运行一次
            func() {
                fmt.Println("Creting Single Instance Now")
                singleInstance = &single{}
            })
    } else {
        fmt.Println("Single Instance already created-2")
    }
    return singleInstance
}
```

1 once的逻辑和第一段代码类似, 还是由double check的步骤,只是通过unit32来记录运行状态以及atomic.LoadUint32/atomic.StoreUint32来进行读写
2 第一段代码无法保证赋值后第一行singleInstance == nil为true, 但是atomic可以, 但是lock可以保证只创建一次(即第二行singleInstance == nil一定为true)
