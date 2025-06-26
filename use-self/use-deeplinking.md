---
description: Integrate Self protocol inside your mobile application using Deeplinking
icon: mobile
---

# Use deeplinking

Import `getUniversalLink` and  `SelfAppBuilder` from `@selfxyz/core`

```typescript
import {getUniversalLink, SelfAppBuilder } from '@selfxyz/core';
```

Instantiate your Self app using `SelfAppBuilder` .

```typescript
const selfApp = new SelfAppBuilder({
    appName: <your-app-name>,
    scope: <your-app-scope>,
    endpoint: <your-endpoint>,
    logoBase64: <url-to-a-png>,
    userIdType: 'hex', // only for if you want to link the proof with the user address
    userId: <user-id>, // uuid or user address
}).build();
```

Get the deeplink from the Self app object.

```typescriptreact
 const deeplink = getUniversalLink(selfApp);
```

You can now use this deeplink to redirect your users to the Self mobile app. If your mobile application uses Kotlin/Java/Swift, reach out to us.
