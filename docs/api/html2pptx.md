# HTML to PPTX 转换服务

## 🚀 Kubernetes 部署

### 1. 构建并推送 Docker 镜像

```bash
# 在项目根目录构建镜像
docker build -t html2pptx-api:latest -f api/Dockerfile .

# 标记并推送到私有仓库
docker tag html2pptx-api:latest your-registry.com/html2pptx-api:latest
docker push your-registry.com/html2pptx-api:latest
```

### 2. K8s 部署配置

#### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: html2pptx-api
  labels:
    app: html2pptx-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: html2pptx-api
  template:
    metadata:
      labels:
        app: html2pptx-api
    spec:
      containers:
      - name: html2pptx-api
        image: your-registry.com/html2pptx-api:latest
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "3000"
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "2000m"
            memory: "2Gi"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 40
          periodSeconds: 30
          timeoutSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
```

#### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: html2pptx-api
spec:
  selector:
    app: html2pptx-api
  ports:
  - port: 80
    targetPort: 3000
  type: ClusterIP
```

#### Ingress（可选）

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: html2pptx-api
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
spec:
  rules:
  - host: html2pptx.your-domain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: html2pptx-api
            port:
              number: 80
```

### 3. 部署命令

```bash
# 应用配置
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml

# 查看状态
kubectl get pods -l app=html2pptx-api
kubectl get svc html2pptx-api

# 查看日志
kubectl logs -l app=html2pptx-api -f
```

---

## 📡 API 调用方法

### Python 调用示例

#### 1. 健康检查

```python
import requests

API_BASE_URL = "http://html2pptx-api.your-domain.com"

def health_check() -> bool:
    """检查 API 服务状态"""
    try:
        response = requests.get(f"{API_BASE_URL}/health", timeout=5)
        response.raise_for_status()
        return True
    except:
        return False
```

#### 2. HTML 转换为 PDF

```python
import requests
import json
import base64

def convert_to_pdf(pages: list, options: dict = None) -> bytes:
    """
    转换 HTML 列表为 PDF
    
    Args:
        pages: HTML 页面列表，格式为 [{"title": "页面标题", "html": "<html>...</html>"}, ...]
        options: 可选配置
        
    Returns:
        PDF 文件二进制数据
    """
    if options is None:
        options = {}
    
    payload = {
        "pages": pages,
        "options": {
            "filename": options.get("filename", "output.pdf"),
            "format": options.get("format", {"width": "10in", "height": "5.625in"}),
            "landscape": options.get("landscape", False),
            "margin": options.get("margin", {"top": "0mm", "right": "0mm", "bottom": "0mm", "left": "0mm"}),
            "printBackground": options.get("printBackground", True),
            "lenient": options.get("lenient", True)
        }
    }
    
    response = requests.post(
        f"{API_BASE_URL}/convert/pdf",
        json=payload,
        headers={"Content-Type": "application/json"},
        timeout=300
    )
    
    response.raise_for_status()
    
    # 解析转换结果（可选）
    results_header = response.headers.get("X-Conversion-Results")
    if results_header:
        decoded = base64.b64decode(results_header).decode('utf-8')
        results = json.loads(decoded)
        success_count = sum(1 for r in results if r["status"] == "success")
        print(f"成功转换: {success_count}/{len(results)} 页")
    
    return response.content


# 使用示例
pages = [
    {"title": "封面", "html": "<html><body style='width:720pt;height:405pt;'>...</body></html>"},
    {"title": "内容1", "html": "<html><body style='width:720pt;height:405pt;'>...</body></html>"}
]

pdf_data = convert_to_pdf(pages, {"filename": "presentation.pdf"})
with open("output.pdf", "wb") as f:
    f.write(pdf_data)
```

#### 3. Markdown 转换为 PDF

```python
import requests

