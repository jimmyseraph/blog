---
title: WebUI自动化测试之玩转文件上传
author: 六道先生
date: 2023-02-22 09:15:24
categories:
- IT
tags:
- Selenium
- PlayWright
- UI Automation Testing
+ description: 从代码角度，详细解析如何在UI自动化测试过程处理上传问题。
---

# 背景
很多测试小伙伴在使用`Selenium`或者`PlayWright`这些WebUI自动化测试框架，做一些Web的自动化测试时，经常会遇到网页上可能提供了一个上传文件的功能，点击这个按钮，会弹出操作系统标准文件选择对话框来进行文件选择，然后进行上传操作。这样的一个功能，无论是`Selenium`还是`PlayWright`都无法完成对操作系统文件选择对话框进行操控。针对这种情况，目前一般是采取三种方案来解决：
+ 忽略，在自动化测试案例中不考虑上传功能，只在手工测试中去覆盖。因为通常上传功能不会大量存在，所以作为很小的一部分功能，自动化测试从实现的成本考虑，不去覆盖这个功能，也是OK的。
+ 直接调用上传接口，通过`http`请求直接完成文件上传。这样做也没问题，但是由于没有走页面上的控件，所以上传成功后，不会触发页面上控件的回调函数，页面上也就不会有什么反应。
+ 通过一些基于图形或坐标的自动化工具（比如`sikuli`、`autoIt`等），嵌入到`Selenium`或者`PlayWright`框架中来处理文件选择对话框。但是这么做的缺点，就是对执行机的分辨率、浏览器本身，以及浏览器窗口的大小都会有要求，并且对操作系统也有要求。测试的稳定性和效率都会严重下降。

那么有没有更好的方法呢？

# 上传文件的原理
## 基础原理
在Web前端中，实现上传是通过`<input type="file">`来实现的。看原生`HTML`代码：
```html
<form method="post" enctype="multipart/form-data">
  <div>
    <label for="file">Choose file to upload</label>
    <input type="file" id="file" name="file" multiple />
  </div>
  <div>
    <button>Submit</button>
  </div>
</form>
```
效果像这样：

![input]

在浏览器层面，当用户点击`<input type="file">`按钮时，会启动一个标准的文件选择对话框来选择文件，文件选择后，浏览器将会把文件的内容读取出来，创建出一个`js`的`File`对象，保存在`input Element`的`files`属性中。

当点击上传时，就可以触发`form`的`action`，发起`HTTP`请求，而请求的`body`部分，就来自于`files`属性中的值。

## 新潮的框架
不管是现在的`VUE`还是`React`框架，其实都是这个原理。这里我们以`React`框架为例，使用蚂蚁的`ant-design`来实现一个上传功能。

首先创建一个`react`项目，项目名称假设叫`demo01`：
```bash
npx create-react-app demo01 --template typescript
```

进入`demo01`目录后添加`antd`依赖：
```bash
yarn add antd
```

找到`App.tsx`，用`ant-design`提供的`Upload`控件实现上传功能：
```typescript
import React, { useState } from 'react';
import { UploadOutlined } from '@ant-design/icons';
import { Button, message, Upload } from 'antd';
import type { RcFile, UploadFile, UploadProps } from 'antd/es/upload/interface';

const App: React.FC = () => {
  const [fileList, setFileList] = useState<UploadFile[]>([]);
  const [uploading, setUploading] = useState(false);

  const handleUpload = () => {
    const formData = new FormData();
    fileList.forEach((file) => {
      formData.append('files[]', file as RcFile);
    });
    setUploading(true);
    // You can use any AJAX library you like
    fetch('https://www.mocky.io/v2/5cc8019d300000980a055e76', {
      method: 'POST',
      body: formData,
    })
      .then((res) => res.json())
      .then(() => {
        setFileList([]);
        message.success('upload successfully.');
      })
      .catch(() => {
        message.error('upload failed.');
      })
      .finally(() => {
        setUploading(false);
      });
  };

  const props: UploadProps = {
    onRemove: (file) => {
      const index = fileList.indexOf(file);
      const newFileList = fileList.slice();
      newFileList.splice(index, 1);
      setFileList(newFileList);
    },
    beforeUpload: (file) => {
      setFileList([...fileList, file]);

      return false;
    },
    fileList,
  };

  return (
    <>
      <Upload {...props}>
        <Button icon={<UploadOutlined />}>Select File</Button>
      </Upload>
      <Button
        type="primary"
        onClick={handleUpload}
        disabled={fileList.length === 0}
        loading={uploading}
        style={{ marginTop: 16 }}
      >
        {uploading ? 'Uploading' : 'Start Upload'}
      </Button>
    </>
  );
};

export default App;
```

