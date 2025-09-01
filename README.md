

import React, { useEffect, useMemo, useRef, useState } from "react";
import { createRoot } from "react-dom/client";
import Tesseract from "tesseract.js";
import {
  Card,
  CardContent,
  CardHeader,
  CardTitle,
} from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import { Label } from "@/components/ui/label";
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
  DialogFooter,
} from "@/components/ui/dialog";
import {
  Tabs,
  TabsContent,
  TabsList,
  TabsTrigger,
} from "@/components/ui/tabs";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import { Badge } from "@/components/ui/badge";
import { Progress } from "@/components/ui/progress";
import { BarChart, Bar, XAxis, YAxis, Tooltip, ResponsiveContainer, CartesianGrid } from "recharts";
import { motion } from "framer-motion";
import { Plus, Upload, Camera, Calculator, BarChart3, Trash2, Edit3, Save, RefreshCcw } from "lucide-react";

/**
 * 记账小程序（前端单文件）
 * - OCR: Tesseract.js（在浏览器本地识别）
 * - 持久化：localStorage
 * - UI: shadcn/ui + Tailwind
 *
 * 功能与需求对照：
 * 1) 上传小票照片 → 识别出 商品名/价格/日期 → 自动按 8% 税率计算税后金额并记录（可编辑）
 * 2) 支持手动输入
 * 3) 商品名可手动修改
 * 4) 四类（食材/饮料/香烟/其他），展示每类总额与占比
 * 5) 一键查看【日/周/月】总金额
 * 6) 生成条形统计图（分类支出）
 * 7) 初始金库 15,000 日元（每月重置），每次记账自动扣减并计算本月剩余天数的日均可用额度
 */

// ---------- 常量与工具 ----------
const TAX_RATE = 0.08; // 8%
const CATEGORIES = ["食材", "饮料", "香烟", "其他"] as const;
const STORE_KEY = "liying-budget-tracker-v1";
const BUDGET_INIT = 15000; // 每月初始金库

function ymd(date = new Date()) {
  const y = date.getFullYear();
  const m = String(date.getMonth() + 1).padStart(2, "0");
  const d = String(date.getDate()).padStart(2, "0");
  return `${y}-${m}-${d}`;
}
function monthKey(date = new Date()) {
  const y = date.getFullYear();
  const m = String(date.getMonth() + 1).padStart(2, "0");
  return `${y}-${m}`;
}
function lastDayOfMonth(date = new Date()) {
  return new Date(date.getFullYear(), date.getMonth() + 1, 0).getDate();
}
function startOfWeek(date = new Date()) {
  // 以周一为一周开始（日本常用）
  const d = new Date(date);
  const day = d.getDay(); // 0(日) ~ 6(六)
  const diff = (day === 0 ? -6 : 1 - day);
  d.setDate(d.getDate() + diff);
  d.setHours(0, 0, 0, 0);
  return d;
}
function endOfWeek(date = new Date()) {
  const s = startOfWeek(date);
  const e = new Date(s);
  e.setDate(s.getDate() + 6);
  e.setHours(23, 59, 59, 999);
  return e;
}
function toJPY(n) {
  return Math.round(n);
}
function fmtJPY(n) {
  return new Intl.NumberFormat("ja-JP", { style: "currency", currency: "JPY", maximumFractionDigits: 0 }).format(Math.round(n || 0));
}
function within(d, from, to) {
  const x = new Date(d).getTime();
  return x >= from.getTime() && x <= to.getTime();
}

// 简单分类规则（可覆盖）
const keywordCategory = (name) => {
  const s = (name || "").toLowerCase();
  if (/米|肉|豚|牛|鶏|魚|野菜|卵|豆|麺|パン|弁当|惣菜|おにぎり|寿司|カレー|鍋|味噌|醤油|油|調味|食パン|りんご|バナナ/.test(s)) return "食材";
  if (/水|お茶|緑茶|紅茶|コーヒー|ジュース|飲料|牛乳|ビール|酒|ワイン|酎ハイ|サイダー/.test(s)) return "饮料";
  if (/たばこ|タバコ|煙草|シガレット|cigarette/i.test(name)) return "香烟";
  return "其他";
};

