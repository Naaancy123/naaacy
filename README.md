[wechat_classify.py](https://github.com/user-attachments/files/33139474/wechat_classify.py)
# naaacy
使用 Python 编写的小工具，用于清洗、分类和整理群聊记录，提取有价值的信息与资源。
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""群聊记录整理工具

功能：
  1. 读取微信/QQ 群聊导出的记录（json / csv / txt）
  2. 按 表情包/功能相关/工作要求/闲聊/其他 分类
  3. 从工作消息中提取待办事项（TODO）
  4. 脱敏（手机号/身份证/邮箱）
  5. 支持关键词搜索、分类筛选、导出 JSON
"""

import argparse
import csv
import json
import re
import sys
from datetime import datetime

# Windows 控制台默认 GBK，无法输出 emoji/特殊符号，强制 UTF-8
if hasattr(sys.stdout, 'reconfigure'):
    sys.stdout.reconfigure(encoding='utf-8')


def mask(text):
    """脱敏：手机号、身份证、邮箱"""
    if not text:
        return text
    text = re.sub(r'1[3-9]\d{9}', '***手机号***', text)
    text = re.sub(r'\b[1-9]\d{5}(18|19|20)\d{2}(0[1-9]|1[0-2])(0[1-9]|[12]\d|3[01])\d{3}[\dXx]\b', '***身份证***', text)
    text = re.sub(r'[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+', '***邮箱***', text)
    return text


def classify(content):
    """分类消息"""
    c = content.lower()
    # 表情包
    if any(p in content for p in ['[微笑]', '[大笑]', '[捂脸]', '[哭]', '[笑哭]', '[OK]', '[玫瑰]', '[烟花]', '[蛋糕]', '[福]', '[礼物]', '[色]', '[亲]', '[害羞]', '[调皮]', '[赞]', '[鼓掌]', '[汗]', '[冷]', '[尴尬]', '[怒]', '[鄙视]', '[泪奔]', '[无奈]', '[卖萌]', '[晕]', '[睡]', '[吐]', '[偷笑]', '[再见]', '[好的]', '[谢谢]', '[不客气]', '[哈哈]', '[嘻嘻]', '[嘿嘿]', '[哦]', '[嗯]', '[是]', '[对]', '[行]']) or \
       any(e in content for e in ['😂', '😄', '😀', '🤣', '😍', '😭', '😅', '😊', '❤️', '🌹', '🎉', '烟花', '蛋糕', '礼物', '福']):
        return 'emoji'
    # 功能相关
    if any(kw in content for kw in ['功能', '版本', '更新', '上线', '修复', '新增', '优化', 'bug']):
        return 'functional'
    # 工作要求
    work_kw = ['请', '完成', '提交', '会议', '任务', '计划', '安排', '要求', '节点', '进度', '汇报', '项目', '需求', '开发', '测试', '客户']
    if any(kw in content for kw in work_kw) and not re.search(r'请([喝吃看听玩聊])', content):
        return 'work'
    # 闲聊
    chat_kw = ['在吗', '干嘛', '在忙吗', '没事', '好的', '行', '可以', 'OK', '谢谢', '不谢', '哈哈', '嘻嘻', '嘿嘿', '哦', '嗯', '是', '对']
    if any(kw in content for kw in chat_kw):
        return 'chit-chat'
    return 'other'


def extract_todo(content, sender):
    """从工作消息中提取TODO（尽量提取“任务内容”而非人名）"""
    patterns = [
        # 请[负责人]在/于[时间][前]完成/提交[任务]
        r'请.{0,6}?(?:在|于|下?周[一二三四五六日天]|明天|后天|今天|本周|下周).{0,12}?(?:前|之前)?(?:完成|提交|汇报|安排|处理|开发|测试)\s*(.+)$',
        # 完成/提交/汇报/安排[任务]
        r'(?:完成|提交|汇报|安排|处理|开发|测试)\s*(.+)$',
        # 请[人][做][任务]
        r'请.{0,6}?(?:完成|提交|汇报|安排|处理)\s*(.+)$',
    ]
    for p in patterns:
        m = re.search(p, content)
        if m:
            todo = m.group(1).strip()
            todo = re.sub(r'[。.!！\s]+$', '', todo)
            if len(todo) >= 2:
                return {'text': mask(todo), 'source': mask(sender), 'timestamp': datetime.now().strftime('%Y-%m-%d %H:%M'), 'original': mask(content)}
    # 兜底：含动作词则去掉“请”和首尾称呼
    if re.search(r'完成|提交|汇报|安排|处理', content):
        todo = re.sub(r'^请|请$', '', content).strip()
        todo = re.sub(r'[。.!！\s]+$', '', todo)
        if len(todo) >= 2:
            return {'text': mask(todo), 'source': mask(sender), 'timestamp': datetime.now().strftime('%Y-%m-%d %H:%M'), 'original': mask(content)}
    return None


def read_file(path):
    """读取聊天记录文件"""
    encodings = ['utf-8', 'gbk', 'utf-8-sig']
    content = None
    for enc in encodings:
        try:
            with open(path, 'r', encoding=enc) as f:
                content = f.read()
            break
        except Exception:
            continue
    if content is None:
        raise Exception("无法读取文件，请确保为 UTF-8/GBK 编码")

    ext = path.split('.')[-1].lower()
    if ext == 'json':
        data = json.loads(content)
        return data if isinstance(data, list) else data.get('messages', [])
    elif ext == 'csv':
        lines = content.strip().split('\n')
        header = lines[0].lower()
        rows = csv.reader(lines[1:] if any(col in header for col in ['sender', 'message', 'content']) else lines)
        return [{'timestamp': r[0] if len(r) > 0 else datetime.now().strftime('%Y-%m-%d %H:%M'),
                 'sender': r[1] if len(r) > 1 else '未知',
                 'content': r[2] if len(r) > 2 else ''} for r in rows if len(r) > 1]
    else:  # txt
        messages = []
        for line in content.strip().split('\n'):
            line = line.strip()
            if not line:
                continue
            # 支持：[时间] 发送者: 内容 / 时间 发送者: 内容 / 发送者: 内容（中英文冒号均可）
            m = re.match(r'\[?(\d{4}-\d{2}-\d{2}[\sT]\d{2}:\d{2})]?\s+([^:：]+)[:：]\s*(.*)', line) or \
                re.match(r'([^:：]+)[:：]\s*(.*)', line)
            if m:
                groups = m.groups()
                if len(groups) == 3:
                    ts, sender, content = groups
                else:
                    sender, content = groups
                    ts = datetime.now().strftime('%Y-%m-%d %H:%M')
                messages.append({'timestamp': ts, 'sender': sender.strip(), 'content': content.strip()})
            elif messages:
                messages[-1]['content'] += '\n' + line
        return messages


def process(messages):
    """处理消息：分类 + TODO提取"""
    results = {
        'todo_items': [],
        'emoji_messages': [],
        'chit_chat_messages': [],
        'work_messages': [],
        'functional_messages': [],
        'other_messages': []
    }
    for msg in messages:
        c = mask(msg.get('content', ''))
        s = mask(msg.get('sender', '未知'))
        cat = classify(c)
        entry = {'timestamp': msg.get('timestamp', datetime.now().strftime('%Y-%m-%d %H:%M')),
                 'sender': s, 'content': c}
        if cat == 'emoji':
            results['emoji_messages'].append(entry)
        elif cat == 'functional':
            results['functional_messages'].append(entry)
        elif cat == 'work':
            results['work_messages'].append(entry)
            todo = extract_todo(c, s)
            if todo:
                results['todo_items'].append(todo)
        elif cat == 'chit-chat':
            results['chit_chat_messages'].append(entry)
        else:
            results['other_messages'].append(entry)
    return results


def search(results, keyword):
    """搜索关键词（兼容 todo 与普通消息的不同字段结构）"""
    if not keyword:
        return results
    k = keyword.lower()
    res = {key: [] for key in results}
    for cat, items in results.items():
        for item in items:
            # todo 项字段为 text/source，其余为 content/sender；待办额外检查原始内容 original
            text = item.get('text', item.get('content', ''))
            src = item.get('source', item.get('sender', ''))
            orig = item.get('original', '')
            if k in text.lower() or k in src.lower() or k in orig.lower():
                res[cat].append(item)
    return res


def format_output(results, category, keyword=''):
    """格式化输出"""
    total = sum(len(v) for v in results.values())
    print("+" + "=" * 70)
    print(f" 群聊记录整理报告（共 {total} 条消息）")
    print("=" * 70)

    def show(title, items, highlight=False, todo=False):
        if not items:
            return
        print(f" {title} ({len(items)}条)")
        print("-" * 40)
        for item in items:
            if todo:
                print(f"[{item.get('timestamp', '')}] {item.get('source', '未知')} → 待办:")
                c = item.get('text', '')
            else:
                print(f"[{item.get('timestamp', '')}] {item.get('sender', '未知')}:")
                c = item.get('content', '')
            if keyword and highlight:
                c = re.sub(keyword, lambda m: f"\033[91m{m.group()}\033[0m", c, flags=re.I)
            print(f"  {c}")
            print()

    if category in ['all', 'todo']:
        show("📋 待办事项", results['todo_items'], True, todo=True)
    if category in ['all', 'emoji']:
        show("😂 表情包", results['emoji_messages'])
    if category in ['all', 'chit-chat']:
        show("☕ 闲聊", results['chit_chat_messages'])
    # 已提取为待办的工作消息，不再重复显示在“工作要求”里
    todo_orig = {t.get('original', '') for t in results['todo_items']}
    work_no_todo = [m for m in results['work_messages'] if m.get('content', '') not in todo_orig]
    if category in ['all', 'work']:
        show("📝 工作要求（非待办）", work_no_todo)
    if category in ['all', 'functional']:
        show("🛠️ 功能相关", results['functional_messages'])
    if category in ['all', 'other']:
        show("⚙️ 其他", results['other_messages'])


def main():
    parser = argparse.ArgumentParser(description='群聊记录整理工具（修复版）')
    parser.add_argument('file', help='聊天记录文件路径')
    parser.add_argument('-c', '--cat', choices=['all', 'todo', 'emoji', 'chit-chat', 'work', 'functional', 'other'], default='all')
    parser.add_argument('-s', '--search', help='关键词搜索')
    parser.add_argument('-o', '--out', help='导出JSON结果')
    args = parser.parse_args()

    print("正在读取聊天记录...")
    try:
        messages = read_file(args.file)
    except Exception as e:
        print(f" 读取失败：{e}")
        sys.exit(1)
    if not messages:
        print(" 未读取到消息")
        sys.exit(1)

    print(" 正在处理...")
    results = process(messages)
    if args.search:
        results = search(results, args.search)

    format_output(results, args.cat, args.search)

    if args.out:
        with open(args.out, 'w', encoding='utf-8') as f:
            json.dump(results, f, ensure_ascii=False, indent=2)
        print(f"\n 已导出至：{args.out}")


if __name__ == '__main__':
    main()