看一下效果：

![antd-upload]

在浏览器上右键查看下这个`Upload`组件所对应的`HTML`代码：
```html
<div class="ant-upload ant-upload-select">
    <span tabindex="0" class="ant-upload" role="button">
        <input type="file" accept="" style="display: none;">
        <button type="button" class="ant-btn css-dev-only-do-not-override-1km3mtt ant-btn-default">
            <span role="img" aria-label="upload" class="anticon anticon-upload">
                <svg viewBox="64 64 896 896" focusable="false" data-icon="upload" width="1em" height="1em" fill="currentColor" aria-hidden="true"><path d="M400 317.7h73.9V656c0 4.4 3.6 8 8 8h60c4.4 0 8-3.6 8-8V317.7H624c6.7 0 10.4-7.7 6.3-12.9L518.3 163a8 8 0 00-12.6 0l-112 141.7c-4.1 5.3-.4 13 6.3 13zM878 626h-60c-4.4 0-8 3.6-8 8v154H214V634c0-4.4-3.6-8-8-8h-60c-4.4 0-8 3.6-8 8v198c0 17.7 14.3 32 32 32h684c17.7 0 32-14.3 32-32V634c0-4.4-3.6-8-8-8z"></path></svg>
            </span>
            <span>Select File</span>
        </button>
    </span>
</div>
```

可以看到在这个`span`里，有两个关键的组件，一个是`<input type="file">`，另一个是按钮，所以`antd`的`Upload`组件，它的核心还是基本的`<input type="file">`组件。正所谓万变不离其中嘛。

现在我们操作页面，选择一个文件，选完后是这样的：

![select-file]

`antd`的`Upload`组件会在我们选完文件后，在按钮下面出现一个我们所选的文件，以附件的形式展现出来。我们来看下，是否当前的`<input type="file">`元素，它的`files`属性里面是否有这个文件对象了。

在F12弹出的控制台（Console）中，执行下面的语句：

```javascript
const fileInput = document.querySelector("input[type='file']");
fileInput.files
```
可以在控制台看到如下的输出，显示了`files`数组的内容：

![files]

瞧，这时候已经有一个`File`元素在数组中了，这就是我们选择的文件。所以这些框架中提供的上传组件，其实本质上也是利用了原生的`<input type="file">`元素，处理上其实本质上都是一样的。

# 模拟文件选择

既然我们明白了这个原理，那就可以用js来模拟这一行为了。（别忘了，先在浏览器是刷新一下页面，让一切重新开始）

选择文件，无非就是在`<input type="file">`元素的`files`属性里面增加一个`File`对象。我们可以去https://developer.mozilla.org/ 官网去查一下`File`对象如何构建。

官网介绍`File`对象有两个构造器：
```javascript
new File(bits, name)
new File(bits, name, options)
```
参数说明：
> `bits`: 一个可循环的对象，比如一个Array，可用的有ArrayBuffers, TypedArrays, DataViews, Blobs, strings，或者是这些前者的混合元素。这些元素将会被放进文件对象中。注意，如果使用字符串，那么它的编码将会是`UTF-8`，与通常Javascript的`UF-16`字符串不同。

> `name`: 文件的名字，或是指向文件的路径，字符串表示。 <br>
> `options`: 可选参数，用来说明文件的一些属性，可用的有下面两个：<br>
>> `type`: 字符串，文件的`MIME`类型，默认为""。<br>
>> `lastModified`: 一个数字，表达最后修改时间，用毫秒表示。默认为`Date.now()`。<br>

根据这个说明，我们在控制台创建一个文件对象：
```javascript
const file = new File(["abcde"], "demo.txt", {
    type: "text/plain",
});
```
文件名叫`demo.txt`，一个文本文件，内容是“abcde”，5个字节大小。