// ---------- 数据模型 ----------
// Entry = { id, date: "YYYY-MM-DD", items: [{id, name, priceBeforeTax, priceAfterTax, category}], note? }

function loadState() {
  try { return JSON.parse(localStorage.getItem(STORE_KEY) || "{}"); } catch { return {}; }
}
function saveState(s) {
  localStorage.setItem(STORE_KEY, JSON.stringify(s));
}

// 初始化或按月重置预算
function usePersistentState() {
  const [state, setState] = useState(() => {
    const now = new Date();
    const mk = monthKey(now);
    const prev = loadState();
    if (!prev.monthKey || prev.monthKey !== mk) {
      const fresh = { monthKey: mk, entries: [], budgetInit: BUDGET_INIT };
      saveState(fresh);
      return fresh;
    }
    if (prev.budgetInit == null) prev.budgetInit = BUDGET_INIT;
    if (!Array.isArray(prev.entries)) prev.entries = [];
    return prev;
  });

  useEffect(() => { saveState(state); }, [state]);
  return [state, setState];
}

// ---------- OCR 解析 ----------
const jpDateRegexes = [
  /(\d{4})[\/\.\-年](\d{1,2})[\/\.\-月](\d{1,2})日?/,
  /(\d{1,2})[\/\.\-月](\d{1,2})日?[\/\.\-](\d{2,4})/,
  /(\d{1,2})[\/\.\-](\d{1,2})[\/\.\-](\d{2,4})/
];

const moneyRegex = /([-+]?\d+[\d,]*\.?\d*)\s*(円|JPY)?/;

function parseReceiptText(text) {
  const lines = text
    .split(/\r?\n/)
    .map((l) => l.replace(/[\uFF10-\uFF5A]/g, (ch) => String.fromCharCode(ch.charCodeAt(0) - 0xFEE0))) // 全角转半角
    .map((l) => l.replace(/[,，]/g, ""))
    .map((l) => l.trim())
    .filter(Boolean);

  // 找日期
  let foundDate = null;
  for (const r of jpDateRegexes) {
    for (const l of lines) {
      const m = l.match(r);
      if (m) {
        let y, mo, d;
        if (r === jpDateRegexes[0]) { y = +m[1]; mo = +m[2]; d = +m[3]; }
        else if (r === jpDateRegexes[1]) { mo = +m[1]; d = +m[2]; y = +m[3]; if (y < 100) y += 2000; }
        else { mo = +m[1]; d = +m[2]; y = +m[3]; if (y < 100) y += 2000; }
        const dt = new Date(y, mo - 1, d);
        if (!isNaN(dt.getTime())) { foundDate = ymd(dt); break; }
      }
    }
    if (foundDate) break;
  }

  // 解析行 → 尝试提取【名称】【价格】
  const skipWords = /小計|合計|税込|税抜|内税|外税|ポイント|釣り|お預り|レジ|店|電話|tel|領収|お買上|barcode|品番|伝票|点数|数量|qty|tax|total|sum|sub/gi;

  const items = [];
  for (const l of lines) {
    if (skipWords.test(l)) { continue; }
    // 行末数字优先作为价格
    const parts = l.split(/\s+/);
    let priceStr = null;
    for (let i = parts.length - 1; i >= 0; i--) {
      const m = parts[i].match(moneyRegex);
      if (m) { priceStr = m[1]; break; }
    }
    if (!priceStr) continue;
    const price = parseFloat(priceStr);
    if (isNaN(price) || price <= 0) continue;

    // 名称 = 去掉价格后的部分
    let name = l.replace(priceStr, "").replace(/円|JPY/i, "").trim();
    name = name.replace(/\s{2,}/g, " ");
    if (!name) name = "未命名";

    const category = keywordCategory(name);
    const priceAfterTax = toJPY(price * (1 + TAX_RATE));
    items.push({ id: crypto.randomUUID(), name, category, priceBeforeTax: price, priceAfterTax });
  }

  return { date: foundDate || ymd(), items };
}

