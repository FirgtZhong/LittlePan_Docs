# API使用文档

**示例网盘地址：<https://cloud.firgt.cn>**
实际的API接口地址应改成：https://你自己的网盘网址/api.php

## 说明

**示例API接口地址：https://cloud.firgt.cn/api.php** API支持JSON、JSONP、FORM 3种返回方式，支持Web跨域调用，也支持程序中直接调用。 
请求方式：POST multipart/form-data 
请求参数说明：

| 字段名      | 变量名      | 是否必填 | 示例值         | 描述                           |
| -------- | -------- | ---- | ----------- | ---------------------------- |
| 文件       | file     | 是    |             | multipart格式文件                |
| 是否首页显示   | show     | 否    | 1           | 默认为是                         |
| 是否设置密码   | ispwd    | 否    | 0           | 默认为否                         |
| 下载密码     | pwd      | 否    | 123456      | 默认留空                         |
| 返回格式     | format   | 否    | json        | 有json、jsonp、form三种选择 默认为json |
| 跳转页面url  | backurl  | 否    | http\://... | 上传成功后的跳转地址 只在form格式有效        |
| callback | callback | 否    | callback    | 只在jsonp格式有效                  |

返回参数说明：

| 字段名   | 变量名     | 类型     | 示例值                              | 描述             |
| ----- | ------- | ------ | -------------------------------- | -------------- |
| 上传状态  | code    | Int    | 0                                | 0为成功，其他为失败     |
| 提示信息  | msg     | String | 上传成功！                            | 如果上传失败会有错误提示   |
| 文件MD5 | hash    | String | f1e807cb0d6ba52d71bdb02864e6bda8 |                |
| 文件名称  | name    | String | exapmle1.jpg                     |                |
| 文件大小  | size    | Int    | 58937                            | 单位：字节          |
| 文件格式  | type    | String | jpg                              |                |
| 下载地址  | downurl | String | http\://.....                    |                |
| 预览地址  | viewurl | String | http\://.....                    | 只有图片、音乐、视频文件才有 |

## 使用教程

调用本API可以使用多种方法，以下是几种常见的实现方式，附带示例代码：

### 方法一：使用cURL（推荐）

cURL是PHP中处理HTTP请求的标准扩展，功能强大且灵活：

```
<?php
// 要上传的本地文件路径
$filePath = '/path/to/local/file.jpg';

// 检查文件是否存在
if (!file_exists($filePath)) {
    die("文件不存在: $filePath");
}

// API请求URL
$apiUrl = 'https://cloud.firgt.cn/api.php';

// 创建cURL句柄
$ch = curl_init();

// 准备表单数据
$postData = [
    'file' => new CURLFile($filePath), // PHP 5.5+ 使用CURLFile类
    'format' => 'json',
    'ispwd' => 1,
    'pwd' => 'SecurePass123'
];

// 设置cURL选项
curl_setopt_array($ch, [
    CURLOPT_URL => $apiUrl,
    CURLOPT_POST => true,
    CURLOPT_POSTFIELDS => $postData,
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HEADER => false,
    CURLOPT_SSL_VERIFYPEER => true, // 验证SSL证书
    CURLOPT_SSL_VERIFYHOST => 2,    // 验证主机名
    CURLOPT_TIMEOUT => 60           // 请求超时时间（秒）
]);

// 执行请求
$response = curl_exec($ch);
$httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);

// 检查是否有错误
if (curl_errno($ch)) {
    die('cURL错误: ' . curl_error($ch));
}

// 关闭cURL句柄
curl_close($ch);

// 处理响应
if ($httpCode === 200) {
    $result = json_decode($response, true);
    if ($result && $result['code'] === 0) {
        echo "文件上传成功！\n";
        echo "下载URL: {$result['downurl']}\n";
        echo "预览URL: {$result['viewurl']}\n";
    } else {
        echo "上传失败: {$result['msg']}\n";
    }
} else {
    echo "HTTP请求失败，状态码: $httpCode\n";
    echo "响应内容: $response\n";
}
?>
```

### 方法二：使用file\_get\_contents（简单场景）

对于简单的上传需求，可以使用`file_get_contents`配合`stream_context_create`：

