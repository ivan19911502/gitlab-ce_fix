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
