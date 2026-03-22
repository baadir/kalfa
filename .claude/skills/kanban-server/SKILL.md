# Skill: kanban-server

Kanban board server'ını kontrol et, kapalıysa otomatik başlat.

## Tetikleyici

`/kanban-push` tarafından çağrılır. Ayrıca doğrudan `/kanban-server` olarak da çalıştırılabilir.

## Araçlar

- `Bash(curl:*)`
- `Bash(npm:*)`
- `Bash(nohup:*)` / `Bash(start:*)`
- `Bash(uname:*)`
- `Bash(sleep:*)`

## Adımlar

### Adım 1: Port kontrolü

```bash
curl -s --max-time 2 http://localhost:2903/api/board
```

- **HTTP 200 gelirse:** `✅ Kanban server zaten çalışıyor (port 2903).` → çık
- **Hata veya timeout:** Adım 2'ye geç

### Adım 2: Server'ı arka planda başlat

`$CLAUDE_PROJECT_DIR` veya mevcut proje kökünde `kanban/` dizini var mı kontrol et.

**Unix/macOS:**
```bash
cd "$CLAUDE_PROJECT_DIR" && nohup npm run kanban > /tmp/kanban-server.log 2>&1 &
```

**Windows (Git Bash):**
```bash
cd "$CLAUDE_PROJECT_DIR" && start //B npm run kanban
```

### Adım 3: Hazır olmasını bekle (max 10 saniye)

1 saniyelik aralıklarla `GET /api/board` isteği gönder:

```bash
for i in 1 2 3 4 5 6 7 8 9 10; do
  sleep 1
  curl -s --max-time 1 http://localhost:2903/api/board && echo "READY" && break
done
```

- **"READY" gelirse:** `✅ Kanban server başlatıldı (port 2903).` → çık
- **10 saniye geçerse:** `❌ Server başlatılamadı. Log: /tmp/kanban-server.log` → hata ile çık

## Çıktı

Başarı: `✅ Kanban server [zaten çalışıyor | başlatıldı] (port 2903).`
Hata: `❌ Server başlatılamadı. [hata detayı]`
