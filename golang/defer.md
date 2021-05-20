```
// 返回0
func a() int {
	a := 1
	defer func() {
		a = 2
		fmt.Println(a)
	}()
	defer func() {
		a = 3
		fmt.Println(a)
		if err := recover(); err != nil {

			fmt.Println(err)
		}
	}()

	panic("ERR")

	return a
}


func b() int {
	a := 1
	defer func(b int) {
		a = 2
	}(a)
	//返回值 = xxx
	//调用defer函数(这里可能会有修改返回值的操作)
	//return 返回值
	return a
}
```
