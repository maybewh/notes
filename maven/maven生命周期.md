## maven生命周期

​		clean compile test package install deploy 每次执行后面的命令的时，在它之前的命令都会被执行（除clean之外，clean是单独的）。

## maven概念模型

![image-20210126191518544](C:\Users\abc\AppData\Roaming\Typora\typora-user-images\image-20210126191518544.png)

​		项目对象模型：

​		指的就是pom.xml配置文件，包含项目自身的信息，如modelVersion, artifactId等标签；项目自身所依赖的jar信息，如dependencies标签；项目运行的环境，如build中的plugins标签里面配置jdk、Tomcat等。

​       依赖管理模型：

​		先从本地仓库找jar包，若没有则会从中央仓库下载jar包。公司中则会从私服拉取（b2b)

​       一键构建：

![image-20210126192643699](C:\Users\abc\AppData\Roaming\Typora\typora-user-images\image-20210126192643699.png)

​        当无法连网时，使用-DarchetypeCatalog=internal，可以直接使用本地已下载好的骨架来创建工程。![image-20210126193202406](C:\Users\abc\AppData\Roaming\Typora\typora-user-images\image-20210126193202406.png)

## maven解决jar冲突

​	冲突场景：导入的两个jar都依赖了同一jar，但版本又不一样。则根据下面的原则解决。

```
* 第一声明优先原则：哪个jar坐标在靠最上面的位置，这个jar就是先声明的，那么就使用它依赖的jar包
* 第二种解决方式：路径近者优先原则。直接依赖路径比传递依赖路径近
  * 直接依赖：项目中直接导入的jar
  * 传递依赖：项目中没有直接导入的jar，可以通过直接依赖的jar包传递过来的jar包
* 第三种解决方式【推荐使用】：使用dependency标签下的<exclusion>标签将其移除
```

​    



可以使用dependency下的socope标签来完成

scope有compile、test、provided、runtime、system

![image-20210126201554244](C:\Users\abc\AppData\Roaming\Typora\typora-user-images\image-20210126201554244.png)

## maven运行环境修改

​	也就是通过plugin引入Tomcat插件以及编译环境JDK版本的设置

## pom标签说明

​		1）dependencyManagement-->锁定jar包版本。maven工程可以分为父子依赖关系，凡是依赖别的项目后，拿到的别的项目的依赖包，都属于传递依赖。即A项目依赖于B项目，则B项目的依赖的都会传递到A项目中。若A项目的开发者再在该项目中再次导入了相同的jar包，就会把B项目中的传递依赖的jar覆盖掉，为了防止以上情况出现，我们可以把项目B中的jar锁住。那么，其他依赖于B项目的项目中，即便是有同名jar直接依赖，也无法覆盖。只有锁定jar的功能，并不会导入jar包。