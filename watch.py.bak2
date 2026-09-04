#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""各プラットフォームの仕様変更・規約変更・UI変更を毎日チェックして一覧HTMLを作る。

使い方:
    python3 watch.py            # 全プラットフォームをチェック
    python3 watch.py mercari    # 指定したものだけ
"""
import difflib
import glob
import html
import json
import os
import re
import shutil
import subprocess
import sys
import time
import urllib.request
from datetime import datetime, timezone, timedelta

BASE = os.path.dirname(os.path.abspath(__file__))
PLATFORM_DIR = os.path.join(BASE, "platforms")
DATA_DIR = os.path.join(BASE, "data")
HISTORY = os.path.join(DATA_DIR, "history.json")
SITE = BASE
REPORT = os.path.join(SITE, "index.html")
KEEP_DAYS = 90
CONFIG = os.path.join(BASE, "config.json")
CHROME = os.environ.get("PW_CHROME") or (
    "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
    if sys.platform == "darwin" else "chromium-browser")
JST = timezone(timedelta(hours=9))
UA = ("Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 "
      "(KHTML, like Gecko) Chrome/126.0 Safari/537.36")
KEEP_SHOTS = 8
UI_BIG = 5   # 画面の部品がこの数以上入れ替わったら「大幅な変更」とみなす

# どのサイトでも毎回変わるだけで意味のない行（IDや残り時間など）は比べない
COMMON_IGNORE = [
    r"^[0-9]{6,}$",                                   # 長い数字だけの行（セッション番号）
    r"^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-",         # UUID
    r"^\s*$",
    r"^(Copyright|\u00a9)",
]


# ---------- 取得 ----------

def fetch_dom(url, tries=3):
    """ページを取る。失敗したら少し待って取り直す（一時的な混雑やbot判定よけ）。"""
    last = None
    for i in range(tries):
        if i:
            time.sleep(4 * i)
        try:
            req = urllib.request.Request(
                url, headers={"User-Agent": UA, "Accept-Language": "ja,en;q=0.8",
                              "Accept": "text/html,application/xhtml+xml,*/*;q=0.8"})
            with urllib.request.urlopen(req, timeout=60) as r:
                dom = r.read().decode("utf-8", "replace")
            if len(dom) < 500:
                raise RuntimeError("中身がほとんど空でした")
            return dom
        except Exception as e:
            last = e
    raise RuntimeError("ページを取得できませんでした（%s）: %s" % (last, url))


def fetch_json(url, tries=3):
    last = None
    for i in range(tries):
        if i:
            time.sleep(4 * i)
        try:
            req = urllib.request.Request(url, headers={"User-Agent": UA})
            with urllib.request.urlopen(req, timeout=60) as r:
                return json.loads(r.read().decode("utf-8"))
        except Exception as e:
            last = e
    raise RuntimeError("取得できませんでした（%s）: %s" % (last, url))


def shot_dir(pid):
    d = os.path.join(DATA_DIR, pid, "shots")
    os.makedirs(d, exist_ok=True)
    return d


def start_screenshot(pid, sid, url, stamp):
    """Chrome を画面なしで開いてページ全体の写真を撮る（裏で走らせる）。"""
    out = os.path.join(shot_dir(pid), "%s_%s.png" % (sid, stamp))
    cmd = [CHROME, "--headless=new", "--disable-gpu", "--no-sandbox", "--no-first-run",
           "--user-data-dir=" + os.path.join(DATA_DIR, "_chrome", sid + "_" + stamp),
           "--window-size=1280,1600", "--hide-scrollbars",
           "--virtual-time-budget=15000", "--screenshot=" + out, url]
    return subprocess.Popen(cmd, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL), out


def past_shots(pid, sid):
    return sorted(glob.glob(os.path.join(shot_dir(pid), sid + "_*.png")))


def prune_shots(pid, sid):
    for old in past_shots(pid, sid)[:-KEEP_SHOTS]:
        os.remove(old)


def rel(path):
    return os.path.relpath(path, BASE) if path else ""


def shot_stamp(path):
    m = re.search(r"_(\d{8})-(\d{4})\.png$", path or "")
    if not m:
        return ""
    d, t = m.group(1), m.group(2)
    return "%s/%s/%s %s:%s" % (d[:4], d[4:6], d[6:], t[:2], t[2:])


# ---------- 抽出 ----------

def to_text(dom):
    s = re.sub(r"(?is)<(script|style|noscript|svg)[^>]*>.*?</\1>", " ", dom)
    s = re.sub(r"(?i)<(br|/p|/div|/li|/h[1-6]|/tr)[^>]*>", "\n", s)
    s = re.sub(r"(?s)<[^>]+>", " ", s)
    s = html.unescape(s)
    lines = [re.sub(r"[ \t　]+", " ", ln).strip() for ln in s.split("\n")]
    return [ln for ln in lines if ln]


def to_skeleton(dom):
    """画面の「部品の顔ぶれ」。商品名や件数は無視する。"""
    parts = set()
    for tag, attrs in re.findall(r"<([a-zA-Z][a-zA-Z0-9-]*)((?:\s[^>]*)?)>", dom):
        t = tag.lower()
        if t in ("script", "style", "meta", "link", "path", "svg", "br", "img"):
            continue
        tid = re.search(r'data-testid="([^"]+)"', attrs)
        if tid:
            parts.add(t + "#" + tid.group(1))
        elif "-" in t:
            parts.add(t)
        else:
            role = re.search(r'role="([^"]+)"', attrs)
            if role:
                parts.add(t + "[" + role.group(1) + "]")
    return sorted(parts)


def to_links(dom, pattern):
    rx = re.compile(pattern)
    seen, items = set(), []
    for block in re.split(r'(?i)<li\b[^>]*postList__item', dom)[1:]:
        m = re.search(r'href="([^"]+)"', block)
        if not m:
            continue
        href = html.unescape(m.group(1))
        if href.startswith("/"):
            href = "https://jp-news.mercari.com" + href
        if not rx.match(href) or href in seen:
            continue
        t = re.search(r"(?is)<h2[^>]*>(.*?)</h2>", block)
        title = html.unescape(re.sub(r"(?s)<[^>]+>", " ", t.group(1))) if t else ""
        title = re.sub(r"\s+", " ", title).strip()
        d = re.search(r'datetime="([0-9-]+)"', block)
        c = re.search(r'postList__cat[^>]*>([^<]+)<', block)
        if len(title) < 4:
            continue
        seen.add(href)
        items.append({"title": title, "url": href,
                      "date": d.group(1) if d else "",
                      "cat": c.group(1).strip() if c else ""})
    return items


def prepare_text(dom, src):
    """規約・料金ページの本文だけを取り出す。

    clip  : {"start": 正規表現, "end": 正規表現} 見出しから見出しまでを切り出す
    ignore: 正規表現の配列。毎回変わるだけの行（IDや広告）を捨てる
    need  : この文字が本文になければ取得失敗とみなす（ログイン画面・認証画面よけ）
    """
    lines = to_text(dom)
    clip = src.get("clip") or {}
    if clip.get("start"):
        rx = re.compile(clip["start"])
        for i, ln in enumerate(lines):
            if rx.search(ln):
                lines = lines[i:]
                break
    if clip.get("end"):
        rx = re.compile(clip["end"])
        for i, ln in enumerate(lines):
            if i and rx.search(ln):
                lines = lines[:i]
                break
    ig = [re.compile(x) for x in src.get("ignore", [])] + [re.compile(x) for x in COMMON_IGNORE]
    lines = [ln for ln in lines if not any(r.search(ln) for r in ig)]
    need = src.get("need")
    if need and not any(need in ln for ln in lines):
        raise RuntimeError("いつもの本文が見つかりません（ログイン画面や認証画面の可能性）: %s" % need)
    if len(lines) < 5:
        raise RuntimeError("本文がほとんど取れませんでした")
    return lines


AMAZON_HELP_API = "https://sellercentral.amazon.co.jp/help/hub/mons-api/GetHelpTopic"


def amazon_help(guid):
    """セラーセントラルの規約ページ。画面はJavaScriptで作られていて素のHTMLには本文がないので、
    画面が裏で呼んでいるAPIを直接たたく。ログインは不要。"""
    body = json.dumps({"guid": guid, "locale": "ja-JP",
                       "statuses": ["Live"], "consistentRead": False}).encode()
    ref = "https://sellercentral.amazon.co.jp/help/hub/reference/external/" + guid
    last = None
    for i in range(3):
        if i:
            time.sleep(4 * i)
        try:
            req = urllib.request.Request(
                AMAZON_HELP_API, data=body,
                headers={"User-Agent": UA, "Content-Type": "application/json;charset=UTF-8",
                         "Accept": "application/json, text/plain, */*",
                         "Origin": "https://sellercentral.amazon.co.jp", "Referer": ref})
            with urllib.request.urlopen(req, timeout=60) as r:
                d = json.loads(r.read().decode("utf-8"))
            topic = d.get("topic") or {}
            lines = to_text(topic.get("content", ""))
            if len(lines) < 5:
                raise RuntimeError("本文がほとんど取れませんでした")
            return topic.get("title", ""), lines
        except Exception as e:
            last = e
    raise RuntimeError("Amazonの規約を取得できませんでした（%s）: %s" % (last, guid))


# ---------- 比較 ----------

def snap_path(pid, sid):
    d = os.path.join(DATA_DIR, pid)
    os.makedirs(d, exist_ok=True)
    return os.path.join(d, sid + ".json")


def load_prev(pid, sid):
    p = snap_path(pid, sid)
    if os.path.exists(p):
        with open(p, encoding="utf-8") as f:
            return json.load(f)
    return None


def save_snap(pid, sid, data):
    with open(snap_path(pid, sid), "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=1)


def diff_lines(old, new, limit=40):
    added, removed = [], []
    for line in difflib.unified_diff(old, new, n=0, lineterm=""):
        if line.startswith("+") and not line.startswith("+++"):
            added.append(line[1:])
        elif line.startswith("-") and not line.startswith("---"):
            removed.append(line[1:])
    return added[:limit], removed[:limit]


def check_source(pid, src, shot=None):
    sid, kind = src["id"], src["kind"]
    prev = load_prev(pid, sid)
    ev = {"source": sid, "label": src["label"], "kind": kind, "url": src.get("url", ""),
          "notify": src.get("notify", True),
          "changed": False, "first_run": prev is None,
          "added": [], "removed": [], "items": [], "note": "",
          "shot_now": "", "shot_prev": ""}

    if kind == "news":
        items = to_links(fetch_dom(src["url"]), src["link_pattern"])
        if prev:
            old = {i["url"] for i in prev["items"]}
            new_items = [i for i in items if i["url"] not in old]
            if new_items:
                ev["changed"] = True
                ev["items"] = new_items[:20]
        save_snap(pid, sid, {"items": items})

    elif kind == "text":
        lines = prepare_text(fetch_dom(src["url"]), src)
        if prev:
            added, removed = diff_lines(prev["lines"], lines)
            if added or removed:
                ev["changed"] = True
                ev["added"], ev["removed"] = added, removed
        save_snap(pid, sid, {"lines": lines})

    elif kind == "amazonhelp":
        title, lines = amazon_help(src["guid"])
        ev["url"] = "https://sellercentral.amazon.co.jp/help/hub/reference/external/" + src["guid"]
        ev["note"] = title
        if prev:
            added, removed = diff_lines(prev["lines"], lines)
            if added or removed:
                ev["changed"] = True
                ev["added"], ev["removed"] = added, removed
        save_snap(pid, sid, {"lines": lines})

    elif kind == "appstore":
        url = "https://itunes.apple.com/lookup?id=%s&country=%s" % (src["app_id"], src.get("country", "jp"))
        res = fetch_json(url)["results"][0]
        cur = {"version": res["version"], "released": res["currentVersionReleaseDate"],
               "notes": res.get("releaseNotes", "")}
        ev["url"] = res.get("trackViewUrl", "")
        ev["note"] = "現在 %s（%s 公開）" % (cur["version"], cur["released"][:10])
        if prev and prev.get("version") != cur["version"]:
            ev["changed"] = True
            ev["added"] = ["バージョン %s → %s" % (prev.get("version"), cur["version"]),
                           "更新内容: " + (cur["notes"] or "（記載なし）")]
        save_snap(pid, sid, cur)

    elif kind == "ui":
        sk = to_skeleton(fetch_dom(src["url"]))
        shots = past_shots(pid, sid)
        ev["shot_now"] = rel(shots[-1]) if shots else ""
        ev["shot_prev"] = rel(shots[-2]) if len(shots) > 1 else ""
        if prev:
            added, removed = diff_lines(prev["skeleton"], sk, limit=25)
            n = len(set(added) | set(removed))
            if added or removed:
                ev["changed"] = True
                ev["added"] = ["増えた部品: " + a for a in sorted(set(added))[:20]]
                ev["removed"] = ["消えた部品: " + r for r in sorted(set(removed))[:20]]
                # 部品が5つ以上まとめて入れ替わったら「大幅な変更」とみなす。
                # バナー差し替えのような小さな変更（1〜4個）は Slack に出さない。
                ev["ui_big"] = n >= UI_BIG
                ev["notify"] = ev["ui_big"]
        ev["note"] = "部品 %d 種類を監視中" % len(sk)
        save_snap(pid, sid, {"skeleton": sk})
        prune_shots(pid, sid)

    else:
        raise ValueError("不明な種類: " + kind)
    return ev



# ---------- ネット上のページを更新 ----------

def publish_site(day):
    """site フォルダを GitHub に送り、公開ページを更新する。"""
    if os.environ.get("PW_NOPUSH"):
        print("公開ページ: GitHub Actions 側で更新するためここでは送りません")
        return
    if not os.path.isdir(os.path.join(SITE, ".git")):
        return
    try:
        subprocess.run(["git", "-C", SITE, "add", "-A"], check=True,
                       stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
        st = subprocess.run(["git", "-C", SITE, "status", "--porcelain"],
                            capture_output=True, text=True).stdout.strip()
        if not st:
            print("公開ページ: 変わりがないため更新しませんでした")
            return
        subprocess.run(["git", "-C", SITE,
                        "-c", "user.email=y.nakamura@busoken.com",
                        "-c", "user.name=platform-watch",
                        "commit", "-qm", day], check=True,
                       stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
        subprocess.run(["git", "-C", SITE, "push", "-q", "origin", "main"], check=True,
                       stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, timeout=120)
        print("公開ページ: 更新しました")
    except Exception as e:
        print("公開ページ: 更新に失敗しました -> %s" % e)


# ---------- Slack への報告 ----------

def load_config():
    cfg = {}
    if os.path.exists(CONFIG):
        with open(CONFIG, encoding="utf-8") as f:
            cfg = json.load(f)
    if os.environ.get("PW_SLACK_WEBHOOK"):
        cfg["slack_webhook_url"] = os.environ["PW_SLACK_WEBHOOK"]
    cfg.setdefault("slack_channel_label", "改革最新情報")
    cfg.setdefault("post_when_no_change", False)
    cfg.setdefault("site_url", "https://ynakamura-code.github.io/pw-a41f6c2b/")
    return cfg


def slack_bullets(r):
    """1つの情報源の変更を、箇条書きの行にする。リンクの文字はページ名にする。"""
    name = "%s／%s" % (r["platform_name"], r["label"])
    out = []
    if r["kind"] == "news":
        for i in r.get("items", [])[:8]:
            d = ("（%s）" % i["date"]) if i.get("date") else ""
            out.append("・<%s|%s>%s" % (i["url"], i["title"], d))
    elif r["kind"] == "text":
        out.append("・記載が変わりました → <%s|%s>" % (r["url"], name) if r.get("url")
                   else "・%s の記載が変わりました" % name)
        for a in r.get("added", [])[:3]:
            out.append("　＋ %s" % a[:140])
        for d in r.get("removed", [])[:3]:
            out.append("　－ %s" % d[:140])
    elif r["kind"] == "appstore":
        ver = r["added"][0] if r.get("added") else "アプリが更新されました"
        out.append("・%s → <%s|%s>" % (ver, r["url"], name) if r.get("url")
                   else "・%s" % ver)
        if len(r.get("added", [])) > 1:
            out.append("　%s" % r["added"][1][:140])
    elif r["kind"] == "ui":
        where = r["label"].replace("Web画面の作り", "").strip("（）")
        out.append("・*%s の画面が大きく変わりました*" % (where or "画面"))
        out.append("　出品や取引の手順に影響するかもしれません。"
                   "変更前後の画面写真を共有ページで見比べてください。")
        if r.get("url"):
            out.append("　→ <%s|%s／%s>" % (r["url"], r["platform_name"], where or "画面"))
    return out


def slack_text(results, now):
    """Slack に流す文章を組み立てる。変更がなければ None を返す。"""
    live = [r for r in results if r.get("notify", True)]
    todo = [r for r in live if r.get("changed") and r.get("manual")]
    fyi = [r for r in live if r.get("changed") and not r.get("manual")]
    errs = [r for r in live if r.get("error")]
    if not (todo or fyi):
        return None

    def blocks(rows):
        by = []
        for r in rows:
            if not by or by[-1][0] != r["platform_name"]:
                by.append((r["platform_name"], []))
            by[-1][1].extend(slack_bullets(r))
        return "\n".join("【%s】\n%s" % (n, "\n".join(ls)) for n, ls in by)

    d = now[:10].split("-")
    out = [":mega: *改革最新情報*（%s/%s）" % (d[1].lstrip("0"), d[2].lstrip("0")), ""]

    if todo:
        out.append("<!channel>")
        out.append(":red_circle: *要確認 — マニュアル修正が必要かも（%d件）*" % len(todo))
        out.append(blocks(todo))
    else:
        out.append(":white_check_mark: 本日、マニュアル修正が必要な変更はありません。")

    if fyi:
        out.append("")
        out.append(":eyes: *参考 — 目を通すだけでOK（%d件）*" % len(fyi))
        out.append(blocks(fyi))

    if errs:
        out.append("")
        out.append(":warning: 取得できなかったもの: " + "、".join(r["label"] for r in errs))

    site = load_config().get("site_url", "")
    if site:
        out.append("")
        out.append("詳細確認はこちらから")
        out.append(site)
    return "\n".join(out)


def post_to_slack(results, now, send=True):
    cfg = load_config()
    url = cfg.get("slack_webhook_url", "")
    if not url:
        return
    text = slack_text(results, now)
    if text is None:
        if not cfg.get("post_when_no_change"):
            print("Slack: 変更なしのため投稿しませんでした")
            return
        text = "*プラットフォーム変更ウォッチ*  %s\n本日は変更ありませんでした。" % now
    if not send:
        print("---- Slack に送る文章（送信はしていません） ----")
        print(text)
        print("---- ここまで ----")
        return
    body = json.dumps({"text": text}).encode("utf-8")
    req = urllib.request.Request(url, data=body,
                                 headers={"Content-Type": "application/json"})
    try:
        with urllib.request.urlopen(req, timeout=30) as r:
            print("Slack: 投稿しました（%s）" % r.status)
    except Exception as e:
        print("Slack: 投稿に失敗しました -> %s" % e)


# ---------- 実行 ----------

def load_history():
    if os.path.exists(HISTORY):
        with open(HISTORY, encoding="utf-8") as f:
            return json.load(f)
    return []


def run(only=None, shots=True, send=True):
    now_dt = datetime.now(JST)
    now = now_dt.strftime("%Y-%m-%d %H:%M")
    stamp = now_dt.strftime("%Y%m%d-%H%M")
    history = load_history()
    results = []

    plats = []
    for fn in sorted(os.listdir(PLATFORM_DIR)):
        if not fn.endswith(".json"):
            continue
        with open(os.path.join(PLATFORM_DIR, fn), encoding="utf-8") as f:
            p = json.load(f)
        if not only or p["id"] == only:
            plats.append(p)
    plats.sort(key=lambda x: (x.get("order", 100), x["name"]))

    # 画面の写真はまとめて先に撮る（1枚40秒ほどかかるため同時に走らせる）
    procs = []
    if shots:
        shutil.rmtree(os.path.join(DATA_DIR, "_chrome"), ignore_errors=True)
    for plat in plats:
        for src in plat["sources"]:
            if shots and src["kind"] == "ui":
                print("  画面を撮影中: %s" % src["label"])
                procs.append(start_screenshot(plat["id"], src["id"], src["url"], stamp)[0])
    for p in procs:
        try:
            p.wait(timeout=180)
        except Exception:
            p.kill()
    shutil.rmtree(os.path.join(DATA_DIR, "_chrome"), ignore_errors=True)

    for plat in plats:
        print("== %s ==" % plat["name"])
        for src in plat["sources"]:
            try:
                time.sleep(1)   # 相手のサーバーに負担をかけないよう少し間をあける
                ev = check_source(plat["id"], src)
                ev.update({"platform": plat["id"], "platform_name": plat["name"], "checked_at": now})
                state = "初回登録" if ev["first_run"] else ("変更あり" if ev["changed"] else "変更なし")
                print("  [%s] %s" % (state, src["label"]))
                results.append(ev)
                if ev["changed"]:
                    history.insert(0, ev)
            except Exception as e:
                print("  [失敗] %s -> %s" % (src["label"], e))
                results.append({"platform": plat["id"], "platform_name": plat["name"],
                                "source": src["id"], "label": src["label"], "kind": src["kind"],
                                "url": src.get("url", ""), "error": str(e), "changed": False,
                                "first_run": False, "added": [], "removed": [], "items": [],
                                "note": "", "shot_now": "", "shot_prev": "", "checked_at": now})

    history = history[:300]
    os.makedirs(DATA_DIR, exist_ok=True)
    with open(HISTORY, "w", encoding="utf-8") as f:
        json.dump(history, f, ensure_ascii=False, indent=1)
    write_report(results, history, now)
    print("\n一覧を更新しました: %s" % REPORT)
    publish_site(now[:10])
    post_to_slack(results, now, send)


KIND = {"news": ("お知らせ", "news"), "text": ("規約・ルール", "rule"),
        "amazonhelp": ("規約・ルール", "rule"),
        "appstore": ("アプリ更新", "app"), "ui": ("画面（UI）", "ui")}


MANUAL_WORDS = ("規約", "ガイド", "禁止", "ルール", "手数料", "本人確認", "改定", "変更",
                "廃止", "終了", "開始", "出品", "配送", "梱包", "支払", "振込", "売上")


def impact_of(r):
    """マニュアル修正が要りそうかを判定する。要りそうなら True。"""
    if r["kind"] in ("text", "amazonhelp"):
        return True
    if r["kind"] == "ui":
        return bool(r.get("ui_big"))
    if r["kind"] == "news":
        for i in r.get("items", []):
            t = i.get("title", "") + i.get("cat", "")
            if any(w in t for w in MANUAL_WORDS):
                return True
        return False
    return False


def esc(s):
    return html.escape(str(s))


def card_body(r):
    k = r["kind"]
    out = []
    if k == "news":
        out.append("<ul class='news'>")
        for i in r["items"]:
            cat = "<span class='cat'>%s</span>" % esc(i["cat"]) if i.get("cat") else ""
            out.append("<li><span class='date'>%s</span>%s<a href='%s' target='_blank'>%s</a></li>"
                       % (esc(i.get("date", "")), cat, esc(i["url"]), esc(i["title"])))
        out.append("</ul>")
    elif k == "ui":
        if r["shot_prev"] and r["shot_now"]:
            out.append(
                "<div class='shots'>"
                "<figure><figcaption>前回 %s</figcaption>"
                "<a href='%s' target='_blank'><img src='%s' loading='lazy'></a></figure>"
                "<figure><figcaption>今回 %s</figcaption>"
                "<a href='%s' target='_blank'><img src='%s' loading='lazy'></a></figure></div>"
                % (esc(shot_stamp(r["shot_prev"])), esc(r["shot_prev"]), esc(r["shot_prev"]),
                   esc(shot_stamp(r["shot_now"])), esc(r["shot_now"]), esc(r["shot_now"])))
            out.append("<p class='hint'>画像はクリックで拡大できます。商品の並びは毎回変わるので、"
                       "見るのは枠組み（メニュー・ボタン・並べ方）です。</p>")
        elif r["shot_now"]:
            out.append("<div class='shots'><figure><figcaption>今回 %s（比較用の前回画像はまだありません）"
                       "</figcaption><a href='%s' target='_blank'><img src='%s' loading='lazy'></a>"
                       "</figure></div>" % (esc(shot_stamp(r["shot_now"])), esc(r["shot_now"]), esc(r["shot_now"])))
    for a in r["added"]:
        out.append("<p class='ln add'>%s</p>" % esc(a))
    for d in r["removed"]:
        out.append("<p class='ln del'>%s</p>" % esc(d))
    if r.get("url") and k in ("text", "ui", "appstore"):
        out.append("<p class='golink'><a href='%s' target='_blank'>"
                   "この変更があったページを開く ↗</a></p>" % esc(r["url"]))
    return "".join(out)


def site_days():
    ds = [os.path.basename(f)[:-5] for f in glob.glob(os.path.join(SITE, "20*.html"))]
    return sorted(set(ds), reverse=True)


PICKER_RE = re.compile(r"<div class='picker'>.*?</div>", re.S)


def build_picker(days, cur):
    """日付プルダウンの HTML を作る。days は新しい順。"""
    opts = "".join("<option value='%s.html'%s>%s%s</option>"
                   % (esc(d), " selected" if d == cur else "", esc(d),
                      "（最新）" if d == days[0] else "")
                   for d in days)
    return ("<div class='picker'>見る日付 <select onchange=\"location.href=this.value\">%s</select>"
            "<span class='pn'>%d日分を保存しています（最大%d日）</span></div>"
            % (opts, len(days), KEEP_DAYS))


def refresh_pickers(days):
    """過去のページのプルダウンも、今ある日付すべてに差し替える。"""
    for d in days:
        path = os.path.join(SITE, d + ".html")
        if not os.path.exists(path):
            continue
        with open(path, encoding="utf-8") as f:
            doc = f.read()
        doc2 = PICKER_RE.sub(lambda m: build_picker(days, d), doc, count=1)
        if doc2 != doc:
            with open(path, "w", encoding="utf-8") as f:
                f.write(doc2)


def prune_site():
    keep = site_days()[:KEEP_DAYS]
    for f in glob.glob(os.path.join(SITE, "20*.html")):
        if os.path.basename(f)[:-5] not in keep:
            os.remove(f)
    for d in glob.glob(os.path.join(SITE, "shots", "20*")):
        if os.path.basename(d) not in keep:
            shutil.rmtree(d, ignore_errors=True)


def write_report(results, history, now):
    os.makedirs(SITE, exist_ok=True)
    day = now[:10]
    for r in results:
        r["manual"] = impact_of(r) if r.get("changed") else False
        r["anchor"] = "p-%s-%s" % (r["platform"], r["source"])
        if r["kind"] == "ui" and r.get("changed"):
            dst = os.path.join(SITE, "shots", day)
            os.makedirs(dst, exist_ok=True)
            for key, tag in (("shot_prev", "prev"), ("shot_now", "now")):
                src = os.path.join(BASE, r[key]) if r.get(key) else ""
                if src and os.path.exists(src):
                    name = "%s_%s_%s.png" % (r["platform"], r["source"], tag)
                    shutil.copyfile(src, os.path.join(dst, name))
                    r[key] = "shots/%s/%s" % (day, name)
                else:
                    r[key] = ""
        else:
            r["shot_prev"] = r["shot_now"] = ""

    order, by = [], {}
    for r in results:
        pid = r["platform"]
        if pid not in by:
            order.append(pid)
            by[pid] = {"name": r["platform_name"], "todo": [], "fyi": [], "quiet": [], "errs": []}
        if r.get("error"):
            by[pid]["errs"].append(r)
        elif not r.get("changed"):
            by[pid]["quiet"].append(r)
        elif r["manual"]:
            by[pid]["todo"].append(r)
        else:
            by[pid]["fyi"].append(r)

    n_todo = sum(len(by[p]["todo"]) for p in order)
    n_fyi = sum(len(by[p]["fyi"]) for p in order)
    hit_plats = [by[p]["name"] for p in order if by[p]["todo"] or by[p]["fyi"]]

    # ---- 上段：全体のひとこと ----
    if n_todo:
        banner = ("<div class='banner hit'><b>%d件</b> マニュアルの確認が要りそうな変更があります"
                  "<span>対象: %s ／ ほかに参考情報が%d件</span></div>"
                  % (n_todo, esc("、".join(hit_plats)), n_fyi))
    elif n_fyi:
        banner = ("<div class='banner mid'>マニュアルに関わる変更はありません"
                  "<span>参考情報が%d件（%s）</span></div>" % (n_fyi, esc("、".join(hit_plats))))
    else:
        banner = ("<div class='banner ok'>変更はありませんでした"
                  "<span>%d のプラットフォームすべてで、マニュアルの修正は不要です</span></div>" % len(order))

    # ---- プラットフォーム別のカード（増えるほど列が並ぶ） ----
    tiles = []
    for pid in order:
        b = by[pid]
        if b["todo"]:
            cls, st = "todo", "要確認 %d件" % len(b["todo"])
        elif b["fyi"]:
            cls, st = "fyi", "参考 %d件" % len(b["fyi"])
        elif b["errs"]:
            cls, st = "err", "取得できず"
        else:
            cls, st = "ok", "変更なし"
        lines = []
        for r in b["todo"] + b["fyi"]:
            k = KIND.get(r["kind"], (r["kind"], "news"))
            lines.append("<li><a href='#%s'><span class='dot %s'></span>%s</a></li>"
                         % (esc(r["anchor"]), k[1], esc(r["label"])))
        body = ("<ul class='tlist'>%s</ul>" % "".join(lines)) if lines else                "<p class='tnone'>見た %d か所すべて変わっていません</p>" % len(b["quiet"])
        tiles.append("<div class='tile %s'><h3>%s</h3><span class='tstate'>%s</span>%s</div>"
                     % (cls, esc(b["name"]), esc(st), body))

    # ---- 下段：プラットフォームごとの詳細 ----
    def cards_of(rows, manual):
        out = []
        for r in rows:
            label, cls = KIND.get(r["kind"], (r["kind"], "news"))
            flag = ("<span class='flag todo'>マニュアル要確認</span>" if manual
                    else "<span class='flag fyi'>参考</span>")
            if r["url"]:
                title = "<h3><a class='tlink' href='%s' target='_blank'>%s ↗</a></h3>" % (
                    esc(r["url"]), esc(r["label"]))
                link = "<a class='src' href='%s' target='_blank'>公式ページを開く</a>" % esc(r["url"])
            else:
                title, link = "<h3>%s</h3>" % esc(r["label"]), ""
            out.append(
                "<article class='card %s' id='%s'><header>%s<span class='badge %s'>%s</span>"
                "%s%s</header><div class='body'>%s</div></article>"
                % (cls, esc(r["anchor"]), flag, cls, esc(label), title, link, card_body(r)))
        return "".join(out)

    details = []
    for pid in order:
        b = by[pid]
        if not (b["todo"] or b["fyi"] or b["errs"]):
            continue
        details.append("<h2 class='plathead'>%s</h2>" % esc(b["name"]))
        if b["errs"]:
            details.append("<div class='errs'><b>取得できなかったもの</b>%s</div>" % "".join(
                "<div>%s — %s</div>" % (esc(r["label"]), esc(r["error"])) for r in b["errs"]))
        if b["todo"]:
            details.append("<h3 class='sechead todo'>マニュアル修正が要りそうな変更</h3>" + cards_of(b["todo"], True))
        if b["fyi"]:
            details.append("<h3 class='sechead'>参考（画面・アプリなど）</h3>" + cards_of(b["fyi"], False))

    quiet_all = []
    for pid in order:
        for r in by[pid]["quiet"]:
            name = by[pid]["name"] + " / " + r["label"]
            note = r.get("note") or "変更なし"
            if r.get("url"):
                quiet_all.append("<a class='chip' href='%s' target='_blank'>%s ↗<small>%s</small></a>"
                                 % (esc(r["url"]), esc(name), esc(note)))
            else:
                quiet_all.append("<span class='chip'>%s<small>%s</small></span>"
                                 % (esc(name), esc(note)))

    hist = {}
    for h in history[:60]:
        hist.setdefault(h["checked_at"][:10], []).append(h)
    hrows = []
    for hday in sorted(hist, reverse=True):
        hrows.append("<div class='hday'><b>%s</b><ul>" % esc(hday))
        for h in hist[hday]:
            label = KIND.get(h["kind"], (h["kind"],))[0]
            first = ""
            if h.get("items"):
                first = h["items"][0]["title"]
            elif h.get("added"):
                first = h["added"][0]
            elif h.get("removed"):
                first = h["removed"][0]
            name = h["platform_name"] + " / " + h["label"]
            link = h.get("items", [{}])[0].get("url") if h.get("items") else h.get("url")
            title = ("<a href='%s' target='_blank'>%s ↗</a>" % (esc(link), esc(name))) if link else esc(name)
            hrows.append("<li><span class='tag'>%s</span>%s<span class='sub'>%s</span></li>"
                         % (esc(label), title, esc(first[:100])))
        hrows.append("</ul></div>")

    days = sorted(set(site_days() + [day]), reverse=True)[:KEEP_DAYS]
    picker = build_picker(days, day)

    doc = """<!doctype html><html lang="ja"><meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>プラットフォーム変更ウォッチ</title>
