学习笔记

## 合格的使用goroutine进行并发
    1. 把并发扔给调用者
    2. 能够控制goroutine什么时候退出
    3. 搞清楚goroutine什么时候退出
    
## 示例代码
    func main() {
    	t := NewTracker()
    	go t.Run()
    
    	_ = t.Event(context.Background(), "test")
    	_ = t.Event(context.Background(), "test")
    	_ = t.Event(context.Background(), "test")
    	ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(time.Second*2))
    	defer cancel()
    
    	t.ShutDown(ctx)
    }
    
    func NewTracker() *Tracker {
    	return &Tracker{
    		ch:   make(chan string, 10),
    		stop: make(chan struct{}),
    	}
    }
    
    type Tracker struct {
    	ch   chan string
    	stop chan struct{}
    }
    
    func (t *Tracker) Run() {
    	for msg := range t.ch {
    		time.Sleep(time.Second)
    		fmt.Println(msg)
    	}
    	t.stop <- struct{}{}
    }
    
    func (t *Tracker) Event(ctx context.Context, msg string) error {
    	select {
    	case t.ch <- msg:
    		return nil
    	case <-ctx.Done():
    		return ctx.Err()
    	}
    }
    
    func (t *Tracker) ShutDown(ctx context.Context) {
    	close(t.ch)
    	select {
    	case <-t.stop:
    	case <-ctx.Done():
    	}
    }
    
## atomic.value采用内存方式的锁效率较高

    v := aotimic.Value()
    v.Store(make(map))
    func read(key string){
        m := v.load().(map)    
        return m[key]
    }
    
    func insert(key, val string){
        m := v.load().(map)
        m2 := make(map)
        for 
    }
    
    
    func (){
        for {
            m := v.load()
            
        }
    }

    