// ---------- 组件 ----------
function Header({ monthSpent, monthBalance, dailyLimit, budgetInit }) {
  const daysLeft = lastDayOfMonth(new Date()) - new Date().getDate();
  return (
    <div className="grid gap-4 md:grid-cols-3">
      <Card className="shadow-md">
        <CardHeader className="pb-2"><CardTitle className="text-lg">本月累计支出</CardTitle></CardHeader>
        <CardContent className="text-2xl font-bold">{fmtJPY(monthSpent)}</CardContent>
      </Card>
      <Card className="shadow-md">
        <CardHeader className="pb-2"><CardTitle className="text-lg">金库余额（起始 {fmtJPY(budgetInit)}）</CardTitle></CardHeader>
        <CardContent>
          <div className="text-2xl font-bold">{fmtJPY(monthBalance)}</div>
          <Progress value={Math.max(0, 100 - (monthBalance / budgetInit) * 100)} className="mt-2" />
        </CardContent>
      </Card>
      <Card className="shadow-md">
        <CardHeader className="pb-2"><CardTitle className="text-lg">剩余{daysLeft}天·建议日均不超</CardTitle></CardHeader>
        <CardContent className="text-2xl font-bold">{fmtJPY(dailyLimit)}</CardContent>
      </Card>
    </div>
  );
}

function EntryRow({ entry, onEdit, onDelete }) {
  const total = entry.items.reduce((s, it) => s + (it.priceAfterTax || 0), 0);
  return (
    <div className="rounded-2xl border p-3 flex flex-col gap-2">
      <div className="flex items-center justify-between">
        <div className="font-semibold">{entry.date}</div>
        <div className="flex items-center gap-2">
          <Badge variant="secondary">{fmtJPY(total)}</Badge>
          <Button size="sm" variant="outline" onClick={() => onEdit(entry)}><Edit3 className="w-4 h-4 mr-1"/>编辑</Button>
          <Button size="sm" variant="destructive" onClick={() => onDelete(entry.id)}><Trash2 className="w-4 h-4 mr-1"/>删除</Button>
        </div>
      </div>
      <div className="grid md:grid-cols-2 gap-2">
        {entry.items.map((it) => (
          <div key={it.id} className="flex items-center justify-between text-sm bg-muted/40 rounded-xl px-3 py-2">
            <div className="truncate mr-2">{it.name} <span className="opacity-60">· {it.category}</span></div>
            <div className="font-medium">{fmtJPY(it.priceAfterTax)}</div>
          </div>
        ))}
      </div>
    </div>
  );
}

function TotalsPanel({ entries }) {
  const today = ymd();
  const now = new Date();
  const wFrom = startOfWeek(now);
  const wTo = endOfWeek(now);
  const mKey = monthKey(now);

  const sumBy = (pred) => toJPY(entries.filter(pred).flatMap(e => e.items).reduce((s, it) => s + (it.priceAfterTax||0), 0));

  const daySum = sumBy(e => e.date === today);
  const weekSum = sumBy(e => within(e.date, wFrom, wTo));
  const monthSum = sumBy(e => monthKey(new Date(e.date)) === mKey);

  return (
    <div className="grid gap-3 md:grid-cols-3">
      <Card className="shadow-sm"><CardHeader className="pb-2"><CardTitle className="text-base">今天合计</CardTitle></CardHeader><CardContent className="text-2xl font-bold">{fmtJPY(daySum)}</CardContent></Card>
      <Card className="shadow-sm"><CardHeader className="pb-2"><CardTitle className="text-base">本周合计</CardTitle></CardHeader><CardContent className="text-2xl font-bold">{fmtJPY(weekSum)}</CardContent></Card>
      <Card className="shadow-sm"><CardHeader className="pb-2"><CardTitle className="text-base">本月合计</CardTitle></CardHeader><CardContent className="text-2xl font-bold">{fmtJPY(monthSum)}</CardContent></Card>
    </div>
  );
}

