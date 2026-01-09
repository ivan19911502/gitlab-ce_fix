# GitLab Backup & Restore Guide

> Полное руководство по восстановлению GitLab из backup и решению проблем с зашифрованными токенами

[![GitLab Version](https://img.shields.io/badge/GitLab-17.6+-orange)](https://gitlab.com)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.28+-blue)](https://kubernetes.io)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

## 📋 Содержание

- [Обзор проблемы](#обзор-проблемы)
- [Симптомы](#симптомы)
- [Диагностика: Поиск проблемных токенов](#диагностика-поиск-проблемных-токенов)
- [Решение: Sign-up Settings (Error 500)](#решение-sign-up-settings-error-500)
- [Решение: GitLab Runner](#решение-gitlab-runner)
- [Проверка после исправления](#проверка-после-исправления)
- [Правильный Backup](#правильный-backup)
- [Правильное Restore](#правильное-restore)
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

| Поле | Назначение | Проблема |
|------|-----------|----------|
| `encrypted_ci_jwt_signing_key` | RSA ключ для JWT подписей CI | Sign-up Settings Error 500 |
| `encrypted_ci_job_token_signing_key` | RSA ключ для CI job токенов | Sign-up Settings Error 500 |
| `runners_registration_token_encrypted` | Токен для регистрации Runner | Runner не регистрируется |

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
```

2. GitLab Runner не регистрируется
   
При попытке зарегистрировать runner:
```
gitlab-runner register \
  --url https://your-gitlab.com \
  --registration-token <TOKEN>

# ERROR: Registering runner... failed
# runner=xxx status=500 Internal Server Error
```

💡 Причина

Токены в БД зашифрованы старым db_key_base, а GitLab пытается расшифровать их новым ключом.

Схема проблемы:
```
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
```

🔍 Диагностика: Поиск проблемных токенов

Команда 1: Найти ВСЕ сломанные токены

Эта команда проверит все зашифрованные поля и покажет, какие из них сломаны:
```
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
```
Пример вывода:

```
=== Диагностика зашифрованных токенов ===
✅ runners_registration_token: OK
✅ error_tracking_access_token: OK
❌ ci_jwt_signing_key: BROKEN (CipherError)
❌ ci_job_token_signing_key: BROKEN (CipherError)

============================================================
Итого:
  ✅ Рабочих: 2
  ❌ Сломанных: 2

🔧 SQL для исправления:
============================================================
UPDATE application_settings
SET
  encrypted_ci_jwt_signing_key = NULL,
  encrypted_ci_job_token_signing_key = NULL
WHERE id = 1;
```

Команда 2: Найти проблемный валидатор
Эта команда покажет, какой именно валидатор падает при проверке:
```
kubectl exec -it -n gitlab gitlab-0 -- gitlab-rails runner '
setting = ApplicationSetting.current
setting.signup_enabled = false

puts "=== Тестирование валидаторов ==="

ApplicationSetting.validators.each_with_index do |v, i|
  validator_name = v.class.name
  
  print "#{i+1}. #{validator_name}... "
  
  begin
    v.validate(setting)
    puts "OK"
  rescue OpenSSL::Cipher::CipherError => e
    puts "❌ CIPHER ERROR!"
    puts "\n🚨 НАЙДЕН ПРОБЛЕМНЫЙ ВАЛИДАТОР!"
    puts "Validator: #{v.class.name}"
    puts "Options: #{v.options.inspect}"
    
    if v.respond_to?(:attributes)
      puts "Attributes: #{v.attributes.inspect}"
    end
    
    break
  rescue => e
    puts "Error: #{e.class.name}"
  end
end
'
```
Пример вывода:
```
=== Тестирование валидаторов ===
1. ActiveModel::BlockValidator... OK
2. DurationValidator... OK
...
134. RsaKeyValidator... OK
135. RsaKeyValidator... ❌ CIPHER ERROR!

🚨 НАЙДЕН ПРОБЛЕМНЫЙ ВАЛИДАТОР!
Validator: RsaKeyValidator
Options: {:allow_nil=>true}
Attributes: [:ci_job_token_signing_key]
```

🔧 Решение: Sign-up Settings (Error 500)
Проблемные токены для CI/CD:

`encrypted_ci_jwt_signing_key`
`encrypted_ci_job_token_signing_key`
Шаг 1: Удалите старые токены
```
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
```

Ожидаемый результат:

```
ci_jwt | ci_job 
--------+--------
 f      | f
```

Шаг 2: Перезапустите Puma

```
kubectl exec -n gitlab gitlab-0 -- gitlab-ctl restart puma
```

Шаг 3: Создайте новые токены

```
kubectl exec -it -n gitlab gitlab-0 -- gitlab-rails runner '
require "openssl"

puts "=== Создание CI/CD токенов ==="

setting = ApplicationSetting.current

# Генерируем RSA ключи (2048 бит)
puts "1. Создание ci_jwt_signing_key..."
jwt_key = OpenSSL::PKey::RSA.new(2048)

puts "2. Создание ci_job_token_signing_key..."
job_key = OpenSSL::PKey::RSA.new(2048)

# Сохраняем (GitLab автоматически зашифрует текущим db_key_base)
setting.ci_jwt_signing_key = jwt_key.to_pem
setting.ci_job_token_signing_key = job_key.to_pem

puts "\nСохранение..."

if setting.save(validate: false)
  puts "✅ Токены сохранены!"
  
  # Проверка
  setting.reload
  puts "\nПроверка:"
  puts "✅ ci_jwt_signing_key: #{setting.ci_jwt_signing_key.bytesize} bytes"
  puts "✅ ci_job_token_signing_key: #{setting.ci_job_token_signing_key.bytesize} bytes"
  
  # Тест валидации
  puts "\nТестирование валидации..."
  setting.signup_enabled = false
  
  if setting.valid?
    puts "✅ Валидация PASSED!"
    
    if setting.save
      puts "✅ Сохранение SUCCESSFUL!"
      puts "\n🎉 CI/CD токены готовы! Sign-up Settings работают!"
    end
  else
    puts "❌ Валидация всё ещё падает"
    puts "Ошибки:"
    setting.errors.full_messages.each { |e| puts "  - #{e}" }
  end
else
  puts "❌ Ошибка сохранения"
  setting.errors.full_messages.each { |e| puts "  - #{e}" }
end
'
```
Ожидаемый вывод:
```
=== Создание CI/CD токенов ===
1. Создание ci_jwt_signing_key...
2. Создание ci_job_token_signing_key...

Сохранение...
✅ Токены сохранены!

Проверка:
✅ ci_jwt_signing_key: 1679 bytes
✅ ci_job_token_signing_key: 1679 bytes

Тестирование валидации...
✅ Валидация PASSED!
✅ Сохранение SUCCESSFUL!

🎉 CI/CD токены готовы! Sign-up Settings работают!
```
Шаг 4: Перезапустите Puma
```
kubectl exec -n gitlab gitlab-0 -- gitlab-ctl restart puma
```


🏃 Решение: GitLab Runner

Проблемный токен:

`runners_registration_token_encrypted`

Шаг 1: Проверка токена

```
kubectl exec -it -n gitlab gitlab-0 -- gitlab-rails runner '
setting = ApplicationSetting.current

puts "=== Проверка runners_registration_token ==="

enc_value = setting.runners_registration_token_encrypted

if enc_value.present?
  puts "Токен существует: #{enc_value.bytesize} bytes"
  
  begin
    token = setting.runners_registration_token
    puts "✅ Токен читается: #{token[0..10]}..."
  rescue OpenSSL::Cipher::CipherError
    puts "❌ Токен сломан (зашифрован старым ключом)"
    puts "\n🚨 Необходимо пересоздать токен!"
  end
else
  puts "⚪ Токен отсутствует - необходимо создать"
end
'
```
Шаг 2: Удаление старого токена
```
kubectl exec -it -n gitlab gitlab-0 -- gitlab-rails dbconsole
UPDATE application_settings 
SET runners_registration_token_encrypted = NULL 
WHERE id = 1;

-- Проверка
SELECT runners_registration_token_encrypted IS NOT NULL as has_token
FROM application_settings WHERE id = 1;
```
Ожидаемый результат:
```
 has_token 
-----------
 f
 ```
Шаг 3: Перезапуск GitLab
```
kubectl exec -n gitlab gitlab-0 -- gitlab-ctl restart puma
```

Шаг 4: Создание нового токена
```
kubectl exec -it -n gitlab gitlab-0 -- gitlab-rails runner '
require "securerandom"

setting = ApplicationSetting.current

# Генерируем новый токен (20 символов)
new_token = Devise.friendly_token(20)

puts "=== Создание токена для Runner ==="
puts "Новый токен: #{new_token}"

# Сохраняем (GitLab автоматически зашифрует текущим db_key_base)
setting.runners_registration_token = new_token

if setting.save(validate: false)
  puts "\n✅ Токен сохранён и зашифрован!"
  
  # Проверка
  setting.reload
  decrypted = setting.runners_registration_token
  puts "✅ Проверка расшифровки: #{decrypted[0..10]}..."
  
  puts "\n" + "="*60
  puts "📋 Токен для регистрации Runner:"
  puts "="*60
  puts decrypted
  puts "="*60
  
  puts "\n💡 Используйте этот токен для регистрации Runner"
else
  puts "❌ Ошибка сохранения"
  setting.errors.full_messages.each { |e| puts "  - #{e}" }
end
'
```

Ожидаемый вывод:
```
=== Создание токена для Runner ===
Новый токен: zXyAbC123dEfGhIjK456

✅ Токен сохранён и зашифрован!
✅ Проверка расшифровки: zXyAbC123d...

============================================================
📋 Токен для регистрации Runner:
============================================================
zXyAbC123dEfGhIjK456
============================================================

💡 Используйте этот токен для регистрации Runner
```