```
<?php
$filePath = '/path/to/local/file.jpg';
$apiUrl = 'https://cloud.firgt.cn/api.php';

// 创建表单数据
$boundary = uniqid();
$delimiter = '--' . $boundary;

// 构建请求体
$fileData = file_get_contents($filePath);
$fileName = basename($filePath);

$postData = "--$boundary\r\n";
$postData .= "Content-Disposition: form-data; name=\"file\"; filename=\"$fileName\"\r\n";
$postData .= "Content-Type: application/octet-stream\r\n\r\n";
$postData .= $fileData . "\r\n";
$postData .= "--$boundary\r\n";
$postData .= "Content-Disposition: form-data; name=\"format\"\r\n\r\n";
$postData .= "json\r\n";
$postData .= "--$boundary\r\n";
$postData .= "Content-Disposition: form-data; name=\"ispwd\"\r\n\r\n";
$postData .= "1\r\n";
$postData .= "--$boundary\r\n";
$postData .= "Content-Disposition: form-data; name=\"pwd\"\r\n\r\n";
$postData .= "SecurePass123\r\n";
$postData .= "--$boundary--\r\n";

// 设置HTTP上下文选项
$options = [
    'http' => [
        'method' => 'POST',
        'header' => "Content-Type: multipart/form-data; boundary=$boundary\r\n" .
                    "Content-Length: " . strlen($postData) . "\r\n",
        'content' => $postData,
        'timeout' => 60
    ]
];

// 创建上下文
$context = stream_context_create($options);

// 发送请求
$response = file_get_contents($apiUrl, false, $context);

// 处理响应
if ($response) {
    $result = json_decode($response, true);
    if ($result && $result['code'] === 0) {
        echo "上传成功: {$result['msg']}\n";
        echo "下载链接: {$result['downurl']}\n";
    } else {
        echo "上传失败: {$result['msg']}\n";
    }
} else {
    echo "请求失败\n";
}
?>
```

### 方法三：使用HTTP客户端库（如Guzzle）

对于更复杂的场景，推荐使用第三方库Guzzle：

```
<?php
require 'vendor/autoload.php'; // 引入Guzzle

use GuzzleHttp\Client;
use GuzzleHttp\Psr7\Utils;

$filePath = '/path/to/local/file.jpg';
$apiUrl = 'https://cloud.firgt.cn/api.php';

// 创建HTTP客户端
$client = new Client();

try {
    // 发送请求
    $response = $client->post($apiUrl, [
        'multipart' => [
            [
                'name' => 'file',
                'contents' => fopen($filePath, 'r'),
                'filename' => basename($filePath)
            ],
            [
                'name' => 'format',
                'contents' => 'json'
            ],
            [
                'name' => 'ispwd',
                'contents' => '1'
            ],
            [
                'name' => 'pwd',
                'contents' => 'SecurePass123'
            ]
        ]
    ]);

    // 处理响应
    $result = json_decode($response->getBody(), true);
    if ($result && $result['code'] === 0) {
        echo "上传成功: {$result['msg']}\n";
        echo "下载链接: {$result['downurl']}\n";
    } else {
        echo "上传失败: {$result['msg']}\n";
    }
} catch (\Exception $e) {
    echo "请求异常: " . $e->getMessage() . "\n";
}
?>
```

### 使用注意事项

1. **文件路径**：确保PHP进程有读取本地文件的权限。
2. **超时设置**：大文件上传时，建议增加超时时间（如`CURLOPT_TIMEOUT`）。
3. **错误处理**：生产环境中应完善错误日志记录，例如：
   ```
   // 记录错误日志
   error_log("API请求失败: " . json_encode($result));
   ```
4. **SSL验证**：生产环境务必保持`CURLOPT_SSL_VERIFYPEER`为`true`，避免中间人攻击。
5. **依赖安装**：使用Guzzle时，需先通过Composer安装：
   ```
   composer require guzzlehttp/guzzle
   ```

### 典型应用场景

1. **批量文件上传**：遍历目录中的文件，逐个调用API上传。
2. **定时同步**：通过定时任务，将服务器本地文件同步到文件存储系统。
3. **CMS集成**：在内容管理系统中，通过后台上传文件到指定存储。

根据您的具体需求，选择合适的方法实现即可。cURL是最通用的方案，Guzzle则提供了更友好的面向对象接口。

### 使用API进行文件删除

我在原有的带密码功能基础上进行了改进，增加了删除文件时的必要验证：

* DELETE请求参数：
* `file_pwd`: 文件上传时设置的密码（删除时必选，需要文件设置了密码）

使用方法：

* 如果系统没有设置全局删除密码，只需提供文件密码：

```
curl -X DELETE \
  -d "hash=3a78c73f88489c8f33c0e430683b5d5&file_pwd=123456" \
  https://cloud.firgt.cn/api.php
```

响应示例：

```
{
  "code": 0,
  "msg": "文件删除成功"
}
```

错误响应示例：

```
{
  "code": -1,
  "msg": "文件密码错误"
}
```

这样就实现了对设置了密码的文件进行输入密码删除的功能，同时保留了系统管理员删除验证。 如果您的文件未设密码，又希望删除的话，可以在网盘管理界面中删除
