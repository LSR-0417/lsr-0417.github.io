IPFS HLS Test

測試連上自己的 ipfs node 的 HLS 切片影片進行串流

`npx servve` 啟動伺服器

## 完整系統環境

| 名稱         | 用途                                       |
| ------------ | ------------------------------------------ |
| yt-dlp       | 影片備份軟體，完整下載包含影片與 meta data |
| ffmpeg       | 影片轉檔軟體，轉為數種解析度的串流格式     |
| ipfs desktop | 加速器                            |
| node.js      | 執行網頁伺服器                             |

## 下載影片

使用 yt-dlp 備份影片，指令細節說明如下：

```bash
yt-dlp \
  --write-subs \
  --all-subs \
  --sub-format "vtt" \
  --write-thumbnail \
  --write-info-json \
  --no-write-comments \
  --write-description \
  --merge-output-format mp4 \
  -o "my_archive/%(title)s.%(ext)s" \
  "你的YouTube網址"
```

參數功能說明：

- `--write-subs --all-subs`：精準命中你的需求，只抓取「創作者手動上傳」的所有語言正式字幕。
- `--sub-format "vtt"`：強制將字幕轉為我們網頁播放器需要的 WebVTT 格式。
- `--write-thumbnail`：下載最高畫質的影片封面縮圖。
- `--write-info-json`：下載包含影片標題、標籤、章節等中介資料的終極 JSON 檔。
- `--no-write-comments`：（關鍵防護） 強制排除所有觀眾留言，讓你的 JSON 檔案體積大幅縮小，前端網頁讀取時會變成「秒開」。
- `--write-description`：額外獨立存一份純文字的說明欄檔案。
- `--merge-output-format mp4`：確保最高畫質的影像與聲音下載後，會封裝成最標準好用的 .mp4 檔案。
- `-o "my_archive/%(title)s.%(ext)s"`：將所有產生的檔案統一丟進 my_archive 資料夾，並以影片的原始標題作為檔名。

資料夾樹狀圖

```plantext
my_archive/
  ├── 影片標題.mp4          (影片與聲音本體)
  ├── 影片標題.webp         (高畫質縮圖)
  ├── 影片標題.info.json    (包含留言、觀看數、章節的終極資料庫)
  ├── 影片標題.description  (純文字的說明欄)
  ├── 影片標題.zh-TW.vtt    (繁中字幕)
  └── 影片標題.ja.vtt       (日文字幕)
```

## 影片轉檔

第一步：預先建立 5 個資料夾

首先，準備好用來存放這 5 種畫質切片的資料夾結構（0 是 4K，4 是 480p）：

```bash
mkdir -p my_4k_abr/stream_0 my_4k_abr/stream_1 my_4k_abr/stream_2 my_4k_abr/stream_3 my_4k_abr/stream_4
```

第二步：執行 4K 全解析度 ABR 轉檔指令

請把 temp.mp4 換成你的 4K 原始影片檔案。這段轉檔會非常吃 CPU 和時間，執行後可以去喝杯咖啡：

```bash
ffmpeg -i temp.mp4 \
  -map 0:v:0 -map 0:a:0 \
  -map 0:v:0 -map 0:a:0 \
  -map 0:v:0 -map 0:a:0 \
  -map 0:v:0 -map 0:a:0 \
  -map 0:v:0 -map 0:a:0 \
  -c:v libx264 -c:a aac \
  -filter:v:0 scale=-2:2160 -b:v:0 15000k -maxrate:v:0 16000k -bufsize:v:0 30000k \
  -filter:v:1 scale=-2:1440 -b:v:1 8000k  -maxrate:v:1 8600k  -bufsize:v:1 16000k \
  -filter:v:2 scale=-2:1080 -b:v:2 5000k  -maxrate:v:2 5300k  -bufsize:v:2 10000k \
  -filter:v:3 scale=-2:720  -b:v:3 2800k  -maxrate:v:3 3000k  -bufsize:v:3 5600k \
  -filter:v:4 scale=-2:480  -b:v:4 1400k  -maxrate:v:4 1500k  -bufsize:v:4 2800k \
  -f hls \
  -hls_time 5 \
  -hls_playlist_type vod \
  -hls_segment_filename "my_4k_abr/stream_%v/segment_%03d.ts" \
  -master_pl_name master.m3u8 \
  -var_stream_map "v:0,a:0 v:1,a:1 v:2,a:2 v:3,a:3 v:4,a:4" \
  "my_4k_abr/stream_%v/playlist.m3u8"
```
