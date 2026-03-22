---
description: Plan dosyasındaki task'ları kanban board'a otomatik yükle ve browser'ı aç
argument-hint: "[plan-dosyası (opsiyonel)]"
allowed-tools:
  - Read
  - Bash(curl:*)
  - Bash(ls:*)
  - Bash(find:*)
  - Bash(start:*)
  - Bash(open:*)
  - Bash(xdg-open:*)
  - Bash(uname:*)
---

Planlama çıktısındaki task'ları kanban board'a toplu olarak yükle. Server kapalıysa otomatik başlat.

## Adımlar

### Adım 1: Plan dosyasını belirle

Argüman verilmişse o dosyayı kullan.

Argüman verilmemişse şu sırayla en son değiştirilen `.md` dosyasını bul:

1. `briefs/*.md`
2. `proposals/*.md`
3. `launches/*.md`

Hiçbiri yoksa kullanıcıya sor: "Hangi plan dosyasını kullanayım?"

Eğer kullanıcı dosya vermek yerine "bunları ekle" diye konuşmada task listesi verdiyse — konuşma çıktısından parse et, dosyaya gerek yok.

### Adım 2: Kanban server kontrolü

```bash
curl -s --max-time 3 http://localhost:2903/api/board
```

- **200 gelirse:** devam et
- **Hata / timeout:** `.claude/skills/kanban-server/SKILL.md` skill'ini çağır — server otomatik başlatılır. Skill yoksa önce yarat, ardından çağır.

### Adım 3: Task'ları parse et

**Kalfa `/brief` formatı** (`briefs/*.md`) — MoSCoW bölümlerine göre priority:

| Bölüm başlığı | priority | columnKey |
|---------------|----------|-----------|
| `Olmazsa Olmaz` / `Must Have` | `"high"` | `"task"` |
| `Olmalı` / `Should Have` | `"medium"` | `"todo"` |
| `Olabilir` / `Could Have` | `"low"` | `"todo"` |
| `Olmayacak` / `Won't Have` | atla | — |

Her bölümdeki `- ` veya `* ` ile başlayan satırlar → task `text`
Alt girintili satırlar veya kabul kriterleri (`- [ ]`) → task `note` olarak birleştir

**Genel markdown fallback** (diğer dosyalar):

```
- [ ] Task adı     → text, columnKey: "todo", priority: null
- Task adı         → text, columnKey: "todo", priority: null
* Task adı         → text, columnKey: "todo", priority: null
```

**Konuşma çıktısından parse** (dosya yoksa):
Kullanıcının verdiği veya Claude'un ürettiği task listesindeki maddeleri doğrudan al.

### Adım 4: Mevcut task'ları çek — duplicate önleme

```bash
curl -s http://localhost:2903/api/board
```

Board'daki mevcut `task.text` değerlerini listele. Parse edilen task'larla karşılaştır:
- Aynı text varsa → atla, logla: `⏭ Zaten mevcut: [task adı]`
- Yeni task ise → Adım 5'e geç

### Adım 5: Her yeni task'ı POST et

```bash
curl -s -X POST http://localhost:2903/api/tasks \
  -H "Content-Type: application/json" \
  -d '{
    "text": "<task adı>",
    "columnKey": "<task|todo>",
    "priority": "<high|medium|low|null>",
    "note": "<açıklama veya kabul kriterleri>"
  }'
```

Her başarılı yanıt için logla: `✅ Eklendi: [task adı]`
Hata yanıtı için logla: `❌ Hata: [task adı] — [hata mesajı]`

### Adım 6: Browser'ı aç

```bash
# Platform tespiti
case "$(uname -s 2>/dev/null || echo Windows)" in
  Darwin)  open http://localhost:2903 ;;
  Linux)   xdg-open http://localhost:2903 ;;
  *)       start http://localhost:2903 ;;
esac
```

### Adım 7: Özet

Kullanıcıya sonucu göster:

```text
📋 Kanban Push Tamamlandı
  ✅ Eklendi  : 8 task
  ⏭ Atlandı  : 2 task (zaten mevcut)
  ❌ Hata     : 0 task
  🌐 Board    : http://localhost:2903
```
