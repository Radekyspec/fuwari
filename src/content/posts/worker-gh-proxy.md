---
title: Workers代理GitHub API
published: 2025-12-21
description: '使用Cloudflare Workers代理GitHub API，并添加token增加速率限制'
image: ''
tags: ["Cloudflare", "Cloudflare Workers", "GitHub", "GitHub API"]
category: 'Workers'
draft: false 
lang: 'zh_CN'
---

## 前言

改编自[Cloudflare Workers Proxy](https://github.com/ymyuuu/Cloudflare-Workers-Proxy)

::github{repo="ymyuuu/Cloudflare-Workers-Proxy"}

在使用本脚本之前，请务必详细阅读以下安全注意事项和免责声明，以确保使用脚本时的安全和法律合规性。

## 注意事项

- 请确保部署的 Workers 脚本在部署时是有效的，并且有足够的资源来处理请求。

- 请注意不要滥用该服务，确保只将它用于合法和合适的用途。

## 免责声明

- **责任限制**：作者不对脚本可能导致的任何安全问题、数据损失、服务中断、法律纠纷或其他损害负责。使用此脚本需自行承担风险。

- **不当使用**：使用者需了解，本脚本可能被用于非法活动或未经授权的访问。作者强烈反对和谴责任何不当使用脚本的行为，并鼓励合法合规的使用。

- **合法性**：请确保遵守所有适用的法律、法规和政策，包括但不限于互联网使用政策、隐私法规和知识产权法。确保您拥有对目标地址的合法权限。

- **自担风险**：使用此脚本需自行承担风险。作者和 Cloudflare 不对脚本的滥用、不当使用或导致的任何损害承担责任。

**此免责声明针对非中国大陆地区用户，如在中国大陆地区使用，需遵守相关地区法律法规，且由使用者自行承担相应风险与责任。**

## 代码实现

需要在worker内添加Secret，`Name`需设定为`GITHUB_TOKEN`，在GitHub内设置Token的访问权限为当前仓库只读

添加了白名单域名防止被滥用代理任意网站

```js title="worker.js"
const ALLOW_HOSTS = new Set([
    // 'github.com',
    // 'github.githubassets.com',
    // 'avatars.githubusercontent.com',
    'raw.githubusercontent.com',
    // 'user-images.githubusercontent.com',
    // 'camo.githubusercontent.com',
    // 'gist.githubusercontent.com',
    'api.github.com',
]);

export default {
    async fetch(request, env, ctx) {
      return await handleRequest(request, env);
    },
  };

async function handleRequest(request, env) {
    try {
        const url = new URL(request.url);

        // 如果访问根目录，返回HTML
        if (url.pathname === "/") {
            return new Response(getRootHtml(), {
                headers: {
                    'Content-Type': 'text/html; charset=utf-8'
                }
            });
        }

        // 从请求路径中提取目标 URL
        let actualUrlStr = decodeURIComponent(url.pathname.replace("/", ""));

        // 判断用户输入的 URL 是否带有协议
        actualUrlStr = ensureProtocol(actualUrlStr, url.protocol);

        // 保留查询参数
        actualUrlStr += url.search;

        let target;
        try {
            target = new URL(actualUrlStr);
        } catch (e) {
            return forbidden("Invalid target URL");
        }
        if (!ALLOW_HOSTS.has(target.hostname)) {
            return forbidden(`Forbidden host: ${target.hostname}`);
        }

        // 创建新 Headers 对象，排除以 'cf-' 开头的请求头
        const newHeaders = filterHeaders(request.headers, name => !name.startsWith('cf-'));
        newHeaders.set("Authorization", `Bearer ${env.GITHUB_TOKEN}`);

        // 创建一个新的请求以访问目标 URL
        const modifiedRequest = new Request(actualUrlStr, {
            headers: newHeaders,
            method: request.method,
            body: request.body,
            redirect: 'manual'
        });

        // 发起对目标 URL 的请求
        const response = await fetch(modifiedRequest);
        let body = response.body;

        // 处理重定向
        if ([301, 302, 303, 307, 308].includes(response.status)) {
            body = response.body;
            // 创建新的 Response 对象以修改 Location 头部
            return handleRedirect(response, body);
        } else if (response.headers.get("Content-Type")?.includes("text/html")) {
            body = await handleHtmlContent(response, url.protocol, url.host, actualUrlStr);
        }

        // 创建修改后的响应对象
        const modifiedResponse = new Response(body, {
            status: response.status,
            statusText: response.statusText,
            headers: response.headers
        });

        // 添加禁用缓存的头部
        setNoCacheHeaders(modifiedResponse.headers);

        // 添加 CORS 头部，允许跨域访问
        setCorsHeaders(modifiedResponse.headers);

        return modifiedResponse;
    } catch (error) {
        // 如果请求目标地址时出现错误，返回带有错误消息的响应和状态码 500（服务器错误）
        return jsonResponse({
            error: error.message
        }, 500);
    }
}

function forbidden(message) {
    const r = new Response(message, {
        status: 403,
        headers: { 'Content-Type': 'text/plain; charset=utf-8' }
    });
    setNoCacheHeaders(r.headers);
    setCorsHeaders(r.headers);
    return r;
}

// 确保 URL 带有协议
function ensureProtocol(url, defaultProtocol) {
    return url.startsWith("http://") || url.startsWith("https://") ? url : defaultProtocol + "//" + url;
}

// 处理重定向
function handleRedirect(response, body) {
    const location = new URL(response.headers.get('location'));
    const modifiedLocation = `/${encodeURIComponent(location.toString())}`;
    return new Response(body, {
        status: response.status,
        statusText: response.statusText,
        headers: {
            ...response.headers,
            'Location': modifiedLocation
        }
    });
}

// 处理 HTML 内容中的相对路径
async function handleHtmlContent(response, protocol, host, actualUrlStr) {
    const originalText = await response.text();
    const regex = new RegExp('((href|src|action)=["\'])/(?!/)', 'g');
    let modifiedText = replaceRelativePaths(originalText, protocol, host, new URL(actualUrlStr).origin);

    return modifiedText;
}

// 替换 HTML 内容中的相对路径
function replaceRelativePaths(text, protocol, host, origin) {
    const regex = new RegExp('((href|src|action)=["\'])/(?!/)', 'g');
    return text.replace(regex, `$1${protocol}//${host}/${origin}/`);
}

