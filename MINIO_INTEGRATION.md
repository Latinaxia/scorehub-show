# MinIO集成文档

## 1. 概述

本文档说明了如何将ScoreHub应用程序的文件存储系统从本地存储迁移到MinIO对象存储服务。

## 2. 问题解答

### Q: 图片上传到MinIO后，如何与用户或内容关联？

A: MinIO只负责存储文件本身，文件与用户/内容的关联关系仍然存储在数据库中：
- 用户表的`avatar`字段存储MinIO中的文件路径（如：`avatars/1.png`）
- 内容表的`images`字段存储MinIO中的文件路径，多个路径用逗号分隔

### Q: 数据库中应该存储什么？

A: 存储MinIO中的文件路径（Object Key），而不是URL，因为：
- URL可能会变化（域名、端口变化）
- 可以通过配置动态生成URL
- 路径更短，节省存储空间

访问时再将路径转换为完整的URL（如：`http://localhost:9000/scorehub/avatars/1.png`）

## 3. 后端配置

### 3.1 Maven依赖

在`pom.xml`中添加MinIO SDK依赖：

```xml
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>8.5.7</version>
</dependency>
```

### 3.2 配置文件

在`application.properties`中添加MinIO配置：

```properties
# MinIO配置
minio.endpoint=http://localhost:9000
minio.accessKey=admin
minio.secretKey=12345678
minio.bucketName=scorehub
```

### 3.3 配置类

创建`MinioConfig.java`配置类，用于初始化MinIO客户端：

```java
@Configuration
public class MinioConfig {
    @Value("${minio.endpoint}")
    private String endpoint;
    
    @Value("${minio.accessKey}")
    private String accessKey;
    
    @Value("${minio.secretKey}")
    private String secretKey;
    
    @Bean
    public MinioClient minioClient() {
        return MinioClient.builder()
                .endpoint(endpoint)
                .credentials(accessKey, secretKey)
                .build();
    }
    
    @Bean
    public String minioBucketName() {
        return "scorehub";
    }
}
```

### 3.4 MinIO工具类

创建`MinioUtil.java`，提供文件上传、删除、URL获取等功能：

```java
@Component
public class MinioUtil {
    
    @Autowired
    private MinioClient minioClient;
    
    @Autowired
    private String minioBucketName;
    
    @Autowired
    private String minioEndpoint;
    
    // 上传文件
    public String uploadFile(MultipartFile file, String prefix);
    
    // 上传用户头像
    public String uploadAvatar(MultipartFile file, Long userId);
    
    // 上传内容图片
    public List<String> uploadContentImages(MultipartFile[] files, Long contentId);
    
    // 删除文件
    public void deleteFile(String objectName);
    
    // 获取文件访问URL
    public String getFileUrl(String objectName);
}
```

## 4. 文件上传API

### 4.1 上传用户头像

- **URL**: `/api/upload/avatar`
- **方法**: `POST`
- **参数**:
  - `file`: 文件对象
  - `userId`: 用户ID
- **返回**: `{ url: "...", path: "..." }`

### 4.2 上传内容图片

- **URL**: `/api/upload/images`
- **方法**: `POST`
- **参数**:
  - `files`: 文件数组
  - `contentId`: 内容ID
- **返回**: `{ urls: [...], paths: [...] }`

### 4.3 发布内容更新

- **URL**: `/api/content/publish`
- **方法**: `POST`
- **返回**: 现在会返回内容ID `{ id: 123 }`，方便后续上传图片

## 5. 文件路径结构

在MinIO中，文件按以下结构组织：

```
scorehub/
├── avatars/
│   ├── 1.png
│   └── 2.png
└── contents/
    ├── 1/
    │   ├── 1.png
    │   └── 2.png
    └── 2/
        └── 1.png
```

- `avatars/{userId}.png`: 用户头像，使用用户ID命名
- `contents/{contentId}/{index}.png`: 内容图片，使用内容ID+索引命名

## 6. 前端集成

### 6.1 用户头像上传

在`ProfileView.vue`中：

1. 移除base64编码逻辑
2. 使用FormData上传文件到`/api/upload/avatar`
3. 将返回的路径保存到用户信息中

```javascript
const handleAvatarUpload = async (event) => {
    const file = event.target.files[0];
    if (!file) return;
    
    if (!userStore.user) {
        alert('请先登录');
        return;
    }
    
    const formData = new FormData();
    formData.append('file', file);
    formData.append('userId', userStore.user.id);
    
    const response = await fetch('/api/upload/avatar', {
        method: 'POST',
        body: formData
    });
    
    const result = await response.json();
    if (result.code === 200) {
        user.value.avatar = result.data.path;
        userStore.user.avatar = result.data.path;
        alert('头像更新成功');
    }
};
```

### 6.2 内容图片上传

在`PublishView.vue`中：

1. 添加图片预览功能
2. 先发布内容获取ID，再上传图片
3. 将图片路径插入到内容中

```javascript
const publishAndUploadImages = async () => {
    // 1. 发布内容
    const publishResult = await publishContent();
    const contentId = publishResult.data.id;
    
    // 2. 上传图片
    if (form.value.images.length > 0) {
        await uploadImages(contentId);
    }
    
    alert('发布成功！');
    router.push('/');
};
```

## 7. MinIO管理

### 7.1 启动MinIO

确保本地已安装并启动MinIO：

```bash
# 使用Docker启动MinIO
docker run -p 9000:9000 -p 9001:9001 \
  quay.io/minio/minio server /data --console-address ":9001"
```

### 7.2 创建Bucket

1. 访问 http://localhost:9001
2. 使用admin/12345678登录
3. 创建名为`scorehub`的Bucket
4. 或者直接运行应用，应用会自动创建Bucket

## 8. 注意事项

1. **权限设置**: 确保MinIO的bucket有public权限，否则无法直接通过URL访问
2. **文件大小限制**: 配置文件中设置了10MB的单个文件限制
3. **图片格式**: 只支持常见的图片格式
4. **安全**: 在生产环境中，应使用预签名URL而不是直接公开bucket

## 9. 数据库字段更新

无需数据库结构变更，现有的字段已足够使用：

- `user.avatar`: VARCHAR(255) - 存储头像路径
- `content.images`: VARCHAR(500) - 存储图片路径，多个用逗号分隔

## 10. 访问图片URL

图片访问URL格式：
`{minio.endpoint}/{bucket}/{path}`

例如：
`http://localhost:9000/scorehub/avatars/1.png`
