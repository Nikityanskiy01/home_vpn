# 6. Публикация на GitHub

## Вариант без команд

1. На GitHub нажмите **New repository**.
2. Назовите его, например, `homelab-vpn-gateway`, выберите Private (или Public, только если проверили отсутствие секретов).
3. Не создавайте README на GitHub: он уже есть в этой папке.
4. Нажмите **Add file → Upload files**, перетащите содержимое папки `vpn-gateway-homelab`, затем **Commit changes**.

## Вариант через Git

В PowerShell, находясь в папке репозитория:

```powershell
git init
git add .
git status
git commit -m "Add Russian HomeLab VPN gateway guide"
git branch -M main
git remote add origin https://github.com/YOUR_GITHUB_LOGIN/homelab-vpn-gateway.git
git push -u origin main
```

Если GitHub предложит авторизацию, завершите её в браузере. Перед `git add` убедитесь, что рядом нет экспортов конфигов, `.env`, backup или ключей; `.gitignore` — помощь, а не гарантия.
