学习笔记: Go error最佳实践

1. 语言见的对比

        c 语言常见于将对象当作参数传入，根据int返回值的值判断执行结果
        C++ 引入了exception，但是函数调用方不知道exception的类型
        Java 对exception引入了返回类型的声明，但会造成无关大小的函数抛出exception，非常繁琐
        
2. Go常见异常

        场景1：
            野生goroutine的异常无法捕获
            func main(){
                xxx 
                go func(){
                    x()
                    panic("")
                }()
            }
            
            解决方法：
            func main(){
                xxx
                Go(x)
            }
            
            func Go(x func()){
                go func(){
                    if err := recover(); err != nil{
                        
                    }
                    x()
                }()
            }
            
3. sentinel Error

        预定义的error类型
            package xxx
            
            var ErrNoType = errors.New("no type")
            
            func bar() error{
                return ErrNoType
            }
            
            package yyy
            
            func foo(){
                if err == xxx.ErrNoType{
                    
                }
            }
        缺陷：
            上层无法自定义携带信息，eg：无法使用类似formatErr的形式使用
            在两个包之间造成了依赖
            
4. Error Types

        调用者使用断言判断error类型
        
        func main(){
            err := foo()
            swith err.(type)
            case nil:
            //xxx
            case *MyErr:
            
        }
        
5. Opaque Error

        error只表征调用成功与否
        
        err := foo()
        if err != nil{
            xxx
        }
        
        扩展：可以在Error上实现接口方法判断错误是否实现了特定的行为
        
        type Timeout interface{
            func Timeout() bool
        }
        
        func IsTimeout(err error) bool {
            te, ok := err.Timeout()
            return ok && te.Timeout()
        }
        
6. Handling Error 

        1.通过封装io.Writer消除err的判断
        type ErrWriter struct{
            io.Writer
            err error
        }
        
        func (ew *ErrWriter) Write(buf []byte) (int, error){
            if ew.err != nil{
                return 0, e.err
            }
            var n int
            n, ew.err := ew.Writer.Write(buf)
            return n, nil
        }
        
        func foo(w io.Writer) erorr{
            ew := &ErrWriter(w)
            //_, err := fmt.Fprintf(w, "")
            
            fmt.Fprintf(ew, "")
            return ew.err
        }
        
        2. Wrap Error
            1. 最上层err := foo()打印日志时没有任何上下文
            2. 第一次优化底层 return fmt.Errorf("xxx err: %v", err), 上层没有文件名和行号信息
            3. 使用github.com/pkg/errors对error进行wrap,只有底层函数wrap， return errors.Wrap(err, "open file err") Wrap会携带堆祝栈等信息
            4. 非最上层只返回,在最上层打印error
            5. 只有application进行wrap error，kit性的基础库只返回根因 return err
            
            
        3. 理念
            1.错误要被日志记录
            2.应用应该处理错误，保证100%完整性
            3.之后不再有错误报告
            
7. errors

        go 1.13新增的errors.Is,errors.Is会判断是否err携带Unwrap方法，然后循环取root err判断err是否与target err是一个对象
        fmt.Errorf() 兼容errors.Is 提供Unwrap返回root err，可以使用errors.Is判断root err是否一致
        
        example:
            package xxx
            
            func DoSomething(userId string) error{
                err := users.Authtication(userId)
                if err == users.ErrPermission{
                    return fmt.Errof("%s err:%w", userId, err)
                }
            }
            
            package main
            func main(){
                if errors.Is(xxx.DoSometing(userId), users.ErrPermission){
                    //处理err
                }
            }
            
8. Wrap: github.com/pkg/errors.Wrap() vs fmt.Errorf()

        pkg/errors 携带堆栈信息 
        fmt.Errorf 不携带堆栈信息
            
9. DAO层没有数据如何响应

        最好响应err, nil具有二义性;例如map对象的nil和map的初始化空对象
        
            