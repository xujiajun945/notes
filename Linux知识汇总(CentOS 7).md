# Linux知识汇总(CentOS 7)

## 系统相关

1. **系统环境变量: /etc/profile 文件**

   导入环境变量: 
   ```
   # export语法, 系统变量为PATH
   export [VAR]=...
   ```

   导入新的环境变量后需要重启profile文件: 

   ```
   source /etc/profile  或  .  /etc/profile  (source指令与 . 指令含义相同)
   ```

2. **根据端口号查进程**

   ```
   netstat -anp | grep [port]  例如: 查询端口8080  netstat -anp | grep 8080
   ```

3. **防火墙(firewall)相关指令**:

   - **启动防火墙**: 

     ```
     systemctl start firewall.service
     ```

   - **关闭防火墙**: 

     ```
     systemctl stop firewall.service
     ```

   - **添加放行端口**: 

     ```
     firewall-cmd  --zone=public  --add-port=[port]/tcp  --permanent

     命令含义:
     –zone # 作用域
     –add-port=8080/tcp  # 添加端口，格式为：端口/通讯协议
     –permanent # 永久生效，没有此参数重启后失效　
     ```
     
   - **锁定防火墙服务**(防止添加端口等操作): 

     ```
     systemctl mask firewall
     ```

   - **取消防火墙锁定**: 

     ```
     systemctl unmask firewall
     ```

   - **查看开放的端口列表**: 

     ```
     firewall-cmd --list-ports
     ```

   - **查看防火墙状态**: 

     ```
     systemctl status firewalld
     ```
     
   - **重启防火墙**:

     ```
     firewall-cmd --reload
     ```

   - 

4. **nohup相关命令**:

   - 不挂断地运行命令, no hangup的缩写, 意即"不挂断"

     ```
     # 不挂断运行sh文件
     nohup ./xx.sh
     ```

   - <font color=#2196F3> &  用于后台运行命令, 但当用户挂起(退出)时, 命令会中断, 可以将nohup命令与&命令结合使用</font>

     ```
     # 后台运行且用户挂起后运行不中断
     nohup ./xxx.sh & 
     ```
     
   - 指定输出文件
     
     ```
     # 默认输出文件问相同目录下的nohup.out文件, 我们为了记录日志, 可以指定输出路径
     nohup ./xxx.sh > mylog.log &
     ```
     
   - **<font color=green>重定向标准错误到标准输出</font>**

     ```
     nohup command > myout.file 2>&1 & 
     
     0 – stdin (standard input)
     1 – stdout (standard output)
     2 – stderr (standard error)
     2>&1是将标准错误（2）重定向到标准输出（&1），标准输出（&1）再被重定向输入到myout.file文件中
     ```

5.  **查看内存**:

   ```
   # 查看内存
   free
   
   # 带单位查看
   free -h
   
   # 间隔观察 -s 命令用于指定间隔的秒数
   free -s 3 
   ```

   

6. 