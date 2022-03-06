---
title: 用jmeter-grpc-request性能测试的严重问题
author: 六道先生
date: 2022-03-04 18:00:38
categories:
- IT
tags:
- 性能测试
- gRPC
- jmeter
+ description: 近期有小伙伴在用jmeter-grpc-request插件做gRPC接口的性能测试，但是跟我说刚跑没多久就卡死了，毕竟是公司的同事，赶紧披挂上阵帮助小伙伴解决问题。
---
# 起因
今日收到一个同事的求救信息，说正在做gRPC接口测试，用的是jmeter的一个第三方插件，叫[jmeter-grpc-request](https://github.com/zalopay-oss/jmeter-grpc-request/releases)，平日用着挺好用的，今天设置了100个线程，持续跑，结果才跑了5000来个请求，就卡住了。 <br>
卡住了？什么是卡住了呢？<br>
我仔细问了，才知道是jmeter整个没有响应了，只能强行杀进程才能停止。这是怎么回事呢？
# 场景重现
我问同事要了jmeter的脚本文件，并且下载了这个gRPC取样器的插件，在我本机试了一下，果然，线程数量很少时候，运行正常，但是数量多了一些（仅仅到了50），很快就出现了jmeter无响应的情况。<br>
根据经验，立刻切到启动jmeter的命令行界面，看到的提示是内存溢出（OutOfMemoryException）。<br>
这是很奇怪的现象，按理来说，仅仅是50个线程，才跑了几千的请求量，怎么就内存溢出了呢？我们知道，jmeter默认的运行内存是1G（HEAP="-Xms1g -Xmx1g -XX:MaxMetaspaceSize=256m"）几千的请求量，什么东西占用了1G的Heap空间？
# 内存分析
jmeter工具还是挺方便的，在出现了内存溢出后，自动dump出了此时的JVM情况，在当前的运行目录下生成了`java_pid<id>.hprof`文件（id是当时jmeter的进程ID）。<br>
所以我用jhat命令来读取hprof文件，看看到底什么东西占用了这么多内存：
```Bash
$ jhat -port 7001 java_pid<id>.hprof
```
> 这里我用了`-port`参数指定了7001端口，因为默认的7000端口已经被我机器上别的程序占用了。<br>

经过一段时间的等待（dump出来的hprof文件文件太大了，上G），命令行提示已读取完成，此时jhat会启动一个Web服务器，打开浏览器输入`http://localhost:7001`就可以看到jvm中加载的所有对象。<br>
当然，我要看的是Heap中这些对象的占用空间情况，所以通过最底下的链接，切换到了heap内存分析页：<br>
{% asset_img hprof.jpg hprof %}

> 很明显，排在前几行的就是“罪魁祸首”了。<br>
也许有的小伙伴不清楚，`class [B`是什么，其实就是java中的byte数组，居然占了300多MB的内存空间。<br>
其次，就是`FiledDescriptor`对象，占了200多MB的内存空间。这两个大户，耗费了一半的Heap内存，难怪内存不够了。<br>

那么现在的问题就在于两个地方，
1. 这个byte数组，存了什么，为何这么多；
2. 这个`FiledDescriptor`为何这么多实例。

这就只能从代码角度来分析了。
# 代码分析
根据对jmeter组件开发的了解，代码从继承`AbstractSampler`的Sampler开始。当在jmeter中开始运行取样器时，执行的就是sample方法，仔细看看sampler中的sample方法：
```Java
@Override
public SampleResult sample(Entry ignored) {
    GrpcResponse grpcResponse = new GrpcResponse();
    SampleResult sampleResult = new SampleResult();
    try {
        initGrpcClient();
        sampleResult.setSampleLabel(getName());
        String grpcRequest = clientCaller.buildRequestAndMetadata(getRequestJson(),getMetadata());
        sampleResult.setSamplerData(grpcRequest);
        sampleResult.setRequestHeaders(clientCaller.getMetadataString());
        sampleResult.sampleStart();
        grpcResponse = clientCaller.call(getDeadline());
        sampleResult.sampleEnd();
        sampleResult.setSuccessful(true);
        sampleResult.setResponseData(grpcResponse.getGrpcMessageString().getBytes(StandardCharsets.UTF_8));
        sampleResult.setResponseMessage("Success");
        sampleResult.setDataType(SampleResult.TEXT);
        sampleResult.setResponseCodeOK();
    } catch (RuntimeException e) {
        errorResult(grpcResponse, sampleResult, e);
    }
    return sampleResult;
}
```
`initGrpcClient()`方法，应该是作者写的一个初始化gRPCClient的方法，进去看看：
```Java
private void initGrpcClient() {
    if (clientCaller == null) {
        clientCaller = new ClientCaller(
            getHostPort(),
            getProtoFolder(),
            getLibFolder(),
            getFullMethod(),
            isTls(),
            isTlsDisableVerification());
    }
}
```
嗯，果然是new了一个ClientCaller对象。里面的get方法，is方法都不用管，明显是从sampler界面中获取参数值，跟我们要探寻的东西无关。那么我们要到ClientCaller的构造方法里面看看：
```Java
public ClientCaller(
    String HOST_PORT, 
    String TEST_PROTO_FILES, 
    String LIB_FOLDER, 
    String FULL_METHOD, 
    boolean TLS, 
    boolean TLS_DISABLE_VERIFICATION) {
    this.init(HOST_PORT, TEST_PROTO_FILES, LIB_FOLDER, FULL_METHOD, TLS, TLS_DISABLE_VERIFICATION);
}

private void init(
    String HOST_PORT, 
    String TEST_PROTO_FILES, 
    String LIB_FOLDER, 
    String FULL_METHOD, 
    boolean TLS, 
    boolean TLS_DISABLE_VERIFICATION) {
    try {
        tls = TLS;
        disableTtlVerification = TLS_DISABLE_VERIFICATION;
        hostAndPort = HostAndPort.fromString(HOST_PORT);
        metadataMap = new LinkedHashMap<>();
        channelFactory = ChannelFactory.create();
        ProtoMethodName grpcMethodName =
                ProtoMethodName.parseFullGrpcMethodName(FULL_METHOD);

        // Fetch the appropriate file descriptors for the service.
        final DescriptorProtos.FileDescriptorSet fileDescriptorSet;

        try {
            fileDescriptorSet = ProtocInvoker.forConfig(TEST_PROTO_FILES, LIB_FOLDER).invoke();
        } catch (Throwable t) {
            shutdownNettyChannel();
            throw new RuntimeException("Unable to resolve service by invoking protoc", t);
        }

        // Set up the dynamic client and make the call.
        ServiceResolver serviceResolver = ServiceResolver.fromFileDescriptorSet(fileDescriptorSet);
        methodDescriptor = serviceResolver.resolveServiceMethod(grpcMethodName);

        createDynamicClient();

        // This collects all known types into a registry for resolution of potential "Any" types.
        registry = JsonFormat.TypeRegistry.newBuilder()
                .add(serviceResolver.listMessageTypes())
                .build();
    } catch (Throwable t) {
        shutdownNettyChannel();
        throw t;
    }
}
```
嗯，代码有点长，主要方法是init，里面看到了一个有嫌疑的语句：
```Java
fileDescriptorSet = ProtocInvoker.forConfig(TEST_PROTO_FILES, LIB_FOLDER).invoke();
```
`invoke`方法返回了一个`FileDescriptorSet`对象，这个对象，是Google的ProtoBuf的核心对象。那么他到底是怎么获得的？这引起了我的兴趣。所以果断跟了进去：
```Java
public FileDescriptorSet invoke() throws ProtocInvocationException {
    Path wellKnownTypesInclude;
    Path googleTypesInclude;
    try {
        wellKnownTypesInclude = setupWellKnownTypes();
    } catch (IOException e) {
        throw new ProtocInvocationException("Unable to extract well known types", e);
    }

    Path descriptorPath;
    try {
        descriptorPath = Files.createTempFile("descriptor", ".pb.bin");
    } catch (IOException e) {
        throw new ProtocInvocationException("Unable to create temporary file", e);
    }

    // Large folder processing, solve CreateProcess error=206
    final ImmutableSet<String> protoFilePaths = scanProtoFiles(discoveryRoot);
    ImmutableList<String> protocArgs = null;

    if (protoFilePaths.size() > largeFolderLimit) {
        try {
            File argumentsFile = createFileWithArguments(protoFilePaths.toArray(new String[0]));
            protocArgs = ImmutableList.<String>builder()
                    .add("@" + argumentsFile.getAbsolutePath())
                    .addAll(includePathArgs(wellKnownTypesInclude))
                    .add("--descriptor_set_out=" + descriptorPath.toAbsolutePath().toString())
                    .add("--include_imports")
                    .build();
        } catch (IOException e) {
            logger.error("Unable to create protoc parameter file", e);
        }
    }

    if (protocArgs == null) {
        protocArgs = ImmutableList.<String>builder()
                .addAll(protoFilePaths)
                .addAll(includePathArgs(wellKnownTypesInclude))
                .add("--descriptor_set_out=" + descriptorPath.toAbsolutePath().toString())
                .add("--include_imports")
                .build();
    }

    invokeBinary(protocArgs);
    try {
        return FileDescriptorSet.parseFrom(Files.readAllBytes(descriptorPath));
    } catch (IOException e) {
        throw new ProtocInvocationException("Unable to parse the generated descriptors", e);
    }
}
```
一大段代码，从开始到`invokeBinary(protocArgs)`，都为了做一件事情，根据从界面获取的proto文件的根目录，来生成对应的protoc命令行的参数，像`descriptor_set_out`，`include_imports`，这些不都是protoc工具的参数嘛。那么，`invokeBinary(protocArgs)`这里必然是执行了protoc命令，而protocArgs就是那些传入的参数，根据这些参数，可以知道protoc命令将指定的proto源文件编译为`.pb`的二进制文件，并存放在descriptorPath目录下。注意，不管多少个proto文件，都会被编译到一个pb文件中，这个文件，就是所谓的FileDescriptorSet的序列化后的内容。<br>
然后就是
```Java
return FileDescriptorSet.parseFrom(Files.readAllBytes(descriptorPath));
```
parseFrom方法就是从descriptorPath目录下读取pb文件，readAllBytes就是按字节读取文件内容的方法，返回byte数组，并反序列化成FileDescriptorSet对象。<br>
代码逻辑上讲，没什么问题，但是问题就在于，当进行多线程压测时，每运行一次，就会编译并读取一次pb文件，发上千次请求的时候，岂不是读取了上千次？另外，我们公司的proto文件大概有数百个，都放在一个目录下面，而进行压测的小伙伴也没有把依赖的proto文件单独拉出来，直接读取了整个大目录，导致这个编译后的pb文件里面囊括了公司的全部proto文件，这么一来，岂不是很容易导致内存溢出吗？这就是真正出问题的原因。
# 修改插件源码
处理方式嘛，自然也就比较简单了，把这个FileDescriptorSet对象，当然还有一些其他对象，我把这些本来的fields改造成单例，也就是静态对象：
```Java
private static DescriptorProtos.FileDescriptorSet fileDescriptorSet;
private static Descriptors.MethodDescriptor methodDescriptor;
private static ServiceResolver serviceResolver;
private static JsonFormat.TypeRegistry registry;
```
而代码中原来给这些对象赋值的地方，则改成是否为空的判断：
```Java
if(ClientCaller.fileDescriptorSet == null) {
    ClientCaller.fileDescriptorSet = ProtocInvoker.forConfig(TEST_PROTO_FILES, LIB_FOLDER).invoke();
}
```
这样一旦判断有值，就不会反复进行读取值了。<br>
改完了源码，将插件重新进行编译，又发现了源码居然还少一个依赖，也不知道作者日常是怎么编译成功的，哈哈，赶紧补上，并且顺带把他的依赖库升了一下级。<br>
然后执行：
```Bash
$ mvn package
```
这样打包成jar包后，重新复制到jmeter的`lib/ext`目录下，测试了一下100线程300s的压测，顺利完成。嗯，看来问题解决了。
# 不足
因为FileDescriptorSet被我简单的改成了单例，所以一旦读取过值之后，就不会再次赋值，导致在jmeter中，用过一次这个插件后，要想改测别的proto，会失败，因为不会重新编译并读取了。所以要重新打开jmeter才能改新的测试接口。算是一个小问题吧，不过影响不大，暂时先改到这个程度了。