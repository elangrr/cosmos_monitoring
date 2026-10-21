# secrets/

Put `telegram_bot_token` here (one line, just the token) **before** the first `docker compose up`. This folder is git-ignored.

```bash
echo "<your bot token>" > secrets/telegram_bot_token
chmod 600 secrets/telegram_bot_token
sudo chown 65534:65534 secrets/telegram_bot_token   # Alertmanager runs as nobody (65534)
```
