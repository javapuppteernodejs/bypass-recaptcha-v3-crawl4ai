# Bypassing reCAPTCHA v3 with CapSolver in Crawl4AI Workflows

## Introduction

reCAPTCHA v3 is a background verification mechanism that scores user behavior to differentiate humans from bots. This guide details integrating CapSolver with Crawl4AI to efficiently solve reCAPTCHA v3 for web scraping and automation.

## reCAPTCHA v3 Technical Overview

Unlike v2, reCAPTCHA v3 operates invisibly, analyzing user interactions (mouse movements, scrolling, IP, browser fingerprint) to generate a risk score (0.0-1.0). Low scores can lead to blocked requests. Traditional automation struggles to mimic human behavior for high scores.

### Crawl4AI & CapSolver Synergy

**Crawl4AI** [^1] offers advanced browser control and stealth features, crucial for simulating human behavior. Its `js_code` injection facilitates token acquisition.

**CapSolver** [^2] provides high-success-rate reCAPTCHA v3 solutions with fast response times and easy API integration.

Combining these tools enables a robust reCAPTCHA v3 bypass strategy.

### reCAPTCHA v3 Workflow

Google's JavaScript embeds on a page, collects user data, and sends it for analysis, returning a score and `gRecaptchaResponse` token. This token is sent to the backend for validation; low scores result in rejection.

## Integration Guide

This guide details solving reCAPTCHA v3 with CapSolver in Crawl4AI.

### 1. Prerequisites