现在我们需要把我们创建的这个`File`对象放到`<input type="file">`元素的`files`属性里。但是`files`的数组根据官方描述，不能直接修改，所以我们迂回一下，用`DataTransfer`来进行赋值，这也是官方推荐的方式，现在我们先创建`DataTransfer`，然后把`File`对象放进`DataTransfer`中：
```javascript
var dataTransfer = new DataTransfer();
dataTransfer.items.add(file);
```

接下来，我们就可以对`files`属性赋值了：
```javascript
const fileInput = document.querySelector("input[type='file']");
fileInput.files = dataTransfer.files;
```
这时候如果我们和之前一样，执行`fileInput.files`，就可以看到这个数组里有了一个元素，就是我们之前定义的那个`File`对象。可是，页面此时没有任何变化，和之前操作文件选择对话框不同，这时候不会有一个文件名显示在页面中。

什么原因呢？因为我们是使用js直接赋值进去的，没有触发`<input type="file">`元素的`change`事件，所以也就没有出现页面的任何变化。

这个事件我们可用使用js来触发。用到的方法，是元素的`dispatchEvent`方法。先看方法原型：
```javascript
dispatchEvent(event)
```
参数说明：
> `event`: 需要派发的`Event`对象。

再看下`Event`对象该怎么构建，从官网看构造器：
```javascript
new Event(type)
new Event(type, options)
```
参数说明：
> `type`: 字符串，表示事件的名字

> `options`: 可选参数，是一个参数对象，可用以下属性：<br>
>> `bubbles`: 可选参数，布尔值，是否事件需要向上层节点冒泡，默认为false。<br>
>> `cancelable`: 可选参数，布尔值，是否事件可被取消，默认是false。<br>
>> `composed`: 可选参数，布尔值，是否事件可以穿透影子DOM，触发根节点监听器，默认值为false。

我们观察一下`<input type="file">`元素挂了哪些事件：
![listeners]

看，这个元素上只有一个`invalid`事件，而没有`change`事件，而在`<div id="root">`节点上，却有各种事件，其中包括了`change`事件。这就是`react`框架的典型特点，所有事件挂在`<div id="root">`节点上。那我们需要让`<input type="file">`节点的事件“冒泡”，向`<div id="root">`节点传递。所以我们创建一个这样的事件对象：
```javascript
var event = new Event("change", {
    bubbles: true,
});
```
并派发该事件：
```javascript
fileInput.dispatchEvent(event);
```
好了，这时候我们可以看到页面上发生了变化，就像我们使用文件选择对话框选了文件一样，在按钮下面出现了一个叫`demo.txt`的文件。

后面如果要进行上传操作，直接操作`Start Upload`按钮就可以了，这里就不赘述了。

# 自动化实操
明白了这个原理，我们就可以在自动化框架中实现文件上传了，这里我分别用`Selenium`和`PlayWright`两套框架来实操，语言统一使用`Java`。

## Selenium
`Selenium`大家都比较熟悉了，我们直接写一个方法，来实现文件的上传：
```java
/**
 * 处理文件上传时的文件选择
 * @param driver        WebDriver对象
 * @param inputElement  页面<input type='file'>元素
 * @param file          需要准备上传的文件
 * @param mimeType      上传文件的MIME类型
 * @throws IOException  文件读取异常
 */
public static void uploadFile(WebDriver driver, WebElement inputElement, File file, String mimeType) throws IOException {
    // 从给定的文件中读取全部的字节，存入一个List中
    FileInputStream fileInputStream = new FileInputStream(file);
    int c = -1;
    List<Object> bytes = new ArrayList<>();
    while ((c = fileInputStream.read()) != -1) {
        bytes.add(c);
    }
    fileInputStream.close();

    // 准备js脚本
    String js = "var bytes = new Uint8Array(arguments[0]); \n" +
                "const file = new File([bytes], arguments[1], {\n" +
                "    type: arguments[2],\n" +
                "}); \n" +
                "var dataTransfer = new DataTransfer(); \n" +
                "dataTransfer.items.add(file); \n" +
                "arguments[3].files = dataTransfer.files; \n" +
                "var event = new Event(\"change\", {\n" +
                "    bubbles: true,\n" +
                "}); \n" +
                "arguments[3].dispatchEvent(event);";

    ((JavascriptExecutor)driver).executeScript(js, bytes, file.getName(), mimeType, inputElement);
}
```
> 我们采用Java本身的读文件能力，将文件所有的字节读出来，存入一个`List`中，然后通过参数传入js脚本，让js中的`File`对象的第一个文件内容参数，就是使用了这个文件的全部字节数组。

