
#!/bin/bash
# ============================================
# 发票备份一键归档
# 用法：在发票系统网页里点【导出JSON备份】后，双击本文件
# 作用：自动找到刚导出的 JSON → 按日期归档到 桌面/发票备份/
# ============================================

BACKUP_DIR="$HOME/Desktop/发票备份"
mkdir -p "$BACKUP_DIR"
TODAY=$(date +%F)

echo "🔍 正在查找最近导出的发票备份文件..."

# 在 下载/桌面 找最近 1 小时内修改的 json，按修改时间倒序
CANDIDATE=$(find "$HOME/Downloads" "$HOME/Desktop" -maxdepth 2 -name "*.json" -mtime -0.05 2>/dev/null \
  | xargs ls -t 2>/dev/null | head -1)

# 兜底：没找到最近1小时的，就找全局最新的 json
if [ -z "$CANDIDATE" ]; then
  CANDIDATE=$(find "$HOME/Downloads" "$HOME/Desktop" -maxdepth 2 -name "*.json" 2>/dev/null \
    | xargs ls -t 2>/dev/null | head -1)
fi

if [ -z "$CANDIDATE" ]; then
  osascript -e 'display notification "下载和桌面都没找到 JSON 文件" with title "发票备份" subtitle "❌ 未找到导出文件"'
  echo "❌ 没找到任何 JSON 文件。请先在网页里点【导出JSON备份】，再双击我。"
  read -n 1 -s -r -p "按任意键关闭..."
  exit 1
fi

# 校验内容确实是发票数据（含 taxStatus 字段的数组）
if ! head -c 2000 "$CANDIDATE" | grep -q "taxStatus"; then
  osascript -e "display notification \"$CANDIDATE 不像发票备份\" with title \"发票备份\" subtitle \"❌ 文件校验失败\""
  echo "❌ 找到的最新 JSON 不是发票系统导出的：$CANDIDATE"
  read -n 1 -s -r -p "按任意键关闭..."
  exit 1
fi

# 统计发票条数
COUNT=$(python3 -c "import json;print(len(json.load(open('$CANDIDATE'))))" 2>/dev/null || echo "?")

# 目标文件名：发票备份_2026-08-28.json，同日重复则 -2、-3
TARGET="$BACKUP_DIR/发票备份_${TODAY}.json"
N=2
while [ -e "$TARGET" ]; do
  TARGET="$BACKUP_DIR/发票备份_${TODAY}-${N}.json"
  N=$((N+1))
done

cp "$CANDIDATE" "$TARGET"

# 报告距上次备份天数
LAST=$(ls -t "$BACKUP_DIR"/发票备份_*.json 2>/dev/null | sed -n 2p)
GAP_MSG=""
if [ -n "$LAST" ]; then
  LAST_DATE=$(basename "$LAST" | grep -o '[0-9]\{4\}-[0-9]\{2\}-[0-9]\{2\}' | head -1)
  if [ -n "$LAST_DATE" ]; then
    GAP=$(( ( $(date +%s) - $(date -j -f %F "$LAST_DATE" +%s 2>/dev/null || echo $(date +%s)) ) / 86400 ))
    [ "$GAP" -gt 7 ] && GAP_MSG="（⚠️ 距上次备份已 ${GAP} 天）"
  fi
fi

TOTAL=$(ls "$BACKUP_DIR"/发票备份_*.json 2>/dev/null | wc -l | tr -d ' ')
osascript -e "display notification \"已归档 ${COUNT} 张发票，共 ${TOTAL} 份历史备份\" with title \"发票备份 ✅\" subtitle \"${TODAY}${GAP_MSG}\""
echo "✅ 归档完成：$TARGET（${COUNT} 张发票，第 ${TOTAL} 份备份）${GAP_MSG}"
echo ""
read -n 1 -s -r -p "按任意键关闭..."
