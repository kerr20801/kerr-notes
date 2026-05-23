# 遠端 Patch Script 的兩個陷阱

> **背景**：無法直接編輯遠端伺服器的檔案時，寫一個 patch script 用 SSH 執行是常見做法。但有幾個坑不踩到不知道。

---

## 為什麼用 patch script

SSH heredoc 寫 Python 有個問題：

```bash
ssh server << 'EOF'
python3 -c "
x = f'''
SELECT {col} FROM table
WHERE id = {id}
'''
print(x)
"
EOF
```

Python f-string 的 `{col}` 和 bash 的變數展開會衝突。加了 `'EOF'` 引號可以防 bash 展開，但 Python 的 f-string 語法本身又跟 triple-quote heredoc 結束符號很容易搞混。

正確做法是在本機寫 `.py` 檔，SCP 過去執行：

```bash
# 本機
vim /tmp/patch_xxx.py

# 推過去執行
scp /tmp/patch_xxx.py server:/tmp/
ssh server "python3 /tmp/patch_xxx.py"
```

但這樣做也有坑。

---

## 坑一：triple-quote 裡的 f-string

patch script 通常長這樣：

```python
old = '''
def process():
    sql = f'''SELECT * FROM users WHERE id={uid}'''
    return sql
'''

new = '''
def process():
    sql = f'''SELECT * FROM users WHERE id=? '''
    return sql
'''

content = open('/path/to/file.py').read()
content = content.replace(old, new)
open('/path/to/file.py', 'w').write(content)
```

問題：`old` 和 `new` 裡面的 `f'''` 和外層的 `'''` 衝突，Python 解析器會在第一個 `'''` 就結束字串，後面的內容變成語法錯誤。

**解法**：patch 字串裡有 SQL f-string 的，改用字串拼接：

```python
old = (
    "def process():\n"
    "    sql = f'''SELECT * FROM users WHERE id={uid}'''\n"
    "    return sql\n"
)

new = (
    "def process():\n"
    "    sql = 'SELECT * FROM users WHERE id=?'\n"
    "    return sql\n"
)
```

或者用不同的引號：

```python
old = """
def process():
    sql = f'''SELECT * FROM users WHERE id={uid}'''
    return sql
"""
# 外層用 """，裡面的 ''' 就不會衝突
```

---

## 坑二：regex DOTALL 截斷 routes

patch script 有時需要用 regex 找到某個 function 或 route 的範圍，然後替換：

```python
import re

content = open('app.py').read()

# 想找 /api/users 這個 route
pattern = r'@app\.route\("/api/users"\).*?^@app'
match = re.search(pattern, content, re.DOTALL | re.MULTILINE)
```

問題：`re.DOTALL` 讓 `.` 匹配換行，`.*?` 是 non-greedy，但如果 function 很長，這個 pattern 可能一路匹配到下一個 `@app` decorator 才停，把中間的 routes 全部吃掉。

更嚴重的情況：replace 之後，被吞掉的那些 routes 就消失了。

**解法**：不要用 regex 找範圍，改用 `find()` 定點替換：

```python
old_str = '''@app.route("/api/users")
def get_users():
    return jsonify(users)'''

new_str = '''@app.route("/api/users")
def get_users():
    return jsonify(users), 200'''

assert old_str in content, f"找不到目標字串，請確認"
content = content.replace(old_str, new_str, 1)  # 只替換第一個
```

`assert` 很重要：如果找不到 `old_str`，寧可報錯，也不要靜默地什麼都沒改。

---

## 最小可靠的 patch script 結構

```python
#!/usr/bin/env python3
"""
Patch: 說明這個 patch 在改什麼
Date: 2026-05-XX
"""

import sys

TARGET = '/path/to/target/file.py'

old = (
    "# 舊的程式碼\n"
    "result = old_function()\n"
)

new = (
    "# 新的程式碼\n"
    "result = new_function()\n"
)

# 讀檔
with open(TARGET) as f:
    content = f.read()

# 驗證目標存在
if old not in content:
    print(f"ERROR: 找不到目標字串，請確認 {TARGET} 版本正確")
    sys.exit(1)

# 確認只有一個匹配（防止替換到不該替換的地方）
count = content.count(old)
if count > 1:
    print(f"WARNING: 找到 {count} 個匹配，請確認 old_str 夠唯一")
    sys.exit(1)

# 替換
content = content.replace(old, new, 1)

# 寫回
with open(TARGET, 'w') as f:
    f.write(content)

print(f"OK: {TARGET} 已更新")
```

---

## 一行總結

> Patch script 的重點不是「怎麼改」，而是「改錯了怎麼知道」。`assert` 和 `count == 1` 是最低限度的安全網。

---

*記錄於 2026-05，多次踩坑後整理*
