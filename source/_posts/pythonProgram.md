---
title: 丝芭商城竞价数据统计
date: 2025-03-10 14:31:23
abstracts: 收集丝芭特殊场竞价公演以及竞价拍立得数据
tags: [python,爬虫]
cover: ./images/resume_icon.jpg
---

:::warning
本系统仅适用于丝芭商城，其他商城请自行修改代码。
本系统爬虫仅用于学习交流，请勿用于非法用途。
:::

# 丝芭商城数据导出工具  

## 项目简介  
本项目是一个用于 **自动化抓取和导出丝芭商城（https://shop.48.cn）拍卖数据** 的 Python 工具。  
用户可以获取 **指定拍卖页面** 的信息，并将其整理后导出为 **Excel 文件**。  

## 功能介绍  
- **自动获取拍卖数据**：解析商品详情页，提取关键信息，如竞拍价格、出价记录等。  
- **数据存储**：将抓取的数据存入 Excel，方便后续分析。  
- **Tampermonkey 脚本支持**：支持在浏览器中使用 Tampermonkey 直接获取数据。  
- **多线程优化**：提升请求效率，减少等待时间。  
- **防反爬策略**：加入请求间隔控制，降低被封风险。  

## 代码主要实现部分  

### 🎯 **1.获取竞价成功与失败数据**

#### 📌 **函数：`auto_bid_until_end`**
此函数用于自动执行竞价过程，直到达到指定的成功竞价数量，并收集所有竞价数据，包括成功和失败的竞价信息。  

##### 📌 **函数参数**  
- `driver`：WebDriver 实例，用于与网页交互。  
- `target_successful_count`：目标成功竞价的数量。  
- `bid_type`：竞价类型（座位类别）。  
- `theater_name`：剧院名称。  

##### 📜 **代码实现**  

```python
def auto_bid_until_end(driver, target_successful_count, bid_type, theater_name):
    successful_bids_data = []  # 存储所有成功竞价信息
    unsuccessful_bids_data = []  # 存储所有未成功竞价信息
    total_bids_data = []
    wait = WebDriverWait(driver, 10)

    # 获取最大页数
    max_page_element = driver.find_element(By.XPATH, '/html/body/div[2]/div/div[3]/div[3]/div[2]/div[2]/div[4]/span[3]')
    max_page = int(max_page_element.text)
    now_page = 1

    # 📌 **获取成功竞价数据**
    while now_page <= max_page:
        u_blist = driver.find_element(By.ID, "u_blist")
        successful_bids_data.extend(parse_successful_bids(u_blist))

        if len(successful_bids_data) >= target_successful_count:
            print("✅ 已获取所有成功竞价信息")
            break

        # 检查 `u_blistM` 是否存在额外的成功竞价信息
        u_blistM = driver.find_element(By.ID, "u_blistM")
        successful_bids_data.extend(parse_successful_bids(u_blistM))

        if len(successful_bids_data) >= target_successful_count:
            print("✅ 已获取所有成功竞价信息")
            break

        # 📌 **翻页**
        next_button = wait.until(EC.element_to_be_clickable((By.XPATH, '//*[@id="a_b_n"]')))
        next_button.click()
        print(f"🔄 加载第 {now_page} 页...")
        now_page += 1
        time.sleep(0.15)

    # 📌 **获取未成功竞价数据**
    while now_page <= max_page:
        u_blist = driver.find_element(By.ID, "u_blist")
        unsuccessful_bids_data.extend(parse_unsuccessful_bids(u_blist, successful_bids_data))

        # 检查 `u_blistM` 是否存在额外的未成功竞价信息
        u_blistM = driver.find_element(By.ID, "u_blistM")
        unsuccessful_bids_data.extend(parse_unsuccessful_bids(u_blistM, successful_bids_data))

        # 📌 **翻页**
        next_button = wait.until(EC.element_to_be_clickable((By.XPATH, '//*[@id="a_b_n"]')))
        next_button.click()
        print(f"🔄 加载第 {now_page} 页...")
        now_page += 1
        time.sleep(0.15)

    # **去重处理**
    unsuccessful_bids_data = deduplication(unsuccessful_bids_data)
    total_bids_data = successful_bids_data + unsuccessful_bids_data

    # 📌 **获取座位信息**
    seats = get_seat_positon(theater_name, bid_type, target_successful_count)

    # **分配座位号**
    for idx, bid in enumerate(total_bids_data):
        bid["座位类型"] = bid_type
        if idx > len(seats) - 1:
            bid["座位号"] = "竞价失败"
        else:
            bid["座位号"] = seats[idx]  # 添加座位号

    return total_bids_data
```

