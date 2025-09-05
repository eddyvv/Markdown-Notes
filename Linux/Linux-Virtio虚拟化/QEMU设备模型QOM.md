# QEMU设备模型QOM

QOM是QEMU在C的基础上自己实现的一套面向对象机制，负责将device、bus等设备都抽象成为对象。对象的初始化分为四步：

1. 将 TypeInfo 注册 TypeImpl
2. 实例化 ObjectClass
3. 实例化 Object
4. 添加 Property

















# 参考

[QEMU建模之设备创建总体流程 - CodeAntenna](https://codeantenna.com/a/U3KxKIbire)

[QEMU设备的对象模型QOM | TO DO (juniorprincewang.github.io)](http://juniorprincewang.github.io/2018/07/23/qemu源码添加设备/)