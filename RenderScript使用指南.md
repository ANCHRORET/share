# RenderScript使用指南 #
## 一、入口 ##
[https://developer.android.google.cn](https://developer.android.google.cn) -> API指南 -> 库 -> 功能 ->
v8 支持库 -> [RenderScript 开发者指南](https://developer.android.google.cn/guide/topics/renderscript/compute.html#access-rs-apis)  
**本文章是对上面内容的翻译**  
## 二、RenderScript介绍 ##  
RenderScript是一个框架，这个框架是为了可以在性能好的android机器上执行密集计算的任务。RenderScript是以并行的数据计算为设计原则，尽管对串行的工作也有益。RenderScript在机器的可用处理器上并行运行，例如多核的cpu和GPU。这个允许你更多的关注算法而不是更多的关注安排工作的执行顺序。
RenderScript特别擅长执行图像处理，计算摄影以及计算机视觉。  

----------
使用RenderScript之前，要明白下面几个理念：  
- 使用c99-derived language来完成性能更好的运算代码。下面的章节***Writing a RenderSctipt Kernel***描述了怎么使用它来写一个运算内核。  
- 控制API用于组织RenderScript资源和控制内核执行器的生命周期。运行在三种不同的语言下：Java,android NDK中的C++和c99派生出的内核语言。下面的章节分别描述了第一种***在Java中使用RenderScript（Using RenderScript from Java Code）***和第三种***单一来源的RenderScript(Single-Source renderScript)***。  
  
## 三、Writing a RenderScript Kernel ##  
渲染脚本的内核通常会在**<project_root>/src/**文件加下的**.rs**文件中，每个**.rs**被称为一个脚本。每个脚本包含它自己的一组内核，函数，和变量。一个脚本包含：  

- pragma声明（**\#pragme version(1)**）:脚本中使用渲染脚本内核使用的语言版本，而且只能是**1**  
- pragma声明（**\#pragma rs java_package_name(com.example.app)**）：定义从这个脚本中映射出对应Java类的包名。要切记，**.rs**文件必须是应用包里的代码的一部分，而不是位于依赖库工程中。  
- 0或更多个**执行函数**：执行函数是单线程的渲染脚本函数，可以在Java代码中通过任意参数调用这个函数。执行函数通常用于在较大的进程管线内的初始化安装和串行计算。  
- 0或更多个**脚本全局**：每个脚本全局等同于c中的一个全局变量。可以在Java中访问脚本全局，脚本全局经常作为参数传给渲染脚本内核。  
- 0或更多个**计算内核**：计算内核是单个函数或函数集合，计算内核可以穿过数据集合在渲染脚本运行时并行执行，有两种计算内核：映射内核（也被成为foreach 内核）和减少内核。  
<br />映射内核是一个并行的函数，它运行于相同维度的**Allocation**的集合中。默认的，这些维度中的每一个坐标上的元素执行这个函数一次。它通常（不一定）用于将输入**Allocation**的集合转换为输出**Allocation**的集合。
    - 下面是一个简单映射内核的例子：  
            uchar4 RS_KERNEL invert(uchar4 in, uint32_t x, uint32_t y) {
              uchar4 out = in;
              out.r = 255 - in.r;
              out.g = 255 - in.g;
              out.b = 255 - in.b;
              return out;
            }  
在很多层面，这与标准的c函数相同。在函数原型中使用的**RS_KERNEL**指明这个函数是一个**渲染脚本映射内核**而不是一个**可执行函数**。基于启动内核时的输入**Allocation**，**in**参数被自动填充，x和y参数会在下面讨论。内核的返回值会被写入到输出**Allocation**的对应位置。内核运行在整个的输入**Allocation**上，在**Allocation**的每一个元素上执行这个内核函数一次。  
<br /> 映射内核可能有一个或更多的输入**Allocation**，单个或两个输出**Allocation**。渲染脚本运行的时候会检查并确保所有的输入和输出**Allocation**有相同的维度，而且每个元素的类型也要与内核的原型(**uchar4 in**)匹配；如果不匹配，会抛出异常。**注意：**在Android6.0（API level 23）之前，映射内核只具有不高于一个的输入分配内存。
<br />如果你需要更多的输入和输出**Allocation**，这些对象要和名字为**rs_allocation**的脚本全局绑定，并且在内核或者可执行函数中，只能通过**rsGetElementAt_*type*()**或者**rsSetElementAt_*type*()**来访问。**注意：**为了方便，RS_KERNEL是渲染脚本自动产生的宏定义。  
            #define RS_KERNEL __attribute__((kernel))  
  
  **减少内核**是一组函数，它运行在相同维度的输入**Allocation**上。默认的，**累加器函数**会在**Allocation**中的每一个坐标的元素上执行一次。这个内核一般用来减少输入**Allocation**的集合将之变成一个。
  - 下面是一个简单地减少内核的例子，功能是将输入的元素加起来。  
        #pragma rs reduce(addint) accumulator(addintAccum)
        
        static void addintAccum(int *accum, int val) {
          *accum += val;
        }  
减少内核包含一个或更多用户写的函数。**\#pragma rs reduce**通过指定它的名字（例如，**addint**），组成内核的函数的名字和角色（例如，一个**accumulator**函数**addintAccum**）来定义内核。函数必须是静态的。减少内核需要**累加器**函数；它或许还有别的函数，取决于你想让他干什么。  
<br />减少内核的累加器函数必须返回**void**，必须具有最少两个参数。第一个参数（例如 accum）是累加数据项的指针，第二个参数（例如 val）会基于内核启动时传进来的输入**Allocation**参数自动填充。累加数据项是在渲染脚本运行时产生的；默认值为0。默认的，这个内核运行于整个输入**Allocation**上，分配内存中的每个元素执行累加器函数一次。默认的，累加数据项的最终的值被视为**减少内核**的结果，返回到Java中。渲染脚本运行时会检查并确保输入**Allocation**中的元素和累加器函数的原型匹配，如果不匹配，渲染脚本会抛出异常。  
<br />减少内核拥有一个以上的输入**Allocation**，但是没有输出**Allocation**。  
想了解**减少内核**更多信息，查看下面的部分***Reduction Kernels in Depth***。  
**减少内核**的支持版本是Android7.0或以上。  
  
  **映射内核**的函数或者**减少内核**的累加器函数会获取当前执行到哪个坐标（**Allocaiton**的坐标），通过传入int/uint32_t类型的参数x,y,和z，这些参数是可选的。  
<br />**映射内核**的函数或者**减少内核**的累加器函数也可以采用可选的**rs_kernel_context**类型的上下文作为参数。一些运行时API需要这个参数来查询当前执行器的属性--例如，**rsGetDimX**。（上下文参数在Android6.0（API level 23）或以上才可用）。  
- 可选的**init()**函数。init()函数是一种特殊的可执行函数，渲染脚本会在脚本第一次初始化的时候调用它。这样可以在脚本创建的时候自动进行一些运算。  
- 0或更多个**静态**脚本全局和**静态**函数。静态的脚本全局和普通的脚本全局一样，但是它不能从java中访问。静态的函数是一个标准的c函数，它可以被脚本中的任何内核和可执行函数调用，但是不暴露给Java的API。如果脚本全局或者函数不需要再java代码中调用，强烈建议将其声明为**静态的**。  
  
#### 设置浮点指针的精度 ####
在脚本中你可以控制浮点类型的精度，如果不需要使用IEEE 754-2008标准（default），这点就很有用了。下面的pragma可以被设置为不同的浮点指针精度。  
- **\#pragma rs_fp_full**(不指定时的默认值)：对于需要浮点精度的应用程序，如IEEE 754-2008标准所述  
- **\#pragma rs_fp_relaxed**:对于不需要严格的IEEE 754-2008标准的应用程序，可以使用更低的精度,这种模式可以使用**刷新到零**或者**四舍五入到零**。  
- **\#pragma rs_fp_imprecise**:对于不需要严格精度的应用。这种模式使得rs_fp_relaxed模式下的所有东西遵循下面的规则：  
    - 结果是-0.0的可以用+0.0代替  
    - INF（无穷大）和NAN（无效数字）没有被定义  
  
许多应用使用rs_fp-relaxed也不会有什么副作用。对于仅需要放松进度就可以优化的架构非常有用（就像SIMD CPU instructions）  

## 四、获取RenderScript APIs ##
开发android应用时，你可以有两种方法获取到RenderScript API：  
1、当你开发的app最低版本是Android3.0（API level 11）时，你可以直接使用android.renderscript  
2、在support library里面，有android.support.v8.renderscript，最低支持到Android2.3（API level 9）。  

下面是你要考虑的点：  
- 如果你使用**Support Library APIs**，无论你想使用是RenderScript的什么功能，都会兼容到android2.3版本。这样可以允许你的应用运行在更多的设备上。  
- 部分RenderScript特性将会无效  
- 如果你使用 **Support Library APIs**，与使用 **native APIs** 相比，你的apk可能会更大  
  
#### 使用RenderScript Support Library APIS ####  
你必须配置你的开发环境才可以获取到支持库。下面这些是必须的：  
- Android SDK Tools version >= 22.2  
- Android SDK Build-tools version >= 18.1.0  
  
你可以通过[Android SDK Manager](https://developer.android.google.cn/tools/help/sdk-manager.html)来检查和更新这些工具的版本  
<br />如何使用支持库  
1. 确认上面工具的版本满足  
2. 更新android构建指引的设置
   - 打开你的应用的module的**build.gradle**
   - 将下面的RenderScript的设置添加进去  

            android {
                compileSdkVersion 23
                buildToolsVersion "23.0.3"
            
                defaultConfig {
                    minSdkVersion 9
                    targetSdkVersion 19
            
                    renderscriptTargetApi 18
                    renderscriptSupportModeEnabled true
            
                }
            }  
上面的设置控制了构建过程的下列行为  
    - **renderscriptTargetApi**-会产生指定的字节码版本。在保证功能的前提下，我们建议将这个值设置为需要支持库（**需要将renderscriptSupportModeEnabled 设置为 true**）的最低版本。这个设置的有效值是大于Android3.0（API level 11）的整型值，小于最新发布的API的正式版。如果app中manifest指定的**最低sdk版本**与这个值不同（我认为意思是小于这个值），**build.gradle**中设置的值会被忽略，并且被设置为manifest中的**最低sdk版本**。  
    - **renderscriptSupportModeEnabled**-如果应用所运行的设备不支持目标版本，回馈到兼容版本产生特定的字节码。  
    - **buildToolsVersion**-构建工具的版本。这个版本需要>=18.1.0。如果这个选项没有指定，会使用安装的构建工具的最高版本。你应该指定这个选项来确保在不同的开发机器上能有一致的配置。  
3. 如果你的app要使用RenderScript，添加下面的支持库：  
        import android.support.v8.renderscript.*;  
  
## 五、在Java中使用（Using RenderScript from Java Code ##  
在Java使用RenderScript需要依赖**android.renderscript**或者**android.support.v8.renderscript**包里面的API。下面是基本的使用模式：  
1. **Initialize a RenderScript context**  这个**RenderScript对象**通过**RenderScript.create(Context)**方法创建，确保**RenderScript对象**可以被使用而且会提供一个对象来控制后续的RenderScript的对象的生命周期。你应该将context的创建视为一个潜在的长时间运行的操作，因为他会在不同的硬件上创建资源；如果可能的话，初始化操作不应该在app的关键路径（我猜是不是说要延迟初始化）。通常，一个应用在一个阶段只会有一个**RenderScript Context**实例。  
2. **Create at least one Allocation to be passed to a script**  **Allocation**是RenderScript用到的对象，它用来给定量的数据提供存储空间。脚本中的内核将**Allocation**对象作为它的输出和输入。可以内核中通过***rsGetElementAt_type()***和***rsSetElementAt_type()***来访问这个对象。**Allocation**允许将数组从Java代码传递到RenderScript代码，反之亦然。**Allocation**对象通常使用**createTyped()**或者**createFromBitmap()**方法来创建。  
3. **Create whatever scripts are necessary.**当使用RenderScript的时候，有两种类型的脚本：  
    - **ScriptC**：这是一个用户定义的脚本，在上面***Writing a RenderScript Kernel***有描述过如何写一个脚本。为了在Java代码中方便的获取这个脚本，每个脚本都有一个通过RenderScript编译器映射出来的Java类；这个类的名字是**ScriptC_*filename***。例如，如果上面的内核位于**invert**。rs和**RenderScript context**也已经在mRenderScript中创建，初始化脚本的代码是：
            Script_invert invert = new Script_invert(mRenderScript)
    - **ScriptIntrinsic**：这个脚本创建于RenderScript内核，为了一般的操作，就像**Gaussian blur（高斯模糊），convolution(卷积)和 image blending（图像混合）**。想了解更多的信息，看一下**ScriptIntrinsic**的子类。
4. **Populate Allocations with data**  除了通过**createFromBitmap()**方法创建的对象，Allocation被创建时也可以通过空数据来填充。通过**Allocation**的**copy**方法来填充，这个**copy**方法是同步的。  
5. **Set any necessary script globals**  在**Script_*filename***类中，你可以使用**set_*globalname***方法来设置。你的脚本中有一个**脚本全局**名字是**threshold**，可以通过**set_threshold(int)**方法来设置；通过**set_lookup(Allocation)**方法设置一个as_allocation类型的**脚本全局**变量**lookup**。这些**set方法**是**异步的**。  
6. **Launch the appropriate kernels and invokable functions**  
通过**ScriptC_*filename***类中的方法**forEach_*mappingKernelName*()**或**reduce_*reductionKernelName*()**来启动给定内核。这个启动时**异步**的。根据内核中设定的参数，这个方法采用一个或更多的**Allocation**，所有的**Allocation**都必须设定相同的维度。默认的，内核在这些维度的每个坐标上执行；想在这些坐标的子集上运行内核，需要传递合适的参数**Script.LaunchOptions**作为**forEach**和**reduce**方法的最后一个参数。使用**ScriptC_*filename***类中方法**invoke_*functionName***来启动时，这些启动是异步的。  
7. **Retrieve data from Allocation objects and [*javaFuturetype*](http://#) objects**。为了在Java代码中从**Allocation**获得数据，你要使用Allocation中的**'copy'**方法将数据copy到java中。想要从**减少内核**中获取结果，必须使用***javaFutureType*.get()**方法。上面说的**'copy'**和**get()**方法是**同步的**。  
8. **Tear down the RenderScript context**。你可以通过下面的方法销毁**RenderScript context**，**destroy()**或者允许**RenderScript context对象进行垃圾收集**。之后，你使用任何属于这个context的对象都会抛出异常。

#### 异步执行模式 Asynchronous execution model ####  
方法**forEach,invoke,reduce,and set**都是异步的，每个方法都有可能在完成目标操作之前返回（返回是不是在说结束）。但是，单个操作是串行的，依照他们启动的顺序。  
**Allocation**类提供**'copy'**方法copy数据给Alloction对象，或者从Allocation对象中copy出数据。**'copy'**方法是同步的，是串行的（相对于上述的异步操作方法操作相同Allocation）。  
映射的***javaFutureType***类提供一个**get()**方法从reduction中获取结果，**get()**方法是同步的，是串行的（相对于异步的reduction）。  
  
## 六、单一来源的渲染脚本（Single-Source RenderScript） ##  
Android 7.0 (API 24) 介绍了一种新的编程特性-Single-Source RenderScript,新的特性中，在script定义的时候，kernels从script中产生而不是从Java中产生。现在这种方式局限于**映射内核**中，在这个部分中简称为"kernels"。这种新的特征也支持在**script**中创建**rs_allocation**类型的**Allocation**。现在可以在单独的脚本中实现整个算法，即使需要启动多个kernels。优点是双重的：1.更多的可读代码，因为他用一种语言来实现算法；2.潜在的，存在实现更快的代码的可能，因为在多个kernels启动时，Java和RenderScript之间的转换变少了。  
<br />在Single-Source RenderScript中，你写一个**Writing a RenderScript Kernel**部分中描述的kernels。写一个叫做 **rsForEach() **可执行的功能来启动kernels。这个方法采用一个内核函数作为其第一个参数，接下来是输入**Allocation**和输出**Allocation**。里一个相似的方法 **reForEachWithOptions()** 采用一个额外的参数 **rs_script_call_it**，这个参数指定一个输入**Allocation**和输出**Allocation**的元素子集作为内核函数执行的空间。  
<br />按照 **Using RenderScript from Java Code** 部分中的步骤，可以调用Java中特定执行函数来开启RenderScript 运算。在 **launch the appropriate kernels** 这一步中，调用一个可执行函数 **invoke_*function_name*()** 来开启整个运算，包括启动kernels。  
<br />从一个内核启动到另一个内核启动，经常需要使用 **Allocation** 保存和及时传递结果。你可以通过 **rsCreateAllocation()** 来创建他们。这个API的一种方便使用的方式是**reCreateAllocation_<T><W>(...)**,***T***是元素的数据类型，***W***是这个元素的矢量宽度。这个函数采用X,Y和Z这三个维度作为参数，对于1D和2D的allocation,Y或Z维度的尺寸会被忽略。例如，**reCreateAllocation_uchar4(16384)**创建了一个含有16384个一维元素的分配空间，每一个元素都是uchar4类型的。  
<br />分配的空间由系统自动管理，你不必特地的去释放他们。但是，你可以调用 **rsClearObject(rs_allocation\* alloc)** 来表明你不再需要这块内存，这样系统能尽快的将这块资源释放掉。  
<br />**Writing a RenderScript Kernel**部分有一个转换图片简单例子。下面使用 **Single-Source RenderScript** 拓展这个例子，对这个图片做更多的操作，它包含另外一个内核，**greyscale**,用来将一个彩色的图片转换为黑白的。**process()**方法将两个内核连续的应用于输入的图片（输入的图片**Allocation**），产生一个输出图片。输入 **Allocation** 和输出 **Allocation** 作为 **rs_allocation** 的参数传入。  

    // File:singleSource.rs
    
    #pragma version(1)
    #pragma rs java_package_name(com.android.rssample)
    
    static const float4 weight = {0.299f, 0.587f, 0.114f, 0.0f};
    
    uchar4 RS_KERNEL invert(uchar4 in, uint32_t x, uint32_t y) {
      uchar4 out = in;
      out.r = 255 - in.r;
      out.g = 255 - in.g;
      out.b = 255 - in.b;
      return out;
    }
    
    uchar4 RS_KERNEL greyscale(uchar4 in) {
      const float4 inF = rsUnpackColor8888(in);
      const float4 outF = (float4){ dot(inF, weight) };
      return rsPackColorTo8888(outF);
    }
    
    void process(rs_allocation inputImage, rs_allocation outputImage) {
      const uint32_t imageWidth = rsAllocationGetDimX(inputImage);
      const uint32_t imageHeight = rsAllocationGetDimY(inputImage);
      rs_allocation tmp = rsCreateAllocation_uchar4(imageWidth, imageHeight);
      rsForEach(invert, inputImage, tmp);
      rsForEach(greyscale, tmp, outputImage);
    }  
你可以像下面一样在Java中调用**process()**函数：  
  
    // File SingleSource.java
    
    RenderScript RS = RenderScript.create(context);
    ScriptC_singlesource script = new ScriptC_singlesource(RS);
    Allocation inputAllocation = Allocation.createFromBitmapResource(
        RS, getResources(), R.drawable.image);
    Allocation outputAllocation = Allocation.createTyped(
        RS, inputAllocation.getType(),
        Allocation.USAGE_SCRIPT | Allocation.USAGE_IO_OUTPUT);
    script.invoke_process(inputAllocation, outputAllocation);  
这个例子展示了完全通过RenderScript语言来实现一个涉及到两个内核的算法。没有 **Single-Source RenderScript** ，你必须在Java中启动两个内核，分别启动两个内核会使整个算法变得难以理解。**Single-Source RenderScript** 特性不单使代码更容易看懂，它还消除了内核启动期间 **Java** 和 **script** 之间的过渡。一些迭代算法会启动内核上百次，使得这种转换的开销相当的大。

## 七、Reduction Kernels in Depth ##
...