### 🎯 **2.统计当前页面竞价数据前的准备活动**

#### 📌 **函数：`stats_one_good`**
此函数用于获取丝芭商城的单页面下的竞价数据，并将其保存到 Excel 文件中。

##### 📜 **代码实现**
```python
def stats_one_good(driver):
    # 📌 获取剧院名称和 Excel 文件名
    theater_name = driver.find_element(By.XPATH, "/html/body/div[2]/div/div[2]/div[2]/ul/li[2]/p").text
    excel_name = driver.find_element(By.XPATH, "/html/body/div[2]/div/div[2]/div[2]/ul/li[1]").text

    # 📌 获取商品标题
    title_name_element = driver.find_element(By.CLASS_NAME, "i_tit")
    title_name = title_name_element.text.strip()  # 获取文本内容并去除首尾空格

    # 📌 获取商品详细信息并判断是否为生日公演
    item_info = driver.find_element(By.XPATH, '//*[@id="TabTab03Con1"]').get_attribute('outerHTML')
    soup = BeautifulSoup(item_info, 'html.parser')
    item_info_text = soup.get_text()
    birthday = "生日潮流包" in item_info_text

    # 📌 确定竞价类型
    bid_type = get_seat_type(title_name)

    # 📌 根据剧场和商品类型选择相应的竞价数量
    if "SNH" in theater_name and "星梦剧院" in title_name and "MINILIVE" not in title_name:
        bid_number = get_bid_number_SNH(bid_type, driver)
        if birthday:
            theater_name = "SNHbirthday"

    elif "SNH" in theater_name and "星梦空间" in title_name and "MINILIVE" not in title_name:
        bid_number = get_bid_number_HGH(bid_type)
        theater_name = "HGH"

    elif "BEJ" in theater_name and "生日会" not in title_name:
        bid_number = get_bid_number_BEJ(driver)

    elif "MINILIVE" in title_name:
        bid_number = get_bid_number_MiniLive(driver)
        theater_name = "MINILIVE"

    elif "拍立得" in title_name:
        bid_number = get_bid_number_pld(driver)
        theater_name = "拍立得"

    elif "生日会" in title_name:
        bid_number = get_bid_number_birthparty(driver, theater_name)
        theater_name = "生日会"

    print(f"🎫 竞价席位总数：{bid_number}")

    # 📌 竞价流程
    if bid_number != 0:
        max_bid_num = bid_number
        total_bids_data = auto_bid_until_end(driver, max_bid_num, bid_type, theater_name)

        # 📌 获取商品 ID 并保存 Excel 数据
        item_id = get_item_name(driver)
        save_excel(total_bids_data, item_id, excel_name)

    # 📌 刷新页面，准备下一次操作
    driver.refresh()
```

### 📊 **3.保存竞价数据到 Excel**

#### 📌 **函数：`save_excel`**
此函数用于将竞价成功的数据保存到 Excel 文件，支持写入商品名称、表头、竞价信息，并更新统计信息（最小/最大出价、最早/最晚出价等）。

##### 📜 **代码实现**
```python
def save_excel(successful_bids_data, item_name, output_file="bidding_results.xlsx"):
    """
    将竞价成功数据保存到 Excel 文件。

    :param successful_bids_data: 竞价成功的数据列表
    :param item_name: 商品名称
    :param output_file: 输出文件名（默认："bidding_results.xlsx"）
    """

    # 📌 将数据转换为 Pandas DataFrame
    df = pd.DataFrame(successful_bids_data)
    
    # 📌 创建 Excel 工作簿和工作表
    wb = Workbook()
    ws = wb.active
    ws.title = "Bidding Results"
    
    # 📌 在 Excel 文件的第一行写入商品名称
    ws.append([item_name])
    
    # 📌 写入表头并加粗
    header = ["出价状态", "出价人", "出价时间", "出价金额", "座位类型", "座位号"]
    ws.append(header)
    
    for cell in ws[2]:  # 第二行是标题行
        cell.font = Font(bold=True)
    
    # 📌 将竞价数据写入 Excel
    for row in dataframe_to_rows(df, index=False, header=False):
        ws.append(row)
    
    # 📌 更新最小/最大出价、最早/最晚出价的出价人信息
    ws = update_min_max_info(df, ws)

    # 📌 保存 Excel 文件
    wb.save(output_file + ".xlsx")
    print(f"✅ 竞价成功信息已保存至 {output_file}.xlsx")
```



