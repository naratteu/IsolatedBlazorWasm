# IsolatedBlazorWasm

https://news.hada.io/topic?id=17565  
https://github.com/GoogleChromeLabs/telnet-client

https://github.com/user-attachments/assets/eae875bb-477e-49e2-b2c4-b9395444e3af

위 링크는 `direct-socket`을 실험해볼수있는 예제프로젝트입니다

혹시 `Blazor WebAssembly`에서도 `direct-socket`을 사용할수 있을지, 우선 `isolated-webapp` 으로 배포해봅시다.

```bat
dotnet new blazorwasm --pwa --no-https
dotnet run --Urls http://localhost:5000
```

위와같이 PWA(Progressive Web Apps) 블레이저 템플릿을 생성해 실행하고, 별도의 터미널에서 크롬의 *격리된-웹앱* 등록을 시도해봅니다.

```bat
path "C:\Program Files\Google\Chrome\Application"
chrome --install-isolated-web-app-from-url=http://localhost:5000 ^
    --enable-features=IsolatedWebApps,IsolatedWebAppDevMode ^
    --enable-logging=stderr
```

크롬131.0.6778.205를 기준으로, `--enable-features` 로 실험적 피쳐인  [isolated-web-apps](chrome://flags/#enable-isolated-web-apps), [isolated-web-app-dev-mode](chrome://flags/#enable-isolated-web-app-dev-mode)를 활성화 해야 격리된 웹앱을 등록할수 있습니다.
허나 기존 PWA템플릿 만으로는 격리된 웹앱으로써 등록할수 없습니다. `--enable-logging` 을 사용하여 어떤 요구사항이 발생하는지 확인해봅니다.

```log
[38008:62788:0107/175635.204:WARNING:external_pref_loader.cc(294)] You are using an old-style extension deployment method (external_extensions.json), which will soon be deprecated. (see http://developer.chrome.com/docs/extensions/how-to/distribute/install-extensions)
[38008:50744:0107/175635.684:INFO:CONSOLE(1)] "Manifest: Line: 1, column: 1, Syntax error.", source: isolated-app://iikvnzipoj236odpzfmunjeqhpjx5xldgbsvsukxr2hfg6f2z2iqaaac/.well-known/manifest.webmanifest (1)
[38008:50744:0107/175635.686:ERROR:isolated_web_app_installation_manager.cc(471)] Isolated Web App command line installation failed: App is not installable: The manifest could not be fetched, parsed, or the document is on an opaque origin..
```

`manifest.webmanifest`를 `.well-known`의 하위로 탐색하고있습니다. 어째서 다른걸까요? 변경해줍니다.

```bat
mkdir wwwroot\.well-known
move wwwroot\manifest.webmanifest wwwroot\.well-known\manifest.webmanifest
```

```log
[40936:30792:0107/180155.470:ERROR:isolated_web_app_installation_manager.cc(471)] Isolated Web App command line installation failed: App is not installable: No supplied icon is at least 144px square in PNG, SVG or WebP format.

[50592:58360:0107/183205.371:ERROR:isolated_web_app_installation_manager.cc(471)] Isolated Web App command line installation failed: Manifest `version` is not present. manifest_url: isolated-app://r74oyi6fkfzctov3fvmyfg3whqlmmkv4ykzldon72udsalyajp6aaaac/.well-known/manifest.webmanifest

[576:35764:0107/183524.225:ERROR:isolated_web_app_installation_manager.cc(471)] Isolated Web App command line installation failed: Scope should resolve to the origin. scope: isolated-app://32cq3rnna6b3fjwe2zpuo2w6umxe4z3jhexm2fhzxkixdhqzo2zqaaac/.well-known/, origin: isolated-app://32cq3rnna6b3fjwe2zpuo2w6umxe4z3jhexm2fhzxkixdhqzo2zqaaac
```

`icon-....png` 를 `/icon-....png` 로 바꾸고 `version`과 `scope` 를 추가해주니 어쨋든 설치에는 성공합니다. 설치된 웹앱은 `chrome://apps` 에서 확인가능합니다.

[telnet-client 예제](https://github.com/GoogleChromeLabs/telnet-client/blob/main/assets/.well-known/manifest.webmanifest) 를 참고해 `permissions_policy` 도 추가해준다면 `direct-socket` 도 사용가능할것입니다.

설치에는 성공했으나, 기대한대로 런타임에러가 발생합니다.

![{64910B28-6004-4E07-A3A7-622741399EA4}](https://github.com/user-attachments/assets/aed3913f-82da-46e8-ad8b-1de57a322cb9)

```log
Refused to set the document's base URI to 'isolated-app://5qxrhe6khtrpn67mx6s25ytsxmvotxkr435qlwtdx3vxvd3s757aaaac/' because it violates the following Content Security Policy directive: "base-uri 'none'".
```

기준경로는 `manifest`를 따르므로 `<base>`를 명시하지 말라는듯 합니다. `wwwroot/index.html`의 `<base href="/" />` 를 제거하고 manifest에 입력된 경로들을 절대경로`/` 로 변경합니다.

```log
Refused to execute inline script because it violates the following Content Security Policy directive: "script-src 'self' 'wasm-unsafe-eval'". Either the 'unsafe-inline' keyword, a hash ('sha256-9mThMC8NT3dPbcxJOtXiiwevtWTAPorqkXGKqI388cI='), or a nonce ('nonce-...') is required to enable inline execution.
```

페이지에서 직접 호출하는 자바스크립트 코드도 없어야 한다고 합니다.

```log
blazor.webassembly.js:1 This document requires 'TrustedHTML' assignment.
```

이제와서 생각해보니 동적으로 HTML요소를 제어하는 코드를 함부로 신뢰하면 안될거같다고 합니다. 기존의 악습을 전승시키려면 `.js` 파일에 아래 코드를 삽입해야합니다. [출처](https://greasyfork.org/en/discussions/development/220765-this-document-requires-trustedhtml-assignment)

```js
if (window.trustedTypes && window.trustedTypes.createPolicy) {  
    window.trustedTypes.createPolicy('default', {  
        createHTML: (string, sink) => string  
    }); 
}
```

위 코드를 `index.html`에 명시적으로 포함된 `_framework/blazor.webassembly.js` 에 주입해야하는데, 해당 파일은 프로젝트상에 존재하지 않고 빌드후에 포함됨으로 `bin\Debug\net9.0\wwwroot\_framework\blazor.webassembly.js` 경로를 찾아가 `!function(){"use strict";` 직후에 동작하도록 삽입합니다.

![{BE83C531-FD59-431F-BEB0-12586F738D1C}](https://github.com/user-attachments/assets/9a405d6c-f4eb-4ae4-98c7-53609504d182)

## Hello, world!

호환성을 보증할순 없으나, 어쨋든 블레이저 웹어셈블리 프로젝트가 *격리된-웹앱* 으로써 실행됩니다!

이제 [JS interop](https://learn.microsoft.com/aspnet/core/blazor/javascript-interoperability) 을 활용하여 *직접-쏘오켓* 을 활용할수 있을것입니다. 미래에는 닷넷의 `Socket`객체와의 통합도 기대가능하지 않을까 합니다.

## Todo

- 오프라인 설치
- 수행결과 단계적 커밋
- Github Page 배포 후 웹앱 등록
- 동작 이상 체크
