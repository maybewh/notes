# ImportAware接口和@Import注解

​		@Import可以导入配置类，即导入用@Configuration配置的类，如果@Configuration注解的类实现了ImportAware接口，那么它就可以获取导入该配置类的数据配置。

