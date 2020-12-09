## 基于 errgroup 实现一个 http server 的启动和关闭 ，以及 linux signal 信号的注册和处理，要保证能够 一个退出，全部注销退出。

    func main() {
    	var errs = make(chan error, 1)
    	group, _ := errgroup.WithContext(context.Background())
    	group.Go(func() error {
    		err := http.ListenAndServe(":8080", nil)
    		return err
    	})
    
    	go func() {
    		errs <- group.Wait()
    	}()
    
    	sig := make(chan os.Signal)
    	signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)
    
    	go func() {
    		errs <- fmt.Errorf("err:%s", <-sig)
    	}()
    	fmt.Println("err", <-errs)
    }