import json
import hashlib
import os

def parse_sites(filename, is_adult=False):
    if not os.path.exists(filename):
        print(f"警告: 文件 {filename} 不存在")
        return []
    
    with open(filename, "r", encoding="utf-8") as f:
        data = json.load(f)
    
    items = data if isinstance(data, list) else data.get("sites", data.get("data", []))
    parsed = []

    for item in items:
        # 如果已被原项目标注 _comment 或 _baseUrl 软禁用，则跳过
        if "_comment" in item or "_baseUrl" in item:
            continue
        
        # 提取真实 API 地址
        api_url = item.get("baseUrl") or item.get("api") or item.get("url")
        if not api_url or not isinstance(api_url, str):
            continue
        
        # 过滤状态中包含失败标识的源
        status = str(item.get("status", ""))
        if any(err in status for err in ["❌", "失效", "超时", "🚨", "🚫"]):
            continue

        name = item.get("name", "未命名")
        display_name = f"[18+] {name}" if is_adult else name
        prefix = "cms_a_" if is_adult else "cms_"
        site_key = prefix + hashlib.md5(api_url.encode("utf-8")).hexdigest()[:8]

        parsed.append({
            "key": site_key,
            "name": display_name,
            "type": 1,
            "api": api_url,
            "searchable": 1,
            "quickSearch": 1,
            "filterable": 1
        })
    return parsed

def main():
    lite_sites = parse_sites("lite.json", is_adult=False)
    adult_sites = parse_sites("adult.json", is_adult=True)

    # 1. 生成纯净版 tvbox.json
    tvbox_lite = {
        "spider": "",
        "wallpaper": "https://picsum.photos/1920/1080",
        "sites": lite_sites
    }
    with open("tvbox.json", "w", encoding="utf-8") as f:
        json.dump(tvbox_lite, f, ensure_ascii=False, indent=2)

    # 2. 合并去重生成 tvbox-adult.json
    all_sites = list(lite_sites)
    seen_urls = set(s["api"] for s in lite_sites)
    for s in adult_sites:
        if s["api"] not in seen_urls:
            seen_urls.add(s["api"])
            all_sites.append(s)

    tvbox_adult = {
        "spider": "",
        "wallpaper": "https://picsum.photos/1920/1080",
        "sites": all_sites
    }
    with open("tvbox-adult.json", "w", encoding="utf-8") as f:
        json.dump(tvbox_adult, f, ensure_ascii=False, indent=2)

    print(f"生成成功: tvbox.json ({len(lite_sites)} 个)，tvbox-adult.json ({len(all_sites)} 个)")

if __name__ == "__main__":
    main()
