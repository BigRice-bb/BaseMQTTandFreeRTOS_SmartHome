# BaseMQTTandFreeRTOS_SmartHome
BaseMQTTandFreeRTOS_SmartHome

FreeRTOS--两个线程
1,ATRecvParser--不断读取串口数据--放入环形缓冲区 并解析
2,Task_thread--用于MQTT订阅和发布消息,以及消息处理


MQTT应用层
mqtt_connect->platform_net_socket_connect
mqtt_subscribe-->订阅主题-->topic1_handler--回调函数,用于处理主题数据
mqtt_publish-->发布主题

网络层:
platform_net_socket_connect->连接路由器和服务器->ATSendCmd
platform_net_socket_recv_timeout->ATReadData
platform_net_socket_write_timeout->ATSendCmd->ATSendData
platform_net_socket_close->ATSendCmd->AT+CIPCLOSE

AT命令层:
ATSendCmd->HAL_AT_Send->阻塞等待at_ret_mutex
发送AT命令,等待esp8266的返回-ATRecvParser函数解析-释放at_ret_mutex信号量
AT+CWMODE=3       1. 配置 WiFi 模式  返回OK
AT+CWQAP       断开连接  返回OK
AT+CWJAP="SSID","password"     2. 连接路由器     返回WIFI CONNECTED   WIFI  GOT IP   OK
AT+CIPSTART="TCP","192.168.3.116",8080     3. ESP8266 设备作为 TCP client 连接到服务器   返回CONNECT   OK
AT+CIPSEND=4     4.ESP8266 设备向服务器器发送数据     
返回值类型:
AT_OK        0
AT_ERR      -1
AT_TIMEOUT  -2

ATRecvParser--核心处理函数->HAL_AT_Secv->USART3_Read->一次读取一个字符放入字符数组中
判定条件  当读到\r\n  代表读到一条完整的返回  开始解析
strstr(buf, "OK\r\n" )--> 返回AT_OK -->释放信号量at_ret_mutex
strstr(buf, "ERROR\r\n")--> 返回AT_ERROR -->释放信号量at_ret_mutex
GetSpecialATString-->ProcessSpecialATString(buf)

硬件抽象层:
HAL层--硬件抽象层
HAL_AT_Send->USART3_Write
HAL_AT_Secv->USART3_Read

硬件层--usart3
USART3_Write
USART3_Read(等待信号量uart_recv_mutex)->USART3_IRQHandler(接收中断,释放uart_recv_mutex)
完成一个字节的读写
实现最底层的对串口的读写