function CategoryChart({ entries }) {
  const catTotals = useMemo(() => {
    const map = Object.fromEntries(CATEGORIES.map(c => [c, 0]));
    for (const e of entries) {
      for (const it of e.items) {
        map[it.category] = (map[it.category] || 0) + (it.priceAfterTax || 0);
      }
    }
    const total = Object.values(map).reduce((a,b)=>a+b,0) || 1;
    return CATEGORIES.map(c => ({ category: c, amount: toJPY(map[c]||0), percent: Math.round((map[c]||0)/total*100) }));
  }, [entries]);

  return (
    <Card className="shadow-md">
      <CardHeader className="pb-2 flex items-center justify-between">
        <CardTitle className="text-lg flex items-center"><BarChart3 className="w-5 h-5 mr-2"/>分类支出（金额/占比）</CardTitle>
      </CardHeader>
      <CardContent>
        <div className="text-sm mb-3 flex flex-wrap gap-2">
          {catTotals.map(ct => (
            <Badge key={ct.category} variant="secondary">{ct.category}: {fmtJPY(ct.amount)} · {ct.percent}%</Badge>
          ))}
        </div>
        <div className="w-full h-72">
          <ResponsiveContainer width="100%" height="100%">
            <BarChart data={catTotals}>
              <CartesianGrid strokeDasharray="3 3" />
              <XAxis dataKey="category" />
              <YAxis />
              <Tooltip formatter={(v) => fmtJPY(v)} />
              <Bar dataKey="amount" />
            </BarChart>
          </ResponsiveContainer>
        </div>
      </CardContent>
    </Card>
  );
}

function EditEntryDialog({ open, onOpenChange, entry, onSave }) {
  const [draft, setDraft] = useState(entry);
  useEffect(() => setDraft(entry), [entry]);
  if (!draft) return null;

  const updateItem = (id, patch) => {
    setDraft({ ...draft, items: draft.items.map(it => it.id === id ? { ...it, ...patch } : it) });
  };
  const addItem = () => {
    setDraft({ ...draft, items: [...draft.items, { id: crypto.randomUUID(), name: "", category: "其他", priceBeforeTax: 0, priceAfterTax: 0 }] });
  };
  const deleteItem = (id) => {
    setDraft({ ...draft, items: draft.items.filter(it => it.id !== id) });
  };
  const recalc = (it) => {
    const p = parseFloat(it.priceBeforeTax || 0);
    return toJPY(p * (1 + TAX_RATE));
  };

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="max-w-3xl">
        <DialogHeader>
          <DialogTitle>编辑记录 · {draft.date}</DialogTitle>
        </DialogHeader>
        <div className="space-y-3">
          <div className="grid grid-cols-2 gap-2">
            <div>
              <Label>日期</Label>
              <Input type="date" value={draft.date} onChange={(e)=>setDraft({...draft, date: e.target.value})}/>
            </div>
            <div>
              <Label>备注</Label>
              <Input value={draft.note||""} onChange={(e)=>setDraft({...draft, note: e.target.value})}/>
            </div>
          </div>

          <div className="space-y-2">
            <div className="flex items-center justify-between">
              <Label>条目</Label>
              <Button size="sm" variant="outline" onClick={addItem}><Plus className="w-4 h-4 mr-1"/>新增一项</Button>
            </div>
            <div className="grid gap-2">
              {draft.items.map((it) => (
                <div key={it.id} className="grid md:grid-cols-12 gap-2 items-center border rounded-xl p-2">
                  <Input className="md:col-span-4" placeholder="商品名" value={it.name} onChange={(e)=>updateItem(it.id,{ name: e.target.value })} />
                  <Select value={it.category} onValueChange={(v)=>updateItem(it.id,{ category: v })}>
                    <SelectTrigger className="md:col-span-2"><SelectValue placeholder="分类"/></SelectTrigger>
                    <SelectContent>
                      {CATEGORIES.map(c => <SelectItem key={c} value={c}>{c}</SelectItem>)}
                    </SelectContent>
                  </Select>
                  <Input className="md:col-span-2" type="number" min="0" step="1" placeholder="税前价格" value={it.priceBeforeTax}
                         onChange={(e)=>updateItem(it.id,{ priceBeforeTax: e.target.value, priceAfterTax: toJPY(parseFloat(e.target.value||0)*(1+TAX_RATE)) })} />
                  <div className="md:col-span-2 font-medium text-right pr-2">{fmtJPY(recalc(it))}</div>
                  <Button className="md:col-span-2" variant="destructive" size="icon" onClick={()=>deleteItem(it.id)}><Trash2 className="w-4 h-4"/></Button>
                </div>
              ))}
            </div>
          </div>

          <DialogFooter>
            <Button onClick={()=>onSave(draft)}><Save className="w-4 h-4 mr-1"/>保存</Button>
          </DialogFooter>
        </div>
      </DialogContent>
    </Dialog>
  );
}

