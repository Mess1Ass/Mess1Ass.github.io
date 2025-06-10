---
title: 丝芭商城切票
date: 2025-03-10 14:31:23
abstracts: 切票
tags: [python,爬虫,playwright]
cover: ./images/ticketPlayWright.jpg
---

:::warning
本系统仅适用于丝芭商城，未测试，不保证可行性，重点介绍playwright。
本系统爬虫仅用于学习交流，请勿用于非法用途。
:::

# 丝芭商城门票抢购

## 项目简介  
本项目旨在使用playwright技术 **自动化购买丝芭公演门票** 。  

## 功能介绍  
- **自动购买门票**：调用购买api接口，实现自动化购买。  
- **playwright 支持**：相比于selenium，playwright 更加适合自动化测试，速度更快，更稳定。  
- **多线程优化**：提升请求效率，减少等待时间。  
- **防反爬策略**：加入请求间隔控制，降低被封风险。 

## 代码主要实现部分  

### 🎯 **1.playwright安装**

##### 📜 **代码实现** 

```cmd
# 安装playwright
pip install playwright

# 安装playwright所需的浏览器插件
playwright install
```

### 🎯 **2.playwright启动浏览器**

##### 📜 **代码实现**  

```python
async with async_playwright() as p:
    # 以无头模式运行 可更换启动的浏览器
    browser = await p.chromium.launch(executable_path=".\ms-playwright\chromium-1169\chrome-win\chrome.exe", headless=False)
    context = await browser.new_context()
    page = await context.new_page()
    try:
        # ✅ 用于 API 请求的独立 context
        api_context = await p.request.new_context()
        ###  页面操作代码

        ###  页面操作代码

        await asyncio.sleep(60)  # 保持页面一段时间，便于观察
        await browser.close()


    except Exception as e:
        print("异常或浏览器关闭:", e)
    finally:
        await browser.close()
```

### 🎯 **3.playwright调用购票接口**

#### 📌 **函数：`make_request_with_retries`**
此函数用于调用购票api，直到成功/没有库存，并将返回的信息保存成日志，并返回日志信息。  

##### 📌 **函数参数**  
- `context: APIRequestContext`：必须为APIRequestContext类型，其他类型没有get/post方法。  
- `ticket_Add_url`：购票url。  
- `ticket_Add_params`：接口传参。  
- `ticket_Add_headers`：请求头。

##### 📜 **代码实现**  

```python
async with async_playwright(context: APIRequestContext, ticket_Add_url, ticket_Add_params, ticket_Add_headers) as p:
    retries = 0
    backoff = 0.2
    while retries < max_retries:
        try:
            response = await context.post(ticket_Add_url, params=ticket_Add_params, headers=ticket_Add_headers)
            current_time = datetime.now()
            if response.ok:
                data = await response.json()
                if 'success' in data.get("Message", ""):
                    log_entry = f"数据={data}"
                    log_data.append(log_entry)
                    return log_data
                else:
                    log_entry = f"数据={data}"
                    log_data.append(log_entry)
                    retries += 1
                    delay = backoff * (2 ** retries) + random.random()*0.1
                    await asyncio.sleep(delay)
            else:
                error_entry = f"状态码: {response.status}"
                log_data.append(error_entry)
                retries += 1
                delay = backoff * (2 ** retries) + random.random()*0.1
                await asyncio.sleep(delay)
        except Exception as e:
            error_entry = f"{e}"
            log_data.append(error_entry)
            retries += 1
            delay = backoff * (2 ** retries) + random.random()*0.1
            await asyncio.sleep(delay)
    return log_data
```