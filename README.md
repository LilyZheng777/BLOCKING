import time
import random
import gspread
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from webdriver_manager.chrome import ChromeDriverManager
import sys

# ================== 配置区 ==================
SPREADSHEET_URL = "https://docs.google.com/spreadsheets/d/1-PkASmVbIGicDzOc-VpRMs7vOzggMljvDOf7YwrkLpg"
API_KEY = "你的_API_KEY_在这里"          # ←←← 必须改这里！

BATCH_SIZE = 25          # 强烈建议 20~30，太多容易被封
START_ROW = 2            # 从第2行开始（跳过标题）
DELAY_MIN = 2.8
DELAY_MAX = 6.0
# ===========================================

def block_user(driver, username):
    try:
        clean_user = str(username).strip('@').strip()
        if not clean_user:
            return False
            
        driver.get(f"https://www.threads.net/@{clean_user}")
        time.sleep(3.5)
        
        # 点击右上角「更多」
        more_button = WebDriverWait(driver, 15).until(
            EC.element_to_be_clickable((By.XPATH, 
                "//button[contains(@aria-label,'更多') or contains(@aria-label,'More')] | //div[@role='button' and contains(., '更多')]"
            ))
        )
        more_button.click()
        time.sleep(2)
        
        # 点击「封锁」
        block_option = WebDriverWait(driver, 12).until(
            EC.element_to_be_clickable((By.XPATH, "//span[contains(text(),'封锁') or contains(text(),'Block')]"))
        )
        block_option.click()
        time.sleep(1.8)
        
        # 确认封锁
        confirm = WebDriverWait(driver, 10).until(
            EC.element_to_be_clickable((By.XPATH, "//button[contains(text(),'封锁') or contains(text(),'Block')]"))
        )
        confirm.click()
        
        print(f"✅ 成功封锁: @{clean_user}")
        return True
        
    except Exception as e:
        print(f"❌ 失败 @{username} → {e}")
        return False

# ===================== 主程序 =====================
print("正在连接 Google Sheet...")
gc = gspread.api_key(API_KEY)
sh = gc.open_by_url(SPREADSHEET_URL)
worksheet = sh.sheet1   # 第一个工作表

data = worksheet.get_all_records()
usernames = []
for row in data[START_ROW-2:]:
    if row.get("帳號名稱"):
        usernames.append(row["帳號名稱"])

print(f"✅ 共读取到 {len(usernames)} 个用户")
print(f"本次将封锁前 {BATCH_SIZE} 个用户（可修改 BATCH_SIZE）\n")

confirm = input("输入 Y 开始执行: ")
if confirm.upper() != "Y":
    print("已取消")
    sys.exit(0)

# 启动浏览器
options = webdriver.ChromeOptions()
options.add_argument("--start-maximized")
# options.add_argument("--user-data-dir=threads_profile")  # 推荐：保存登录状态

driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()), options=options)

try:
    driver.get("https://www.threads.net")
    input("\n请在打开的浏览器中**登录你的 Threads 账号**，完成后按 Enter 继续...\n")
    
    success = 0
    for i, user in enumerate(usernames[:BATCH_SIZE], 1):
        print(f"[{i}/{BATCH_SIZE}] 处理中...")
        if block_user(driver, user):
            success += 1
        
        # 随机延时
        sleep_time = random.uniform(DELAY_MIN, DELAY_MAX)
        time.sleep(sleep_time)
        
        if i % 8 == 0:
            print(f"💤 休息 20 秒，保护账号...")
            time.sleep(20)
    
    print(f"\n本次运行结束！成功封锁 {success} 个用户")

finally:
    input("\n按 Enter 关闭浏览器...")
    driver.quit()