function OCRUploader({ onParsedDraft }) {
  const [imgUrl, setImgUrl] = useState(null);
  const [busy, setBusy] = useState(false);
  const [rawText, setRawText] = useState("");
  const fileRef = useRef(null);

  const handleFile = async (file) => {
    if (!file) return;
    const url = URL.createObjectURL(file);
    setImgUrl(url);
    setBusy(true);
    try {
      const { data } = await Tesseract.recognize(file, "jpn+eng", { logger: () => {} });
      setRawText(data.text || "");
      const draft = parseReceiptText(data.text || "");
      onParsedDraft(draft);
    } catch (e) {
      alert("OCR 失败：" + e.message);
    } finally {
      setBusy(false);
    }
  };

  return (
    <Card className="shadow-md">
      <CardHeader className="pb-2"><CardTitle className="text-lg">上传小票识别</CardTitle></CardHeader>
      <CardContent className="space-y-3">
        <div className="flex items-center gap-2">
          <Input type="file" accept="image/*" ref={fileRef} onChange={(e)=>handleFile(e.target.files?.[0])}/>
          <Button variant="outline" onClick={()=>fileRef.current?.click()}><Upload className="w-4 h-4 mr-1"/>选择图片</Button>
        </div>
        {imgUrl && (
          <div className="grid md:grid-cols-2 gap-3">
            <div>
              <div className="text-sm mb-1 opacity-70">预览</div>
              <img src={imgUrl} alt="receipt" className="rounded-2xl w-full object-contain max-h-80 border"/>
            </div>
            <div>
              <div className="text-sm mb-1 opacity-70">OCR 原文（可用于排查）</div>
              <Textarea className="min-h-80" value={rawText} onChange={(e)=>setRawText(e.target.value)} />
            </div>
          </div>
        )}
        {busy && <div className="text-sm animate-pulse">正在识别中… 请稍候</div>}
      </CardContent>
    </Card>
  );
}

