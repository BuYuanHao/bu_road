1、 final 修饰方法可以重载，不可以重写
2、当声明一个 final 成员时，必须在构造函数退出前设置它的值。
3、

```java
//出错
byte b1=1;
byte b2=3;
//因为b1、b2可以自动转换成int类型的变量，运算时java虚拟机对它进行了转换，结果导致把一个int赋值给byte
byte b3=b1+b2;

//正确 
final byte b1=1;
final byte b2=3;
byte b3=b1+b2;

```