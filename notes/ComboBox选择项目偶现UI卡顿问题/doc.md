# Avalonia中ComboBox选择项目偶现UI卡顿问题
## 1. 背景

界面的ComboBox的SelectedItem绑定实现了INotifyPropertyChanged接口的ViewModel的属性。在该属性的set中，执行了一系列的逻辑处理，包括几个后台查询接口，ui状态操作等等，代码很多，但关键耗时任务都放在Task中执行。

表现：点击下拉，点击任意项目，Popup关闭，项目被选中时，界面会偶现卡顿，时长大概1s左右。概率不高，在软件启动后，第一次打开该页面，出现的可能性最高，此后出现概率较低。软件静止一段时间（几分钟即可）后，重新操作，又有很大概率出现该问题。

## 2. 代码示意

ViewModel关键代码逻辑抽象如下所示。

```csharp
public UserModel SelectedModel
{
	get { return _selectedModel }
    set 
    {
    	if (_selectedModel != value)
        {
            _selectedMOdel = value;
            
            // 更新相关属性
            this.RelatedProperty1 = value.P1;
			this.RelatedProperty2 = value.P2;
			this.RelatedProperty3 = value.P3;
            this.RelatedProperty4 = value.P4;
			this.RelatedProperty5 = value.P5;
            
            this.OnPropertyChanged(nameof(SelectedModel));
        }
    }
}

public override void OnPropertyChanged(PropertyChangedArgs e)
{
    // 查询服务并将结果反馈到UI。
    QueryFromServiceAndSetToUi(e.PropertyName, e.NewValue);
}

private void QueryFromServiceAndSetToUi(string name, object value)
{
    Task.Run(()=>
    {
    	// Step 1: 在Task中执行查询服务的操作。
    }).ContinueWith(t=>{
    	Dispatcher.UIThread.Post(()=>
		{
            // Step 2: 任务完成时，委托主线程更新UI。
        });
    });
}
```

## 3. 可能性分析

以上代码是抽象的示意代码，实际上代码量非常大，排查困难。有以下几点考虑待排查：

1）Step 2 部分逻辑执行时间较长，直接卡顿了UI；

2）主线程等待了锁，锁在任务线程中释放；

3）错误地在主线程使用了Task.Wait()函数；

4）单纯是设备问题。由于该问题是在公司云内虚拟远程桌面中暴露的，最后考虑可能是设备本身性能问题。因为该设备本身就比较卡顿，执行其他软件也会有卡顿的情况。

对于以上几种可能性依次排查后，发现，1）情况被排除，该步骤代码是一些基本的赋值代码，统计时长都在10ms以内级别。2）情况排查后没有发现错误加锁导致主线程排队的情况。3）情况没有。4）种情况不到最后不想作为结论。

## 4. 借助工具

到这里仅仅通过阅读代码难以找到问题的根本原因。借助工具Performance Profiler查看程序执行的Timeline，如下图1所示，在最后一次卡顿中，UI Freeze显示的冻结时间约为**659ms**。

![图1](img\img1.png)

图1

接着我们查看Hotspots模块，如图2所示，该时段主线程中直接被System.Threading.Monitor.Wait()等待了659ms。这就是UI卡住的直接原因。

![图2](img\img2.png)

图2

再看图3中Wait函数下的调用堆栈，结论比较清晰了，Avalonia的Popup也是一个Win32窗口，其关闭时，会执行渲染器（CompositingRenderer）的销毁，其Dispose函数中，执行了Task.Wait()函数。因此，这个卡顿并不是我们自己的代码直接阻塞了UI线程。那么Avalonia渲染线程被阻塞的原因是什么呢？

![图3](img\img4.png)

图3

## 5. Avalonia的CompositingRenderer解析

以下是CompositingRenderer的Dispose代码。从注释可以看到，合成渲染器释放时，会等待合成批次被应用和渲染以保证渲染目标不再被使用并且可以被安全地处理。问题是，为何这里会导致卡顿，且卡顿是有一定规律（启动第一次或者静置一段时间）的偶发性的呢？

```CSharp
public class CompositingRenderer
{
    public void Dispose()
    {
        Stop();
        CompositionTarget.Dispose();

        // Wait for the composition batch to be applied and rendered to guarantee that
        // render target is not used anymore and can be safely disposed
        if (Compositor.Loop.RunsInBackground)
            _compositor.Commit().Wait();
    }    
}
```

## 6. 关于.Net内置线程池

问题到这里算比较清晰了。**在实现为Win32窗口的Popup关闭时，清理合成渲染器的时候会等待现有批次任务执行完毕在继续执行，这里可能造成UI卡顿**。我们先说结论。造成这里被阻塞的原因是，**Task被挂起在队列里面没有及时得到执行，因此这里Task.Wait()仍然在队列里面等待执行，导致主线程阻塞**。

简单来说，这是.Net内置线程池的特性。.Net的Task对象会被扔进内置的线程池（ThreadPool）中执行，而线程池中闲置的线程是由其MaxThreadCount和MinThreadCount控制的，一开始，线程池会准备最小数量的闲置线程备用。当任务进入队列时，线程池将任务依次分配给队列中的任务。而当闲置的线程被分配完了，但是队列中仍然存在任务时，线程池将会按一定时长（约1s）间隔来创建新的线程去执行队列中的任务，直到队列清空或者线程总数达到MaxThreadCount。

因此，线程池队列中的任务会有两种情况会被阻塞等待。一是线程达到最大数量，二是当前线程数量不够，线程仍然在增长过程中，这是有间隔时间的。

## 7. 问题结论

那么在我们的示例中，ComboBox选中项目的同时，通知ViewModel的SelectedModel属性，此时在其set中创建了5个（示例数量）Task去执行后台查询，紧接着Popup被关闭，其Dispose提交了合成渲染的任务，如果此时线程池线程被前面的任务占用完毕，没有空闲线程，该任务就被挂起等待，直到有任务释放了线程，或者线程池创建了新线程，该任务才会被取出执行。

## 8. 加深理解

我们尝试100%复现这个问题来加深理解。在SelectedModel的set中，修改代码如下。

```CSharp
public UserModel SelectedModel
{
	get { return _selectedModel }
    set 
    {
    	if (_selectedModel != value)
        {
            _selectedMOdel = value;
            
            // --- 新增代码 ---
            for(int i = 0; i < 10; i++)
            {
                Task.Run(()=>
                {
                	Thread.Sleep(1000);
                })
            }
            // --- 新增代码 ---
            
            this.OnPropertyChanged(nameof(SelectedModel));
        }
    }
}
```

此时，这个卡顿在一开始的几次将会必现，且等待时长将超过1s。下图简单示意了任务在线程池中的调度过程。根据图示，最后的Task11（渲染线程提交的任务），将会在5s后开始执行，即UI将会被阻塞5s。当然这些数据是示意数据。实际上，ThreadPool调度时，创建线程的间隔时长是一个根据系统环境、程序运行情况、等待线程数量等多种因素来计算获得的，实际上等待时间可能大于或者小于5s，但一般情况下差距不会太大。

![img4](img\img5.png)

图4

## 9. 解决方案

据以上分析，在关闭Popup或者关闭窗口同时触发的变更中，创建大量（Count >= 4）后台任务时，建议手动设置ThreadPool的MinThreadCount，保证可用线程数量大于待执行数量+1，避免ui线程提交的任务被阻塞。当然，用完应当及时还原ThreadPool的MinThreadCount。