function ManualInput({ onCreate }) {
  const [date, setDate] = useState(ymd());
  const [items, setItems] = useState([{ id: crypto.randomUUID(), name: "", category: "其他", priceBeforeTax: 0, priceAfterTax: 0 }]);

  const updateItem = (id, patch) => setItems(items.map(it => it.id === id ? { ...it, ...patch } : it));
  const addItem = () => setItems([...items, { id: crypto.randomUUID(), name: "", category: "其他", priceBeforeTax: 0, priceAfterTax: 0 }]);
  const deleteItem = (id) => setItems(items.filter(it => it.id !== id));

  const canSave = items.some(it => (it.name?.trim()?.length || 0) > 0 && (parseFloat(it.priceBeforeTax) > 0));

  return (
    <Card className="shadow-md">
      <CardHeader className="pb-2"><CardTitle className="text-lg">手动输入</CardTitle></CardHeader>
      <CardContent className="space-y-3">
        <div className="grid md:grid-cols-3 gap-2">
          <div>
            <Label>日期</Label>
            <Input type="date" value={date} onChange={(e)=>setDate(e.target.value)} />
          </div>
          <div className="md:col-span-2 flex items-end justify-end">
            <Button onClick={()=>setItems([{ id: crypto.randomUUID(), name: "", category: "其他", priceBeforeTax: 0, priceAfterTax: 0 }])} variant="outline"><RefreshCcw className="w-4 h-4 mr-1"/>清空条目</Button>
          </div>
        </div>

        <div className="space-y-2">
          <div className="flex items-center justify-between">
            <Label>条目</Label>
            <Button size="sm" variant="outline" onClick={addItem}><Plus className="w-4 h-4 mr-1"/>新增一项</Button>
          </div>
          {items.map((it) => (
            <div key={it.id} className="grid md:grid-cols-12 gap-2 items-center border rounded-xl p-2">
              <Input className="md:col-span-4" placeholder="商品名" value={it.name} onChange={(e)=>updateItem(it.id,{ name: e.target.value })} />
              <Select value={it.category} onValueChange={(v)=>updateItem(it.id,{ category: v })}>
                <SelectTrigger className="md:col-span-2"><SelectValue placeholder="分类"/></SelectTrigger>
                <SelectContent>
                  {CATEGORIES.map(c => <SelectItem key={c} value={c}>{c}</SelectItem>)}
                </SelectContent>
              </Select>
              <Input className="md:col-span-2" type="number" min="0" step="1" placeholder="税前价格"
                     value={it.priceBeforeTax}
                     onChange={(e)=>updateItem(it.id,{ priceBeforeTax: e.target.value, priceAfterTax: toJPY(parseFloat(e.target.value||0)*(1+TAX_RATE)) })} />
              <div className="md:col-span-2 font-medium text-right pr-2">{fmtJPY(toJPY((parseFloat(it.priceBeforeTax)||0)*(1+TAX_RATE)))}</div>
              <Button className="md:col-span-2" variant="destructive" size="icon" onClick={()=>deleteItem(it.id)}><Trash2 className="w-4 h-4"/></Button>
            </div>
          ))}
        </div>

        <div className="flex justify-end">
          <Button disabled={!canSave} onClick={()=>onCreate({ id: crypto.randomUUID(), date, items: items.map(it=>({ ...it, priceAfterTax: toJPY((parseFloat(it.priceBeforeTax)||0)*(1+TAX_RATE)) })) })}>
            <Save className="w-4 h-4 mr-1"/>保存记录
          </Button>
        </div>
      </CardContent>
    </Card>
  );
}