def convert_markdown_to_pdf(markdown: str, options: dict = None) -> bytes:
    """
    转换 Markdown 文本为 PDF
    
    Args:
        markdown: Markdown 文本内容
        options: 可选配置
        
    Returns:
        PDF 文件二进制数据
    """
    if options is None:
        options = {}
    
    payload = {
        "markdown": markdown,
        "options": {
            "filename": options.get("filename", "output.pdf"),
            "format": options.get("format", "A4"),
            "landscape": options.get("landscape", False),
            "margin": options.get("margin", {
                "top": "20mm",
                "right": "20mm",
                "bottom": "20mm",
                "left": "20mm"
            }),
            "printBackground": options.get("printBackground", True),
            "theme": options.get("theme", "default"),  # 可选: default, github, elegant
            "customCss": options.get("customCss", "")
        }
    }
    
    response = requests.post(
        f"{API_BASE_URL}/convert/markdown-pdf",
        json=payload,
        headers={"Content-Type": "application/json"},
        timeout=300
    )
    
    response.raise_for_status()
    
    # 解析响应头信息
    total_pages = response.headers.get("X-Total-Pages", "?")
    markdown_length = response.headers.get("X-Markdown-Length", "?")
    print(f"PDF 总页数: {total_pages}, Markdown 长度: {markdown_length} 字符")
    
    return response.content


# 使用示例
markdown = """
# 项目文档

## 简介

这是一个使用 **Markdown** 编写的项目文档。

## 功能特点

- 支持标准 Markdown 语法
- 支持代码高亮
- 支持表格和引用

## 代码示例

```python
def hello():
    print('Hello, World!')
```

## 表格

| 名称 | 类型   | 描述       |
| ---- | ------ | ---------- |
| id   | number | 唯一标识符 |
| name | string | 用户名称   |

> 这是一段引用文本。
> """

pdf_data = convert_markdown_to_pdf(markdown, {
    "filename": "document.pdf",
    "theme": "github"  # 使用 GitHub 风格主题
})

with open("output.pdf", "wb") as f:
    f.write(pdf_data)
```

**支持的主题**:
- `default` - 默认主题
- `github` - GitHub 风格
- `elegant` - 优雅主题（衬线字体）

**自定义样式**:
```python
custom_css = """
h1 {
    color: #2c3e50;
    border-bottom: 2px solid #3498db;
}
"""
pdf_data = convert_markdown_to_pdf(markdown, {
    "theme": "default",
    "customCss": custom_css
})
```

#### 4. HTML 转换为 PPTX

```python
import requests
import json
import base64

def convert_to_pptx(slides: list, options: dict = None) -> bytes:
    """
    转换 HTML 列表为 PPTX
    
    Args:
        slides: HTML 幻灯片列表，格式为 [{"title": "标题", "html": "<html>...</html>"}, ...]
        options: 可选配置
        
    Returns:
        PPTX 文件二进制数据
    """
    if options is None:
        options = {}
    
    payload = {
        "slides": slides,
        "options": {
            "title": options.get("title", "Presentation"),
            "author": options.get("author", "html2pptx"),
            "filename": options.get("filename", "output.pptx"),
            "lenient": options.get("lenient", True)
        }
    }
    
    response = requests.post(
        f"{API_BASE_URL}/convert",
        json=payload,
        headers={"Content-Type": "application/json"},
        timeout=300
    )
    
    response.raise_for_status()
    
    # 解析转换结果（可选）
    results_header = response.headers.get("X-Conversion-Results")
    if results_header:
        decoded = base64.b64decode(results_header).decode('utf-8')
        results = json.loads(decoded)
        success_count = sum(1 for r in results if r["status"] == "success")
        print(f"成功转换: {success_count}/{len(results)} 页")
    
    return response.content


# 使用示例
slides = [
    {"title": "封面", "html": "<html><body style='width:720pt;height:405pt;'>...</body></html>"},
    {"title": "内容1", "html": "<html><body style='width:720pt;height:405pt;'>...</body></html>"}
]

