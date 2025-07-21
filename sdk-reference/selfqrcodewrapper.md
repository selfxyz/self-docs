# SelfQRcodeWrapper

A React component wrapper that prevents server-side rendering issues with QR code generation in Next.js applications.

## Props

The `SelfQRcodeWrapper` component accepts the following props:

| Parameter      | Type            | Required | Default         | Description                                           |
| -------------- | --------------- | -------- | --------------- | ----------------------------------------------------- |
| `selfApp`      | SelfApp         | Yes      | -               | The SelfApp configuration object                      |
| `onSuccess`    | () => void      | Yes      | -               | Callback function executed on successful verification |
| `onError`      | (error: { error_code?: string, reason?: string }) => void      | Yes      | -               | Callback function executed on verification error     |
| `type`         | 'websocket' \| 'deeplink' | No       | 'websocket'     | Type of QR code to generate                           |
| `websocketUrl` | string          | No       | WS\_DB\_RELAYER | Custom WebSocket URL for verification                 |
| `size`         | number          | No       | 300             | QR code size in pixels                                |
| `darkMode`     | boolean         | No       | false           | Enable dark mode styling                              |
| `children`     | React.ReactNode | No       | -               | Custom children to render                             |
