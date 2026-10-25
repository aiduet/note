+++
title = "A small frontend tweak for the blog"
date = "2024-01-11"
slug = "a-small-frontend-tweak-for-the-blog"
tags = ["notes"]
+++

I switched the blog theme to [Shiro](https://github.com/innei/Shiro). On the phone it said “Browser version is too low”.

The phone was on a current OS and browser.

I checked the theme code and the User-Agent:

Mozilla/5.0 (iPhone; CPU iPhone OS 17_4 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) CriOS/123.0.6312.52 Mobile/15E148 Safari/604.1

That is Chrome on iOS. It reports as `CriOS`.

See: https://chromium.googlesource.com/chromium/src.git/+/HEAD/docs/ios/user_agent.md

Safari also showed the pop-up. iOS Safari and macOS Safari UAs differ a bit. This check fixed it:

```javascript
function isSupportedBrowser() {
  const ua = navigator.userAgent;
  const macSafariRegex = /Version\/(\d+).*Safari/;
  const iosVersionRegex = /OS (\d+)_/;
  const chromeRegex = /Chrome\/(\d+)|CriOS\/(\d+)/;
  const firefoxRegex = /Firefox\/(\d+)/;
  const edgeRegex = /Edg\/(\d+)/;
  const operaRegex = /Opera\/(\d+)/;
  const criosRegex = /CriOS\/(\d+)/;

  // macOS Safari
  if (ua.includes('Macintosh') && ua.includes('Safari') && !ua.includes('Chrome') && !ua.includes('Edg')) {
    const match = ua.match(macSafariRegex);
    return match && parseInt(match[1], 10) >= 16;
  }

  // iOS Safari
  if ((ua.includes('iPhone') || ua.includes('iPad') || ua.includes('iPod')) && ua.includes('Safari') && !ua.includes('CriOS') && !ua.includes('Edg')) {
    const match = ua.match(iosVersionRegex);
    return match && parseInt(match[1], 10) >= 16;
  }

  // Chrome or Chrome on iOS (CriOS)
  let match = ua.match(chromeRegex);
  if (match) {
    const version = parseInt(match[1] || match[2], 10);
    return version >= 110;
  }

  // Edge
  match = ua.match(edgeRegex);
  if (match) {
    return parseInt(match[1], 10) >= 110;
  }

  // Firefox
  match = ua.match(firefoxRegex);
  if (match) {
    return parseInt(match[1], 10) >= 113;
  }

  // Opera
  match = ua.match(operaRegex);
  if (match) {
    return parseInt(match[1], 10) >= 102;
  }
  
  return false;
}
```