pptx_data = convert_to_pptx(slides, {
    "title": "我的演示文稿",
    "author": "作者名",
    "filename": "presentation.pptx"
})

with open("output.pptx", "wb") as f:
    f.write(pptx_data)
```

#### 5. 从目录批量转换 HTML 文件

```python
import re
from pathlib import Path

def load_html_files(directory: str, pattern: str = None) -> list:
    """
    从目录加载 HTML 文件
    
    Args:
        directory: HTML 文件所在目录
        pattern: 可选的文件名筛选模式
        
    Returns:
        按幻灯片编号排序的 HTML 文件列表
    """
    dir_path = Path(directory)
    if not dir_path.exists():
        raise FileNotFoundError(f"目录不存在: {directory}")
    
    html_files = list(dir_path.glob("*.html"))
    
    if pattern:
        html_files = [f for f in html_files if pattern in f.name]
    
    # 排除 collection 文件
    html_files = [f for f in html_files if "collection" not in f.name.lower()]
    
    # 按文件名中的数字排序
    def get_slide_number(filepath):
        match = re.search(r'slide_(\d+)_', filepath.name)
        return int(match.group(1)) if match else 999
    
    html_files.sort(key=get_slide_number)
    return html_files


def read_html_files(files: list) -> list:
    """读取 HTML 文件内容"""
    pages = []
    for filepath in files:
        name = filepath.stem
        match = re.search(r'slide_\d+_(.+?)_\d{8}_\d{6}', name)
        title = match.group(1) if match else name
        
        html_content = filepath.read_text(encoding='utf-8')
        pages.append({"title": title, "html": html_content})
    
    return pages


# 使用示例
html_files = load_html_files("./slides", pattern="20251215")
pages = read_html_files(html_files)

pdf_data = convert_to_pdf(pages, {"filename": "presentation.pdf"})
pptx_data = convert_to_pptx(pages, {"title": "演示文稿", "filename": "presentation.pptx"})
```

### cURL 调用示例

```bash
# 健康检查
curl http://html2pptx-api.your-domain.com/health

# 转换为 PPTX
curl -X POST http://html2pptx-api.your-domain.com/convert \
  -H "Content-Type: application/json" \
  -d '{
    "slides": [{"title": "Slide 1", "html": "<html><body style=\"width:720pt;height:405pt;\"><h1>Hello</h1></body></html>"}],
    "options": {"title": "My Presentation", "lenient": true}
  }' \
  --output output.pptx

# 转换为 PDF
curl -X POST http://html2pptx-api.your-domain.com/convert/pdf \
  -H "Content-Type: application/json" \
  -d '{
    "pages": [{"title": "Page 1", "html": "<html><body style=\"width:720pt;height:405pt;\"><h1>Hello</h1></body></html>"}],
    "options": {"filename": "output.pdf", "printBackground": true}
  }' \
  --output output.pdf

# Markdown 转换为 PDF
curl -X POST http://html2pptx-api.your-domain.com/convert/markdown-pdf \
  -H "Content-Type: application/json" \
  -d '{
    "markdown": "# Hello World\n\nThis is a **Markdown** document.\n\n## Features\n\n- Item 1\n- Item 2",
    "options": {"filename": "output.pdf", "theme": "github", "format": "A4"}
  }' \
  --output output.pdf
```

---

## 📋 API 端点

| 端点                    | 方法 | 说明                |
| ----------------------- | ---- | ------------------- |
| `/health`               | GET  | 健康检查            |
| `/convert`              | POST | HTML → PPTX 转换    |
| `/convert/pdf`          | POST | HTML → PDF 转换     |
| `/convert/markdown-pdf` | POST | Markdown → PDF 转换 |
| `/convert/files`        | POST | 文件上传转换        |

## ⚠️ HTML 规范要求

所有 HTML 幻灯片必须遵循 16:9 尺寸规范：

```css
body {
  width: 720pt;
  height: 405pt;
  margin: 0;
  padding: 0;
}
```