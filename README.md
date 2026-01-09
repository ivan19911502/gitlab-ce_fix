# GitLab Backup & Restore Guide

> Полное руководство по восстановлению GitLab из backup и решению проблем с зашифрованными токенами

[![GitLab Version](https://img.shields.io/badge/GitLab-17.6+-orange)](https://gitlab.com)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.28+-blue)](https://kubernetes.io)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

## 📋 Содержание

- [Обзор проблемы](#обзор-проблемы)
- [Симптомы](#симптомы)
- [Причина](#причина)
- [Решение: Sign-up Settings (Error 500)](#решение-sign-up-settings-error-500)
- [Решение: GitLab Runner](#решение-gitlab-runner)
- [Правильный Backup](#правильный-backup)
- [Правильное Restore](#правильное-restore)
- [Диагностика](#диагностика)
- [FAQ](#faq)

---

## 🔍 Обзор проблемы

При восстановлении GitLab из backup вы можете столкнуться с проблемой **расшифровки токенов**. Это происходит, когда:

1. ✅ База данных восстановлена из backup
2. ❌ Файл `gitlab-secrets.json` **НЕ восстановлен** или восстановлен **другой**

### Что такое `gitlab-secrets.json`?

Это файл, содержащий ключи шифрования GitLab, включая критически важный ключ **`db_key_base`**, который используется для шифрования/расшифровки токенов в базе данных.

**Путь к файлу:**
/etc/gitlab/gitlab-secrets.json

### Зашифрованные поля в `application_settings`:

| Поле | Назначение |
|------|-----------|
| `runners_registration_token` | Токен для регистрации GitLab Runner |
| `error_tracking_access_token` | Токен для error tracking |
| `encrypted_ci_jwt_signing_key` | RSA ключ для JWT подписей CI |
| `encrypted_ci_job_token_signing_key` | RSA ключ для CI job токенов |

---

## 🚨 Симптомы

### 1. Ошибка 500 в Sign-up Settings

При попытке открыть настройки регистрации пользователей:
https://your-gitlab.com/admin/application_settings/general#js-signup-settings

**Ошибка в логах:**
```log
Started PATCH "/admin/application_settings/general"
Processing by Admin::ApplicationSettingsController#general as HTML
Completed 500 Internal Server Error

OpenSSL::Cipher::CipherError ():
  lib/gitlab/crypto_helper.rb:27:in 'aes256_gcm_decrypt'
  app/models/concerns/token_authenticatable.rb:40
2. GitLab Runner не регистрируется
При попытке зарегистрировать runner:
gitlab-runner register \
  --url https://your-gitlab.com \
  --registration-token <TOKEN>

# ERROR: Registering runner... failed
# runner=xxx status=500 Internal Server Error
💡 Причина
Токены в БД зашифрованы старым db_key_base, а GitLab пытается расшифровать их новым ключом.
Схема проблемы:
┌─────────────────┐     old db_key_base      ┌──────────────┐
│   Old GitLab    │ ────────────────────────> │   Encrypted  │
│                 │      (зашифровал)         │   Token in   │
└─────────────────┘                           │   Database   │
                                              └──────────────┘
                                                      │
                                                      │ попытка расшифровки
                                                      │
                                                      ▼
┌─────────────────┐     new db_key_base      ┌──────────────┐
│   New GitLab    │ ◄──────────────────────── │   ❌ ERROR   │
│                 │   (не может расшифровать) │  CipherError │
└─────────────────┘                           └──────────────┘
🔧 Решение: Sign-up Settings (Error 500)
Проблемные токены:
encrypted_ci_jwt_signing_key
encrypted_ci_job_token_signing_key
Вариант 1: Не использую CI/CD
Просто удалите токены (валидаторы пропустят их, если они NULL):
# Подключаемся к БД
kubectl exec -it -n gitlab gitlab-0 -- gitlab-rails dbconsole
UPDATE application_settings 
SET 
  encrypted_ci_jwt_signing_key = NULL,
  encrypted_ci_jwt_signing_key_iv = NULL,
  encrypted_ci_job_token_signing_key = NULL,
  encrypted_ci_job_token_signing_key_iv = NULL
WHERE id = 1;

-- Проверка
SELECT 
  encrypted_ci_jwt_signing_key IS NOT NULL as ci_jwt,
  encrypted_ci_job_token_signing_key IS NOT NULL as ci_job
FROM application_settings WHERE id = 1;

\q
# Перезапуск Puma
kubectl exec -n gitlab gitlab-0 -- gitlab-ctl restart puma
✅ Готово! Sign-up Settings должны открыться без ошибок.
Вариант 2: Использую CI/CD
Удалите старые и создайте новые токены:
Шаг 1: Удалите старые токены (SQL выше)
Шаг 2: Перезапустите Puma
kubectl exec -n gitlab gitlab-0 -- gitlab-ctl restart puma
sleep 30
Шаг 3: Создайте новые токены
kubectl exec -it -n gitlab gitlab-0 -- gitlab-rails runner '
require "openssl"

puts "=== Создание CI токенов ==="

setting = ApplicationSetting.current

# Генерируем RSA ключи (2048 бит)
jwt_key = OpenSSL::PKey::RSA.new(2048)
job_key = OpenSSL::PKey::RSA.new(2048)

# Сохраняем (GitLab автоматически зашифрует текущим db_key_base)
setting.ci_jwt_signing_key = jwt_key.to_pem
setting.ci_job_token_signing_key = job_key.to_pem

if setting.save(validate: false)
  puts "✅ Токены созданы успешно!"
  
  # Проверка
  setting.reload
  puts "✅ ci_jwt_signing_key: #{setting.ci_jwt_signing_key.bytesize} bytes"
  puts "✅ ci_job_token_signing_key: #{setting.ci_job_token_signing_key.bytesize} bytes"
  
  # Тест валидации
  setting.signup_enabled = false
  if setting.valid?
    puts "✅ Валидация прошла успешно!"
  end
else
  puts "❌ Ошибка сохранения"
  setting.errors.full_messages.each { |e| puts "  - #{e}" }
end
'
✅ Проверка: Откройте Sign-up Settings - должна работать без ошибок!
🏃 Решение: GitLab Runner
Проблемный токен:
runners_registration_token_encrypted
Полное решение:
Шаг 1: Проверка токена
kubectl exec -it -n gitlab gitlab-0 -- gitlab-rails runner '
setting = ApplicationSetting.current

enc_value = setting.runners_registration_token_encrypted

if enc_value.present?
  puts "Токен существует: #{enc_value.bytesize} bytes"
  
  begin
    token = setting.runners_registration_token
    puts "✅ Токен читается: #{token[0..10]}..."
  rescue OpenSSL::Cipher::CipherError
    puts "❌ Токен сломан (зашифрован старым ключом)"
  end
else
  puts "⚪ Токен отсутствует"
end
'
Шаг 2: Удаление старого токена
kubectl exec -it -n gitlab gitlab-0 -- gitlab-rails dbconsole
UPDATE application_settings 
SET runners_registration_token_encrypted = NULL 
WHERE id = 1;

\q
Шаг 3: Перезапуск GitLab
kubectl exec -n gitlab gitlab-0 -- gitlab-ctl restart puma
sleep 30
Шаг 4: Создание нового токена
kubectl exec -it -n gitlab gitlab-0 -- gitlab-rails runner '
require "securerandom"

setting = ApplicationSetting.current

# Генерируем новый токен (20 символов)
new_token = Devise.friendly_token(20)

puts "Новый токен: #{new_token}"

# Сохраняем (GitLab автоматически зашифрует)
setting.runners_registration_token = new_token

if setting.save(validate: false)
  puts "✅ Токен сохранён и зашифрован!"
  
  # Проверка
  setting.reload
  decrypted = setting.runners_registration_token
  puts "✅ Проверка: #{decrypted[0..10]}..."
else
  puts "❌ Ошибка сохранения"
end
'
Шаг 5: Получение токена для регистрации
Способ 1: Через веб-интерфейс (рекомендуется)
Откройте Admin Area → CI/CD → Runners
Нажмите "Register an instance runner"
Скопируйте registration token
Способ 2: Из консоли
kubectl exec -it -n gitlab gitlab-0 -- gitlab-rails runner '
puts ApplicationSetting.current.runners_registration_token
'
Шаг 6: Регистрация Runner
gitlab-runner register \
  --url https://your-gitlab.com \
  --registration-token <НОВЫЙ_ТОКЕН> \
  --executor docker \
  --description "My Docker Runner" \
  --docker-image "alpine:latest" \
  --docker-privileged
✅ Проверка: Runner должен появиться в Admin Area → Runners
📦 Правильный Backup
Что нужно включить в backup:
Компонент	Команда	Важность
База данных	gitlab-rake gitlab:backup:create	🔴 Критично
gitlab-secrets.json	/etc/gitlab/gitlab-secrets.json	🔴 Критично
gitlab.rb	/etc/gitlab/gitlab.rb	🟡 Важно
SSL сертификаты	/etc/gitlab/ssl/	🟢 Опционально
Скрипт полного backup:
#!/bin/bash
# Полный backup GitLab для Kubernetes

BACKUP_DIR="/backup/gitlab/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

echo "🔵 Создание backup GitLab..."

# 1. Backup БД
echo "1/4 Создание backup базы данных..."
kubectl exec -n gitlab gitlab-0 -- gitlab-rake gitlab:backup:create SKIP=uploads,builds,artifacts,lfs,registry

# 2. Копируем gitlab-secrets.json (КРИТИЧНО!)
echo "2/4 Копирование gitlab-secrets.json..."
kubectl cp gitlab/gitlab-0:/etc/gitlab/gitlab-secrets.json \
  "$BACKUP_DIR/gitlab-secrets.json"

# 3. Копируем gitlab.rb
echo "3/4 Копирование gitlab.rb..."
kubectl cp gitlab/gitlab-0:/etc/gitlab/gitlab.rb \
  "$BACKUP_DIR/gitlab.rb"

# 4. Копируем backup файлы БД
echo "4/4 Копирование backup файлов..."
BACKUP_FILE=$(kubectl exec -n gitlab gitlab-0 -- ls -t /var/opt/gitlab/backups/ | head -1)
kubectl cp gitlab/gitlab-0:/var/opt/gitlab/backups/$BACKUP_FILE \
  "$BACKUP_DIR/$BACKUP_FILE"

echo "✅ Backup завершён: $BACKUP_DIR"
echo "📋 Содержимое:"
ls -lh "$BACKUP_DIR"
🔄 Правильное Restore
⚠️ ВАЖНО: Порядок действий критичен!
Шаг 1: Восстановить gitlab-secrets.json ПЕРВЫМ
# Копируем secrets ДО восстановления БД!
kubectl cp gitlab-secrets.json gitlab/gitlab-0:/etc/gitlab/gitlab-secrets.json

# Применяем конфигурацию
kubectl exec -n gitlab gitlab-0 -- gitlab-ctl reconfigure

# Проверяем, что файл на месте
kubectl exec -n gitlab gitlab-0 -- cat /etc/gitlab/gitlab-secrets.json | grep db_key_base
Шаг 2: Восстановить конфигурацию (опционально)
kubectl cp gitlab.rb gitlab/gitlab-0:/etc/gitlab/gitlab.rb
kubectl exec -n gitlab gitlab-0 -- gitlab-ctl reconfigure
Шаг 3: Восстановить БД
# Копируем backup файл
kubectl cp 1234567890_2024_01_09_17.6.2_gitlab_backup.tar \
  gitlab/gitlab-0:/var/opt/gitlab/backups/

# Останавливаем процессы
kubectl exec -n gitlab gitlab-0 -- gitlab-ctl stop puma
kubectl exec -n gitlab gitlab-0 -- gitlab-ctl stop sidekiq

# Восстанавливаем (укажите timestamp из имени файла)
kubectl exec -n gitlab gitlab-0 -- \
  gitlab-rake gitlab:backup:restore BACKUP=1234567890_2024_01_09_17.6.2

# Перезапускаем GitLab
kubectl exec -n gitlab gitlab-0 -- gitlab-ctl restart
Шаг 4: Проверка после восстановления
# Проверка GitLab
kubectl exec -n gitlab gitlab-0 -- gitlab-rake gitlab:check

# Проверка токенов
kubectl exec -it -n gitlab gitlab-0 -- gitlab-rails runner '
setting = ApplicationSetting.current

tokens = {
  "Runner Token" => :runners_registration_token,
  "Error Tracking" => :error_tracking_access_token,
  "CI JWT Key" => :ci_jwt_signing_key,
  "CI Job Token Key" => :ci_job_token_signing_key
}

puts "=== Проверка токенов ==="
tokens.each do |name, token|
  begin
    value = setting.send(token)
    if value.present?
      puts "✅ #{name}: OK (#{value.bytesize} bytes)"
    else
      puts "⚪ #{name}: NULL"
    end
  rescue OpenSSL::Cipher::CipherError
    puts "❌ #{name}: BROKEN!"
  end
end
'
🔍 Диагностика
Команда для поиска всех сломанных токенов:
kubectl exec -it -n gitlab gitlab-0 -- gitlab-rails runner '
puts "=== Диагностика зашифрованных токенов ==="

setting = ApplicationSetting.current
broken = []
ok = []

ApplicationSetting.column_names.select { |c| c.end_with?("_encrypted") }.each do |enc_col|
  value = setting.send(enc_col)
  
  next if value.blank?
  
  attr = enc_col.gsub("_encrypted", "")
  
  begin
    setting.send(attr)
    ok << attr
    puts "✅ #{attr}: OK"
  rescue OpenSSL::Cipher::CipherError
    broken << enc_col
    puts "❌ #{attr}: BROKEN (CipherError)"
  rescue => e
    puts "⚠️  #{attr}: #{e.class.name}"
  end
end

puts "\n" + "="*60
puts "Итого:"
puts "  ✅ Рабочих: #{ok.size}"
puts "  ❌ Сломанных: #{broken.size}"

if broken.any?
  puts "\n🔧 SQL для исправления:"
  puts "="*60
  sql_sets = broken.map { |f| "#{f} = NULL" }.join(",\n  ")
  puts "UPDATE application_settings"
  puts "SET"
  puts "  #{sql_sets}"
  puts "WHERE id = 1;"
end
'
Проверка конкретного токена:
# Проверка runners_registration_token
kubectl exec -it -n gitlab gitlab-0 -- gitlab-rails runner '
setting = ApplicationSetting.current

begin
  token = setting.runners_registration_token
  puts "✅ Token: #{token[0..20]}..."
rescue OpenSSL::Cipher::CipherError
  puts "❌ Token encrypted with old key!"
end
'
❓ FAQ
Q: Можно ли восстановить токены без gitlab-secrets.json?
A: Нет, невозможно. Токены зашифрованы AES-256-GCM и без оригинального db_key_base их не расшифровать. Единственное решение - удалить старые токены и создать новые.
Q: Что делать, если после восстановления все токены сломаны?
A: Это значит, что gitlab-secrets.json не был восстановлен или восстановлен неправильный файл. Решение:
Проверьте, что secrets восстановлен:
kubectl exec -n gitlab gitlab-0 -- cat /etc/gitlab/gitlab-secrets.json | grep db_key_base
Если secrets правильный, но токены не читаются - удалите все и пересоздайте (используйте команду диагностики выше).
Q: GitLab работает, но CI/CD pipeline падают с ошибкой аутентификации
A: Проверьте токены CI:
kubectl exec -it -n gitlab gitlab-0 -- gitlab-rails runner '
s = ApplicationSetting.current
