# AGENTS.md: claude-antigravity-agents

Bu depo Mark Fulton'ın `markfulton/claude-antigravity-agents` projesinin Kuzey Tarhun tarafından çatallanmış (fork) ve geliştirilmiş sürümüdür.

## Kararlar ve Notlar

[Antigravity, 3 Kas 2026]
- Repo GitHub üzerinde public olarak `ktarxhun/claude-antigravity-agents` adıyla çatallandı.
- Orijinal projedeki eksiklikler giderildi:
  1. Düşünme seviyesi yönetimi eklendi: `--effort low|medium|high|xhigh|max`.
  2. Model havuzu güncellendi: `gemini-3.8-flash-high`, `gemini-4-argon-high` ve modern model slug'ları entegre edildi.
  3. Çok turlu oturum devamlılığı eklendi: Belirsiz `--continue` yerine deterministik `--conversation <id>` bağlandı.
  4. Çalışma modları ayrıldı: `--mode plan` (salt okunur mimari) ve `--mode accept-edits` (doğrudan kod yazımı).
  5. Sıfır bağımlılıklı Python çalıştırıcısı (`bin/agy-delegate`) yazıldı, `~/.local/bin/agy-delegate` altına bağlandı.
  6. Otomatik kota denetimi ve geri dönüş (fallback) eklendi: 429/kota aşımında çıkış kodu 42 döner; küçük işlerde Claude kendi içinde çözer, büyük işlerde kullanıcıya sorar.
  7. Proaktif refleks: Skill tanımı rutin frontend ve amelelik işlerinde Claude'un kullanıcı komutunu beklemeden kendiliğinden devreye girmesi üzerine güncellendi.
  8. Skill yerel ortama kuruldu: `~/.claude/skills/antigravity-agents/SKILL.md`.
  9. Tüm dokümantasyondaki em dash (uzun çizgi) karakterleri temizlendi.