<style>
:root{--bg:#f6f7f9;--card:#fff;--line:#e3e6ea;--tx:#1c1e21;--sub:#6b7280;
 --news:#2563eb;--rule:#7c3aed;--app:#059669;--ui:#d97706;--red:#d94a3d}
*{box-sizing:border-box}
body{font-family:-apple-system,"Hiragino Sans","Noto Sans JP",sans-serif;margin:0;
 background:var(--bg);color:var(--tx);line-height:1.7}
.wrap{max-width:1080px;margin:0 auto;padding:32px 20px 80px}
h1{font-size:22px;margin:0 0 4px}
.time{color:var(--sub);font-size:13px;margin:0 0 20px}
.banner{border-radius:12px;padding:18px 22px;font-size:16px;margin-bottom:22px}
.banner span{display:block;font-size:13px;opacity:.8;margin-top:2px}
.banner.hit{background:#fff1f0;border:1px solid #f3c9c4;color:#a52a1f}
.banner.mid{background:#fff8e8;border:1px solid #ecd9a8;color:#8a6100}
.banner.ok{background:#eef7f0;border:1px solid #cbe5d3;color:#256b3d}
.banner b{font-size:24px}
.tiles{display:grid;gap:14px;grid-template-columns:repeat(auto-fit,minmax(240px,1fr));margin-bottom:12px}
.tile{background:var(--card);border:1px solid var(--line);border-top:4px solid #cbd5e1;
 border-radius:12px;padding:14px 16px}
.tile.todo{border-top-color:var(--red)}
.tile.fyi{border-top-color:var(--ui)}
.tile.ok{border-top-color:#34a853}
.tile.err{border-top-color:#b8860b}
.tile h3{margin:0;font-size:16px}
.tstate{display:inline-block;font-size:12px;font-weight:700;border-radius:999px;
 padding:2px 10px;margin:6px 0 8px;background:#eef1f5;color:#4b5563}
.tile.todo .tstate{background:#fdeaea;color:#a52a1f}
.tile.fyi .tstate{background:#fdf1de;color:#8a6100}
.tile.ok .tstate{background:#e8f5ec;color:#256b3d}
ul.tlist{list-style:none;margin:0;padding:0}
ul.tlist li{font-size:13px;padding:3px 0}
ul.tlist a{color:var(--tx);text-decoration:none}
ul.tlist a:hover{text-decoration:underline}
.dot{display:inline-block;width:8px;height:8px;border-radius:50%%;margin-right:7px}
.dot.news{background:var(--news)}.dot.rule{background:var(--rule)}
.dot.app{background:var(--app)}.dot.ui{background:var(--ui)}
.tnone{margin:0;font-size:13px;color:var(--sub)}
h2{font-size:15px;color:var(--sub);font-weight:600;margin:34px 0 12px}
h2.plathead{font-size:19px;color:var(--tx);border-bottom:2px solid var(--line);
 padding-bottom:8px;margin-top:40px}
.sechead{font-size:14px;margin:18px 0 10px;color:var(--sub);font-weight:600}
.sechead.todo{color:#a52a1f;border-left:5px solid var(--red);padding-left:10px}
.card{background:var(--card);border:1px solid var(--line);border-radius:12px;
 margin-bottom:16px;overflow:hidden;scroll-margin-top:20px}
.card header{display:flex;align-items:center;gap:10px;flex-wrap:wrap;
 padding:14px 18px;border-bottom:1px solid var(--line);background:#fbfcfd}
.card h3{font-size:16px;margin:0;flex:1}
.flag{font-size:12px;font-weight:700;border-radius:5px;padding:3px 10px}
.flag.todo{background:var(--red);color:#fff}
.flag.fyi{background:#e9ecf1;color:#5b6472}
.badge{font-size:12px;font-weight:700;color:#fff;border-radius:5px;padding:3px 9px}
.badge.news{background:var(--news)}.badge.rule{background:var(--rule)}
.badge.app{background:var(--app)}.badge.ui{background:var(--ui)}
.src{font-size:13px;color:#fff;background:var(--news);text-decoration:none;
 border-radius:6px;padding:5px 12px;white-space:nowrap}
.src:hover{opacity:.85}
.tlink{color:var(--tx);text-decoration:none}
.tlink:hover{text-decoration:underline}
.golink{margin:12px 0 0}
.golink a{display:inline-block;font-size:13px;color:var(--news);text-decoration:none;
 border:1px solid #c7d7f0;background:#f2f7ff;border-radius:8px;padding:7px 14px}
.golink a:hover{background:#e6f0ff}
.body{padding:16px 18px}
ul.news{list-style:none;margin:0;padding:0}
ul.news li{padding:8px 0;border-bottom:1px dashed var(--line);display:flex;gap:10px;
 align-items:baseline;flex-wrap:wrap}
ul.news li:last-child{border:0}
.date{color:var(--sub);font-size:13px;font-variant-numeric:tabular-nums}
.cat{font-size:11px;background:#eef1f5;color:#4b5563;border-radius:4px;padding:1px 7px}
ul.news a{color:var(--tx);text-decoration:none;border-bottom:1px solid #cbd5e1}
.ln{margin:4px 0;padding:8px 12px;border-radius:8px;font-size:14px}
.ln.add{background:#eaf7ee;border-left:4px solid #34a853}
.ln.del{background:#fdeeee;border-left:4px solid #ea4335;text-decoration:line-through;color:#7a3b38}
.shots{display:grid;grid-template-columns:1fr 1fr;gap:14px;margin-bottom:12px}
.shots figure{margin:0}
.shots figcaption{font-size:12px;color:var(--sub);margin-bottom:6px}
.shots img{width:100%%;border:1px solid var(--line);border-radius:8px;display:block}
.hint{font-size:12px;color:var(--sub);margin:0}
.chips{display:flex;flex-wrap:wrap;gap:8px}
.chip{background:#fff;border:1px solid var(--line);border-radius:999px;
 padding:6px 14px;font-size:13px;color:var(--sub);text-decoration:none;display:inline-block}
a.chip:hover{border-color:var(--news);color:var(--news)}
.chip small{display:block;font-size:11px;color:#9aa1ab}
.errs{background:#fffaf0;border:1px solid #f0dfb8;border-radius:10px;padding:12px 16px;
 font-size:13px;color:#8a6100;margin-bottom:16px}
details summary{cursor:pointer;font-size:14px;color:var(--sub);margin-bottom:10px}
.hday{margin-bottom:14px}
.hday b{font-size:13px;color:var(--sub)}
.hday ul{list-style:none;margin:6px 0 0;padding:0}
.hday li{background:#fff;border:1px solid var(--line);border-radius:8px;padding:8px 12px;
 margin-bottom:6px;font-size:14px}
.hday li a{color:var(--tx);text-decoration:none;border-bottom:1px solid #cbd5e1}
.hday li a:hover{color:var(--news);border-color:var(--news)}
.tag{font-size:11px;background:#eef1f5;border-radius:4px;padding:2px 7px;margin-right:8px;color:#4b5563}
.sub{display:block;font-size:12px;color:var(--sub);margin-top:2px}
@media(max-width:700px){.shots{grid-template-columns:1fr}}
.picker{background:#fff;border:1px solid var(--line);border-radius:10px;padding:10px 14px;
 margin-bottom:18px;font-size:14px}
.picker select{font-size:14px;padding:4px 8px;border:1px solid var(--line);border-radius:6px;
 margin-left:6px;background:#fff}
.pn{font-size:12px;color:var(--sub);margin-left:10px}
</style>
<div class="wrap">
<h1>プラットフォーム変更ウォッチ</h1>
<p class="time">最終チェック %s</p>
%s
%s
<div class="tiles">%s</div>
%s
<h2>変更がなかったところ</h2>
<details open><summary>すべて見る（%d か所）— クリックで公式ページへ</summary><div class="chips">%s</div></details>
<h2>これまでに見つかった変更</h2>
%s
</div></html>""" % (esc(now), picker, banner, "".join(tiles), "".join(details),
                    len(quiet_all), "".join(quiet_all),
                    "".join(hrows) or "<p class='hint'>まだありません</p>")
    with open(os.path.join(SITE, day + ".html"), "w", encoding="utf-8") as f:
        f.write(doc)
    with open(REPORT, "w", encoding="utf-8") as f:
        f.write(doc)
    prune_site()
    refresh_pickers(site_days()[:KEEP_DAYS])


if __name__ == "__main__":
    argv = [a for a in sys.argv[1:] if not a.startswith("-")]
    run(argv[0] if argv else None, shots="--noshot" not in sys.argv,
        send="--nopost" not in sys.argv)
