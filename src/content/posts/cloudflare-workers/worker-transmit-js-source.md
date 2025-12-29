---
title: Workers流式传输
published: 2025-12-21
description: '使用Cloudflare Workers从R2流式传输任意大小文件'
image: ''
tags: ["Cloudflare", "Cloudflare Workers"]
category: 'Workers'
draft: false 
lang: 'zh_CN'
---

记录一下自己部署在workers上从R2获取更新文件的服务，使用流式传输避免请求体过大的问题

需要在worker内添加对R2的binding，`NAME`需设定为`STARTLIVE_BUCKET`以直接从env内拿到文件对象

```js title="worker.js"
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    // 假设你的 R2 桶中有一个 /files/yourfile.zip
    const key = url.pathname.substring(1); // 获取文件名

    const file = await env.STARTLIVE_BUCKET.get(key);
    if (file) {
      const headers = new Headers();
      // 设置文件名，让浏览器触发下载
      headers.set('Content-Disposition', `attachment; filename="${key.split('/').pop()}"`);
      // 设置文件类型
      headers.set('Content-Type', 'application/octet-stream');

      return new Response(file.body, {
        headers,
        // 确保大文件可以流式传输
        // body: file.body, // 已经是流式了
        // headers: headers
      });
    } else {
      return new Response("File not found", { status: 404 });
    }
  },
};
```

