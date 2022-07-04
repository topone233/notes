Linux

| 命令    |      |      |
| ------- | ---- | ---- |
| ls、dir |      |      |
| cd      |      |      |
| pwd     |      |      |
| history |      |      |



1. 查看8080端口的占用情况

   ```shell
    lsof -i :8080 
   ```

2. 根据PID强制杀死进程

   ```sh
   kill -9 [PID] 
   ```

3. 启动JAR包

   ```shell
   // 方法一：当前ssh窗口被锁定。可以通过ctrl + c中断程序执行，或者直接关闭窗口，程序退出。
   java -jar shop.jar
   
   // 方法二：&表示在后台运行，当前ssh窗口不锁定。但是当窗口关闭时，程序终止运行。
   java -jar shop.jar &
   
   // 方法三：nohup表示不挂断的运行，当账户退出或终端关闭时，程序仍然运行。
   nohup java -jar shop.jar &
   
   // 方法四：nohup执行时，默认所有输出都重定向到nohup.out中，通过 > 指定输出文件
   // /dev/null 代表空设备文件，输出到这里 也就是不输出，相当于一个黑洞。
   nohup java -jar shop.jar >/dev/null & 
   ```