*   **CapSolver Account & API Key**: Register at [CapSolver](https://www.capsolver.com/?utm_source=github_tutorial&utm_medium=referral&utm_campaign=crawl4ai_recaptcha_v3) and obtain your API Key.
*   **Install Crawl4AI**:
    ```bash
    pip install crawl4ai
    ```
*   **Install CapSolver Python SDK**:
    ```bash
    pip install capsolver
    ```

### 2. Identify reCAPTCHA v3 Parameters

Locate the reCAPTCHA v3 `siteKey` (e.g., `data-sitekey` attribute) and `websiteURL`. Also, identify the `action` parameter (e.g., `homepage`, `login`, `submit`) specific to the user interaction.

Example: `https://2captcha.com/demo/recaptcha-v3` has `siteKey`: `6LfB5_IbAAAAAMCtsjEHEHKqcB9iQocwwxTiihJu`, `action`: `homepage`.

### 3. Crawl4AI Integration Code

```python
import asyncio
from crawl4ai import AsyncWebCrawler, CrawlerRunConfig
import capsolver

# Configure CapSolver API Key
capsolver.api_key = "YOUR_CAPSOLVER_API_KEY"

async def solve_recaptcha_v3_with_crawl4ai(target_url: str, site_key: str, action: str = "verify"):
    print(f"Attempting to solve reCAPTCHA v3 on {target_url} (action: {action})...")

    # 1. Create reCAPTCHA v3 task using CapSolver API
    # Docs: https://docs.capsolver.com/en/guide/captcha/ReCaptchaV3/
    try:
        solution = await capsolver.solve({
            "type": "ReCaptchaV3TaskProxyLess",
            "websiteURL": target_url,
            "websiteKey": site_key,
            "pageAction": action,
            "minScore": 0.3
        })
        recaptcha_token = solution["gRecaptchaResponse"]
        print(f"CapSolver obtained reCAPTCHA v3 token: {recaptcha_token[:30]}...")
    except Exception as e:
        print(f"CapSolver reCAPTCHA v3 resolution failed: {e}")
        return None

    # 2. Prepare JavaScript to inject the token
    js_injection_code = f"""
        if (typeof grecaptcha !== \'undefined\' && grecaptcha.execute) {
            grecaptcha.ready(function() {
                grecaptcha.execute(\\'{site_key}\\', {{action: \'{action}\\'}}).then(function(token) {
                    var responseElements = document.querySelectorAll(\\'[name="g-recaptcha-response"]\\'\
');
                    responseElements.forEach(function(element) {
                        element.value = \'{recaptcha_token}\\';
                    });
                    var form = document.querySelector(\\'form\\'\
');
                    if (form) {
                        form.submit();
                    }
                });
            });
        }} else {
            console.error(\\'grecaptcha object not found or not ready.\\'\
');
            var responseElements = document.querySelectorAll(\\'[name="g-recaptcha-response"]\\'\
');
            responseElements.forEach(function(element) {
                element.value = \'{recaptcha_token}\\';
            });
            var form = document.querySelector(\\'form\\'\
');
            if (form) {
                form.submit();
            }
        }
    """

    # 3. Execute crawling task with Crawl4AI and inject JavaScript
    async with AsyncWebCrawler() as crawler:
        config = CrawlerRunConfig(
            url=target_url,
            js_code=js_injection_code,
            enable_interaction=True,
            timeout=60
        )
        try:
            result = await crawler.arun(config)
            print("Crawl4AI scraping result:")
            print(result.markdown[:500])
            return result
        except Exception as e:
            print(f"Crawl4AI scraping failed: {e}")
            return None

async def main():
    CAPSOLVER_API_KEY = "CAP-YOUR_API_KEY"
    PAGE_URL = "https://2captcha.com/demo/recaptcha-v3"
    PAGE_KEY = "6LfB5_IbAAAAAMCtsjEHEHKqcB9iQocwwxTiihJu"
    PAGE_ACTION = "homepage"

    capsolver.api_key = CAPSOLVER_API_KEY
    await solve_recaptcha_v3_with_crawl4ai(PAGE_URL, PAGE_KEY, PAGE_ACTION)

if __name__ == "__main__":
    asyncio.run(main())
```

### Code Analysis & Optimization

1.  **CapSolver Call**:
    *   `type`: `ReCaptchaV3TaskProxyLess`.
    *   `pageAction`: Critical for reCAPTCHA v3 score. Match the specific action on the page.
    *   `minScore`: Specify minimum acceptable score. CapSolver retries if score is too low.
2.  **JavaScript Injection**:
    *   Token obtained asynchronously via `grecaptcha.execute()`. Script waits for `grecaptcha.ready`, executes, then injects token into `g-recaptcha-response` hidden inputs.
    *   **Form Submission**: Generic logic included. **Customize as needed** per target site.
3.  **Crawl4AI Configuration**:
    *   `enable_interaction=True`: Simulates realistic user behavior, improving reCAPTCHA v3 score.
    *   `wait_until_selector`: Waits for a specific element post-submission, enhancing stability.

### Real-World Example: Comment Submission Page

Example: `https://blog.example.com/post/comment` with `siteKey`: `ANOTHER_EXAMPLE_SITE_KEY`, `action`: `comment_submit`.

```python
# ... (Imports and CapSolver configuration as above)

async def main_example_v3():
    CAPSOLVER_API_KEY = "CAP-YOUR_API_KEY"
    COMMENT_PAGE_URL = "https://blog.example.com/post/comment"
    COMMENT_SITE_KEY = "ANOTHER_EXAMPLE_SITE_KEY"
    COMMENT_ACTION = "comment_submit"

    capsolver.api_key = CAPSOLVER_API_KEY

    solution = await capsolver.solve({
        "type": "ReCaptchaV3TaskProxyLess",
        "websiteURL": COMMENT_PAGE_URL,
        "websiteKey": COMMENT_SITE_KEY,
        "pageAction": COMMENT_ACTION,
        "minScore": 0.7
    })
    recaptcha_token = solution["gRecaptchaResponse"]
    print(f"CapSolver obtained reCAPTCHA v3 token: {recaptcha_token[:30]}...")

    js_injection_code = f"""
        if (typeof grecaptcha !== \'undefined\' && grecaptcha.execute) {
            grecaptcha.ready(function() {
                grecaptcha.execute(\\'{COMMENT_SITE_KEY}\\', {{action: \'{COMMENT_ACTION}\\'}}).then(function(token) {
                    var responseElements = document.querySelectorAll(\\'[name="g-recaptcha-response"]\\'\
');
                    responseElements.forEach(function(element) {
                        element.value = \'{recaptcha_token}\\';
                    });
                    document.getElementById(\\'comment_text\\'\
').value = \'This is a test comment.\';
                    document.getElementById(\\'comment_author\\'\
').value = \'TestUser\';
                    var commentForm = document.getElementById(\\'commentForm\\'\
');
                    if (commentForm) {
                        commentForm.submit();
                    } else {
                        var form = document.querySelector(\\'form\\'\
');
                        if (form) {
                            form.submit();
                        }
                    }
                });
            });
        }} else {
            console.error(\\'grecaptcha object not found or not ready.\\'\
');
            var responseElements = document.querySelectorAll(\\'[name="g-recaptcha-response"]\\'\
');
            responseElements.forEach(function(element) {
                element.value = \'{recaptcha_token}\\';
            });
            var form = document.querySelector(\\'form\\'\
');
            if (form) {
                form.submit();
            }
        }
    """

    async with AsyncWebCrawler() as crawler:
        config = CrawlerRunConfig(
            url=COMMENT_PAGE_URL,
            js_code=js_injection_code,
            enable_interaction=True,
            wait_until_selector="#comment_success_message",
            timeout=60
        )
        try:
            result = await crawler.arun(config)
            print("Comment submission page scraping result:")
            print(result.markdown[:500])
        except Exception as e:
            print(f"Comment submission page scraping failed: {e}")

if __name__ == "__main__":
    asyncio.run(main_example_v3())
```

**Key Optimizations**:

*   **Accurate `pageAction`**: Critical for high reCAPTCHA v3 score.
*   **Adjust `minScore`**: Higher `minScore` (e.g., 0.7-0.9) for sensitive actions.
*   **Simulate User Input**: Filling other form fields makes automation more human-like.

## Conclusion

reCAPTCHA v3 presents challenges for web scraping. Combining Crawl4AI's browser control with CapSolver's CAPTCHA solving effectively overcomes these. Key aspects: understanding reCAPTCHA v3, configuring CapSolver (`pageAction`, `minScore`), and using Crawl4AI's `js_code` for token injection and behavior simulation. Testing and adjustments build stable, efficient automation systems.

## References

[^1]: Crawl4AI Documentation. [https://docs.crawl4ai.com/](https://docs.crawl4ai.com/)
[^2]: CapSolver Official Website. [https://www.capsolver.com/](https://www.capsolver.com/?utm_source=github&utm_medium=integration&utm_campaign=crawl4ai-recaptchav3)
[^3]: sarperavci/Capsolver-GoogleRecaptchaV3Bypass. GitHub. [https://github.com/sarperavci/Capsolver-GoogleRecaptchaV3Bypass](https://github.com/sarperavci/Capsolver-GoogleRecaptchaV3Bypass)