export default function App() {
  const [state, setState] = usePersistentState();
  const [ocrDraft, setOcrDraft] = useState(null);
  const [editing, setEditing] = useState(null);

  const entries = state.entries || [];
  const thisMonthKey = state.monthKey || monthKey();
  const monthEntries = entries.filter(e => monthKey(new Date(e.date)) === thisMonthKey);
  const monthSpent = toJPY(monthEntries.flatMap(e=>e.items).reduce((s, it)=>s+(it.priceAfterTax||0),0));
  const monthBalance = Math.max(0, (state.budgetInit||BUDGET_INIT) - monthSpent);
  const daysLeft = lastDayOfMonth(new Date()) - new Date().getDate();
  const dailyLimit = daysLeft > 0 ? toJPY(monthBalance / daysLeft) : 0;

  const addEntry = (entry) => {
    // 修正空白项 & 自动分类
    const cleaned = { ...entry, items: entry.items
      .filter(it => (it.name?.trim()?.length || 0) > 0 && (parseFloat(it.priceBeforeTax) > 0 || parseFloat(it.priceAfterTax) > 0))
      .map(it => ({
        ...it,
        category: CATEGORIES.includes(it.category) ? it.category : keywordCategory(it.name),
        priceBeforeTax: parseFloat(it.priceBeforeTax)||toJPY((parseFloat(it.priceAfterTax)||0)/(1+TAX_RATE)),
        priceAfterTax: toJPY(parseFloat(it.priceAfterTax)||((parseFloat(it.priceBeforeTax)||0)*(1+TAX_RATE)))
      })) };
    if (cleaned.items.length === 0) return;
    setState({ ...state, entries: [cleaned, ...entries] });
    setOcrDraft(null);
  };
  const deleteEntry = (id) => setState({ ...state, entries: entries.filter(e=>e.id!==id) });
  const saveEdit = (draft) => {
    setState({ ...state, entries: entries.map(e => e.id === draft.id ? { ...draft, items: draft.items.map(it => ({ ...it, priceAfterTax: toJPY((parseFloat(it.priceBeforeTax)||0)*(1+TAX_RATE)) })) } : e) });
    setEditing(null);
  };
  const clearAll = () => {
    if (confirm("确定要清空本月所有记录吗？")) {
      setState({ monthKey: monthKey(), entries: [], budgetInit: BUDGET_INIT });
    }
  };

  const totalByCategory = useMemo(()=>{
    const map = Object.fromEntries(CATEGORIES.map(c=>[c,0]));
    for (const e of monthEntries) for (const it of e.items) map[it.category] += (it.priceAfterTax||0);
    return map;
  }, [monthEntries]);

  return (
    <div className="min-h-screen bg-gradient-to-b from-white to-slate-50 text-slate-900 p-4 md:p-8">
      <div className="max-w-5xl mx-auto space-y-6">
        <motion.h1 initial={{ opacity: 0, y: -6 }} animate={{ opacity: 1, y: 0 }} transition={{ duration: 0.4 }}
          className="text-2xl md:text-3xl font-bold tracking-tight">🍱 日常伙食费记账 · 8%税率</motion.h1>

        <Header monthSpent={monthSpent} monthBalance={monthBalance} dailyLimit={dailyLimit} budgetInit={state.budgetInit||BUDGET_INIT} />

        <Tabs defaultValue="ocr" className="mt-2">
          <TabsList className="grid grid-cols-3 w-full">
            <TabsTrigger value="ocr"><Upload className="w-4 h-4 mr-1"/>上传小票</TabsTrigger>
            <TabsTrigger value="manual"><Plus className="w-4 h-4 mr-1"/>手动输入</TabsTrigger>
            <TabsTrigger value="records"><Calculator className="w-4 h-4 mr-1"/>记录与统计</TabsTrigger>
          </TabsList>

          <TabsContent value="ocr" className="space-y-4">
            <OCRUploader onParsedDraft={(d)=>setOcrDraft({ id: crypto.randomUUID(), ...d })} />
            {ocrDraft && (
              <Card className="shadow-md">
                <CardHeader className="pb-2"><CardTitle className="text-lg">识别结果（可编辑后保存）</CardTitle></CardHeader>
                <CardContent className="space-y-3">
                  <div className="grid md:grid-cols-3 gap-2">
                    <div>
                      <Label>日期</Label>
                      <Input type="date" value={ocrDraft.date} onChange={(e)=>setOcrDraft({...ocrDraft, date: e.target.value})}/>
                    </div>
                  </div>

                  <div className="space-y-2">
                    {ocrDraft.items.length === 0 && <div className="text-sm opacity-70">未识别到条目，请手动添加。</div>}
                    {ocrDraft.items.map((it, idx) => (
                      <div key={it.id} className="grid md:grid-cols-12 gap-2 items-center border rounded-xl p-2">
                        <Input className="md:col-span-4" value={it.name} onChange={(e)=>{
                          const items = [...ocrDraft.items]; items[idx] = { ...it, name: e.target.value, category: keywordCategory(e.target.value) }; setOcrDraft({ ...ocrDraft, items });
                        }} />
                        <Select value={it.category} onValueChange={(v)=>{ const items = [...ocrDraft.items]; items[idx] = { ...it, category: v }; setOcrDraft({ ...ocrDraft, items }); }}>
                          <SelectTrigger className="md:col-span-2"><SelectValue placeholder="分类"/></SelectTrigger>
                          <SelectContent>
                            {CATEGORIES.map(c => <SelectItem key={c} value={c}>{c}</SelectItem>)}
                          </SelectContent>
                        </Select>
                        <Input className="md:col-span-2" type="number" min="0" step="1" value={it.priceBeforeTax}
                           onChange={(e)=>{ const v=e.target.value; const items=[...ocrDraft.items]; items[idx] = { ...it, priceBeforeTax: v, priceAfterTax: toJPY((parseFloat(v)||0)*(1+TAX_RATE)) }; setOcrDraft({ ...ocrDraft, items }); }} />
                        <div className="md:col-span-2 font-medium text-right pr-2">{fmtJPY(toJPY((parseFloat(it.priceBeforeTax)||0)*(1+TAX_RATE)))}</div>
                        <Button className="md:col-span-2" variant="destructive" size="icon" onClick={()=>{ const items = ocrDraft.items.filter(x=>x.id!==it.id); setOcrDraft({ ...ocrDraft, items }); }}><Trash2 className="w-4 h-4"/></Button>
                      </div>
                    ))}
                    <div className="flex justify-between">
                      <Button variant="outline" onClick={()=>setOcrDraft({ ...ocrDraft, items: [...ocrDraft.items, { id: crypto.randomUUID(), name: "", category: "其他", priceBeforeTax: 0, priceAfterTax: 0 }] })}><Plus className="w-4 h-4 mr-1"/>新增一项</Button>
                      <Button onClick={()=>addEntry(ocrDraft)}><Save className="w-4 h-4 mr-1"/>保存记录</Button>
                    </div>
                  </div>
                </CardContent>
              </Card>
            )}
          </TabsContent>

          <TabsContent value="manual">
            <ManualInput onCreate={addEntry} />
          </TabsContent>

          <TabsContent value="records" className="space-y-4">
            <TotalsPanel entries={entries} />
            <CategoryChart entries={monthEntries} />

            <Card className="shadow-md">
              <CardHeader className="pb-2 flex items-center justify-between">
                <CardTitle className="text-lg">本月记录</CardTitle>
                <div className="flex gap-2">
                  <Button variant="outline" onClick={()=>setState({ ...state, budgetInit: prompt("设置本月初始金库金额：", String(state.budgetInit||BUDGET_INIT))|0 || state.budgetInit })}>调整金库</Button>
                  <Button variant="destructive" onClick={clearAll}>清空本月</Button>
                </div>
              </CardHeader>
              <CardContent className="space-y-3">
                {monthEntries.length === 0 && <div className="text-sm opacity-70">本月暂无记录</div>}
                {monthEntries.map(e => (
                  <EntryRow key={e.id} entry={e} onEdit={setEditing} onDelete={deleteEntry} />
                ))}
              </CardContent>
            </Card>
          </TabsContent>
        </Tabs>

        <EditEntryDialog open={!!editing} onOpenChange={(v)=>!v&&setEditing(null)} entry={editing} onSave={saveEdit} />

        <div className="text-xs opacity-60 text-center pt-4">本工具在本地浏览器运行与存储。OCR 可能会有误，请在保存前确认条目与金额。</div>
      </div>
    </div>
  );
}

const mount = document.getElementById("root");
if (mount) createRoot(mount).render(<App />);
