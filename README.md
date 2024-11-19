[合集 \- Ideal库 \- Common库(7\)](https://github.com)[1\.开源 \- Ideal库 \- 常用时间转换扩展方法（一）11\-07](https://github.com/hugogoos/p/18531206)[2\.开源 \- Ideal库 \- 常用时间转换扩展方法（二）11\-09](https://github.com/hugogoos/p/18535467)[3\.开源 \- Ideal库 \- 获取特殊时间扩展方法（三）11\-11](https://github.com/hugogoos/p/18538819)[4\.开源 \- Ideal库 \-获取特殊时间扩展方法（四）11\-12](https://github.com/hugogoos/p/18539591):[FlowerCloud机场](https://hushicha.org)[5\.开源 \- Ideal库 \- 常用枚举扩展方法（一）11\-13](https://github.com/hugogoos/p/18542907)[6\.开源 \- Ideal库 \- 常用枚举扩展方法（二）11\-14](https://github.com/hugogoos/p/18545101)7\.开源 \- Ideal库 \- 枚举扩展设计思路及实现难点（三）11\-18收起
今天想和大家分享关于枚举扩展设计思路和在实现过程中遇到的难点。


![](https://img2024.cnblogs.com/blog/386841/202411/386841-20241118232244245-90705571.png)


# ***01***、设计思路


设计思路说起来其实也很简单，就是通过枚举相关信息：枚举值、枚举名、枚举描述、枚举项、枚举类型，进行各种转换，通过一个信息获取其他信息。比如通过枚举项获取枚举描述、通过枚举类型获取枚举名称\-枚举描述键值对用于下拉列表等等。


主要包括以下几类转换实现：


（1）通过枚举值转成：枚举、枚举名称、举描述；


（2）通过枚举名称转成：枚举、枚举值、枚举描述；


（3）通过枚举描述转成：枚举、枚举值、枚举名称；


（4）通过枚举项转成：枚举值、枚举描述；


（5）通过枚举类型转成：枚举值\-枚举名称、枚举值\-枚举描述、枚举名称\-枚举值、枚举名称\-枚举描述、枚举描述\-枚举值、枚举描述\-枚举名称等键值对集合；


# ***02***、实现难点


在实现过程中的确遇到一些难点以及需要注意的小细节。


## 1、枚举名称转枚举中的小细节


在实现枚举名称转枚举的过程中遇到了一个小坑。


枚举本身提供了Enum.TryParse方法用于把枚举名称字符串或者枚举值字符串转为枚举，因此我们选用了此方法实现枚举名称转枚举。但是我们自己这个接口设计目标是把枚举名称转为枚举，因此需要排除误传枚举值字符串的情况。


方案：当转换成功后，我们还需要调用Enum.IsDefined方法检查枚举名称字符串是否是有效的枚举名称字符串，如果不是枚举中现有的有效枚举项，还需要考虑是否为位标志组合情况。


代码如下：



```
//根据枚举名称转换成枚举，转换失败则返回空
public static TEnum? ToEnumByName(this string name)
    where TEnum : struct, Enum
{
    //转换成功则返回结果，否则返回空
    if (Enum.TryParse(name, out var result))
    {
        //检查是否为有效的枚举名称字符串，
        if (Enum.IsDefined(typeof(TEnum), name))
        {
            //返回枚举
            return result;
        }
        else
        {
            //计算是否为有效的位标志组合项
            var isValidFlags = IsValidFlagsMask<ulong, TEnum>(result.ToEnumValue<ulong>());
            //如果是有效的位标志组合项则返回枚举，否则返回空
            return isValidFlags ? result : default(TEnum?);
        }
    }
    //返回空
    return default;
}

```

## 2、枚举描述转枚举中的小细节


在实现枚举描述转枚举的过程中对于带位标志枚举处理需要特别小心。我们知道位标志枚举有个特性是枚举项之间可以通过位操作进行组合得到一个有效的枚举项，而且这个枚举项还不存在于当前定义的枚举中。这就要求我们必须处理好组合情况。


方案：因此可以按照以下步骤进行处理。


（1）优先处理当枚举不带位标志的情况；


（2）再处理带位标志的情况；


位标志组合是通过英文逗号\[,]拼接的，因此还要考虑到如果一个描述字符串中带有英文逗号\[,]，但是本身又不是组合的情况。


（3）先把描述当作不是组合情况，先处理一遍，如果转换成功则结束转换；


（4）如果是组合的情况，则计算出组合的每项枚举值，通过位操作得到其对应的枚举项；


而在组合的情况中还需要考虑，如果有组合的项时无效的情况，则这个枚举描述转换应该标记为无效。


（5）判断组合的每一项都是正确的，否则返回空；


（6）将通过组合计算出来的枚举项转为枚举；


因为通过枚举值转枚举过程中还需要明确枚举值类型才行，因此最后一步还需要根据枚举类型实际的基础类型调用不同的转换方法。


具体代码如下：



```
//根据枚举描述转换成枚举，转换失败返回空
public static TEnum? ToEnumByDesc(this string description)
    where TEnum : struct, Enum
{
    var type = typeof(TEnum);
    var info = GetEnumTypeInfo(type);
    //不是位标志枚举的情况处理
    if (!info.IsFlags)
    {
        return ToEnumDesc(description);
    }
    //是位标志枚举的情况处理
    //不是组合位的情况,本身可能就包含[,]
    var tenum = ToEnumDesc(description);
    if (tenum.HasValue)
    {
        return tenum;
    }
    //如果不包含[,]，则直接返回
    if (!description.Contains(','))
    {
        return default;
    }
    //是组合位的情况
    var names = description.Split(',');
    var values = Enum.GetValues(type);
    //记录有效枚举描述个数
    var count = 0;
    ulong mask = 0L;
    //变量枚举所有项
    foreach (var name in names)
    {
        foreach (Enum value in values)
        {
            //取枚举项描述与目标描述相比较，相同则返回该枚举项
            if (value.ToEnumDesc() == name)
            {
                //有效枚举个数加1
                count++;
                //将枚举值转为long类型
                var valueLong = Convert.ToUInt64(value);
                // 过滤掉负数或无效的值，规范的位标志枚举应该都为非负数
                if (valueLong >= 0)
                {
                    //合并枚举值至mask
                    mask |= valueLong;
                }
                break;
            }
        }
    }
    //如果两者不相等，说明描述字符串不是一个有效组合项
    if (count != names.Length)
    {
        return default;
    }
    var underlyingType = Enum.GetUnderlyingType(type);
    if (underlyingType == typeof(byte))
    {
        return ((byte)mask).ToEnumByValue();
    }
    else if (underlyingType == typeof(sbyte))
    {
        return ((sbyte)mask).ToEnumByValue();
    }
    else if (underlyingType == typeof(short))
    {
        return ((short)mask).ToEnumByValue();
    }
    else if (underlyingType == typeof(ushort))
    {
        return ((ushort)mask).ToEnumByValue();
    }
    else if (underlyingType == typeof(int))
    {
        return ((int)mask).ToEnumByValue();
    }
    else if (underlyingType == typeof(uint))
    {
        return ((uint)mask).ToEnumByValue();
    }
    else if (underlyingType == typeof(long))
    {
        return ((long)mask).ToEnumByValue();
    }
    else if (underlyingType == typeof(ulong))
    {
        return mask.ToEnumByValue();
    }
    return default;
}

```

## 3、枚举转枚举值中的小细节


我们知道在定义枚举的时候可以指定枚举值类型为以下八种类型sbyte、byte、short、ushort、int、uint、long、ulong，因此所有涉及到返回枚举值的方法，我们要考虑到支持不同类型的枚举值类型返回。


同时我们也知道相同的方法名和入参，并不能通过不同的返回类型区分出不同的重载方法。


方案：因此我们可以通过泛型的方法支持不同类型的枚举值类型返回。


## 4、通过枚举值转换的小细节


通过上面我们知道枚举值类型有八种，因此涉及到枚举值转换出其他信息是需要同时兼容这八种类型。


要实现这个功能，可以用上面提到的泛型，如果使用泛型我们可以使用struct类型来限制枚举值泛型TValue，因为struct可以覆盖枚举值的八种类型，但是也引发了另一个问题就是float、double、DateTime都是struct类型，这样就导致这些类型在编辑器中也可以点出ToEnumByValue等相关方法。


作为一个公共封装方法，这种方式显然是对用户不友好的。


因此我们选择另外一种方式：重载方法来实现，通过封装方法多写一些代码来实现对用户友好调用。


![](https://img2024.cnblogs.com/blog/386841/202411/386841-20241118232259531-586066057.png)


## 5、如何高效返回键值对数据


在通过枚举类型转成各种键值对集合时，有些方法需要用到反射，因此如何高效的获取这些信息就变成重中之重。


最直接的想法是直接把每种枚举类型对应的键值对集合直接缓存起来，下次用到的时候直接从缓存中获取即可。


但是仔细详细，一共有3个数据：枚举值、枚举名称、枚举描述。两两组合有6种情况，也就是要6个缓存存储12个数据，这样其实就相当于浪费了3倍的空间。


当然我们可以把这三个值存一份，然后在需要的是直接组合获取，但是我们仔细分析一些是否有必要把所有数据都缓存下来。


枚举值会涉及到类型转换，枚举名称涉及调用ToString方法，枚举描述需要用到反射。这其中最好性能的非枚举描述不可，因此枚举描述肯定需要缓存，枚举值和枚举名称可以考虑缓存。


假如我们只缓存枚举描述，那么枚举值和枚举名称在使用的时候直接转换，那么我们执行要一份记录枚举描述的缓存以及一份记录枚举类型对应其所有枚举项的缓存即可。另外因为枚举对应的是否带位标志标记以及掩码可以同时记录下来。


代码如下：



```
public static partial class EnumExtension
{
    //枚举类型基础信息
    private sealed class EnumTypeInfo
    {
        //是否是位标志
        public bool IsFlags { get; set; }
        //枚举掩码
        public ulong Mask { get; set; }
        //枚举项集合
        public List Items { get; set; } = [];
    }
    //存储枚举项对应的描述
    private static readonly ConcurrentDictionarystring> _descs = new();
    //存储枚举相关信息

```

获取枚举名称\-枚举描述键值对代码如下，首先获取枚举类型基础信息，然后把枚举项集合转为键值对集合。



```
//获取枚举名称+枚举描述
public static Dictionary<string, string> ToEnumNameDescs(this Type type)
{
    //根据type获取枚举类型基础信息
    var info = GetEnumTypeInfo(type);
    //通过枚举项集合转为目标类型
    return info.Items.ToDictionary(r => r.ToString(), r => r.ToEnumDesc());
}

```

## 6、如何识别一个枚举值是否为有效的位标志组合


如何识别一个枚举值是否是一个有效的位标志组合，可以说是整个代码中最难的部分了，其实前面也有提到，主要应用掩码的思想，上一章节已经详解讲解了这里就不在赘述了。


稍晚些时候我会把库上传至Nuget，大家可以直接使用Ideal.Core.Common。


***注***：测试方法代码以及示例源码都已经上传至代码库，有兴趣的可以看看。[https://gitee.com/hugogoos/Ideal](https://github.com)