// 返回 JSON 格式的响应
function jsonResponse(data, status) {
    return new Response(JSON.stringify(data), {
        status: status,
        headers: {
            'Content-Type': 'application/json; charset=utf-8'
        }
    });
}

// 过滤请求头
function filterHeaders(headers, filterFunc) {
    return new Headers([...headers].filter(([name]) => filterFunc(name)));
}

// 设置禁用缓存的头部
function setNoCacheHeaders(headers) {
    headers.set('Cache-Control', 'no-store');
}

// 设置 CORS 头部
function setCorsHeaders(headers) {
    headers.set('Access-Control-Allow-Origin', '*');
    headers.set('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
    headers.set('Access-Control-Allow-Headers', '*');
}

// 返回根目录的 HTML
function getRootHtml() {
    return `<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <link href="https://s4.zstatic.net/ajax/libs/materialize/1.0.0/css/materialize.min.css" rel="stylesheet">
  <title>Proxy Everything</title>
  <link rel="icon" type="image/png" href="https://s2.hdslb.com/bfs/openplatform/1682b11880f5c53171217a03c8adc9f2e2a27fcf.png@100w.webp">
  <meta name="Description" content="Proxy Everything with CF Workers.">
  <meta property="og:description" content="Proxy Everything with CF Workers.">
  <meta property="og:image" content="https://s2.hdslb.com/bfs/openplatform/1682b11880f5c53171217a03c8adc9f2e2a27fcf.png@100w.webp">
  <meta name="robots" content="index, follow">
  <meta http-equiv="Content-Language" content="zh-CN">
  <meta name="copyright" content="Copyright © ymyuuu">
  <meta name="author" content="ymyuuu">
  <link rel="apple-touch-icon-precomposed" sizes="120x120" href="https://s2.hdslb.com/bfs/openplatform/1682b11880f5c53171217a03c8adc9f2e2a27fcf.png@100w.webp">
  <meta name="apple-mobile-web-app-capable" content="yes">
  <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
  <meta name="viewport" content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0, user-scalable=no">
  <style>
      body, html {
          height: 100%;
          margin: 0;
      }
      .background {
          background-size: cover;
          background-position: center;
          height: 100%;
          display: flex;
          align-items: center;
          justify-content: center;
      }
      .card {
          background-color: rgba(255, 255, 255, 0.8);
          transition: background-color 0.3s ease, box-shadow 0.3s ease;
      }
      .card:hover {
          background-color: rgba(255, 255, 255, 1);
          box-shadow: 0px 8px 16px rgba(0, 0, 0, 0.3);
      }
      .input-field input[type=text] {
          color: #2c3e50;
      }
      .input-field input[type=text]:focus+label {
          color: #2c3e50 !important;
      }
      .input-field input[type=text]:focus {
          border-bottom: 1px solid #2c3e50 !important;
          box-shadow: 0 1px 0 0 #2c3e50 !important;
      }
      @media (prefers-color-scheme: dark) {
          body, html {
              background-color: #121212;
              color: #e0e0e0;
          }
          .card {
              background-color: rgba(33, 33, 33, 0.9);
              color: #ffffff;
          }
          .card:hover {
              background-color: rgba(50, 50, 50, 1);
              box-shadow: 0px 8px 16px rgba(0, 0, 0, 0.6);
          }
          .input-field input[type=text] {
              color: #ffffff;
          }
          .input-field input[type=text]:focus+label {
              color: #ffffff !important;
          }
          .input-field input[type=text]:focus {
              border-bottom: 1px solid #ffffff !important;
              box-shadow: 0 1px 0 0 #ffffff !important;
          }
          label {
              color: #cccccc;
          }
      }
  </style>
</head>
<body>
  <div class="background">
      <div class="container">
          <div class="row">
              <div class="col s12 m8 offset-m2 l6 offset-l3">
                  <div class="card">
                      <div class="card-content">
                          <span class="card-title center-align"><i class="material-icons left">link</i>Proxy Everything</span>
                          <form id="urlForm" onsubmit="redirectToProxy(event)">
                              <div class="input-field">
                                  <input type="text" id="targetUrl" placeholder="在此输入目标地址" required>
                                  <label for="targetUrl">目标地址</label>
                              </div>
                              <button type="submit" class="btn waves-effect waves-light teal darken-2 full-width">跳转</button>
                          </form>
                      </div>
                  </div>
              </div>
          </div>
      </div>
  </div>
  <script src="https://s4.zstatic.net/ajax/libs/materialize/1.0.0/js/materialize.min.js"></script>
  <script>
      function redirectToProxy(event) {
          event.preventDefault();
          const targetUrl = document.getElementById('targetUrl').value.trim();
          const currentOrigin = window.location.origin;
          window.open(currentOrigin + '/' + encodeURIComponent(targetUrl), '_blank');
      }
  </script>
</body>
</html>`;
}
```

