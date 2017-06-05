dozer是一个java bean copy类库，性能优于apache的BeanUtils，但是他们两个都会对空属性进行拷贝，这点很不方便。在dozer中可以使用xml映射禁用空属性拷贝，还得配置xml，感觉很麻烦。
其实可以构造一个BeanMappingBuilder，对mapping进行配置。

```
mapping(sources.getClass(), destination.getClass(), mapNull(false), mapEmptyString(false));
```
分别对

> org.dozer.loader.api.TypeMappingOptions.mapEmptyString
> org.dozer.loader.api.TypeMappingOptions.mapNull
设置成false即可。

于是可以封装成一个util

```
public static void copyProperties(final Object sources, final Object destination) {
        WeakReference weakReference = new WeakReference(new DozerBeanMapper());
        DozerBeanMapper mapper = (DozerBeanMapper) weakReference.get();
        mapper.addMapping(new BeanMappingBuilder() {
            @Override
            protected void configure() {
                mapping(sources.getClass(), destination.getClass(), mapNull(false), mapEmptyString(false));
            }
        });
        mapper.map(sources, destination);
        mapper.destroy();
        weakReference.clear();
    }
```