然后在代码中只需要调用就可以了：
```java
public static void main(String[] args) throws IOException{
    ChromeOptions options = new ChromeOptions();
    options.setCapability(ChromeDriverService.CHROME_DRIVER_EXE_PROPERTY, "drivers/chromedriver");
    WebDriver driver = new ChromeDriver(options);
    driver.get("http://localhost:3000");
    WebElement inputElement = driver.findElement(By.cssSelector("input[type='file']"));
    uploadFile(driver, inputElement, new File("files/demo.jpeg"), "image/jpeg");
    new WebDriverWait(driver, Duration.ofSeconds(20L))
            .until(ExpectedConditions.elementToBeClickable(By.cssSelector("#root > button")))
            .click();
    driver.close();
}
```
> 这里假设我的chromedriver文件存放在项目目录的drivers目录下，需要上传的文件放在files目录下，是一个jpeg图片文件。<br>
> 需要注意的是，我在这里做了一个条件等待，之后再去点击上传按钮，这是因为选择了上传文件后，需要有一定时间完成文件字节传入到当前的浏览器JS环境中。

## PlayWright
PlayWright中的实现和Selenium类似，仅有一点小差异而已。我这里展示完整的演示代码：
```java
package vip.testops.pw;

import com.microsoft.playwright.*;

import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class Upload {
    public static void main(String[] args) {
        try (Playwright playwright = Playwright.create()) {
            try (Browser browser = playwright.chromium().launch(
                    new BrowserType.LaunchOptions().setHeadless(false)
            )) {
                Page page = browser.newPage();
                page.navigate("http://localhost:3000");
                ElementHandle inputElement = page.locator("input[type='file']").elementHandle();
                uploadFile(page, inputElement, new File("files/demo.jpeg"), "image/jpeg");
                page.locator("#root > button").click();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 处理文件上传时的文件选择
     * @param page          页面对象
     * @param inputElement  页面<input type='file'>元素
     * @param file          需要准备上传的文件
     * @param mimeType      上传文件的MIME类型
     * @throws IOException  文件读取异常
     */
    public static void uploadFile(Page page, ElementHandle inputElement, File file, String mimeType) throws IOException {
        // 从给定的文件中读取全部的字节，存入一个List中
        FileInputStream fileInputStream = new FileInputStream(file);
        int c = -1;
        List<Object> bytes = new ArrayList<>();
        while ((c = fileInputStream.read()) != -1) {
            bytes.add(c);
        }
        fileInputStream.close();

        // 准备参数
        Map<String, Object> data = new HashMap<>();
        data.put("bytes", bytes);
        data.put("name", file.getName());
        data.put("mimeType", mimeType);
        data.put("inputElement", inputElement);

        // 准备js脚本
        String js = "data => { \n" +
                "var bytes = new Uint8Array(data.bytes); \n" +
                "const file = new File([bytes], data.name, {\n" +
                "    type: data.mimeType,\n" +
                "}); \n" +
                "var dataTransfer = new DataTransfer(); \n" +
                "dataTransfer.items.add(file); \n" +
                "data.inputElement.files = dataTransfer.files; \n" +
                "var event = new Event(\"change\", {\n" +
                "    bubbles: true,\n" +
                "}); \n" +
                "data.inputElement.dispatchEvent(event);\n" +
                "}";

        page.evaluate(js, data);
    }
}
```

# 总结
我们在自动化测试框架中，使用js来避开了操作文件选择对话框的困难，这是处理上传文件最为合理的方式。但是需要注意，因为我们采用了完整的文件字节读取放入js环境中，所以针对大文件，很可能会有内存溢出的问题。js本身在处理大文件时，是使用流（stream）的方式读取文件的，大家可以根据实际项目所需要的方式去实现。


[input]: https://s1.ax1x.com/2023/02/22/pSvpioF.jpg
[antd-upload]: https://s1.ax1x.com/2023/02/22/pSvZyU1.jpg
[select-file]: https://s1.ax1x.com/2023/02/22/pSvexSK.jpg
[files]: https://s1.ax1x.com/2023/02/22/pSvKY1P.jpg
[listeners]: https://s1.ax1x.com/2023/02/22/pSvDbvT.jpg