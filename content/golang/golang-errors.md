---
title: golang-errors
date: 2020-01-06 21:22:07
tags: [golang]
---





###  error 只是一个接口的实现

我们之前在定义一个错误的时候只需要 实现如下的一个函数签名, 就可以认为这个结构体是一个error 类型的

```go

type error interface {
	Error() string
}

```

如这里的T1 就是一个error

```go

type T1 struct {
	Desc string
}

func (t1 T1) Error() string {
	return fmt.Sprintf("T1 error :%s", t1.Desc)
}


```

#### 1.3 之前的我们

##### catch错误

在以前我们经常用nil 去比较代码是否发生错误, 如下面的例子
```golang

func main() {
	var err error
	err =Throw()
	if err != nil {
		panic(err)
	}
	fmt.Println("right")
}

```

##### 判断错误的类型
要判断错误的来源是那个我们可以使用下面的技术

```golang 

var ErrNotFound = errors.New("not found")

if err == ErrNotFound {
    // something wasn't found
}
```
同样我们也可以使用类型断言,来确定这个错误是由那个具体的对象抛出的

```golang

if e, ok := err.(*NotFoundError); ok {
    // e.Name wasn't found
}

```


##### 对于错误的处理
一般情况下, 我们不仅仅要catch 错误,为了这些的可读性行更好, 更容易定位问题的出现位置, 我们都会添加一些附加信息, 如下

```go


func UserRegister(name string) error {
    //.....
    if err :=register(name); err !=nil {
        return errors.New("用户注册失败"+err.Error())
    }
}

```
上面的只是一个简单的例子, 一些常见的例子,你可能还需要带上包的位置,这样对象这些错误,一旦看到,我们很容易定位到问题发生的具体位置, 对于这个需求有https://github.com/pkg/errors 这个包很有用.可以很方便的实现我们的需求,同样你也可以定义不同的结构体,在合适的位置抛出错误.

#### 我们的需求

通过开发的实践, 我们发现上面的2个情况是我们经常面对的, 所以在1.3 的版本对error 的处理,语言自己提供了更多的选择

#### 1.3 之后的我们

#####  Unwrap
这个方法可以 相当与就是 对error 脱衣服,可以看到衣服下面的具体错误类型,你可以一直让他脱衣服, 一直到他脱光光,通过下面的源码可以看到,这个函数是在递归调用.

```go

func Unwrap(err error) error {
	u, ok := err.(interface {
		Unwrap() error
	})
	if !ok {
		return nil
	}
	return u.Unwrap()
}
```


#####  Is
这个是为了类型的断言, 我们不需要再使用==去判断错误的具体类型, 只是需要小小的调用Is,传入2个参数,就可以完成,类型判断


```go

// 原来
errorA == errorB 

func Is(err, target error) bool {
	if target == nil {
		return err == target
	}

	isComparable := reflectlite.TypeOf(target).Comparable()
	for {
		if isComparable && err == target {
			return true
		}
		if x, ok := err.(interface{ Is(error) bool }); ok && x.Is(target) {
			return true
		}
		// TODO: consider supporing target.Is(err). This would allow
		// user-definable predicates, but also may allow for coping with sloppy
		// APIs, thereby making it easier to get away with them.
		if err = Unwrap(err); err == nil {
			return false
		}
	}
}

```



你看看这个源码还是使用了反射来判断底层的错误类型,举个使用的例子


```go


func main(){

    if errors.Is(errorA, errorB){
        fmt.Println("这个错误是B")
    }
}

```


#####  As 
类型断言我们不要了,  我们写的代码更好看了

```golang

func As(err error, target interface{}) bool {
	
	val := reflectlite.ValueOf(target)
	typ := val.Type()
	if typ.Kind() != reflectlite.Ptr || val.IsNil() {
		panic("errors: target must be a non-nil pointer")
	}
    // ....
}

```

```golang


if errors.As(errorA,NotFoundError){
    fmt.Println("errorA 是NotFoundError 的一种类型")
}


```


#####  Wrap 包装错误
fmt 包 默认直接输入error 可以自动包装错误了,添加了一个特殊的打印符

```go 

 return nil, fmt.Errorf("%q: %w", name, ErrNotFound)

```