## Android源码通读笔记

### 1.android系统分Application、Framework、libraries、Linux内核四层

  * 其中Dalivk虚拟机在Libraries中，Linux内核主要是Camera驱动、光感Sensor驱动、重力加速器驱动、Audio、wifi、蓝牙、Display、Bindler等驱动
  * Libraries层主要是C/C++实现，包括init、Audio、AudioFlinger、Surface等系统
  
### 2.init进程
  * init进程是Linux系统也是Android系统的第一个进程，负责创建几个关键进程Zygote、SystemServer进程
  * 负责加载系统配置文件（init.rc文件）、创建属性服务（property service）属性服务将系统配置映射到共享内存空间，设置为只读属性。便于其他进程访问。
  
### 3.property_service  
  + 其中init进程会创建一个property_service作为独立的子进程,加载了default.prop文件，使用mmap映射到共享内存，并使用socket实现ipc通信
  + 目前最多支持加载247项属性--**待考证** 
  + init.rc文件具体可查看 [csdn]（https://blog.csdn.net/stoic163/article/details/78773141） bootanimation是在init.rc中定义的

### 4.zygote进程
  * init进程创建了zygote进程，zygote进程开始涉及java,它创建了system_server进程；zygote和system_server死亡会导致java崩溃。
  其中init进程检查发现zygote如果死亡，会将zygote及其所属的所有子进程全部杀死
  * zygote中有个AppRuntiem类，属于AndroidRuntime的子类，它创建了java Dalvik虚拟机。虚拟机中设置了heapsize=16MB,一般OEM商会改为32MB，stack为8MB
  * 
  
  
  
  
