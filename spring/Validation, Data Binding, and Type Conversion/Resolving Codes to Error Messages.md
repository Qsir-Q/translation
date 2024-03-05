# 将代码解析为错误消息
我们讨论了数据绑定和验证。本节介绍与验证错误对应的输出消息。在上一节所示的示例中，我们拒绝了姓名和年龄字段。如果我们希望通过使用MessageSource输出错误消息，我们可以使用拒绝字段时提供的错误代码(在本例中为'name'和'age')来实现。
当您从Errors接口调用(直接或间接地，例如，通过使用ValidationUtils类)rejectValue或其他拒绝方法之一时，底层实现不仅会注册您传入的代码，还会注册一些额外的错误代码。MessageCodesResolver确定Errors接口注册了哪些错误代码。
默认情况下，使用DefaultMessageCodesResolver，它(例如)不仅用您提供的代码注册消息，而且还注册包含您传递给reject方法的字段名的消息。因此，如果您通过使用rejectValue("age"， "too.darn.old")拒绝一个字段，除了too.darn.old代码，Spring还会注册too.darn.old.age和too.darn.old.age.int(第一个包含字段名，第二个包含字段类型)。这样做是为了方便开发人员定位错误消息。

关于MessageCodesResolver和默认策略的更多信息可以分别在MessageCodesResolver和DefaultMessageCodesResolver的javadoc中找到。